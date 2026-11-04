# 7. Interception — framehooks, audiohooks and bridging

Every `file:line` citation in this guide was verified against the `24` branch.
Newer releases and other branches may differ — line numbers drift — so if you
are reading this against a different tree, treat the citations as approximate
and re-grep for the symbol name.

Three separate mechanisms get between a channel and the rest of Asterisk, and
they have three different ownership contracts. Confusing them is a reliable
source of double-frees.

| Mechanism | Sees | Can replace the frame? | Who frees what |
| --- | --- | --- | --- |
| Framehook | **every** frame type, read and write | yes — any type, or `NULL` | you free the old one if you return a new one |
| Audiohook | `AST_FRAME_VOICE` and `AST_FRAME_DTMF` only | yes, but only a re-encoded voice frame | the list function handles the intermediates |
| Bridge | whatever the bridge tech passes | n/a — frames are duped into queues | `bridge_frame_free()`, never bare `ast_frfree()` |

---

## 7.1 Framehooks

`include/asterisk/framehook.h` / `main/framehook.c`.

A framehook is the general-purpose interception point. It is a per-channel
linked list of callbacks invoked on every frame in both directions.

### The interface

```c
#define AST_FRAMEHOOK_INTERFACE_VERSION 4
/*! This interface is required for attaching a framehook to a channel. */
struct ast_framehook_interface {
	/*! framehook interface version number */
	uint16_t version;
	/*! event_cb represents the function that will be called everytime an event occurs on the framehook. */
	ast_framehook_event_callback event_cb;
	/*! destroy_cb is optional.  This function is called immediately before the framehook
	 * is destroyed to allow for stored_data cleanup. */
	ast_framehook_destroy_callback destroy_cb;
	/*! consume_cb is optional. This function is called to query whether the framehook is consuming
	* frames of a specific type at this time. If this callback is not implemented it is assumed that the
	* framehook will consume frames of all types. */
	ast_framehook_consume_callback consume_cb;
	/*! chan_fixup_cb is optional. This function is called when the channel that a framehook is running
	 * on is masqueraded and should be used to move any essential framehook data onto the channel the
	 * old channel was masqueraded to. */
	ast_framehook_chan_fixup_callback chan_fixup_cb;
	/*! chan_breakdown_cb is optional. This function is called when another channel is masqueraded into
	 * the channel that a framehook is running on and should be used to evaluate whether the framehook
	 * should remain on the channel. */
	ast_framehook_chan_fixup_callback chan_breakdown_cb;
	/*! disable_inheritance is optional. If set to non-zero, when a channel using this framehook is
	 * masqueraded, detach and destroy the framehook instead of moving it to the new channel. */
	int disable_inheritance;
	 /*! This pointer can represent any custom data to be stored on the !framehook. This
	 * data pointer will be provided during each event callback which allows the framehook
	 * to store any stateful data associated with the application using the hook. */
	void *data;
};
```
— `include/asterisk/framehook.h:227-256`

`version` must equal `AST_FRAMEHOOK_INTERFACE_VERSION` exactly or
`ast_framehook_attach()` logs an error and returns -1
(`main/framehook.c:137-141`). There is no backwards compatibility shim:
**bump the version and every out-of-tree framehook stops attaching.** That is
deliberate.

`consume_cb` is the field people most often get wrong when reading other
people's code — it is a *callback*, not a flag:

```c
/*!
 * \brief This callback is called to determine if the framehook is currently consuming
 * frames of a given type
 * \since 12
 *
 * \param data The data pointer provided at framehook initialization.
 * \param type The type of frame.
 *
 * \return 0 if frame type is being ignored
 * \return 1 if frame type is not being ignored
 */
typedef int (*ast_framehook_consume_callback)(void *data, enum ast_frame_type type);
```
— `include/asterisk/framehook.h:202-213`

Note the inverted sense: returning **0 means "I am ignoring this type"**. If
you omit `consume_cb` entirely, the framehook is assumed to consume everything.
This matters because the core uses it to decide whether it can take shortcuts —
see §7.1.5.

### The events

```c
enum ast_framehook_event {
	AST_FRAMEHOOK_EVENT_READ, /*!< frame is intercepted in the read direction on the channel. */
	AST_FRAMEHOOK_EVENT_WRITE, /*!< frame is intercepted on the write direction on the channel. */
	AST_FRAMEHOOK_EVENT_ATTACHED, /*!< framehook is attached and running on the channel, the first message sent to event_cb. */
	AST_FRAMEHOOK_EVENT_DETACHED /*!< framehook is detached from the channel, last message sent to event_cb. */
};
```
— `include/asterisk/framehook.h:151-156`

`ATTACHED` and `DETACHED` are guaranteed to bracket the hook's lifetime, and
both are delivered with `frame == NULL`. Both call sites nonetheless check the
return value and free it:

```c
	frame = framehook->i.event_cb(framehook->chan, NULL, AST_FRAMEHOOK_EVENT_DETACHED, framehook->i.data);
	/* never assume anything about this function. If you can return a frame during
	 * the detached event, then assume someone will. */
	if (frame) {
		ast_frfree(frame);
	}
```
— `main/framehook.c:69-74` (and the same pattern at `:162-168` for `ATTACHED`)

### The contract

This doc comment is the whole contract. Read it before writing a framehook:

```
 * It is completely valid for the frame variable to be set to NULL. Always do a NULL
 * check on the frame before attempted to access it. When the frame variable is present,
 * it is safe to view and manipulate that frame in any way possible.  It is even safe
 * to return a completely different frame, but when that occurs this function is in
 * charge of freeing the previous frame.
 *
 * The ast_channel will always be locked during this callback. Never attempt to unlock the
 * channel for any reason.
```
— `include/asterisk/framehook.h:168-175`

