# 05 — The channel read and write paths

**Target tree:** Asterisk, the `24` branch. Line numbers and quoted code may
differ on newer releases and on other branches; re-grep for the symbol name if
you are reading against a different tree.

Everything in this file lives in `main/channel.c`. This is where frames actually
move.

## Channel file descriptors

A channel exposes a small fixed array of fds. `include/asterisk/channel.h:196-206`:

| Constant | Value | Purpose |
|---|---|---|
| `AST_MAX_FDS` | 11 | Size of the array |
| `AST_ALERT_FD` | 10 | The alert pipe — "there is a frame on the readq" |
| `AST_TIMING_FD` | 9 | Timer events |
| `AST_GENERATOR_FD` | 7 | Generator wakeups |
| `AST_JITTERBUFFER_FD` | 6 | Jitterbuffer wakeups |

Indices 0 through 5 belong to the channel driver. The core reserves the high ones.

`ast_channel_fdno(chan)` records **which fd woke us up**, and `__ast_read()`
branches on it. `ast_waitfor_nandfds()` sets it; `__ast_read()` resets it to -1
when done (`main/channel.c:3810`).

## `ast_waitfor` and friends

All four public waiters funnel into one function
(`main/channel.c:3007`):

```c
struct ast_channel *ast_waitfor_nandfds(struct ast_channel **c, int n, int *fds, int nfds,
	int *exception, int *outfd, int *ms);
```

| Wrapper | Line | Equivalent |
|---|---|---|
| `ast_waitfor_n_fd()` | `main/channel.c:3002` | `(NULL, 0, fds, n, exception, &winner, ms)` |
| `ast_waitfor_n()` | `main/channel.c:3176` | `(c, n, NULL, 0, NULL, NULL, ms)` |
| `ast_waitfor()` | `main/channel.c:3181` | `(&c, 1, NULL, 0, NULL, NULL, &ms)` |
| `ast_waitfordigit_full()` | `main/channel.c:3258` | adds a command fd |

The contract is: **call `ast_waitfor()` before `ast_read()`**. In a devmode build,
calling `ast_read()` without it is loud (`main/channel.c:3576-3594`):

```c
		if (ast_channel_fdno(chan) == -1) {
			ast_log(LOG_ERROR,
				"ast_read() on chan '%s' called with no recorded file descriptor.\n",
				ast_channel_name(chan));
		}
```

with the comment explaining the stakes: "One of the potential problems is blocking
on a dead channel."

---

## `__ast_queue_frame()` — getting onto the readq

`main/channel.c:1028-1156`. Signature:

```c
static int __ast_queue_frame(struct ast_channel *chan, struct ast_frame *fin, int head, struct ast_frame *after);
```

Public wrappers (`main/channel.c:1171-1179`):

```c
int ast_queue_frame(struct ast_channel *chan, struct ast_frame *fin)
{
	return __ast_queue_frame(chan, fin, 0, NULL);
}

int ast_queue_frame_head(struct ast_channel *chan, struct ast_frame *fin)
{
	return __ast_queue_frame(chan, fin, 1, NULL);
}
```

The `after` parameter has no public wrapper; it exists so `__ast_read()` can splice
multi-frame translator output into the *middle* of the queue
(`main/channel.c:4223`).

### Step 1 — HANGUP / END_OF_Q collapsing

`main/channel.c:1044-1073`. If the last frame on the queue is a control frame:

- `END_OF_Q` and we are queuing a `HANGUP` → destroy the `END_OF_Q` and queue the
  hangup in its place. Note the careful `after = NULL` with its comment:
  ```c
  				/*
  				 * This has degenerated to a normal queue append anyway.  Since
  				 * we just destroyed the last frame in the queue we must make
  				 * sure that "after" is NULL or bad things will happen.
  				 */
  				after = NULL;
  ```
- `END_OF_Q` (anything else) or `HANGUP` → **queue nothing, return 0 (success)**.

So queuing onto a hung-up channel silently succeeds without queuing. Do not
interpret a 0 return as "the frame will be delivered."

### Step 2 — Duplicate everything

`main/channel.c:1076-1091`. Every input frame is `ast_frdup()`'d into a local list.
**This is the source of the "callers keep ownership" rule.**

On allocation failure it frees the partial list and returns -1 — and note it does
so with `ast_frfree(AST_LIST_FIRST(&frames))`, relying on `ast_frame_free()`
walking the chain.

### Step 3 — Queue-overload flush

`main/channel.c:1101-1126`. The thresholds are bare literals:

```c
	if ((queued_frames + new_frames > 128 || queued_voice_frames + new_voice_frames > 96)) {
```

128 total frames or 96 voice frames. When tripped, it logs at `LOG_WARNING`:

> Exceptionally long %squeue length (%d voice / %d total) queuing to %s

then flushes up to 64 droppable frames:

