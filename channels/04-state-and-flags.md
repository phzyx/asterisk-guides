# 4. State and flags

**Target tree:** Asterisk `master`, git `364f14214c` (2026-11-02)

A channel carries three independent words of mutable condition, and they are
easy to confuse because all three are integers, all three are read through
accessors with similar names, and all three show up in `core show channel`
output side by side. They are not variations on one idea. They answer three
different questions:

```
   ┌──────────────────────────────────────────────────────────────────────────┐
   │ chan->state        enum ast_channel_state       "where is this leg in   │
   │                    10 enumerated values          the call?"             │
   │                    + one bit at (1 << 16)                               │
   │  accessor: ast_channel_state() / ast_setstate()                         │
   ├──────────────────────────────────────────────────────────────────────────┤
   │ chan->softhangup   7 OR-able AST_SOFTHANGUP_*   "has someone asked for  │
   │                    bits                          this leg to end, and   │
   │                                                  for what reason?"      │
   │  accessor: ast_channel_softhangup_internal_flag*()                      │
   ├──────────────────────────────────────────────────────────────────────────┤
   │ chan->flags        22 live AST_FLAG_* bits      "what is true about     │
   │                    in a struct ast_flags         this leg right now?"   │
   │  accessor: ast_channel_flags() → struct ast_flags *                     │
   └──────────────────────────────────────────────────────────────────────────┘
```

The state is a value, the other two are bitfields. The state has a setter with
side effects; the softhangup word has three setters with none; the flag word has
no whole-word setter at all, because the accessor hands out a pointer to the
struct and you mutate it in place. Getting the three straight is most of what
this chapter is for.

## 4.1 The state field

The enum lives in its own header, not in `channel.h`:

```c
/*!
 * \brief ast_channel states
 *
 * \note Bits 0-15 of state are reserved for the state (up/down) of the line
 *       Bits 16-32 of state are reserved for flags
 */
enum ast_channel_state {
	AST_STATE_DOWN,			/*!< Channel is down and available */
	AST_STATE_RESERVED,		/*!< Channel is down, but reserved */
	AST_STATE_OFFHOOK,		/*!< Channel is off hook */
	AST_STATE_DIALING,		/*!< Digits (or equivalent) have been dialed */
	AST_STATE_RING,			/*!< Remote end is ringing (e.g. listening to audible ringback tone). Also often used for initial state for a new channel. */
	AST_STATE_RINGING,		/*!< Line is ringing */
	AST_STATE_UP,			/*!< Line is up */
	AST_STATE_BUSY,			/*!< Line is busy */
	AST_STATE_DIALING_OFFHOOK,	/*!< Digits (or equivalent) have been dialed while offhook */
	AST_STATE_PRERING,		/*!< Channel has detected an incoming call and is waiting for ring */

	AST_STATE_MUTE = (1 << 16),	/*!< Do not transmit voice data */
};
```

Ten line states plus one bit. The doc comment's claim that "bits 16-32 of state
are reserved for flags" describes an intent that never materialised:
`AST_STATE_MUTE` is the only such bit, nothing in the tree ever sets it, and the
only two places that mention it are `main/channel.c:662` and
`res/snmp/agent.c:396`. In practice, treat the state as a plain enumerated value.

| Value | `ast_state2str` | Meaning |
|---|---|---|
| `AST_STATE_DOWN` | `Down` | down and available; the state a freshly allocated channel is usually given |
| `AST_STATE_RESERVED` | `Rsrvd` | down but reserved — analog hardware, mostly |
| `AST_STATE_OFFHOOK` | `OffHook` | off hook, no digits yet |
| `AST_STATE_DIALING` | `Dialing` | digits have been dialled |
| `AST_STATE_RING` | `Ring` | remote end is ringing; also a common initial state for a new inbound channel |
| `AST_STATE_RINGING` | `Ringing` | *this* line is ringing |
| `AST_STATE_UP` | `Up` | answered; media flows |
| `AST_STATE_BUSY` | `Busy` | busy |
| `AST_STATE_DIALING_OFFHOOK` | `Dialing Offhook` | digits dialled while off hook |
| `AST_STATE_PRERING` | `Pre-ring` | incoming call detected, ring not yet signalled |
| `AST_STATE_MUTE` | `Mute` | `(1 << 16)`; never set anywhere in the tree |

`RING` and `RINGING` are the pair that catches people. `RING` means the far end
is ringing and you are hearing ringback; `RINGING` means your own line is
ringing. The header's own comment on `AST_STATE_RING` admits the ambiguity by
noting it is "also often used for initial state for a new channel."

`ast_state2str` (`main/channel.c:637-670`) is a plain switch, and its tail is the
interesting part:

```c
	case AST_STATE_PRERING:
		return "Pre-ring";
	case AST_STATE_MUTE:
		return "Mute";
	default:
		if (!(buf = ast_threadstorage_get(&state2str_threadbuf, STATE2STR_BUFSIZE)))
			return "Unknown";
		snprintf(buf, STATE2STR_BUFSIZE, "Unknown (%u)", state);
		return buf;
	}
```

`AST_STATE_MUTE` is handled as a *case*, not as a bit to mask off. So a
hypothetical state of `AST_STATE_UP | AST_STATE_MUTE` would not print as "Up"
with a mute marker — it would fall to `default` and print `Unknown (65542)`.
This is the clearest evidence that the "bits 16-32 are flags" design was
abandoned rather than implemented.

### `ast_setstate` and its two side effects

Setting the state is not an assignment:

```c
int ast_setstate(struct ast_channel *chan, enum ast_channel_state state)
{
	int oldstate = ast_channel_state(chan);
	char name[AST_CHANNEL_NAME], *dashptr;

	if (oldstate == state)
		return 0;

	ast_copy_string(name, ast_channel_name(chan), sizeof(name));
	if ((dashptr = strrchr(name, '-'))) {
		*dashptr = '\0';
	}

	ast_channel_state_set(chan, state);

	ast_channel_publish_snapshot(chan);

	/* We have to pass AST_DEVICE_UNKNOWN here because it is entirely possible that the channel driver
	 * for this channel is using the callback method for device state. If we pass in an actual state here
	 * we override what they are saying the state is and things go amuck. */
	ast_devstate_changed_literal(AST_DEVICE_UNKNOWN, (ast_test_flag(ast_channel_flags(chan), AST_FLAG_DISABLE_DEVSTATE_CACHE) ? AST_DEVSTATE_NOT_CACHABLE : AST_DEVSTATE_CACHABLE), name);

	return 0;
}
```

Three things to take from this. First, it is a no-op when the state is unchanged
(`:7427`), so the Stasis traffic below it is edge-triggered rather than
level-triggered. Second, it publishes a full channel snapshot synchronously
(`:7437`) — so a state change is a message-bus event, and `ast_setstate` is not
cheap. Third, it fires a device state change (`:7442`) against the channel name
with the `-<uniqueid>` suffix stripped (`:7430-7433`), which is how hints and
`DEVICE_STATE()` learn that an endpoint went busy. `AST_FLAG_DISABLE_DEVSTATE_CACHE`
is consulted right there, in the only place it really matters.

The declaration carries a lock precondition (`channelstate.h:50-54`):

```c
/*!
 * \brief Change the state of a channel
 * \pre chan is locked
 */
int ast_setstate(struct ast_channel *chan, enum ast_channel_state);
```

### The raw setter, and who is allowed to use it

`ast_channel_state_set` is the bare accessor, and it does exactly nothing else:

```c
void ast_channel_state_set(struct ast_channel *chan, enum ast_channel_state value)
{
	chan->state = value;
}
```

Note that unlike many setters in that file (chapter 01), it does *not* invalidate
a snapshot segment — because the state is not in a segment at all (§4.4). The
only way a state change reaches the bus is through `ast_setstate` publishing a
whole snapshot.

Outside the test suite, `ast_channel_state_set` has four call sites in three
places, all in `main/channel.c`: `:858` in `__ast_channel_alloc_ap`, where the
channel does not exist yet and there is nobody to publish to; `:7075` and
`:7076`, where masquerade swaps the two channels' states; and `:7435`, inside
`ast_setstate` itself. Everything else in the tree — all 90 call sites in `.c`
files — goes through `ast_setstate`. Of those, 62 are in `channels/` and 6 in
`addons/`: setting the state is overwhelmingly a channel driver's job, which
makes sense, since the driver is the only thing that knows what the signalling
protocol is doing.

The state values drivers actually set are lopsided too: counting the
`AST_STATE_*` constants appearing in `ast_setstate(` calls across the tree gives
45 `UP`, 16 `RINGING`, 14 `RING`, 7 `DIALING`, 2 each `DIALING_OFFHOOK` and
`BUSY`, and one each `OFFHOOK` and `DOWN`. `RESERVED`, `PRERING` and `MUTE` are
never an argument to `ast_setstate` anywhere.

### Nothing validates the transition

There is no state machine. `ast_setstate` will take a channel from `UP` back to
`DOWN`, or from `DOWN` straight to `BUSY`, without complaint. The closest thing
to an enforced transition is in the answer path, which refuses to answer from the
wrong state:

```c
	switch (ast_channel_state(chan)) {
	case AST_STATE_RINGING:
	case AST_STATE_RING:
		ast_channel_lock(chan);
		if (ast_channel_tech(chan)->answer_with_stream_topology) {
			res = ast_channel_tech(chan)->answer_with_stream_topology(chan, topology);

		} else if (ast_channel_tech(chan)->answer) {
			res = ast_channel_tech(chan)->answer(chan);
		}
		ast_setstate(chan, AST_STATE_UP);
		ast_channel_unlock(chan);
		break;
	case AST_STATE_UP:
		/* Fall through */
	default:
		ast_debug(2, "Skipping answer, since channel state on %s is %s\n", ast_channel_name(chan), ast_state2str(ast_channel_state(chan)));
		break;
	}
```