Four rules fall out of that:

1. **Check for `NULL` on entry.** You will be called with `NULL` on
   attach/detach, and a previous hook in the list may have returned `NULL`.
2. **If you return the frame you were given, you own nothing.** The core
   continues to own it.
3. **If you return a *different* frame, you must free the one you were given.**
   Nobody else will. Returning `NULL` counts as returning a different frame —
   you must free the input first, or it leaks.
4. **The channel is locked. Do not unlock it.** This means no blocking I/O, no
   calls that might take the channel lock again in a non-recursive path, and no
   calling back into `ast_read()`/`ast_write()` on this channel.

A minimal correct replacement looks like this — illustrative code, not a quote
from the tree:

<!-- illustrative -->
```c
static struct ast_frame *my_cb(struct ast_channel *chan, struct ast_frame *frame,
	enum ast_framehook_event event, void *data)
{
	struct ast_frame *new_frame;

	if (!frame || event != AST_FRAMEHOOK_EVENT_READ) {
		return frame;
	}
	if (frame->frametype != AST_FRAME_VOICE) {
		return frame;
	}

	new_frame = build_my_replacement(frame);
	if (!new_frame) {
		return frame;        /* give it back unchanged, we still own nothing */
	}

	ast_frfree(frame);       /* REQUIRED: we are replacing it */
	return new_frame;
}
```

### The re-run loop

The dispatcher is the single most important function to understand, because its
loop semantics are surprising:

```c
	skip_size = sizeof(int) * framehooks->count;
	skip = ast_alloca(skip_size);
	memset(skip, 0, skip_size);

	do {
		unsigned int num = 0;
		original_frame = frame;

		AST_LIST_TRAVERSE_SAFE_BEGIN(&framehooks->list, framehook, list) {
			if (framehook->detach_and_destroy_me) {
				/* this guy is signaled for destruction */
				AST_LIST_REMOVE_CURRENT(list);
				framehook_detach(framehook, FRAMEHOOK_DETACH_DESTROY);
				continue;
			}

			/* If this framehook has been marked as needing to be skipped, do so */
			if (skip[num]) {
				num++;
				continue;
			}

			frame = framehook->i.event_cb(framehook->chan, frame, event, framehook->i.data);

			if (frame != original_frame) {
				/* To prevent looping we skip any framehooks that have already provided a modified frame */
				skip[num] = 1;
				break;
			}

			num++;
		}
		AST_LIST_TRAVERSE_SAFE_END;
	} while (frame != original_frame);

	return frame;
```
— `main/framehook.c:94-129`

What this means in practice:

- As long as every hook returns the frame unchanged, the list is walked once,
  front to back.
- The moment a hook returns a **different pointer**, the traversal *breaks* and
  the whole list is walked again from the top with the new frame — except that
  the hook which just substituted is marked in `skip[]` and will not be called
  again at all for this frame.
- So a hook that replaces a frame **does not see the result of its own
  substitution**, but every *other* hook does see it, including hooks earlier in
  the list that already ran once with the original.
- Therefore **your callback may be invoked more than once per frame.** If your
  hook has side effects (counters, recording, state machines), it must be
  idempotent or must detect the re-run itself.
- `frame == NULL` is "different from `original_frame`", so returning `NULL`
  triggers the re-run with `NULL`, and the remaining hooks get a `NULL` frame.
  That is the normal way to swallow a frame.

`skip` is sized from `framehooks->count`, which is incremented on attach
(`main/framehook.c:157`) but never decremented. `num` indexes positionally into
the live list. Since detached hooks are removed from the list during the
traversal but `count` does not shrink, `skip` is over-sized rather than
under-sized — safe, but it means `skip[]` indices shift if a hook is removed
mid-walk. Not a correctness problem in practice because removal and substitution
in the same pass is vanishingly rare, but it is worth knowing it is positional
and not keyed by hook id.

### Where the core calls them

| Direction | Call site | Notes |
| --- | --- | --- |
| Read | `main/channel.c:3785` | `f = ast_framehook_list_read_event(..., f);` — after the driver produced `f`, before the frametype switch |
| Write | `main/channel.c:5221` | only when `stream == default_stream`; `fr` itself is reassigned |
| Write (indications) | `main/channel.c:4705` | `ast_indicate_data()` builds a stack CONTROL frame, `ast_frdup()`s it, and pushes it through the write event |

The read-side entry point is the thin wrapper pair at the bottom of the file:

```c
struct ast_frame *ast_framehook_list_write_event(struct ast_framehook_list *framehooks, struct ast_frame *frame)
{
	return framehook_list_push_event(framehooks, frame, AST_FRAMEHOOK_EVENT_WRITE);
}

struct ast_frame *ast_framehook_list_read_event(struct ast_framehook_list *framehooks, struct ast_frame *frame)
{
	return framehook_list_push_event(framehooks, frame, AST_FRAMEHOOK_EVENT_READ);
}
```
— `main/framehook.c:313-321`

The indication path is worth quoting because it is the one place the core
*constructs* a frame purely to run it past the framehooks, then unpacks the
result back into scalars:

```c
	if (!ast_framehook_list_is_empty(ast_channel_framehooks(chan))) {
		/* Do framehooks now, do it, go, go now */
		struct ast_frame frame = {
			.frametype = AST_FRAME_CONTROL,
			.subclass.integer = _condition,
			.data.ptr = (void *) data, /* this cast from const is only okay because we do the ast_frdup below */
			.datalen = datalen
		};

		/* we have now committed to freeing this frame */
		awesome_frame = ast_frdup(&frame);

		/* who knows what we will get back! the anticipation is killing me. */
		awesome_frame = ast_framehook_list_write_event(ast_channel_framehooks(chan),
			awesome_frame);
		if (!awesome_frame
			|| awesome_frame->frametype != AST_FRAME_CONTROL) {
			res = 0;
			goto indicate_cleanup;
		}

		_condition = awesome_frame->subclass.integer;
		data = awesome_frame->data.ptr;
		datalen = awesome_frame->datalen;
	}
```
— `main/channel.c:4692-4716`