```c
		AST_LIST_TRAVERSE_SAFE_BEGIN(ast_channel_readq(chan), cur, frame_list) {
			/* Save the most recent frame */
			if (!AST_LIST_NEXT(cur, frame_list)) {
				break;
			} else if (cur->frametype == AST_FRAME_VOICE || cur->frametype == AST_FRAME_VIDEO || cur->frametype == AST_FRAME_NULL) {
				if (++count > 64) {
					break;
				}
				AST_LIST_REMOVE_CURRENT(frame_list);
				ast_frfree(cur);

				/* Read from the alert pipe for each flushed frame. */
				ast_channel_internal_alert_read(chan);
			}
		}
```

Three things to note:

1. **Only VOICE, VIDEO and NULL are droppable.** Control frames and DTMF survive —
   signalling is never sacrificed to relieve pressure.
2. **The last frame is always kept** (`if (!AST_LIST_NEXT(cur, frame_list)) break;`).
3. **The alert pipe is drained in lockstep.** One byte per frame, in both
   directions. If you ever manipulate the readq directly you must keep that
   invariant or the channel will wake up spuriously forever, or never.

Seeing "Exceptionally long queue length" in logs means something is queuing faster
than the channel's owner is reading — usually a thread that forgot to call
`ast_read()`, or a bridge that is blocked.

### Step 4 — Splice

`main/channel.c:1128-1136`. Three cases: after a given frame, at the head, or at
the tail.

### Step 5 — Wake the reader

`main/channel.c:1138-1151`, in priority order:

```c
	if (ast_channel_alert_writable(chan)) {
		/* Write to the alert pipe for each added frame */
		while (new_frames--) {
			if (ast_channel_alert_write(chan)) { ... break; }
		}
	} else if (ast_channel_timingfd(chan) > -1) {
		ast_timer_enable_continuous(ast_channel_timer(chan));
	} else if (ast_test_flag(ast_channel_flags(chan), AST_FLAG_BLOCKING)) {
		pthread_kill(ast_channel_blocker(chan), SIGURG);
	}
```

1. **Alert pipe** — one byte per frame. The normal case.
2. **Continuous timer mode** — for channels with a timing fd but no alert pipe.
   `__ast_read()` turns it off again once the queue drains
   (`main/channel.c:3654-3658`).
3. **`SIGURG`** — the last resort, for a channel that is blocking with neither.
   `AST_FLAG_BLOCKING` plus `ast_channel_blocker(chan)` identify the thread.

---

## `__ast_read()` — the long one

`main/channel.c:3544`. There is a design note right at the top
(`main/channel.c:3551-3553`):

> this function is very long so make sure there is only one return point at the
> end (there are only two exceptions to this).

Honour that if you patch it. The two exceptions are both in the timing-fd path
where the channel has already been unlocked.

### The four public wrappers

`main/channel.c:4276-4294`:

| Function | `dropaudio` | `dropnondefault` |
|---|---|---|
| `ast_read()` | 0 | 1 |
| `ast_read_stream()` | 0 | 0 |
| `ast_read_noaudio()` | 1 | 1 |
| `ast_read_stream_noaudio()` | 1 | 0 |

`dropaudio` discards voice frames (useful when you only care about signalling).
`dropnondefault` replaces any frame from a non-default stream with a null frame —
so legacy single-stream callers never see multistream media.

### Order of operations

**1. Zombie / hangup check** — `main/channel.c:3556-3574`. The subtlety is that a
soft hangup does *not* immediately bail; it queues `AST_CONTROL_END_OF_Q` so
already-queued frames can still be drained:

```c
		/*
		 * It is possible for chan->_softhangup to be set and there
		 * still be control frames that need to be read.  Instead of
		 * just going to 'done' in the case of ast_check_hangup(), we
		 * need to queue the end-of-Q frame so that it can mark the end
		 * of the read queue.  If there are frames to be read,
		 * ast_queue_control() will be called repeatedly, but will only
		 * queue the first end-of-Q frame.
		 */
```

The "only queue the first" is enforced by the collapsing logic in
`__ast_queue_frame()` described above.

**2. Timing fd** — `main/channel.c:3599-3660`. Two events:

`AST_TIMING_EVENT_EXPIRED` → ack the timer, then call `timingfunc` **with the
channel unlocked** (`main/channel.c:3614-3652`), taking an ao2 ref on
`timingdata` first if `AST_FLAG_TIMINGDATA_IS_AO2_OBJ` is set. This is the path
that produces the "cannot `goto done`" return:

```c
			/* cannot 'goto done' because the channel is already unlocked */
			return &ast_null_frame;
```

If DTMF emulation needs a tick, it re-locks and produces a null frame instead;
otherwise the timer rate is reset to 0 and one last null frame is emitted.

`AST_TIMING_EVENT_CONTINUOUS` → disable continuous mode once the readq has at most
one frame (`main/channel.c:3654-3658`).

**3. Generator fd** — `main/channel.c:3662-3672`. Calls `generate(chan, tmp, -1, -1)`
with `generatordata` temporarily NULLed out so `ast_write()` can get through, and
returns a null frame.