Only `RINGING` and `RING` lead to the driver's `answer` callback and the move to
`UP`; everything else logs at debug level 2 and falls through. Note also the two
guards above this switch in `ast_raw_answer_with_stream_topology`
(`main/channel.c:2646`, `:2652`): an outgoing channel cannot be answered at all,
and a zombie or already-hung-up channel returns `-1`. Those are flag checks, not
state checks, which is the first hint that state and flags are consulted
together everywhere.

## 4.2 The softhangup word

`chan->softhangup` is the "someone has asked for this channel to end" register.
Its bits are declared at `include/asterisk/channel.h:1136-1182`, and the doc
comments are worth reading closely because they divide the bits into two
categories that behave completely differently:

```c
	/*!
	 * Used to break the normal frame flow so an async goto can be
	 * done instead of actually hanging up.
	 */
	AST_SOFTHANGUP_ASYNCGOTO = (1 << 1),
	/*!
	 * Soft hangup requested by system shutdown.  Actual hangup
	 * needed.
	 */
	AST_SOFTHANGUP_SHUTDOWN =  (1 << 2),
	/*!
	 * Used to break the normal frame flow after a timeout so an
	 * implicit async goto can be done to the 'T' exten if it exists
	 * instead of actually hanging up.  If the exten does not exist
	 * then actually hangup.
	 */
	AST_SOFTHANGUP_TIMEOUT =   (1 << 3),
```

| Bit | Flag | Category | Meaning |
|---|---|---|---|
| 0 | `AST_SOFTHANGUP_DEV` | real hangup | the device or an internal reason wants out |
| 1 | `AST_SOFTHANGUP_ASYNCGOTO` | break the flow | *not* a hangup; break the read loop so an async goto can run |
| 2 | `AST_SOFTHANGUP_SHUTDOWN` | real hangup | system shutdown |
| 3 | `AST_SOFTHANGUP_TIMEOUT` | break the flow, then maybe hang up | absolute timeout reached; jump to the `T` extension if it exists, otherwise really hang up |
| 4 | `AST_SOFTHANGUP_APPUNLOAD` | real hangup | the application or driver holding the channel is being unloaded |
| 5 | `AST_SOFTHANGUP_EXPLICIT` | real hangup | a non-associated party asked (AMI, ARI, CLI) |
| 6 | — | — | vacant; was `AST_SOFTHANGUP_UNBRIDGE`, removed by `d4695774e7` |
| 7 | `AST_SOFTHANGUP_HANGUP_EXEC` | status record | the channel is already hung up and is running dialplan hangup logic |

The two categories are the whole point. `DEV`, `SHUTDOWN`, `APPUNLOAD` and
`EXPLICIT` mean "actually tear this down". `ASYNCGOTO` and `TIMEOUT` are
borrowing the hangup machinery purely to interrupt whatever the channel's owner
is blocked in, and the PBX clears them again rather than hanging up:
`main/pbx.c:2849-2850`, `:2946-2947` and `:3013-3014` all test for `ASYNCGOTO`
and immediately call `ast_channel_clear_softhangup`, and `:2950-2963`,
`:3017-3030` and `:3071-3073` do the same for `TIMEOUT` after deciding whether a
`T` extension exists. `HANGUP_EXEC` is neither: it is set by
`main/pbx.c:2777` and `main/pbx_hangup_handler.c:74` as a record that the hangup
already happened, and is read almost entirely out of snapshots by `main/cdr.c`.

And `AST_SOFTHANGUP_ALL` is not a bitmask of the above — it is `0xFFFFFFFF`
(`channel.h:1181`), intended only as an argument to
`ast_channel_clear_softhangup`.

### `ast_check_hangup` has a side effect

This is the function everything in the tree uses to ask "should I stop?":

```c
int ast_check_hangup(struct ast_channel *chan)
{
	if (ast_channel_softhangup_internal_flag(chan))		/* yes if soft hangup flag set */
		return 1;
	if (ast_tvzero(*ast_channel_whentohangup(chan)))	/* no if no hangup scheduled */
		return 0;
	if (ast_tvdiff_ms(*ast_channel_whentohangup(chan), ast_tvnow()) > 0)		/* no if hangup time has not come yet. */
		return 0;
	ast_debug(4, "Hangup time has come: %" PRIi64 "\n", ast_tvdiff_ms(*ast_channel_whentohangup(chan), ast_tvnow()));
	ast_test_suite_event_notify("HANGUP_TIME", "Channel: %s", ast_channel_name(chan));
	ast_channel_softhangup_internal_flag_add(chan, AST_SOFTHANGUP_TIMEOUT);	/* record event */
	return 1;
}
```