Two consequences. A framehook **can rewrite an indication** — change
`AST_CONTROL_RINGING` into `AST_CONTROL_PROGRESS`, say — by mutating
`subclass.integer`. And a framehook can **suppress** an indication entirely by
returning `NULL` or by changing the frametype to anything other than
`AST_FRAME_CONTROL`; the caller then gets `res = 0`, i.e. *success*, and never
learns the indication was dropped.

Note also that `data` and `datalen` are taken from the returned frame, so a
framehook that replaces the frame with a differently-sized payload changes what
`indicate_data_internal()` reads. If you rewrite the payload, keep `datalen`
consistent with it.

### Optimisation queries

Two helpers let the core skip work when no hook cares:

```c
int ast_channel_has_audio_frame_or_monitor(struct ast_channel *chan)
{
	return !ast_audiohook_write_list_empty(ast_channel_audiohooks(chan))
		|| !ast_framehook_list_contains_no_active(ast_channel_framehooks(chan));
}

int ast_channel_has_hook_requiring_audio(struct ast_channel *chan)
{
	return !ast_audiohook_write_list_empty(ast_channel_audiohooks(chan))
		|| !ast_framehook_list_contains_no_active_of_type(ast_channel_framehooks(chan), AST_FRAME_VOICE);
}
```
— `main/channel.c:2517-2527`

`ast_framehook_list_contains_no_active_of_type()` is where `consume_cb` is
actually consulted:

```c
	AST_LIST_TRAVERSE(&framehooks->list, cur, list) {
		if (cur->detach_and_destroy_me) {
			continue;
		}
		if (type && cur->i.consume_cb && !cur->i.consume_cb(cur->i.data, type)) {
			continue;
		}
		return 0;
	}

	return 1;
```
— `main/framehook.c:300-310`

So a hook that implements `consume_cb` and returns 0 for `AST_FRAME_VOICE` lets
the bridge stay in a native (non-media-passing) bridge technology. **This is
the difference between a framehook that costs nothing and a framehook that
forces every call it touches out of native bridging.** If your hook only cares
about CONTROL frames, implement `consume_cb`.

### Masquerades

`ast_framehook_list_fixup(old_chan, new_chan)` at `main/framehook.c:223-272`
moves hooks across a masquerade:

- Hooks already on `new_chan` get `chan_breakdown_cb` (or are detached if
  `disable_inheritance`).
- Hooks on `old_chan` are re-attached to `new_chan` — which means
  `AST_FRAMEHOOK_EVENT_ATTACHED` fires again, with a **new id** — then
  `chan_fixup_cb` is called with that new id, then the old one is detached with
  `FRAMEHOOK_DETACH_PRESERVE` (so `destroy_cb` is *not* called and the `data`
  pointer is not freed).
- If the re-attach fails: `"Failed framehook copy during masquerade. Expect loss
  of features."` and the hook is destroyed (`:261-262`).

The practical rule: **never cache a framehook id outside the hook's own `data`
struct**, and always update it in `chan_fixup_cb`. The `data` pointer survives a
masquerade; the id does not.

### Detach is deferred

```c
			/* we mark for detachment rather than doing explicitly here because
			 * it needs to be safe for this function to be called within the
			 * event callback.  If we allowed the hook to actually be destroyed
			 * immediately here, the event callback would crash on exit. */
			framehook->detach_and_destroy_me = 1;
```
— `main/framehook.c:188-192`

So `ast_framehook_detach()` is safe to call from inside your own `event_cb`. The
hook is reaped on the next frame that passes through the list (the
`detach_and_destroy_me` branch at the top of the traversal), or at
`ast_framehook_list_destroy()` time. Until then your `data` must stay valid.

---

## 7.2 Audiohooks

`include/asterisk/audiohook.h` / `main/audiohook.c`.

Audiohooks are the narrow, media-only mechanism — this is what `MixMonitor`,
`ChanSpy`, `Volume` and the speech/DSP modules use. Unlike framehooks they do
not see signalling, and the core does the format wrangling for you.

### Three types

```c
enum ast_audiohook_type {
	AST_AUDIOHOOK_TYPE_SPY = 0,    /*!< Audiohook wants to receive audio */
	AST_AUDIOHOOK_TYPE_WHISPER,    /*!< Audiohook wants to provide audio to be mixed with existing audio */
	AST_AUDIOHOOK_TYPE_MANIPULATE, /*!< Audiohook wants to manipulate the audio */
};
```
— `include/asterisk/audiohook.h:35-39`

| Type | Role | Mechanism |
| --- | --- | --- |
| SPY | read-only tap | frames are fed into the hook's slinfactory; consumer pulls with `ast_audiohook_read_frame()` |
| WHISPER | injects audio | the core pulls from the hook's slinfactory and *adds* it into the frame |
| MANIPULATE | in-place rewrite | `manipulate_callback` is handed the frame |

### Direction

```c
enum ast_audiohook_direction {
	AST_AUDIOHOOK_DIRECTION_READ = 0, /*!< Reading audio in */
	AST_AUDIOHOOK_DIRECTION_WRITE,    /*!< Writing audio out */
	AST_AUDIOHOOK_DIRECTION_BOTH,     /*!< Both reading audio in and writing audio out */
};
```
— `include/asterisk/audiohook.h:48-52`