**4. Jitterbuffer fd** — `main/channel.c:3673-3675`. Just clears
`AST_FLAG_EXCEPTION`.

**5. Drain one byte from the alert pipe** — `main/channel.c:3677-3682`:

```c
	if (ast_channel_internal_alert_read(chan) == AST_ALERT_READ_FATAL) {
		f = &ast_null_frame;
		goto done;
	}
```

One byte per frame, matching the write side exactly.

**6. Readq first** — `main/channel.c:3685-3758`. The queue is preferred over the
driver, and the dequeue is **selective** because DTMF frames may need to stay
queued:

```c
		AST_LIST_TRAVERSE_SAFE_BEGIN(ast_channel_readq(chan), f, frame_list) {
			if ( (f->frametype == AST_FRAME_DTMF_BEGIN || f->frametype == AST_FRAME_DTMF_END) && skip_dtmf) {
				skipped_dtmf_frame = 1;
				continue;
			}

			AST_LIST_REMOVE_CURRENT(frame_list);
			break;
		}
```

`should_skip_dtmf()` (`main/channel.c:3504-3520`) returns 1 when
`AST_FLAG_DEFER_DTMF` or `AST_FLAG_EMULATE_DTMF` is set, or when less than
`AST_MIN_DTMF_GAP` (45 ms) has passed since the last digit.

If nothing acceptable was found, the handling differs depending on *why*
(`main/channel.c:3704-3720`) — it pokes the alert pipe to come back, unless only
DTMF frames remain, in which case it disables the continuous timer instead.

HANGUP and END_OF_Q from the queue become a NULL return
(`main/channel.c:3722-3736`), stashing `f->data.uint32` as the cause:

```c
			case AST_CONTROL_HANGUP:
				ast_channel_softhangup_internal_flag_add(chan, AST_SOFTHANGUP_DEV);
				cause = f->data.uint32;
				/* Fall through */
			case AST_CONTROL_END_OF_Q:
				ast_frfree(f);
				f = NULL;
				break;
```

Note the `/* XXX why not the same for frames from the channel ? */` at
`main/channel.c:3723` — frames coming directly from `tech->read` are *not*
subjected to this check.

Stream resolution for VOICE/VIDEO happens here too, and a frame whose media type
has no negotiated stream is **dropped** (`main/channel.c:3746-3756`):

> In order to allow media to be passed up the underlying media type has to have a
> format negotiated on the channel itself. In cases where this hasn't happened the
> channel driver is incorrectly passing up a frame for a format that has not been
> negotiated. If this occurs just drop the frame as we have no stream that it came
> from.

**7. Otherwise ask the driver** — `main/channel.c:3759-3798`, in order:

```c
		if (ast_test_flag(ast_channel_flags(chan), AST_FLAG_EXCEPTION)) {
			if (ast_channel_tech(chan)->exception)
				f = ast_channel_tech(chan)->exception(chan);
			...
		} else if (ast_channel_tech(chan) && ast_channel_tech(chan)->read_stream) {
			f = ast_channel_tech(chan)->read_stream(chan);
			...
		} else if (ast_channel_tech(chan) && ast_channel_tech(chan)->read) {
			f = ast_channel_tech(chan)->read(chan);
```

`read_stream` wins over `read` when both exist. A driver with neither gets a
`LOG_WARNING`.

**The frame the driver returns is the driver's.** Typically `mallocd == 0`,
pointing into a per-instance buffer that will be overwritten on the next read.
That is the whole reason `ast_frisolate()` exists.

**8. Framehook READ event** — `main/channel.c:3782-3786`, only for the default
stream:

```c
	if (stream == default_stream) {
		/* Perform the framehook read event here. After the frame enters the framehook list
		 * there is no telling what will happen, <insert mad scientist laugh here>!!! */
		f = ast_framehook_list_read_event(ast_channel_framehooks(chan), f);
	}
```

**9. Reset `fdno`** — `main/channel.c:3806-3810`.

**10. Spill excess driver frames** — `main/channel.c:3818-3825`:

```c
		if (AST_LIST_NEXT(f, frame_list)) {
			ast_queue_frame(chan, AST_LIST_NEXT(f, frame_list));
			ast_frfree(AST_LIST_NEXT(f, frame_list));
			AST_LIST_NEXT(f, frame_list) = NULL;
		}
```

Queue (which copies), then free the originals, then unlink. A clean illustration
of the frdup-on-queue convention.

**11. `dropnondefault` absorption** — `main/channel.c:3827-3833`.

**12. The big frametype switch** — `main/channel.c:3835-4245`.

**13. Debug dump** — `main/channel.c:4258-4261`:

```c
	if (ast_channel_fin(chan) & DEBUGCHAN_FLAG) {
		ast_frame_dump(ast_channel_name(chan), f, "<<");
	}
```

### The control-frame case

`main/channel.c:3835-3902`. Highlights:

- `AST_CONTROL_ANSWER` — deduplicated against the current channel state, sets the
  answer time via `set_channel_answer_time()` and calls `ast_setstate()`. If the
  channel was already up, the frame is swallowed.