It is not a pure predicate. If the channel has an absolute hangup time and that
time has passed, `ast_check_hangup` *sets* `AST_SOFTHANGUP_TIMEOUT` on the way
out (`:456`). So the first caller to notice the timeout converts it into a sticky
softhangup bit, and every subsequent call short-circuits at `:448`. There is also
a `ast_check_hangup_locked` (`:460-467`) that just wraps it in the channel lock —
the unlocked form is the one that assumes you already hold it.

### Requesting a softhangup

```c
int ast_softhangup_nolock(struct ast_channel *chan, int cause)
{
	ast_debug(1, "Soft-Hanging (%#04x) up channel '%s'\n", (unsigned)cause, ast_channel_name(chan));
	/* Inform channel driver that we need to be hung up, if it cares */
	ast_channel_softhangup_internal_flag_add(chan, cause);
	ast_queue_frame(chan, &ast_null_frame);
	/* Interrupt any poll call or such */
	if (ast_test_flag(ast_channel_flags(chan), AST_FLAG_BLOCKING))
		pthread_kill(ast_channel_blocker(chan), SIGURG);
	return 0;
}
```

Three steps, and all three matter. The flag is OR'd in; a null frame is queued so
anything in `ast_read` wakes up and re-runs its hangup check; and if a thread is
blocked in `poll` on this channel, it gets a `SIGURG` to break out of the syscall.
That last line is one of only three consumers of `ast_channel_blocker` in the
tree — `main/channel.c:1163` in `__ast_queue_frame`, `:2457` here, and `:7336`
in `ast_do_masquerade` — and all three are the same `AST_FLAG_BLOCKING`-guarded
`SIGURG`. That flag (§4.3) exists for no other purpose.

The locking wrapper `ast_softhangup` (`:2462-2484`) adds one thing the `nolock`
form does not: it publishes an `ast_channel_hangup_request_type()` blob to
Stasis, carrying the cause, a `soft: 1` marker and the technology-specific cause
if the driver set one. So AMI's `HangupRequest` and ARI's
`ChannelHangupRequest` come from here, and a caller that uses the `nolock` form
directly produces no such event.

### Clearing it, and the `END_OF_Q` coupling

Clearing is not the inverse of setting, because the queued frames have to be
unwound too:

```c
void ast_channel_clear_softhangup(struct ast_channel *chan, int flag)
{
	ast_channel_lock(chan);

	ast_channel_softhangup_internal_flag_clear(chan, flag);

	if (!ast_channel_softhangup_internal_flag(chan)) {
		struct ast_frame *fr;

		/* If we have completely cleared the softhangup flag,
		 * then we need to fully abort the hangup process.  This requires
		 * pulling the END_OF_Q frame out of the channel frame queue if it
		 * still happens to be there. */

		fr = AST_LIST_LAST(ast_channel_readq(chan));
		if (fr && fr->frametype == AST_FRAME_CONTROL &&
				fr->subclass.integer == AST_CONTROL_END_OF_Q) {
			AST_LIST_REMOVE(ast_channel_readq(chan), fr, frame_list);
			ast_frfree(fr);
		}
	}

	ast_channel_unlock(chan);
}
```

The header is blunt about this (`channel.h:1671-1680`): "Never clear a softhangup
flag from a channel directly. Instead, use this function." Direct use of
`ast_channel_softhangup_internal_flag_clear` leaves an `AST_CONTROL_END_OF_Q`
frame sitting at the tail of the read queue, and the next `ast_read` will treat
it as the end of the channel's life even though the softhangup was withdrawn.

The frame gets there from the top of `__ast_read`:

```c
	ast_channel_lock(chan);

	/* Stop if we're a zombie or need a soft hangup */
	if (ast_test_flag(ast_channel_flags(chan), AST_FLAG_ZOMBIE) || ast_check_hangup(chan)) {
		if (ast_channel_generator(chan))
			ast_deactivate_generator(chan);

		/*
		 * It is possible for chan->_softhangup to be set and there
		 * still be control frames that need to be read.  Instead of
		 * just going to 'done' in the case of ast_check_hangup(), we
		 * need to queue the end-of-Q frame so that it can mark the end
		 * of the read queue.  If there are frames to be read,
		 * ast_queue_control() will be called repeatedly, but will only
		 * queue the first end-of-Q frame.
		 */
		if (ast_channel_softhangup_internal_flag(chan)) {
			ast_queue_control(chan, AST_CONTROL_END_OF_Q);
		} else {
			goto done;
		}
```