`BOTH` is the default on init; `ast_audiohook_set_frame_feed_direction()` can
narrow it, but only while the hook is `STATUS_NEW`
(`include/asterisk/audiohook.h:144-150`).

Do not confuse `enum ast_audiohook_direction` (which direction a given frame
came from, passed to the callback) with the `AST_AUDIOHOOK_TRIGGER_READ` /
`TRIGGER_WRITE` flags (when the hook wants to be woken):

```c
enum ast_audiohook_flags {
	AST_AUDIOHOOK_TRIGGER_MODE  = (3 << 0), /*!< When audiohook should be triggered to do something */
	AST_AUDIOHOOK_TRIGGER_READ  = (1 << 0), /*!< Audiohook wants to be triggered when reading audio in */
	AST_AUDIOHOOK_TRIGGER_WRITE = (2 << 0), /*!< Audiohook wants to be triggered when writing audio out */
	AST_AUDIOHOOK_WANTS_DTMF    = (1 << 2), /*!< Audiohook also wants to receive DTMF frames */
	AST_AUDIOHOOK_TRIGGER_SYNC  = (1 << 3), /*!< Audiohook wants to be triggered when both sides have combined audio available */
	/*! Audiohooks with this flag set will not allow for a large amount of samples to build up on its
	 * slinfactories. We will flush the factories if they contain too many samples.
	 */
	AST_AUDIOHOOK_SMALL_QUEUE   = (1 << 4),
	AST_AUDIOHOOK_MUTE_READ     = (1 << 5), /*!< audiohook should be mute frames read */
	AST_AUDIOHOOK_MUTE_WRITE    = (1 << 6), /*!< audiohook should be mute frames written */
	AST_AUDIOHOOK_COMPATIBLE    = (1 << 7), /*!< is the audiohook native slin compatible */

	AST_AUDIOHOOK_SUBSTITUTE_SILENCE = (1 << 8), /*!< Substitute silence for missing audio */
};
```
— `include/asterisk/audiohook.h:54-69`

### The dispatcher

Only two frame types ever reach an audiohook:

```c
struct ast_frame *ast_audiohook_write_list(struct ast_channel *chan, struct ast_audiohook_list *audiohook_list, enum ast_audiohook_direction direction, struct ast_frame *frame)
{
	/* Pass off frame to it's respective list write function */
	if (frame->frametype == AST_FRAME_VOICE) {
		return audio_audiohook_write_list(chan, audiohook_list, direction, frame);
	} else if (frame->frametype == AST_FRAME_DTMF) {
		return dtmf_audiohook_write_list(chan, audiohook_list, direction, frame);
	} else {
		return frame;
	}
}
```
— `main/audiohook.c:1447-1457`

Recall from [02-frame-types.md](02-frame-types.md) that `AST_FRAME_DTMF` is an
alias for `AST_FRAME_DTMF_END` — so **DTMF_BEGIN frames never reach an
audiohook.** Only the END frame does, and only hooks with
`AST_AUDIOHOOK_WANTS_DTMF` set:

```c
		if (ast_test_flag(audiohook, AST_AUDIOHOOK_WANTS_DTMF)) {
			audiohook->manipulate_callback(audiohook, chan, frame, direction);
		}
```
— `main/audiohook.c:1133-1135`

Note the DTMF path only visits the `manipulate_list` — SPY and WHISPER hooks
never see DTMF at all. Its return value is also ignored, and the frame is always
returned unchanged (`main/audiohook.c:1144`).

### The voice path in three parts

`audio_audiohook_write_list()` at `main/audiohook.c:1271` is documented by its own
header comment and does exactly what it says:

1. **Translate to slin.** `audiohook_list_translate_to_slin()` (`:1147`)
   produces `middle_frame`, which may be `start_frame` itself if the frame is
   already slin at the list's internal rate. The list's rate floats up to the
   max it has seen:

   ```c
   	audiohook_list->list_internal_samp_rate =
   		MAX(ast_format_get_sample_rate(frame->subclass.format), audiohook_list->list_internal_samp_rate);
   ```
   — `main/audiohook.c:1161-1162`

   An **interpolated frame short-circuits the whole thing**:

   ```c
   	/* If the translation resulted in an interpolated frame then immediately return as audiohooks
   	 * rely on actual media being present to do things.
   	 */
   	if (!middle_frame->data.ptr) {
   		if (middle_frame != start_frame) {
   			ast_frfree(middle_frame);
   		}
   		return start_frame;
   	}
   ```
   — `main/audiohook.c:1285-1293`

   This is the consumer side of `generate_interpolated_slin()` from
   [06-translation-and-timing.md](06-translation-and-timing.md): a voice frame
   with `samples > 0` but `data.ptr == NULL` is legal, and audiohooks bail on it.

2. **Feed the three lists.** Spies get `ast_audiohook_write_frame()`. Whispers
   are summed into the frame with `ast_slinear_saturated_add()` over a
   stack `short combine_buf[samples]`. Manipulators get the frame directly.
   Any hook whose `status != AST_AUDIOHOOK_STATUS_RUNNING` is removed from the
   list in place, moved to `STATUS_DONE`, and — for manipulators — poked one
   last time with a `NULL` frame so it can tear itself down (`:1375`).