- `AST_CONTROL_READ_ACTION` — executes one of the three
  `AST_FRAME_READ_ACTION_*` actions and then **swallows the frame**, returning a
  null frame. This is the "do it on my own thread" mechanism; see
  [04-control-frames.md](04-control-frames.md).
- Topology request/changed — swallowed when `dropnondefault` is set, since a
  single-stream caller cannot act on them.
- `AST_CONTROL_FLASH` / `AST_CONTROL_WINK` — raise channel events.

### The voice case, in order

`main/channel.c:4073-4234`. This is the single most consequential block in the
file.

1. **Non-default stream short-circuit** (`:4074-4077`) — "let it just go through",
   no processing at all.
2. **Clear stale DTMF emulation** (`:4079-4086`).
3. **`dropaudio` or mid-digit** (`:4088-4093`) → frame becomes null.
4. **DTMF emulation interception** (`:4095-4118`) — either synthesizes the DTMF
   END frame or drops the voice frame to keep the digit clean. Detail below.
5. **Format adaptation** (`:4122-4169`). If the frame matches neither
   `rawreadformat` nor `readformat`, rebuild the read translation path. The
   `core_format` choice is the interesting part (`:4159-4163`):
   ```c
   				core_format = ast_channel_readformat(chan);
   				if (!ast_format_cache_is_slinear(core_format)) {
   					core_format = f->subclass.format;
   				}
   				if (ast_set_read_format_path(chan, f->subclass.format, core_format)) {
   					/* Drop frame.  We couldn't make it compatible with the core. */
   					ast_frfree(f);
   					f = &ast_null_frame;
   					break;
   				}
   ```
   The reasoning in the comment above it is worth reading in the source: if the
   core wants slinear it *needs* slinear (softmix bridges, recording, AMD), but
   otherwise the core does not care and we pass the frame through untranslated so
   the bridge technology can sort out compatibility. It also warns that
   `transcode_via_slin` (on by default) and `genericplc` force slin anyway.
6. **Audiohooks, first chance** (`:4170-4184`) — only if the frame is *already*
   slinear, setting `hooked = 1`:
   > Send frame to audiohooks if present, if frametype is linear, to preserve
   > functional compatibility with previous behavior. If not linear, hold off until
   > transcoding is done where we are more likely to have a linear frame
7. **Translate** (`:4186-4192`) — **with `consume = 1`**:
   ```c
   			f = ast_translate(ast_channel_readtrans(chan), f, 1);
   			if (!f) {
   				f = &ast_null_frame;
   			}
   ```
   `ast_translate()` frees the input. Note it only runs when the frame matches
   `rawreadformat`.
8. **Audiohooks, second chance** (`:4194-4202`) — only if not already hooked.
   "also the last chance".
9. **Requeue multi-frame translator output** (`:4204-4227`) — marks every extra
   frame `AST_FRFLAG_REQUEUED` and splices them in *before* anything queued
   earlier in this call:
   ```c
   			if (!readq_tail) {
   				ast_queue_frame_head(chan, multi_frame);
   			} else {
   				__ast_queue_frame(chan, multi_frame, 0, readq_tail);
   			}
   			ast_frfree(multi_frame);
   			AST_LIST_NEXT(f, frame_list) = NULL;
   ```
   `readq_tail` was captured at `main/channel.c:3813`, before any of this call's
   own queuing — which is exactly what makes the ordering correct.
10. **Generator actions** (`:4229-4233`) — `ast_read_generator_actions(chan, f)`,
    for when there is no timing device and outgoing frames must be generated in
    step with incoming ones.

### The RTCP case

`main/channel.c:4237-4243`:

```c
		case AST_FRAME_RTCP:
			/* Incoming RTCP feedback needs to get to the translator for
			 * outgoing media, which means we treat it as an ast_write */
			if (ast_channel_writetrans(chan)) {
				ast_translate(ast_channel_writetrans(chan), f, 0);
			}
			break;
```

`consume = 0`, and the return value is discarded — `ast_translate()` returns
`&ast_null_frame` for RTCP anyway. The frame continues up to the caller unchanged.

---

## DTMF: the emulation state machine

The core cannot assume a channel driver produces well-formed DTMF. Some report
only completion; some report durations shorter than the configured minimum; some
report digits with no gap between them. So the core maintains a state machine that
manufactures the missing halves.

### The flags

| Flag | Meaning |
|---|---|
| `AST_FLAG_IN_DTMF` | A BEGIN was seen and passed through; waiting for the END |
| `AST_FLAG_EMULATE_DTMF` | The core is synthesizing a digit's duration |
| `AST_FLAG_DEFER_DTMF` | Someone asked for DTMF to be held back entirely |
| `AST_FLAG_END_DTMF_ONLY` | This channel only ever reports END; do not emulate |

### The constants

