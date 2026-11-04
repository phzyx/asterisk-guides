# 8. Debugging and recipes

**Target tree:** Asterisk, the `24` branch. Line numbers and quoted code may
differ on newer releases and on other branches; re-grep for the symbol name if
you are reading against a different tree.

How to watch frames at runtime, the bugs that are actually in the tree, and a
set of patterns to copy.

---

## 8.1 `core set debug channel`

The built-in frame tracer. It is not a logger category — it is a bit smuggled
into the top of the per-channel frame counters.

```c
/*!
 * The high bit of the frame count is used as a debug marker, so
 * increments of the counters must be done with care.
 * Please use c->fin = FRAMECOUNT_INC(c->fin) and the same for c->fout.
 */
#define	DEBUGCHAN_FLAG  0x80000000

/* XXX not ideal to evaluate x twice... */
#define	FRAMECOUNT_INC(x)	( ((x) & DEBUGCHAN_FLAG) | (((x)+1) & ~DEBUGCHAN_FLAG) )

/*!
 * The current value of the debug flags is stored in the two
 * variables global_fin and global_fout (declared in main/channel.c)
 */
extern unsigned long global_fin, global_fout;
```
— `include/asterisk/channel.h:872-886`

So `chan->fin` and `chan->fout` are 31-bit counters of frames read and written,
with bit 31 serving as "trace this channel". `FRAMECOUNT_INC` increments the
counter while preserving the flag, which is why you must never write
`chan->fin++`.

The CLI command:

```
Usage: core set debug channel <all|channel> [off]
       Enables/disables debugging on all or on a specific channel.
```
— `main/cli.c:1632-1633`

`all` sets the flag in the two globals (`main/cli.c:1659-1667`) *and* walks every
existing channel. The globals are copied into each new channel at allocation
time:

```c
	ast_channel_fin_set(tmp, global_fin);
	ast_channel_fout_set(tmp, global_fout);
```
— `main/channel.c:864-865`

which is what the CLI means by `"Debugging on new channels is enabled"`
(`main/cli.c:1677`). Naming a single channel sets the flag on that channel only
and leaves the globals alone, so the setting does not survive a masquerade into
a freshly-allocated channel.

One wrinkle in `channel_set_debug()`:

```c
	if (!(ast_channel_fin(chan) & DEBUGCHAN_FLAG) || !(ast_channel_fout(chan) & DEBUGCHAN_FLAG)) {
```
— `main/cli.c:1602`

The whole body is guarded on at least one of the two flags being *clear*. So
once both are set, a subsequent `core set debug channel <name> off` on that same
channel does nothing and prints nothing. Use `core set debug channel all off`,
which bypasses the per-channel guard for the globals and then re-runs
`channel_set_debug` on every channel — where, both flags being set, the guard
again fails. In practice: **turning per-channel frame debug off again is
unreliable; it is simplest to let the channel hang up.** Worth knowing before
you enable it on a production box.

### The two dump points

```c
	/* High bit prints debugging */
	if (ast_channel_fin(chan) & DEBUGCHAN_FLAG)
		ast_frame_dump(ast_channel_name(chan), f, "<<");
	ast_channel_fin_set(chan, FRAMECOUNT_INC(ast_channel_fin(chan)));
```
— `main/channel.c:4260-4263`, at the bottom of `__ast_read()`

```c
	/* High bit prints debugging */
	if (ast_channel_fout(chan) & DEBUGCHAN_FLAG)
		ast_frame_dump(ast_channel_name(chan), fr, ">>");
```
— `main/channel.c:5251-5253`, in `ast_write_stream()`

`<<` is inbound, `>>` is outbound. Note where they sit:

- The **read** dump is at the single return point, *after* the whole frametype
  switch. So you see the frame as the caller will see it — post-translation,
  post-audiohook, post-DTMF-emulation. You do **not** see frames that were
  consumed internally (a swallowed CONTROL, a DTMF_BEGIN absorbed by the
  emulator); they return `&ast_null_frame` or `NULL` and get dumped as that.
- The **write** dump is *before* the frametype switch, but *after* the framehook
  write event (`:5221`) and *after* the generator interception block (which
  `goto done`s at `:5248`, skipping the dump entirely). So a frame suppressed by
  a generator never appears.