3. **Translate back if touched.** Only if `middle_frame_manipulated` is set —
   `audiohook_list_translate_to_native()` is at `:1193`:

   ```c
   	if (middle_frame_manipulated) {
   		if (!(end_frame = audiohook_list_translate_to_native(audiohook_list, direction, middle_frame, start_frame->subclass.format))) {
   			/* translation failed, so just pass back the input frame */
   			end_frame = start_frame;
   		}
   	} else {
   		end_frame = start_frame;
   	}
   	/* clean up our middle_frame if required */
   	if (middle_frame != end_frame) {
   		ast_frfree(middle_frame);
   		middle_frame = NULL;
   	}
   ```
   — `main/audiohook.c:1400-1412`

So the ownership contract from the *caller's* point of view is: the returned
frame is either the frame you passed in, or a new frame that you now own; the
intermediate slin frame is always cleaned up internally. This is why
`ast_write_stream()` does the `if (f && f != fr) ast_frfree(f);` dance described
in [05-channel-path.md](05-channel-path.md) — it cannot know which case it got.

A manipulator failure is silently swallowed:

```c
			if (!audiohook->manipulate_callback(audiohook, chan, middle_frame, direction)) {
				/*
				 * XXX FAILURES ARE IGNORED XXX
				 * If the manipulation fails then the frame will be returned in its original state.
				 * Since there are potentially more manipulator callbacks in the list, no action should
				 * be taken here to exit early.
				 */
				middle_frame_manipulated = 1;
			}
```
— `main/audiohook.c:1385-1393`

Read that carefully: the callback returns **0 on success**, and
`middle_frame_manipulated` is set inside `if (!...)` — i.e. only on success. The
comment's placement is misleading (it describes the failure case but sits in the
success branch), but the logic is right: on failure the frame is untouched, so no
back-translation is needed. The "failures are ignored" complaint is that the
*caller* is never told.

### The manipulate callback contract

```c
 * \note An audiohook does not have any reference to a private data structure for manipulate
 *       types. It is up to the manipulate callback to store this data via it's own method.
 *       An example would be datastores.
 * \note The input frame should never be freed or corrupted during a manipulate callback.
 *       If the callback has the potential to corrupt the frame's data during manipulation,
 *       local data should be used for the manipulation and only copied to the frame on
 *       success.
 * \note A failure return value indicates that the frame was not manipulated and that
 *       is being returned in its original state.
```
— `include/asterisk/audiohook.h:87-95`

Unlike a framehook, a manipulate callback **must not free or replace the frame**.
It mutates `frame->data.ptr` in place, or it returns -1 and leaves it alone.
There is also no `data` pointer in the signature — state must come from a
channel datastore or be hung off the `ast_audiohook` the caller embedded in its
own struct.

The frame handed to a manipulator is guaranteed slin, but **not guaranteed
8 kHz** unless you opted out:

```c
enum ast_audiohook_init_flags {
	/*! Audiohook manipulate callback is capable of handling slinear at any sample rate.
	 * Without enabling this flag on initialization the manipulation callback is guaranteed
	 * 8khz audio only. */
	AST_AUDIOHOOK_MANIPULATE_ALL_RATES = (1 << 0),
};
```
— `include/asterisk/audiohook.h:71-76`

### Where the core calls them

| Path | Call sites | Notes |
| --- | --- | --- |
| Read, DTMF | `main/channel.c:3936`, `:4008`, `:4053`, `:4109` | four separate points in the DTMF state machine |
| Read, voice (pre-translate) | `main/channel.c:4182` | only when the frame is already slin; sets `hooked = 1` |
| Read, voice (post-translate) | `main/channel.c:4200` | the normal case |
| Write, DTMF_BEGIN / DTMF_END | `main/channel.c:5262`, `:5275` | |
| Write, voice (pre-translate) | `main/channel.c:5337` | slin only |
| Write, voice (post-translate) | `main/channel.c:5399` | iterates the possibly-multi-frame list |
| Teardown | `main/channel.c:4269`, `:5502` | `ast_audiohook_write_list_empty()` → `ast_audiohook_detach_list()` |

The read path's two-call structure exists so that a channel already running slin
avoids a pointless slin→slin round trip. `hooked` records which one fired so the
second is skipped — see [05-channel-path.md](05-channel-path.md) §voice case.

The teardown check is easy to misread:

```c
	if (ast_channel_audiohooks(chan) && ast_audiohook_write_list_empty(ast_channel_audiohooks(chan))) {
```
— `main/channel.c:4269`, `:5502`

`ast_audiohook_write_list_empty()` returns true when the list pointer is `NULL`
*or* all three sub-lists are empty (`main/audiohook.c:1439-1445`), i.e. every
hook has removed itself. `ast_audiohook_detach_list()`
(`main/audiohook.c:910-949`) then drains the three lists, frees the four
translation paths it may be holding, and frees the list struct; the caller
follows up with `ast_channel_audiohooks_set(chan, NULL)`
(`main/channel.c:4271-4272`). So the audiohook *list* is garbage-collected
lazily, on the next read or write after the last hook departs — and it gets
recreated from scratch if a hook is attached again later.

---

## 7.3 Bridging

`main/bridge_channel.c` / `include/asterisk/bridge_channel_internal.h`.

A bridge adds a second queue per channel. Where `__ast_read()` pulls from the
channel's `readq`, the bridge channel thread pulls from `bridge_channel->wr_queue`
and writes to the channel. Frames travel: channel → `ast_read()` → bridge tech →
peer's `wr_queue` → peer's `ast_write_stream()`.

### Action frames

Bridges reuse the frame machinery as a general-purpose work queue, via two frame
types reserved for the purpose:

```c
static int bridge_channel_queue_action_data(struct ast_bridge_channel *bridge_channel,
	enum bridge_channel_action_type action, const void *data, size_t datalen)
{
	struct ast_frame frame = {
		.frametype = AST_FRAME_BRIDGE_ACTION,
		.subclass.integer = action,
		.datalen = datalen,
		.data.ptr = (void *) data,
	};

	return ast_bridge_channel_queue_frame(bridge_channel, &frame);
}
```
— `main/bridge_channel.c:909-920`