| Constant | Value | Location | Role |
|---|---|---|---|
| `AST_DEFAULT_EMULATE_DTMF_DURATION` | 100 | `main/channel.c:102` | Duration used when an END arrives with `len == 0` |
| `DEFAULT_EMULATE_MF_DURATION` | 55 | `main/channel.c:106` | MF equivalent |
| `AST_MIN_DTMF_GAP` | 45 | `main/channel.c:112` | Minimum inter-digit gap in ms |
| `option_dtmfminduration` | `AST_MIN_DTMF_DURATION` | `main/options.c:83` | Configurable minimum digit duration; shown in CLI as "Min DTMF duration" (`main/asterisk.c:559`) |

### The scratch frame

`ast_channel_dtmff(chan)` is a **per-channel preallocated frame** used for
synthesized DTMF. It is reused, so the core can manufacture a DTMF frame without
allocating. Both `queue_dtmf_readq()` and the emulation paths write into it.

`queue_dtmf_readq()` (`main/channel.c:3483-3496`):

```c
static inline void queue_dtmf_readq(struct ast_channel *chan, struct ast_frame *f)
{
	struct ast_frame *fr = ast_channel_dtmff(chan);

	fr->frametype = AST_FRAME_DTMF_END;
	fr->subclass.integer = f->subclass.integer;
	fr->len = f->len;

	/* The only time this function will be called is for a frame that just came
	 * out of the channel driver.  So, we want to stick it on the tail of the
	 * readq. */

	ast_queue_frame(chan, fr);
}
```

Safe only because `ast_queue_frame()` copies. If it did not, the next digit would
overwrite the queued one.

### DTMF_BEGIN handling

`main/channel.c:4017-4032`. A BEGIN is **ignored** (replaced with a null frame) if
DTMF is deferred, END-only, already emulating, or the inter-digit gap has not
elapsed:

```c
			if (ast_test_flag(ast_channel_flags(chan), AST_FLAG_DEFER_DTMF | AST_FLAG_END_DTMF_ONLY | AST_FLAG_EMULATE_DTMF) ||
			    (!ast_tvzero(*ast_channel_dtmf_tv(chan)) &&
			      ast_tvdiff_ms(ast_tvnow(), *ast_channel_dtmf_tv(chan)) < AST_MIN_DTMF_GAP) ) {
				ast_log(LOG_DTMF, "DTMF begin ignored '%c' on %s\n", f->subclass.integer, ast_channel_name(chan));
				ast_frfree(f);
				f = &ast_null_frame;
			} else {
				struct timeval now = ast_tvnow();
				ast_set_flag(ast_channel_flags(chan), AST_FLAG_IN_DTMF);
				ast_channel_dtmf_tv_set(chan, &now);
				ast_log(LOG_DTMF, "DTMF begin passthrough '%c' on %s\n", f->subclass.integer, ast_channel_name(chan));
			}
```

### DTMF_END handling

`main/channel.c:3903-4016`. Three branches.

**(a) Deferred or already emulating** (`:3906-3910`) — requeue it and return null:

```c
			if (ast_test_flag(ast_channel_flags(chan), AST_FLAG_DEFER_DTMF) || ast_test_flag(ast_channel_flags(chan), AST_FLAG_EMULATE_DTMF)) {
				queue_dtmf_readq(chan, f);
				ast_frfree(f);
				f = &ast_null_frame;
			}
```

**(b) No BEGIN was seen, and this channel is not END-only** (`:3911-3952`) — the
core **turns the END into a BEGIN** and promises to send the END later:

```c
				} else {
					/* There was no begin, turn this into a begin and send the end later */
					struct timeval tv = ast_tvnow();
					f->frametype = AST_FRAME_DTMF_BEGIN;
					ast_set_flag(ast_channel_flags(chan), AST_FLAG_EMULATE_DTMF);
					ast_channel_dtmf_digit_to_emulate_set(chan, f->subclass.integer);
					ast_channel_dtmf_tv_set(chan, &tv);
					if (f->len) {
						if (f->len > option_dtmfminduration)
							ast_channel_emulate_dtmf_duration_set(chan, f->len);
						else
							ast_channel_emulate_dtmf_duration_set(chan, option_dtmfminduration);
					} else
						ast_channel_emulate_dtmf_duration_set(chan, AST_DEFAULT_EMULATE_DTMF_DURATION);
```

and then starts a **50 fps timer** so null frames arrive to drive the state
machine even if no voice is flowing (`:3934-3942`):

```c
					/*
					 * Start generating 50 fps timer events (null frames) for dtmf emulating
					 * independently from any existing incoming voice frames.
					 * If channel generator is already activated in regular mode use these
					 * timer events to generate null frames.
					 */
					if (!ast_channel_generator(chan)) {
						ast_timer_set_rate(ast_channel_timer(chan), 50);
					}
```

That block appears **four times** in this region (`:3940`, `:3990`, `:4005`,
`:4067`) — every state transition that owes a future END frame arms the timer.

If the gap since the last digit is too short, the digit is deferred instead
(`:3912-3917`).