Neither dump sees frames moving through `ast_queue_frame()`. If you need to see
what is being queued, you need a framehook or a breakpoint.

### What `ast_frame_dump` actually prints

```c
	/* XXX We should probably print one each of voice and video when the format changes XXX */
	if (f->frametype == AST_FRAME_VOICE) {
		return;
	}
	if (f->frametype == AST_FRAME_VIDEO) {
		return;
	}
	if (f->frametype == AST_FRAME_RTCP) {
		return;
	}
```
— `main/frame.c:754-763`

**Media is silently skipped.** This is the single most important thing to know
about the facility: `core set debug channel` shows you *signalling*, not media.
If you are chasing a codec or timing problem it will tell you nothing. A `NULL`
frame pointer prints as `HANGUP` (`:747-753`).

Everything else prints as:

```
<< [ TYPE: Control (4) SUBCLASS: Ringing (3) ] [SIP/alice-00000001]
>> [ TYPE: DTMF End (1) SUBCLASS: DTMF End (49) '1' ] [SIP/alice-00000001]
```

via `ast_frame_type2str()` and `ast_frame_subclass2str()`, with the optional
`moreinfo` field holding type-specific extras (the digit, for DTMF). Output goes
through `ast_verb(-1, ...)` — level -1 means it bypasses the verbosity setting
entirely and always prints. It is colourised, so expect escape codes in a piped
log.

### Two gaps in the stringifiers

`ast_frame_type2str()` (`main/frame.c:671-728`) has **no case for
`AST_FRAME_CNG`**. A comfort-noise frame prints as
`Unknown Frametype '10'`. Since CNG frames are also not skipped by the media
filter, this is the one media-ish type you *will* see in a trace, and it will be
mislabelled.

`ast_frame_subclass2str()` has a **missing `break`**:

```c
	case AST_FRAME_RTCP:
		ast_copy_string(subclass, "RTCP", slen);
	default:
		ast_copy_string(subclass, "Unknown Subclass", slen);
		break;
	}
```
— `main/frame.c:661-666`

The RTCP case falls through into `default:` and the `"RTCP"` string is
immediately overwritten with `"Unknown Subclass"`. Harmless in practice only
because `ast_frame_dump()` returns early for RTCP before ever calling this — but
any other caller of `ast_frame_subclass2str()` on an RTCP frame gets the wrong
answer. A one-line fix if you want it.

---

## 8.2 Other runtime levers

| Lever | Where | What it gives you |
| --- | --- | --- |
| `core show settings` | `main/asterisk.c:567` | prints `Cache media frames: Enabled/Disabled` |
| `cache_media_frames = no` in `asterisk.conf` | `configs/samples/asterisk.conf.sample:53`, parsed at `main/options.c:363-364` | disables the per-thread frame header cache at runtime |
| `--enable-dev-mode` | configure | turns on `ast_assert()`, including the bridge sync self-wait guard and the `fdno == -1` check |
| `MALLOC_DEBUG` | menuselect | compiles out the frame cache entirely (`NO_FRAME_CACHE`) and tracks every allocation |
| `LOW_MEMORY` | menuselect | also compiles out the frame cache |
| `ast_debug(4, ...)` | `main/channel.c:1137` | `"Discarded %d frames due to queue overload on %s"` |

The cache interaction is the one that bites. Under normal builds, freed frame
headers go onto a per-thread free list of up to `FRAME_CACHE_MAX_SIZE` (10)
rather than back to the allocator, gated at runtime on
`ast_opt_cache_media_frames` (`main/frame.c:130`). The default is on
(`AST_DEFAULT_OPTIONS` in `include/asterisk/options.h:116`). Consequences:

- **A leaked frame header may not show up as a leak**, because the cache holds
  headers legitimately. Set `cache_media_frames = no` before a leak hunt.
- **A use-after-free on a frame header may not crash**, because the memory is
  still mapped and still looks like a frame. Set it to `no` to turn soft
  corruption into hard failures.
- `MALLOC_DEBUG` builds do not have the cache at all, so a bug that only
  reproduces in a production build and vanishes under `MALLOC_DEBUG` is a
  strong hint you are looking at a frame-header lifetime bug.