A stack frame with a borrowed payload, handed to a queue function that duplicates
it — the standard pattern from [03-memory-and-ownership.md](03-memory-and-ownership.md).
`bridge_channel_write_action_data()` at `:981-992` is identical except it calls
`bridge_channel_write_frame()` (immediate, into the bridge) instead of queueing.

The action space has a hard boundary in the middle of it:

```c
enum bridge_channel_action_type {
	/*! Bridged channel is to send a DTMF stream out */
	BRIDGE_CHANNEL_ACTION_DTMF_STREAM,
	/*! Bridged channel is to indicate talking start */
	BRIDGE_CHANNEL_ACTION_TALKING_START,
	/*! Bridged channel is to indicate talking stop */
	BRIDGE_CHANNEL_ACTION_TALKING_STOP,
	/*! Bridge channel is to play the indicated sound file. */
	BRIDGE_CHANNEL_ACTION_PLAY_FILE,
	/*! Bridge channel is to run the indicated application. */
	BRIDGE_CHANNEL_ACTION_RUN_APP,
	/*! Bridge channel is to run the custom callback routine. */
	BRIDGE_CHANNEL_ACTION_CALLBACK,
	/*! Bridge channel is to get parked. */
	BRIDGE_CHANNEL_ACTION_PARK,
	/*! Bridge channel is to execute a blind transfer. */
	BRIDGE_CHANNEL_ACTION_BLIND_TRANSFER,
	/*! Bridge channel is to execute an attended transfer */
	BRIDGE_CHANNEL_ACTION_ATTENDED_TRANSFER,

	/*
	 * Bridge actions put after this comment must never be put onto
	 * the bridge_channel wr_queue because they have other resources
	 * that must be freed.
	 */

	/*! Bridge reconfiguration deferred technology destruction. */
	BRIDGE_CHANNEL_ACTION_DEFERRED_TECH_DESTROY = 1000,
	/*! Bridge deferred dissolving. */
	BRIDGE_CHANNEL_ACTION_DEFERRED_DISSOLVING,
};
```
— `include/asterisk/bridge_channel_internal.h:41-71`

The `= 1000` jump is a deliberate firebreak. Actions below it have payloads that
are pure data, so `ast_frfree()` on the duped frame is sufficient cleanup.
Actions at or above 1000 carry references (an ao2 object, a channel) that the
generic frame free path knows nothing about, so they are dispatched directly and
**must never be queued**. If you add an action, pick your side of the comment
deliberately.

### Synchronous actions

`BRIDGE_ACTION_SYNC` lets a thread queue work onto another channel's bridge
thread and block until it has been *freed* — not merely executed:

```c
struct sync_payload {
	/*! Unique ID for this synchronous action */
	unsigned int id;
	/*! Actual frame data to process */
	unsigned char data[0];
};
```
— `main/bridge_channel.c:88-93`

```c
static int bridge_channel_queue_action_data_sync(struct ast_bridge_channel *bridge_channel,
	enum bridge_channel_action_type action, const void *data, size_t datalen)
{
	struct sync_payload *sync_payload;
	int sync_payload_len = sizeof(*sync_payload) + datalen;
	struct bridge_sync sync_struct;
	struct ast_frame frame = {
		.frametype = AST_FRAME_BRIDGE_ACTION_SYNC,
		.subclass.integer = action,
	};

	/* Make sure we don't end up trying to wait on ourself to deliver the frame */
	ast_assert(!pthread_equal(pthread_self(), bridge_channel->thread));

	sync_payload = ast_alloca(sync_payload_len);
	sync_payload->id = ast_atomic_fetchadd_int(&sync_ids, +1);
	memcpy(sync_payload->data, data, datalen);

	frame.datalen = sync_payload_len;
	frame.data.ptr = sync_payload;

	bridge_sync_init(&sync_struct, sync_payload->id);
	if (ast_bridge_channel_queue_frame(bridge_channel, &frame)) {
		bridge_sync_cleanup(&sync_struct);
		return -1;
	}

	bridge_sync_wait(&sync_struct);
	bridge_sync_cleanup(&sync_struct);
	return 0;
}
```
— `main/bridge_channel.c:937-967`

Everything interesting here is on the stack. `sync_payload` is `ast_alloca`'d
and the frame borrows it; `ast_bridge_channel_queue_frame()` duplicates both.
`sync_struct` is also stack-allocated and registered in a global list by id —
hence the warning on `bridge_sync_cleanup()`:

```
 * Since synchronization objects are stack-allocated, it is vital
 * that this is called before the synchronization object goes
 * out of scope.
```
— `main/bridge_channel.c:142-144`

The `ast_assert(!pthread_equal(pthread_self(), bridge_channel->thread))` is the
self-deadlock guard. There is a second guard against queueing a sync frame into
the bridge rather than onto a channel:

```c
	ast_assert(frame->frametype != AST_FRAME_BRIDGE_ACTION_SYNC);
```
— `main/bridge_channel.c:653`, in `bridge_channel_write_frame()`

And a failsafe, because the whole mechanism depends on somebody freeing the
frame:

```c
/*!
 * \brief Failsafe for synchronous bridge action waiting.
 *
 * When waiting for a synchronous bridge action to complete,
 * if there is a frame resource leak somewhere, it is possible
 * that we will never get notified that the synchronous action
 * completed.
 *
 * If a significant amount of time passes, then we will abandon
 * waiting for the synchrnous bridge action to complete.
 *
 * This constant represents the number of milliseconds we will
 * wait for the bridge action to complete.
 */
#define PLAYBACK_TIMEOUT (600 * 1000)
```
— `main/bridge_channel.c:165-179`