**(c) A BEGIN was seen, or the channel is END-only** (`:3953-4015`). Computes the
real duration, and if it is below `option_dtmfminduration`, records the *actual*
wire duration so emulation triggers (`:3961-3972`):

> detect tones that were received on the wire with durations shorter than
> option_dtmfminduration and set f->len to the actual duration of the DTMF frames
> on the wire. This will cause dtmf emulation to be triggered later on.

Then either swallows the frame and emulates the remaining duration
(`:3977-3992`), or passes it through with `len` clamped up to the minimum
(`:3993-4008`).

### Where the emulated END actually gets produced

Two places, both using the scratch frame.

**From a null frame** (`main/channel.c:4033-4072`) — this is what the 50 fps timer
feeds:

```c
				} else if (ast_tvdiff_ms(now, *ast_channel_dtmf_tv(chan)) >= ast_channel_emulate_dtmf_duration(chan)) {
					ast_channel_emulate_dtmf_duration_set(chan, 0);
					ast_frfree(f);
					f = ast_channel_dtmff(chan);
					f->frametype = AST_FRAME_DTMF_END;
					f->subclass.integer = ast_channel_dtmf_digit_to_emulate(chan);
					f->len = ast_tvdiff_ms(now, *ast_channel_dtmf_tv(chan));
```

**From a voice frame** (`main/channel.c:4095-4118`) — same logic, and if the
duration has *not* elapsed the voice frame is dropped to keep the digit clean:

```c
				} else {
					/* Drop voice frames while we're still in the middle of the digit */
					ast_frfree(f);
					f = &ast_null_frame;
				}
```

### Why `AST_FLAG_EMULATE_DTMF` is cleared late

The same comment appears at `main/channel.c:4034-4037` and `:4079-4082`:

> The EMULATE_DTMF flag must be cleared here as opposed to when the duration is
> reached, because we want to make sure we pass at least one voice frame through
> before starting the next digit, to ensure a gap between DTMF digits.

A deliberate one-frame delay. If you "optimize" it away you will get digits
running together.

### DTMF and audiohooks

Every DTMF branch runs the read-direction audiohook list afterwards, with the
same free-the-old-frame pattern (`main/channel.c:3944-3952`, `:4009-4014`,
`:4054-4060`, `:4105-4110`):

```c
				if (ast_channel_audiohooks(chan)) {
					struct ast_frame *old_frame = f;
					f = ast_audiohook_write_list(chan, ast_channel_audiohooks(chan), AST_AUDIOHOOK_DIRECTION_READ, f);
					if (old_frame != f)
						ast_frfree(old_frame);
				}
```

There is a known defect flagged at `main/channel.c:3946-3948`:

> `\todo` XXX It is possible to write a digit to the audiohook twice if the digit
> was originally read while the channel was in autoservice.

---

## `ast_write_stream()` — the write path

`main/channel.c:5168`. `ast_write()` is just
`ast_write_stream(chan, -1, fr)` (`main/channel.c:5163-5166`).

### The two frame pointers

Internally there are two: `fr`, the **caller's** frame, which is never freed by
this function, and `f`, the possibly-translated, possibly-hooked intermediate.
The invariant is enforced once, at the end (`main/channel.c:5492-5493`):

```c
	if (f && f != fr)
		ast_frfree(f);
```

### Step 1 — Deadlock avoidance

`main/channel.c:5180-5188`:

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

**It returns 0 — success — after giving up.** Ten trylocks with a 1 µs sleep
between them, then the frame is silently dropped and the caller is told everything
is fine. If you are debugging mysteriously missing audio under load, this is a
place to look; `core set debug 1` will show the message.

### Step 2 — Stream resolution

`main/channel.c:5191-5213`. An explicit `stream_num` is honoured unconditionally
(`:5191-5197`):

> If we were told to write to an explicit stream then allow this frame through, no
> matter if the type is expected or not (a framehook could change)

Otherwise VOICE/VIDEO/MODEM get the default stream for their media type, with
MODEM mapped to `AST_MEDIA_TYPE_IMAGE` (`main/channel.c:5207-5209`).

### Step 3 — Framehook WRITE event

`main/channel.c:5215-5220`:

```c
	if ((stream == default_stream) && !(fr = ast_framehook_list_write_event(ast_channel_framehooks(chan), fr))) {
		res = 0;
		goto done;
	}
```

**Note `fr` itself is reassigned.** From here on, "the caller's frame" means
whatever the framehook list returned. A framehook can therefore replace or
suppress the caller's frame entirely, and a suppressed frame yields `res = 0` —
success.

### Step 4 — Generator interception

`main/channel.c:5222-5246`. If a generator is running, it owns the channel's
output and almost everything is discarded:

```c
	if (ast_channel_generatordata(chan) && (fr->frametype != AST_FRAME_RTCP) && (!fr->src || strcasecmp(fr->src, "ast_prod"))) {
		if (ast_test_flag(ast_channel_flags(chan), AST_FLAG_WRITE_INT)) {
				ast_deactivate_generator(chan);
		} else {
			if (fr->frametype == AST_FRAME_DTMF_END) {
				/* There is a generator running while we're in the middle of a digit.
				 * It's probably inband DTMF, so go ahead and pass it so it can
				 * stop the generator */
				ast_channel_unlock(chan);
				res = ast_senddigit_end(chan, fr->subclass.integer, fr->len);
				ast_channel_lock(chan);
			} else if (fr->frametype == AST_FRAME_CONTROL
				&& fr->subclass.integer == AST_CONTROL_UNHOLD) {
				...
			}
			res = 0;	/* XXX explain, why 0 ? */
			goto done;
		}
	}
```

Three escapes: `AST_FLAG_WRITE_INT` deactivates the generator; `src == "ast_prod"`
bypasses the check entirely (that is what `ast_prod()` is for); RTCP always gets
through. Otherwise only DTMF_END and CONTROL/UNHOLD are honoured.

The `/* XXX explain, why 0 ? */` is in the source. The answer is that returning -1
would be interpreted as a write failure and force a soft hangup, which would be
wrong — but nobody has written that down.

### Step 5 — Debug dump

`main/channel.c:5251-5253`:

```c
	/* High bit prints debugging */
	if (ast_channel_fout(chan) & DEBUGCHAN_FLAG)
		ast_frame_dump(ast_channel_name(chan), fr, ">>");
```

### Step 6 — The frametype switch

`main/channel.c:5250-5486`.

| Frametype | Handling | Line |
|---|---|---|
| `CONTROL` | `indicate_data_internal()`, `res = 0` | `:5251-5254` |
| `DTMF_BEGIN` | audiohooks, then `ast_senddigit_begin()` unlocked | `:5255-5266` |
| `DTMF_END` | audiohooks, then `ast_senddigit_end()` unlocked | `:5267-5280` |
| `TEXT` | `write_text` if T.140, else `send_text` | `:5281-5291` |
| `HTML` | `tech->send_html` | `:5292-5297` |
| `VIDEO` | `write_stream`, else `write_video` | `:5298-5313` |
| `MODEM` | `tech_write()` | `:5314-5318` |
| `VOICE` | the long path below | `:5319-5462` |
| `NULL`, `IAX` | ignored, `res = 0` | `:5463-5467` |
| `RTCP` | `tech->write_stream` only, else `res = 0` | `:5468-5477` |
| *default* | writes **the original `fr`** | `:5478-5485` |

Control frames do **not** go to `tech->write`; they go through
`indicate_data_internal()`, i.e. the same path as `ast_indicate_data()`. See
[04-control-frames.md](04-control-frames.md).

The `default:` case is the one with teeth (`main/channel.c:5482-5487`):

```c
	default:
		/* At this point, fr is the incoming frame and f is NULL.  Channels do
		 * not expect to get NULL as a frame pointer and will segfault.  Hence,
		 * we output the original frame passed in. */
		CHECK_BLOCKING(chan);
		res = ast_channel_tech(chan)->write(chan, fr);
```

Note it calls `tech->write` directly rather than `tech_write()`, so a
stream-only driver with no `write` callback will crash here. Any new frametype
that can reach `ast_write()` needs an explicit case.

### The voice write path in order

`main/channel.c:5319-5462`:

1. **Generic PLC** (`:5320-5322`) — `apply_plc(chan, fr)` when
   `ast_opt_generic_plc` and the frame is exactly `slin`. **This mutates the
   caller's frame in place.**