For leak hunting the cache is also why you should prefer
`cache_media_frames = no` over trusting `memory show allocations` on a default
build.

---

## 8.3 Known bugs and rough edges

Collected from the rest of this guide, in rough order of how likely they are to
matter to you.

Every item below was re-confirmed present on the `24` branch at the time of
writing. Confirm against your own tree before acting on any of them — the point
of the citations is that you can.

### 1. `ast_frame_enqueue()` is declared but never defined

```c
struct ast_frame *ast_frame_enqueue(struct ast_frame *head, struct ast_frame *f, int maxlen, int dupe);
```
— `include/asterisk/frame.h:612`

No definition exists anywhere in the tree. Calling it compiles and fails to
link. Use `ast_queue_frame()` or the `AST_LIST_*` macros.

### 2. Unreachable topology ref-bump in `__ast_frisolate()`

```c
		if ((fr->frametype == AST_FRAME_VOICE) || (fr->frametype == AST_FRAME_VIDEO) ||
			(fr->frametype == AST_FRAME_IMAGE)) {
			ao2_bump(out->subclass.format);
		} else if (fr->frametype == AST_FRAME_VOICE && fr->subclass.integer == AST_CONTROL_ANSWER) {
			ao2_bump(out->subclass.topology);
		}
```
— `main/frame.c:222-227`

The `else if` tests `AST_FRAME_VOICE` where it means `AST_FRAME_CONTROL`, so the
preceding `if` has already matched and the branch is **unreachable**. Meanwhile
`__ast_frdup()` (`:357`) and `__frame_free()` (`:141`, `:165`) both correctly
test `AST_FRAME_CONTROL`. Net effect: `ast_frisolate()` never takes a reference
on a topology, but freeing will release one — a latent over-release. It does not
bite today because ANSWER frames carrying a topology reach the heap via
`ast_frdup()`, not `ast_frisolate()`. If you ever add a path that isolates a
CONTROL/ANSWER frame, fix this first.

### 3. `ast_write()` returns success after giving up on a deadlock

`ast_write_stream()` opens with a `trylock` retry loop rather than a plain
`ast_channel_lock()`, and after twelve failed attempts it gives up — `count` is
post-incremented, so `count++ > 10` first holds on the twelfth:

```c
	/*Deadlock avoidance*/
	while(ast_channel_trylock(chan)) {
		/*cannot goto done since the channel is not locked*/
		if(count++ > 10) {
			ast_debug(1, "Deadlock avoided for write to channel '%s'\n", ast_channel_name(chan));
			return 0;
		}
		usleep(1);
	}
```
— `main/channel.c:5180-5188`

That `return 0` is **success**, on a path where nothing was written, and the
caller cannot tell the frame was dropped. There is no leak — `ast_write_stream()`
never owns `fr` on any path, and at the early return the intermediate `f` is still
`NULL` — but a caller that counts writes, or that treats a 0 as "the far end has
it", is wrong under lock contention and has no way to find out. Re-confirmed
present on `24`. See [05-channel-path.md](05-channel-path.md).

### 4. `tech_write()` returns 0 when no stream matches

```c
static int tech_write(struct ast_channel *chan, struct ast_stream *stream,
			struct ast_stream *default_stream, struct ast_frame *frame)
{
	if (ast_channel_tech(chan)->write_stream) {
		return stream ? ast_channel_tech(chan)->write_stream(
			chan, ast_stream_get_position(stream), frame) : 0;
	}

	return ((stream == default_stream) && ast_channel_tech(chan)->write) ?
		ast_channel_tech(chan)->write(chan, frame) : 0;
}
```
— `main/channel.c:5155-5165`

Both `: 0` arms return 0, so a write to an unmapped stream, or to a non-default
stream on a driver that only has `write`, looks like a successful write.
Re-confirmed present on `24`. See [05-channel-path.md](05-channel-path.md).

### 5. The `default:` case in `ast_write_stream()` calls `tech->write` directly

```c
	default:
		/* At this point, fr is the incoming frame and f is NULL.  Channels do
		 * not expect to get NULL as a frame pointer and will segfault.  Hence,
		 * we output the original frame passed in. */
		CHECK_BLOCKING(chan);
		res = ast_channel_tech(chan)->write(chan, fr);
```
— `main/channel.c:5482-5487`