Ten minutes. If you ever see a thread stuck for exactly ten minutes in
`ast_sem_timedwait` under `bridge_sync_wait`, you have leaked a
`BRIDGE_ACTION_SYNC` frame.

### `bridge_frame_free()` — the rule

```c
static void bridge_frame_free(struct ast_frame *frame)
{
	if (frame->frametype == AST_FRAME_BRIDGE_ACTION_SYNC) {
		struct sync_payload *sync_payload = frame->data.ptr;
		struct bridge_sync *sync;

		AST_RWLIST_RDLOCK(&sync_structs);
		AST_RWLIST_TRAVERSE(&sync_structs, sync, list) {
			if (sync->id == sync_payload->id) {
				break;
			}
		}
		if (sync) {
			bridge_sync_signal(sync);
		}
		AST_RWLIST_UNLOCK(&sync_structs);
	}

	ast_frfree(frame);
}
```
— `main/bridge_channel.c:994-1013`

**Inside `bridge_channel.c`, never call `ast_frfree()` on a frame that came off a
bridge queue. Call `bridge_frame_free()`.** Freeing a sync frame with bare
`ast_frfree()` compiles, runs, and wedges the waiting thread for ten minutes.
This is the single easiest bug to introduce in the bridging code.

Note the signalling happens on *free*, not on *dispatch*. That is what makes the
contract "the function will not return until the queued frame is freed"
(`:927`) rather than "until the action completes" — though in practice the
dispatch switch is immediately followed by the free, so they coincide.

### Queueing into a bridge channel

```c
int ast_bridge_channel_queue_frame(struct ast_bridge_channel *bridge_channel, struct ast_frame *fr)
{
	struct ast_frame *dup;

	if (bridge_channel->suspended
		/* Also defer DTMF frames. */
		&& fr->frametype != AST_FRAME_DTMF_BEGIN
		&& fr->frametype != AST_FRAME_DTMF_END
		&& !ast_is_deferrable_frame(fr)) {
		/* Drop non-deferable frames when suspended. */
		return 0;
	}
	if (fr->frametype == AST_FRAME_NULL) {
		/* "Accept" the frame and discard it. */
		return 0;
	}
```
— `main/bridge_channel.c:1015-1030`

Then the stream mapping check, which silently drops media for unmapped streams
(`:1032-1047`), then `ast_frdup(fr)` (`:1049`), then — under the bridge channel
lock — two more drop conditions, both of which go through `bridge_frame_free()`
on the *dup*: the channel is leaving the bridge (`:1055-1060`), or the channel
does not accept text (`:1062-1068`).

Note that all of these return **0, meaning success**. The caller cannot
distinguish "queued" from "dropped on the floor". Only an `ast_frdup()` failure
returns -1.

Deferrability is decided by a function in `channel.c`, not the bridge:

```c
int ast_is_deferrable_frame(const struct ast_frame *frame)
{
	/* Do not add a default entry in this switch statement.  Each new
	 * frame type should be addressed directly as to whether it should
	 * be queued up or not.
	 */
	switch (frame->frametype) {
	case AST_FRAME_BRIDGE_ACTION:
	case AST_FRAME_BRIDGE_ACTION_SYNC:
	case AST_FRAME_CONTROL:
	case AST_FRAME_TEXT:
	case AST_FRAME_TEXT_DATA:
	case AST_FRAME_IMAGE:
	case AST_FRAME_HTML:
		return 1;

	case AST_FRAME_DTMF_END:
	case AST_FRAME_DTMF_BEGIN:
	case AST_FRAME_VOICE:
	case AST_FRAME_VIDEO:
	case AST_FRAME_NULL:
	case AST_FRAME_IAX:
	case AST_FRAME_CNG:
	case AST_FRAME_MODEM:
	case AST_FRAME_RTCP:
		return 0;
	}
	return 0;
}
```
— `main/channel.c:1454-1482`

Same no-`default:`-label discipline as `is_visible_indication()` in
[04-control-frames.md](04-control-frames.md): adding a frame type produces a
compiler warning until you have decided. The split is "signalling and discrete
events survive a suspend; media does not" — media that arrived during a suspend
is stale by the time the suspend lifts, so dropping it is correct.

### Stream mapping on the way in

`bridge_channel_write_frame()` remaps `frame->stream_num` from the channel's
numbering into the bridge's:

```c
	if (bridge_channel->chan && ast_channel_is_multistream(bridge_channel->chan)) {
		unmapped_stream_num = frame->stream_num;
		switch (frame->frametype) {
		case AST_FRAME_VOICE:
		case AST_FRAME_VIDEO:
		case AST_FRAME_TEXT:
		case AST_FRAME_IMAGE:
		case AST_FRAME_RTCP:
			/* These frames need to be mapped to an appropriate write stream */
			if (frame->stream_num < 0) {
				/* Map to default stream */
				frame->stream_num = -1;
				break;
			}
			if (frame->stream_num < (int)AST_VECTOR_SIZE(&bridge_channel->stream_map.to_bridge)) {
				frame->stream_num = AST_VECTOR_GET(
					&bridge_channel->stream_map.to_bridge, frame->stream_num);
				if (0 <= frame->stream_num) {
					break;
				}
			}
```
— `main/bridge_channel.c:663-683`