2. `f = fr;` (`:5324`).
3. **Audiohooks if already slinear** (`:5326-5335`), setting `hooked = 1`.
4. **Format adaptation and translation** (`:5337-5376`). If the frame is not in
   `rawwriteformat`, call `ast_set_write_format()` — and on failure, `break`,
   which induces a hangup (the comment at `:5342-5352` spells out why: "The best
   we can do is request the call to hangup since we could not make it
   compatible"). Then translate with **`consume = 0`**:
   ```c
   			if (ast_channel_writetrans(chan)) {
   				struct ast_frame *trans_frame = ast_translate(ast_channel_writetrans(chan), f, 0);
   				if (trans_frame != f && f != fr) {
   					/*
   					 * If translate gives us a new frame and so did the audio
   					 * hook then we need to free the one from the audio hook.
   					 */
   					ast_frfree(f);
   				}
   				f = trans_frame;
   			}
   ```
   **The asymmetry with the read path is deliberate.** Read passes `consume = 1`
   because the frame came from the driver and the core owns it. Write passes
   `consume = 0` because the frame belongs to the caller.
5. **Audiohooks, second chance** (`:5383-5426`) — the most intricate block in the
   function, because it has to rebuild a *list* of frames while hooks replace
   individual members. The `ast_frisolate()` at `:5404` has its own
   justification:
   > doing an ast_frisolate here seems silly, but we are not guaranteed the
   > new_frame isn't part of local storage, meaning if ast_audiohook_write is
   > called multiple times it may override the previous frame we got from it
   > unless we dup it

   And `freeoldlist` is set only when `f != fr` (`:5387-5389`) — i.e. only free
   frames we created, never the caller's.
6. **Write, possibly multiple frames** (`:5428-5461`):
   ```c
   		CHECK_BLOCKING(chan);
   		if ((f != fr) && AST_LIST_NEXT(f, frame_list)) {
   			...
   			while (cur) {
   				next = AST_LIST_NEXT(cur, frame_list);
   				AST_LIST_NEXT(cur, frame_list) = NULL;
   				if (!skip) {
   					res = tech_write(chan, stream, default_stream, cur);
   					if (res < 0) {
   						ast_channel_softhangup_internal_flag_add(chan, AST_SOFTHANGUP_DEV);
   						skip = 1;
   					} else if (next) {
   						ast_channel_fout_set(chan, FRAMECOUNT_INC(ast_channel_fout(chan)));
   					}
   				}
   				ast_frfree(cur);
   				cur = next;
   			}

   			/* reset f so the code below doesn't attempt to free it */
   			f = NULL;
   		} else {
   			res = tech_write(chan, stream, default_stream, f);
   		}
   		ast_clear_flag(ast_channel_flags(chan), AST_FLAG_BLOCKING);
   ```
   The multi-frame branch frees as it goes and then sets `f = NULL` so the
   epilogue does not double-free. Note the guard is `(f != fr) && ...` — a
   *caller-supplied* frame list is never iterated, because the core does not own
   it.

### `tech_write()`

Defined just above `ast_write()`:

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

If the driver is multistream-capable, `write` is never called — not even as a
fallback. And **both "no stream" cases return 0**, so the frame is silently
dropped and reported as success.

### Step 7 — Epilogue

`main/channel.c:5492-5508`:

```c
	if (f && f != fr)
		ast_frfree(f);

	/* Consider a write failure to force a soft hangup */
	if (res < 0) {
		ast_channel_softhangup_internal_flag_add(chan, AST_SOFTHANGUP_DEV);
	} else {
		ast_channel_fout_set(chan, FRAMECOUNT_INC(ast_channel_fout(chan)));
	}
done:
	if (ast_channel_audiohooks(chan) && ast_audiohook_write_list_empty(ast_channel_audiohooks(chan))) {
		/* The list gets recreated if audiohooks are added again later */
		ast_audiohook_detach_list(ast_channel_audiohooks(chan));
		ast_channel_audiohooks_set(chan, NULL);
	}
	ast_channel_unlock(chan);
	return res;
```

Note that `ast_write()` returning -1 has already flagged a soft hangup by the time
you see it.

### `CHECK_BLOCKING` and `AST_FLAG_BLOCKING`

`CHECK_BLOCKING(chan)` sets `AST_FLAG_BLOCKING` and records
`ast_channel_blocker(chan)` as the current thread, so `__ast_queue_frame()` knows
whom to `pthread_kill(SIGURG)`. Every path that calls into the driver brackets the
call with `CHECK_BLOCKING` / `ast_clear_flag(..., AST_FLAG_BLOCKING)`. If you add
a path that calls `tech->write` without that bracket, queue wakeups will be lost
on channels with no alert pipe.

---

## Masquerade and the readq

A masquerade swaps two channels' guts, including their alert pipes and readqs. The
readq swap has to preserve order *and* alert-pipe parity
(`main/channel.c:7028-7055`):

```c
	/* Swap the alertpipes */
	ast_channel_internal_alertpipe_swap(original, clonechan);

	/*
	 * Swap the readq's.  The end result should be this:
	 *
	 *  1) All frames should be on the new (original) channel.
	 *  2) Any frames that were already on the new channel before this
	 *     masquerade need to be at the end of the readq, after all of the
	 *     frames on the old (clone) channel.
	 *  3) The alertpipe needs to get poked for every frame that was already
	 *     on the new channel, since we are now using the alert pipe from the
	 *     old (clone) channel.
	 */
	{
		AST_LIST_HEAD_NOLOCK(, ast_frame) tmp_readq;

		AST_LIST_HEAD_INIT_NOLOCK(&tmp_readq);
		AST_LIST_APPEND_LIST(&tmp_readq, ast_channel_readq(original), frame_list);
		AST_LIST_APPEND_LIST(ast_channel_readq(original), ast_channel_readq(clonechan), frame_list);

		while ((current = AST_LIST_REMOVE_HEAD(&tmp_readq, frame_list))) {
			AST_LIST_INSERT_TAIL(ast_channel_readq(original), current, frame_list);
			if (ast_channel_alert_write(original)) {
				ast_log(LOG_WARNING, "write() failed: %s\n", strerror(errno));
			}
		}
	}
```

Point 3 is the one that catches people: because the alert pipes were swapped
first, the bytes corresponding to the already-queued frames are now in the *wrong*
pipe, so each must be re-poked. The one-byte-per-frame invariant is maintained by
hand here.

---

Next: [06-translation-and-timing.md](06-translation-and-timing.md).