There is no `if (ast_channel_tech(chan)->write)` guard and no `write_stream`
fallback, so an exotic frametype written to a stream-only channel driver
dereferences a `NULL` `write` pointer. Every other case in the switch either
calls `tech_write()` (which checks) or tests the callback first. See
[05-channel-path.md](05-channel-path.md).

### 6. `ast_translate()` ignores `consume` for RTCP

The RTCP short-circuit returns the frame *before* the `consume` check, so
`ast_translate(path, rtcp_frame, 1)` leaks the input. It is latent because the one
call site that can carry an RTCP frame — `main/channel.c:4241`, the
`case AST_FRAME_RTCP:` arm of `__ast_read()` — passes 0. Of the fourteen
`ast_translate()` call sites outside `main/translate.c`, three pass a non-zero
`consume`: `main/channel.c:4190` sits inside `case AST_FRAME_VOICE:`,
`main/audiohook.c:773` translates a slin frame pulled from a slinfactory, and
`res/res_fax.c:3681` is both guarded by `f->frametype == AST_FRAME_VOICE` and
conditional (`event == AST_FRAMEHOOK_EVENT_WRITE ? 0 : 1`). All three are
frametype-guarded, so this stays a trap for new code rather than a live bug. See
[06-translation-and-timing.md](06-translation-and-timing.md).

### 7. Missing `break` after the RTCP case in `ast_frame_subclass2str()`

`main/frame.c:661-662` — described above.

### 8. No `AST_FRAME_CNG` case in `ast_frame_type2str()`

`main/frame.c:671-728` — described above.

### 9. Audiohook manipulate failures are discarded

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

`middle_frame_manipulated = 1` is set only on success — the callback returns 0 on
success (`include/asterisk/audiohook.h:85-86`), so the assignment sits inside the
`if (!...)`. The failure is what disappears: it is not propagated to the caller of
`ast_audiohook_write_list()` and nothing is logged, so a manipulator that failed
and one that was never reached are indistinguishable from outside.

### 10. The autoservice DTMF audiohook `\todo`

An in-tree `\todo` at `main/channel.c:3934-3935` notes that the DTMF path can
write a digit to the audiohook list twice "if the digit was originally read
while the channel was in autoservice". See
[05-channel-path.md](05-channel-path.md) §DTMF.

---

## 8.4 Recipes

Unlike every other code block in this guide, the blocks in this section are
**written for the guide rather than quoted from the tree** — they are the
shapes the patterns take, with the surrounding error handling and locking left
in but the names invented. Citations here point at the real thing to compare
against.

### Recipe: a stack frame, queued

The commonest pattern in the tree. Build it on the stack, let the queue
duplicate it, never free anything.

<!-- illustrative -->
```c
	struct ast_frame f = {
		.frametype = AST_FRAME_CONTROL,
		.subclass.integer = AST_CONTROL_PROGRESS,
	};

	ast_queue_frame(chan, &f);
	/* nothing to free: __ast_queue_frame() frdup'd it */
```

With a payload — the payload is borrowed, and is also copied by the `frdup`:

<!-- illustrative -->
```c
	struct ast_control_t38_parameters parameters = {
		.request_response = AST_T38_NEGOTIATED,
		.max_ifp = 400,
		.rate = AST_T38_RATE_14400,
	};
	struct ast_frame f = {
		.frametype = AST_FRAME_CONTROL,
		.subclass.integer = AST_CONTROL_T38_PARAMETERS,
		.datalen = sizeof(parameters),
		.data.ptr = &parameters,
	};

	ast_queue_frame(chan, &f);
```

Both are safe because the payload outlives the `ast_queue_frame()` call. The bug
is to stash `&f` or `&parameters` anywhere that outlives the function.

### Recipe: a channel driver returning a frame from `tech->read`

The RTP driver's pattern — a frame embedded in the instance struct, payload
pointing into an instance-owned buffer, `mallocd = 0`:

<!-- illustrative -->
```c
struct my_rtp_instance {
	struct ast_frame f;
	unsigned char rawdata[8192 + AST_FRIENDLY_OFFSET];
	...
};

	/* ... after receiving into rawdata + AST_FRIENDLY_OFFSET ... */
	instance->f.frametype = AST_FRAME_VOICE;
	instance->f.subclass.format = fmt;      /* borrowed, not bumped */
	instance->f.datalen = len;
	instance->f.data.ptr = instance->rawdata + AST_FRIENDLY_OFFSET;
	instance->f.offset = AST_FRIENDLY_OFFSET;
	instance->f.samples = samples;
	instance->f.mallocd = 0;                /* core must not free any of this */
	instance->f.src = "RTP";                /* static string */
	return &instance->f;
```

See `res/res_rtp_asterisk.c:396` for the buffer declaration, `:8201-8208` for
the real media-frame assembly and `:6048-6052` for the DTMF one. The `AST_FRIENDLY_OFFSET` headroom is what
lets a downstream consumer prepend a header without reallocating. Three rules:

- `mallocd = 0` means the core will not free it, so the frame is only valid
  until your next `read` on the same instance. Anything that needs to keep it
  calls `ast_frdup()` or `ast_frisolate()`.
- `src` must be a string literal or otherwise permanently valid — `__ast_frdup()`
  will `strlen()` it.
- You hold no reference on `subclass.format`. The format cache keeps the common
  formats alive; if you are using a dynamically created format, hold your own
  ref for the lifetime of the instance.

### Recipe: a read-only framehook that does not break native bridging

<!-- illustrative -->
```c
static int my_consume(void *data, enum ast_frame_type type)
{
	/* 0 means "ignoring this type" */
	return type == AST_FRAME_CONTROL;
}

static struct ast_frame *my_event(struct ast_channel *chan, struct ast_frame *frame,
	enum ast_framehook_event event, void *data)
{
	struct my_state *state = data;

	switch (event) {
	case AST_FRAMEHOOK_EVENT_ATTACHED:
	case AST_FRAMEHOOK_EVENT_DETACHED:
		return NULL;
	case AST_FRAMEHOOK_EVENT_READ:
	case AST_FRAMEHOOK_EVENT_WRITE:
		break;
	}

	if (frame && frame->frametype == AST_FRAME_CONTROL) {
		state->last_control = frame->subclass.integer;
	}

	return frame;     /* unchanged: we own nothing */
}

static void my_destroy(void *data)
{
	ast_free(data);
}

	/* attach */
	struct my_state *state = ast_calloc(1, sizeof(*state));
	struct ast_framehook_interface i = {
		.version = AST_FRAMEHOOK_INTERFACE_VERSION,
		.event_cb = my_event,
		.destroy_cb = my_destroy,
		.consume_cb = my_consume,
		.data = state,
	};
	int id;

	ast_channel_lock(chan);
	id = ast_framehook_attach(chan, &i);
	ast_channel_unlock(chan);
	if (id < 0) {
		ast_free(state);
	}
```

The `consume_cb` is the whole point: without it, `ast_channel_has_hook_requiring_audio()`
returns true and every bridge this channel joins is forced out of native
bridging. See `ast_framehook_list_contains_no_active_of_type()`,
`main/framehook.c:282-311`.

Note that the channel must be locked across `ast_framehook_attach()`
(`include/asterisk/framehook.h:265`), and that attach calls your `event_cb` with
the `ATTACHED` event *before* returning the id — so your `data` must be fully
initialised first.

### Recipe: a framehook that replaces a frame

<!-- illustrative -->
```c
	new_frame = make_replacement(frame);
	if (!new_frame) {
		return frame;        /* failure: hand it back untouched */
	}

	ast_frfree(frame);       /* we are replacing it, so we free it */
	return new_frame;
```

And to drop a frame entirely:

<!-- illustrative -->
```c
	ast_frfree(frame);
	return NULL;
```

Remember from [07-interception.md](07-interception.md) that returning a different
pointer restarts the list walk, so your hook may be called again for the *next*
frame sooner than you expect, and other hooks will see your replacement.

### Recipe: isolating a frame you need to keep

<!-- illustrative -->
```c
	struct ast_frame *keeper;

	keeper = ast_frdup(f);          /* always a fresh heap frame */
	if (!keeper) {
		return -1;
	}
	/* f is still owned by whoever gave it to us */
```

Use `ast_frisolate()` instead when you want to *avoid* a copy where possible:

<!-- illustrative -->
```c
	struct ast_frame *isolated;

	isolated = ast_frisolate(f);
	if (!isolated) {
		return -1;
	}
	if (isolated != f) {
		ast_frfree(f);           /* only if you owned f */
	}
	/* now use `isolated` and only `isolated` */
```

The `isolated != f` check is mandatory: `ast_frisolate()` may return the same
pointer it was given. Never write `ast_frfree(f); use(isolated);` without it.

### Recipe: walking a frame list

Frames are a list, and `ast_read()`/`ast_translate()` can hand you more than
one:

<!-- illustrative -->
```c
	struct ast_frame *cur, *next;

	for (cur = f; cur; cur = next) {
		next = AST_LIST_NEXT(cur, frame_list);
		AST_LIST_NEXT(cur, frame_list) = NULL;   /* detach before handing on */
		do_something(cur);
		ast_frfree(cur);
	}
```

The `AST_LIST_NEXT(cur, frame_list) = NULL` is the safety pattern from
`__frame_free()`: `ast_frfree()` on a frame whose `frame_list` still points
somewhere will free the whole tail too. Detaching first makes the ownership
explicit. See [03-memory-and-ownership.md](03-memory-and-ownership.md).

### Recipe: inside `bridge_channel.c`

<!-- illustrative -->
```c
	bridge_frame_free(fr);      /* NOT ast_frfree(fr) */
```

Always. A bare `ast_frfree()` on a `BRIDGE_ACTION_SYNC` frame leaves the
originating thread blocked for `PLAYBACK_TIMEOUT` — ten minutes
(`main/bridge_channel.c:179`). See [07-interception.md](07-interception.md).

### Recipe: do not free these

<!-- illustrative -->
```c
	if (f == &ast_null_frame) {
		/* shared global. Do not free, do not modify. */
		return;
	}
```

`&ast_null_frame` has `mallocd == 0` so `ast_frfree()` is technically a no-op on
it, but comparing by pointer and bailing is clearer and protects you from
accidentally mutating it. Never `ast_frdup()` it either — just return it onward.

---

## 8.5 A debugging checklist

**"My frames are not arriving."**
Check, in order: is the queue overloaded (`"Exceptionally long ... queue
length"` at `main/channel.c:1118`)? Is the frame type droppable — only VOICE,
VIDEO and NULL are flushed (`:1124`), and at most 64 of them (`:1125-1127`)? Is
the stream mapped — unmapped media is dropped silently in both
`ast_bridge_channel_queue_frame()` (`main/bridge_channel.c:1043-1046`) and
`bridge_channel_handle_write()` (`:2463-2465`)? Is the bridge channel suspended
and the frame not deferrable (`ast_is_deferrable_frame()`,
`main/channel.c:1454-1482`)?

**"Frames arrive out of order or doubled."**
Suspect the framehook re-run loop — a hook returning a new pointer causes the
whole list to be re-walked (`main/framehook.c:98-127`). Suspect
`AST_FRFLAG_REQUEUED`. Suspect the DTMF emulation state machine producing a
synthetic END.

**"Double free / use after free on a frame."**
Who owned it? Walk the table in
[03-memory-and-ownership.md](03-memory-and-ownership.md). The three usual
culprits: a framehook that freed a frame it then returned; an `ast_frisolate()`
return value used without the `!= f` check; a frame freed while still linked
into a `frame_list`.

**"Leaking memory under load."**
Set `cache_media_frames = no` first, then rebuild with `MALLOC_DEBUG`. The
per-thread cache will otherwise mask ten headers per thread and keep freed
headers looking valid.

**"Audio is silent / distorted but signalling works."**
`core set debug channel` will not help — it skips VOICE and VIDEO
(`main/frame.c:755-760`). Go to the translation path instead: check
`samples` vs `datalen` consistency, check for interpolated frames
(`data.ptr == NULL`, `samples > 0`), check the smoother on the driver side. See
[06-translation-and-timing.md](06-translation-and-timing.md).

**"A thread is stuck for exactly ten minutes."**
A leaked `AST_FRAME_BRIDGE_ACTION_SYNC` frame. Find the `ast_frfree()` that
should have been `bridge_frame_free()`.

---

## Back to the top

[README.md](README.md) — index and the one-page mental model.