Read that branch carefully, because it is the single most load-bearing use of
these three words in the tree. A zombie or a hung-up channel stops the read — but
if the reason is a softhangup bit rather than zombiehood, the function does not
just bail out; it queues `END_OF_Q` so the frames already in the queue can still
be drained in order, with a definite terminator behind them. Control frames
queued before the hangup are not lost. Chapter 05 picks up the rest of this
function.

### The accessors are labelled "evil"

```c
/* Evil softhangup accessors */
int ast_channel_softhangup_internal_flag(struct ast_channel *chan)
{
	return chan->softhangup;
}
void ast_channel_softhangup_internal_flag_set(struct ast_channel *chan, int value)
{
	chan->softhangup = value;
}
void ast_channel_softhangup_internal_flag_add(struct ast_channel *chan, int value)
{
	chan->softhangup |= value;
}
void ast_channel_softhangup_internal_flag_clear(struct ast_channel *chan, int value)
{
	chan ->softhangup &= ~value;
}
```

That comment is the tree's own, not mine. The four functions are a raw read, a
raw overwrite, an OR and an AND-NOT, with no locking, no frame queueing and no
event publication — all of which the wrappers above provide. `_set` in particular
will silently destroy bits another subsystem is relying on; use `_add` and the
public `ast_channel_clear_softhangup` instead.

## 4.3 The flag word

The flags live in a `struct ast_flags`, which is a struct wrapping exactly one
integer (`include/asterisk/utils.h:220-222`):

```c
struct ast_flags {
	unsigned int flags;
};
```

and the accessor hands out a pointer to it (`channel.h:4526`,
`channel_internal_api.c:1165-1168`), which is why there is no whole-word setter
along the lines of `ast_channel_flags_set`. Most code in the tree mutates the
channel's flags in place through the generic `ast_flags` macros: across `.c`
files there are 87 `ast_test_flag(ast_channel_flags(...)`, 60
`ast_set_flag(...)`, 36 `ast_clear_flag(...)` and 7 `ast_set2_flag(...)`.

There are, however, two single-bit setters that handle the locking for you, and
they are easy to miss because they are 300 lines from everything else in
`channel.c` (`:11207-11219`):

```c
void ast_channel_set_flag(struct ast_channel *chan, unsigned int flag)
{
	ast_channel_lock(chan);
	ast_set_flag(ast_channel_flags(chan), flag);
	ast_channel_unlock(chan);
}

void ast_channel_clear_flag(struct ast_channel *chan, unsigned int flag)
{
	ast_channel_lock(chan);
	ast_clear_flag(ast_channel_flags(chan), flag);
	ast_channel_unlock(chan);
}
```

They were added in 13.17.0 and the header is explicit that calling them with the
lock already held is safe (`channel.h:4535-4536`, `:4548-4549`) — which it is,
because the channel lock is recursive (chapter 03). There are 41 call sites
across `main/` and `apps/`. Prefer these to a bare `ast_set_flag` on
`ast_channel_flags(chan)`: they are the only flag mutation in the subsystem that
cannot be wrong about locking.

### The catalog

`channel.h:998-1079` declares 22 live flags. The "set by" column is where the
flag is *written*, which is usually a much smaller set of places than where it is
read.

| Bit | Flag | Uses | What it means / who sets it |
|---|---|---|---|
| 1 | `AST_FLAG_DEFER_DTMF` | 8 | queue incoming DTMF until cleared |
| 2 | `AST_FLAG_WRITE_INT` | 8 | a write should interrupt the generator |
| 3 | `AST_FLAG_BLOCKING` | 18 | a thread is blocked on this channel; set and cleared by `CHECK_BLOCKING` / `ast_clear_flag` around the poll, and read by `ast_softhangup_nolock` to decide whether to signal |
| 4 | `AST_FLAG_ZOMBIE` | 27 | the channel is a masquerade leftover or is being hung up; see below |
| 5 | `AST_FLAG_EXCEPTION` | 8 | an exception is pending on one of the channel's fds |
| 6 | `AST_FLAG_MOH` | 10 | music on hold is playing. The header's own comment: "XXX anthm promises me this will disappear XXX" |
| 7 | `AST_FLAG_SPYING` | 8 | an audiohook spy is attached |
| 9 | `AST_FLAG_IN_AUTOLOOP` | 21 | the channel is inside the PBX's auto-incrementing priority loop; `pbx.c` uses it to decide whether setting a priority means "go there" or "go there next" |
| 10 | `AST_FLAG_OUTGOING` | 27 | this is the outbound leg. Checked first thing in the answer path |
| 12 | `AST_FLAG_IN_DTMF` | 7 | a `DTMF_BEGIN` was read but not yet its `DTMF_END` |
| 13 | `AST_FLAG_EMULATE_DTMF` | 15 | a `DTMF_END` arrived without a `BEGIN`, so the digit length is being faked |
| 14 | `AST_FLAG_END_DTMF_ONLY` | 40 | suppress `DTMF_BEGIN` generation entirely; the most-used flag in the catalog |
| 16 | `AST_FLAG_MASQ_NOSTREAM` | 3 | do not carry an active stream across a masquerade |
| 17 | `AST_FLAG_BRIDGE_HANGUP_RUN` | 4 | the hangup extension already ran at bridge teardown; stops `pbx_run` running it again |
| 20 | `AST_FLAG_DISABLE_WORKAROUNDS` | 5 | reintroduce certain bugs so AGI-style dialplans keep working |
| 21 | `AST_FLAG_DISABLE_DEVSTATE_CACHE` | 17 | device state events from this channel are not cachable; read by `ast_setstate` |
| 22 | `AST_FLAG_BRIDGE_DUAL_REDIRECT_WAIT` | 6 | a dual redirect is in progress and the bridge must wait |
| 23 | `AST_FLAG_ORIGINATED` | 9 | the channel was originated rather than inbound |
| 24 | `AST_FLAG_DEAD` | 17 | well and truly dead; see below |
| 25 | `AST_FLAG_SNAPSHOT_STAGE` | 7 | snapshot publication is suppressed because a batch is being staged |
| 26 | `AST_FLAG_TIMINGDATA_IS_AO2_OBJ` | 5 | `chan->timingdata` is an ao2 object and needs unreffing, not freeing |
| 27 | `AST_FLAG_SUBROUTINE_EXEC` | 9 | the channel is executing a subroutine or macro |