It **mutates the caller's frame** — `frame->stream_num` is rewritten in place and
never put back. `unmapped_stream_num` (`main/bridge_channel.c:664`, or `-1` at
`:703`) is not a restore: it is applied only to the `ast_frdup()`'d copy pushed
onto `deferred_queue` (`:712-718`), so a deferred frame re-enters unmapped while
the caller's frame stays rewritten. The two maps,
`stream_map.to_bridge` and `stream_map.to_channel`, are the channel↔bridge
translation in each direction. The outbound side is applied in
`ast_bridge_channel_queue_frame()` (`:1038-1040`) and again in the write handler
(`:2453-2466`).

### The write handler

`bridge_channel_handle_write()` at `main/bridge_channel.c:2373` is the bridge's
equivalent of `__ast_read()`. It opens with the same balance invariant the
channel readq has:

```c
	/* It's not good to have unbalanced frames and alert_pipe alerts. */
	ast_assert(!AST_LIST_EMPTY(&bridge_channel->wr_queue));
	if (AST_LIST_EMPTY(&bridge_channel->wr_queue)) {
		/* No frame, flush the alert pipe of excess alerts. */
		ast_log(LOG_WARNING, "Weird.  No frame from bridge for %s to process?\n",
			ast_channel_name(bridge_channel->chan));
		ast_alertpipe_read(bridge_channel->alert_pipe);
		ast_bridge_channel_unlock(bridge_channel);
		return;
	}
```
— `main/bridge_channel.c:2382-2391`

One alert byte per frame, exactly as in `__ast_queue_frame()`. Then the
selective dequeue:

```c
	AST_LIST_TRAVERSE_SAFE_BEGIN(&bridge_channel->wr_queue, fr, frame_list) {
		if (bridge_channel->dtmf_hook_state.collected[0]) {
			switch (fr->frametype) {
			case AST_FRAME_BRIDGE_ACTION:
			case AST_FRAME_BRIDGE_ACTION_SYNC:
				/* Defer processing these frames while DTMF is collected. */
				continue;
			default:
				break;
			}
		}
		ast_alertpipe_read(bridge_channel->alert_pipe);
		AST_LIST_REMOVE_CURRENT(frame_list);
		break;
	}
	AST_LIST_TRAVERSE_SAFE_END;

	ast_bridge_channel_unlock(bridge_channel);

	if (!fr) {
		/*
		 * Wait some to reduce CPU usage from a tight loop
		 * without any wait because we only have deferred
		 * frames in the wr_queue.
		 */
		usleep(1);
		return;
	}
```
— `main/bridge_channel.c:2393-2420`

Structurally the same trick as `should_skip_dtmf()` in `__ast_read()`: walk the
queue skipping frames that must wait, take the first eligible one, and drain
exactly one alert byte for it. If a DTMF feature sequence is mid-collection,
action frames are held back — you do not want a `PLAY_FILE` to fire while the
user is still typing `*8`. If *everything* in the queue is deferred, `fr` is
`NULL` and the thread `usleep(1)`s rather than spinning; the alert bytes for the
deferred frames remain unread, so the next poll wakes immediately and re-checks.

Then the dispatch switch (`:2422-2472`):

| Frametype | Handling |
| --- | --- |
| `BRIDGE_ACTION` | `bridge_channel_handle_action(..., fr->subclass.integer, fr->data.ptr)` |
| `BRIDGE_ACTION_SYNC` | same, but payload is `((struct sync_payload *) fr->data.ptr)->data` |
| `CONTROL` | `bridge_channel_handle_control()` |
| `NULL` | dropped |
| `TEXT` | `sendtext_safe()` |
| `TEXT_DATA` | `ast_sendtext_data()` with the `struct ast_msg_data` payload |
| default | stream-map, then `ast_write_stream(bridge_channel->chan, num, fr)` |

and then, unconditionally, `bridge_frame_free(fr);` (`:2473`).

`sendtext_safe()` is worth reading as the canonical handling of the
"TEXT may not be zero-terminated" hazard from
[02-frame-types.md](02-frame-types.md):

```c
static void sendtext_safe(struct ast_channel *chan, const struct ast_frame *f)
{
	if (f->datalen) {
		char *text = f->data.ptr;

		if (text[f->datalen - 1]) {
			/* Not zero terminated, we need to allocate */
			text = ast_strndup(text, f->datalen);
			if (!text) {
				return;
			}
		}

		ast_sendtext(chan, text);

		if (text != f->data.ptr) {
			/* Only free if we allocated */
			ast_free(text);
		}
	} else {
		/* Special case if the frame length is zero (although I
		 * am not sure this is possible?) */
		ast_sendtext(chan, "");
	}
}
```
— `main/bridge_channel.c:2340-2364`

Note that `text[f->datalen - 1]` reads the last byte *within* `datalen` — so a
payload of exactly `datalen` bytes with no terminator is handled, but it does
rely on `datalen` being accurate. Set `datalen` on TEXT frames to include the
terminator if you have one.

### DTMF feature hooks consume the frame

```c
	dtmf[0] = frame->subclass.integer;
	dtmf[1] = '\0';
	if (bridge_channel->dtmf_hook_state.collected[0]
		|| (hook = ao2_find(features->dtmf_hooks, dtmf, OBJ_SEARCH_PARTIAL_KEY))) {
		enum ast_frame_type frametype = frame->frametype;

		bridge_frame_free(frame);
		frame = NULL;
```
— `main/bridge_channel.c:2490-2497`

`bridge_handle_dtmf()` **eats** the frame as soon as the digit could be part of a
feature sequence, saving only the frametype. That is why the far end does not
hear the digits you use to invoke a bridge feature. The caller gets `NULL` back
and must treat that as "frame consumed".

---

## Next

[08-debugging-and-recipes.md](08-debugging-and-recipes.md) — how to see all of
this at runtime, a set of copy-and-adapt patterns, and an addendum on the places
where the frame code does not behave the way its interface suggests.