### The gaps are graves

Bits 0, 8, 11, 15, 18 and 19 are missing from that list, and the reason is a
policy you should assume applies to you: a retired channel flag's bit is never
reused. Bit 15 is still in the header as a commented-out tombstone with an
explanation:

```c
	/* OBSOLETED in favor of AST_CAUSE_ANSWERED_ELSEWHERE
	 * Flag to show channels that this call is hangup due to the fact that the call
	 * was indeed answered, but in another channel */
	/* AST_FLAG_ANSWERED_ELSEWHERE = (1 << 15), */
	/*! This flag indicates that on a masquerade, an active stream should not
	 *  be carried over */
	AST_FLAG_MASQ_NOSTREAM = (1 << 16),
```

The other five were deleted outright, but `git log -S` on `channel.h` recovers
them:

| Bit | Was | Removed by |
|---|---|---|
| 0 | `AST_FLAG_DIGITAL` (a `#define`, not an enumerator) | retired by `b81f233e68` (2005), which commented it out; the dead line was deleted by `3314ea0d59`. Still named in a comment on `struct ast_channel::transfercapability` |
| 8 | `AST_FLAG_NBRIDGE` | `680765d452`, "Remove ast_channel_bridge() and associated code called only by it." |
| 11 | `AST_FLAG_WHISPER` | `2b33aca04c`, "Remove old whisper remnants from channel.h" |
| 15 | `AST_FLAG_ANSWERED_ELSEWHERE` | superseded by `AST_CAUSE_ANSWERED_ELSEWHERE`; the comment survives |
| 18 | `AST_FLAG_FASTAGI`, then `AST_FLAG_BRIDGE_HANGUP_DONT` | `4ff527903e` removed `FASTAGI`; `3d63833bd6`, the bridging API merge, removed `BRIDGE_HANGUP_DONT` |
| 19 | `AST_FLAG_ASYNCAGI` | `4ff527903e`, which took bit 18's occupant with it |

If you add a flag, take bit 28. Do not take bit 0 because it looks free.

### `AST_FLAG_ZOMBIE`: two writers, twenty-three readers

Zombie means "this channel still exists as an object but must not be used". It is
set at exactly two places in the tree. The first is in `ast_hangup`, after the
wait for any in-flight masquerade to finish:

```c
	ast_channel_lock(chan);

	while (ast_channel_masq(chan) || ast_channel_masqr(chan))  {
		CHANNEL_DEADLOCK_AVOIDANCE(chan);
	}

	/* Mark as a zombie so a masquerade cannot be setup on this channel. */
	ast_set_flag(ast_channel_flags(chan), AST_FLAG_ZOMBIE);

	ast_channel_unlock(chan);
```

The second is at the end of `ast_do_masquerade`, on the channel that just donated
its guts to another:

```c
	/*
	 * Now, at this point, the "clone" channel is totally F'd up.
	 * We mark it as a zombie so nothing tries to touch it.
	 *
	 * This must be done before we unlock clonechan to prevent
	 * setting up another masquerade on the clonechan.
	 */
	ast_set_flag(ast_channel_flags(clonechan), AST_FLAG_ZOMBIE);
	ast_queue_frame(clonechan, &ast_null_frame);
```

Both comments say the same thing: the flag's job is to prevent a *further*
masquerade being set up on a channel that is already being dismantled. Chapter 06
covers why that matters. Everything else — 20-odd sites across `main/channel.c`,
`main/bridge.c`, `main/bridge_channel.c`, `main/audiohook.c`, `main/pickup.c`,
`apps/app_chanspy.c`, `apps/app_agent_pool.c` and `apps/app_externalivr.c` — only
reads it, and almost always in the same compound guard:

```c
	if (ast_test_flag(ast_channel_flags(chan), AST_FLAG_ZOMBIE) || ast_check_hangup(chan)) {
```

That exact line appears at `main/channel.c:2652`, `:3554`, `:4782` and `:5191`.
`:3250` and `:6590` are the same test over a channel named `c` and without the
brace, since the `if` governs a single statement; and the negated form
`!... && !ast_check_hangup(...)` appears at `:5895` (over `parent`), `:6488` and
`:6527`. Treat "zombie or hung up" as a single concept when you write new code
against a channel: both halves mean "stop".

### `AST_FLAG_DEAD`: set once, read only from snapshots

`AST_FLAG_DEAD` is set at exactly one place, in the ao2 destructor path, with a
comment that explains the ordering:

```c
	ast_pbx_hangup_handler_destroy(chan);

	/* Things that may possibly raise Stasis messages shouldn't occur after this point */
	ast_set_flag(ast_channel_flags(chan), AST_FLAG_DEAD);

	if (ast_channel_internal_is_finalized(chan)) {
		/* A channel snapshot should not be in the process of being staged now. */
		ast_assert(!ast_test_flag(ast_channel_flags(chan), AST_FLAG_SNAPSHOT_STAGE));

		ast_channel_lock(chan);
		ast_channel_publish_final_snapshot(chan);
		ast_channel_unlock(chan);
	}
```

The flag goes up *before* the final snapshot is published, so the final snapshot
carries it. That is the entire design: of the 17 occurrences of `AST_FLAG_DEAD`,
only this one touches a live channel. Every other read is against a snapshot's
flag copy — `main/manager_channels.c:645-646` and `:687`, `main/cdr.c:1588`,
`:1915`, `:2120`, `:2466`, `:2486`, `main/cel.c:956-957`, `res/stasis/app.c:453`
and `:579`, `res/res_chan_stats.c:100`. `stasis_channels.h:173-176` states the
contract directly: a channel going away is not signalled by a null new snapshot,
it is signalled by `AST_FLAG_DEAD` on the new snapshot.

So `AST_FLAG_DEAD` is not really channel state for your code to test. It is a
tombstone shipped to event consumers. If you want to know whether a live channel
is finished, use `ast_check_hangup` and `AST_FLAG_ZOMBIE`.

### `AST_FLAG_SNAPSHOT_STAGE`: a flag that suppresses publication

```c
void ast_channel_stage_snapshot(struct ast_channel *chan)
{
	ast_set_flag(ast_channel_flags(chan), AST_FLAG_SNAPSHOT_STAGE);
}

void ast_channel_stage_snapshot_done(struct ast_channel *chan)
{
	ast_clear_flag(ast_channel_flags(chan), AST_FLAG_SNAPSHOT_STAGE);
	ast_channel_publish_snapshot(chan);
}
```

Between those two calls, `ast_channel_publish_snapshot` returns early
(`stasis_channels.c:1092-1094`), so a sequence of ten field changes produces one
message instead of ten. Two consequences worth knowing. First, `ast_setstate`
inside a staged region will still assign the state and still fire the device
state change, but its snapshot publication is swallowed — the state change
reaches the bus only when the stage closes. Second, forgetting
`ast_channel_stage_snapshot_done` silences the channel on the bus for the rest of
its life, which is why the destructor asserts the flag is clear
(`main/channel.c:2205`). `main/rtp_engine.c:2819` and `:2826` are the one place
in the tree that deliberately closes a staged region *without* publishing: they
clear the flag directly when the staged work turned out to change nothing
(`if (chanchanges) ... else`, `:2816-2820`, and the same for `bridgechanges` at
`:2823-2827`), so there is no snapshot worth emitting.

## 4.4 All three words in the snapshot

The channel snapshot is segmented and most segments are only rebuilt when a
setter invalidates them (chapter 01). These three words are not in any segment —
they are top-level fields of `struct ast_channel_snapshot`
(`stasis_channels.h:158-161`) — so they are re-read from the live channel on
every single snapshot creation:

```c
	snapshot->state = ast_channel_state(chan);
	snapshot->amaflags = ast_channel_amaflags(chan);
	ast_copy_flags(&snapshot->flags, ast_channel_flags(chan), 0xFFFFFFFF);
	ast_set_flag(&snapshot->softhangup_flags, ast_channel_softhangup_internal_flag(chan));
```

Note the asymmetry on the last two lines. The flag word is copied with
`ast_copy_flags(..., 0xFFFFFFFF)`, which overwrites. The softhangup word is
installed with `ast_set_flag`, which ORs — correct only because the snapshot was
freshly allocated and zeroed. The two lines are not interchangeable, and if you
ever reuse a snapshot object the second one is a bug waiting to happen.

The practical upshot for anyone reading events: `state`, `flags` and
`softhangup_flags` in a snapshot are always current as of the moment the
snapshot was built, even when every segment pointer was bumped from the previous
snapshot unchanged.

## 4.5 Locking

All three words are plain integers in the struct, so none of the three accessor
families does anything atomic. The rules follow from chapter 03.

| Operation | Requirement |
|---|---|
| `ast_setstate` | `\pre chan is locked` (`channelstate.h:52`). It publishes a snapshot and fires a device state change, so this is a lock held across real work |
| `ast_channel_state` read | hold the lock if you intend to act on the value; it is an `int` read, so it will not tear, but it can be stale the instant it returns |
| `ast_check_hangup` | hold the lock — it may write `AST_SOFTHANGUP_TIMEOUT`. `ast_check_hangup_locked` exists for when you do not |
| `ast_softhangup_nolock` | caller holds the lock; it queues a frame and may `pthread_kill` |
| `ast_channel_clear_softhangup` | takes the lock itself (`:2425`) |
| Any `ast_set_flag` / `ast_clear_flag` on `ast_channel_flags(chan)` | hold the channel lock. These are read-modify-write on a shared `unsigned int` with no atomics anywhere |
| `ast_channel_set_flag` / `ast_channel_clear_flag` | nothing — they take the lock themselves, and are safe to call with it already held (`main/channel.c:11207-11219`) |

The flag word is the one people get wrong, because `ast_set_flag` looks like a
single operation and reads like one in the source. It is `flags |= value` on
memory that other threads are also OR-ing into.

## 4.6 Common bugs

| Symptom | Cause |
|---|---|
| A channel never hangs up after an API hangup request | the softhangup bit was set with `_flag_set` rather than `_flag_add`, clearing someone else's bit, or no null frame was queued so the reader never woke |
| `ast_read` returns `NULL` immediately on a channel that should still be live | a softhangup bit was cleared directly, leaving `END_OF_Q` in the read queue |
| Flags intermittently lose a bit under load | `ast_set_flag` on `ast_channel_flags()` without the channel lock |
| No `ChannelStateChange` / AMI `Newstate` event despite the state changing | `ast_channel_state_set` was called instead of `ast_setstate`, or the change happened inside a staged snapshot region |
| A channel goes silent on the message bus for the rest of its life | `ast_channel_stage_snapshot` without a matching `_done` |
| Device state and hints never update for an endpoint | the state was set without `ast_setstate`, so `ast_devstate_changed_literal` never ran |
| `core show channel` reports `Unknown (65542)` | someone OR-ed a bit into the state field; `ast_state2str` has no masking |
| Answer silently does nothing | the channel is `AST_FLAG_OUTGOING`, or its state is not `RING`/`RINGING` |
| Code acts on a channel that is being torn down | tested `ast_check_hangup` but not `AST_FLAG_ZOMBIE`, or tested `AST_FLAG_DEAD` on a live channel, where it is only ever set in the destructor |

## 4.7 Rules

| Rule | Why |
|---|---|
| Change the state only through `ast_setstate`, with the lock held | it is the only path that publishes a snapshot and updates device state |
| Treat the state as a value, never as a bitfield | `AST_STATE_MUTE` is vestigial and `ast_state2str` does not mask |
| Do not assume a state transition is legal or even ordered | nothing validates it; only the answer path checks the source state |
| Add softhangup bits with `_flag_add`, never `_flag_set` | `_set` overwrites every other subsystem's bit |
| Clear softhangup bits only via `ast_channel_clear_softhangup` | it also unwinds the `END_OF_Q` frame |
| Know which softhangup bits mean "really hang up" | `ASYNCGOTO` and `TIMEOUT` are flow-control; the PBX clears them |
| Remember `ast_check_hangup` can write | it sets `AST_SOFTHANGUP_TIMEOUT` when the deadline passes |
| Pair "zombie or hung up" in every guard | they are two independent ways to mean "stop" |
| Do not test `AST_FLAG_DEAD` on a live channel | it is set in the destructor, for snapshot consumers |
| Hold the channel lock for every flag mutation, or use `ast_channel_set_flag`/`_clear_flag` | `ast_set_flag` is a non-atomic read-modify-write; the two wrappers lock for you |
| Take the next unused high bit for a new flag | retired bits are never recycled, and the gaps are deliberate |
| Always pair `ast_channel_stage_snapshot` with `_done` | otherwise the channel stops publishing entirely |

## Next

Chapter 05 (planned; see the [overview](README.md)) covers the fd vector, the
`ast_waitfor` family and timing fds — the machinery that `AST_FLAG_BLOCKING` and
the null-frame-plus-`SIGURG` dance in this chapter exist to interrupt.
