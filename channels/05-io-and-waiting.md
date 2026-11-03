# 05 — I/O and waiting

Media does not arrive on a channel by magic. It arrives because some thread sat
in `poll(2)` on a vector of file descriptors that a channel driver registered,
woke up, and called `ast_read`. Everything in this chapter is about that
handoff.

The handoff is carried by three pieces of channel state, and the whole design
rests on them being written and read in a fixed order:

```
   ┌────────────────────────────────────────────────────────────────────────┐
   │  fds        AST_VECTOR(, int)   what to poll on                        │
   │  fdno       int                 which slot woke us up                  │
   │  flags      AST_FLAG_BLOCKING   "a thread is parked in poll right now" │
   │             AST_FLAG_EXCEPTION  "that slot reported POLLPRI"           │
   └────────────────────────────────────────────────────────────────────────┘

        ast_waitfor_nandfds()                      ast_read()
        ─────────────────────                      ──────────
        reads   fds                                reads   fdno
        writes  fdno                  ────────▶    reads   EXCEPTION
        writes  EXCEPTION                          writes  fdno = -1
        writes  BLOCKING (set, then clear)
        writes  softhangup (TIMEOUT)
```

Three facts are worth holding onto before the detail:

1. **`ast_waitfor` and `ast_read` are two halves of one operation.** `ast_read`
   does not poll. It believes `fdno`, which only `ast_waitfor_nandfds` writes.
   Nothing in the type system enforces the order, and the tree knows it: there
   is an `AST_DEVMODE` debug message for exactly this mistake
   (`main/channel.c:3573-3587`).
2. **`ast_waitfor` is a mutator.** It sets `AST_SOFTHANGUP_TIMEOUT`, sets and
   clears `AST_FLAG_EXCEPTION`, sets `AST_FLAG_BLOCKING` and then clears it,
   and writes `fdno` on *every* ready channel — not only on the one it returns.
3. **The fd vector has a published layout, and part of that layout is wrong.**
   Five slots at the top are reserved by `#define`; two of those are dead, and
   the comment on a third describes the wrong subsystem and has done since 2011.

This chapter assumes [chapter 03](03-references-and-locking.md) for the locking
vocabulary and [chapter 04](04-state-and-flags.md) for `AST_FLAG_BLOCKING` and
the softhangup bits.

## 5.1 The fd vector

The field itself is a growable vector of plain ints
(`main/channel_private.h:142-144`):

```c
	AST_VECTOR(, int) fds;				/*!< File descriptors for channel -- Drivers will poll on
							 *   these file descriptors, so at least one must be non -1.
							 *   See \arg \ref AstFileDesc */
```

The `\arg \ref AstFileDesc` is a dead end. That doxygen page exists, in
`include/asterisk/channel.h:117-120`, and says this and only this:

```c
/*! \page AstFileDesc File descriptors
	Asterisk File descriptors are connected to each channel (see \ref Def_Channel)
	in the \ref ast_channel structure.
*/
```

So the struct comment is the documentation. Two things in it matter. The
drivers poll on these descriptors — the core does the polling on the driver's
behalf, which is why a channel driver has a `read` callback but no event loop.
And at least one entry must be non-`-1`, because a channel with no live
descriptors is a channel `ast_waitfor` cannot wait on; see §5.3.

The vector is initialised with room for `AST_MAX_FDS` entries in
`__ast_channel_internal_alloc`'s helper (`main/channel_internal_api.c:1309`)
and freed in the destructor (`:1446`).

### The reserved slots

The top five slots are claimed by `#define` in the public header
(`include/asterisk/channel.h:196-206`):

```c
#define AST_MAX_FDS		11	/*!< original maximum number of file descriptors */
#define AST_EXTENDED_FDS	12	/*!< the start of extended file descriptor positions */
/*
 * We have AST_MAX_FDS file descriptors in a channel.
 * Some of them have a fixed use:
 */
#define AST_ALERT_FD	(AST_MAX_FDS-1)		/*!< used for alertpipe */
#define AST_TIMING_FD	(AST_MAX_FDS-2)		/*!< used for timingfd */
#define AST_AGENT_FD	(AST_MAX_FDS-3)		/*!< unused - formerly used by agents for pass through */
#define AST_GENERATOR_FD	(AST_MAX_FDS-4)	/*!< unused - formerly used by generator */
#define AST_JITTERBUFFER_FD	(AST_MAX_FDS-5)	/*!< used by generator */
```

`AST_MAX_FDS` is 11, so the original fixed layout was slots 0 through 10, with
6 through 10 reserved and 0 through 5 left to the driver. `AST_EXTENDED_FDS` is
12, which means slot 11 belongs to neither scheme: nothing can ever put a
descriptor there, because the reserved names only reach down to 6 and
`ast_channel_fd_add` starts at 12. Note also that `AST_MAX_FDS` is no longer a
maximum of anything. Outside its own arithmetic in those five `#define`s, the
only remaining use in the tree is the vector's initial capacity at
`main/channel_internal_api.c:1309`.

Here is what those five slots actually do, by grep rather than by comment:

| Slot | Name | Comment says | Reality |
|---|---|---|---|
| 10 | `AST_ALERT_FD` | used for alertpipe | correct; set at `main/channel.c:853` |
| 9 | `AST_TIMING_FD` | used for timingfd | correct; set at `main/channel.c:855`, read at `:3592` |
| 8 | `AST_AGENT_FD` | unused — formerly used by agents | correct; no uses outside the header |
| 7 | `AST_GENERATOR_FD` | unused — formerly used by generator | correct; no uses outside the header |
| 6 | `AST_JITTERBUFFER_FD` | **used by generator** | **wrong** — used by the jitterbuffer framehook |

The last row is a copy-paste that has survived fifteen years.
`AST_JITTERBUFFER_FD` was added by `18d591cb48` (2011-04-20, "Introduction of
the JITTERBUFFER dialplan function"), which bumped `AST_MAX_FDS` from 10 to 11
and appended the new `#define` directly beneath `AST_GENERATOR_FD`, carrying
that line's `/*!< used by generator */` comment along with it. Its real
consumer is `main/abstract_jb.c`, which parks a timer fd there when the
`JITTERBUFFER` function attaches a framehook (`:1340`), clears it on detach
(`:1287`), and checks `fdno` against it inside the framehook callback (`:969`).

The two "unused" annotations are newer and were done deliberately. Sean
Bright's `e200f51ea4` (2025-02-06, "channel.c: Remove dead AST_GENERATOR_FD
code") deleted a block in `ast_read` that could never run — "Nothing ever sets
the `AST_GENERATOR_FD`" — and noted in passing that `AST_AGENT_FD` was dead
too. That commit edited lines 204 and 205 and left 206 alone.

### Accessors, and the `-1` invariant

Writing a slot goes through `ast_channel_internal_fd_set`
(`main/channel_internal_api.c:1028-1040`), with `ast_channel_set_fd`
(`main/channel.c:2417-2421`) as a public one-line passthrough:

```c
void ast_channel_internal_fd_set(struct ast_channel *chan, int which, int value)
{
	int pos;

	/* This ensures that if the vector has to grow with unused positions they will be
	 * initialized to -1.
	 */
	for (pos = AST_VECTOR_SIZE(&chan->fds); pos < which; pos++) {
		AST_VECTOR_REPLACE(&chan->fds, pos, -1);
	}

	AST_VECTOR_REPLACE(&chan->fds, which, value);
}
```

The pre-fill loop is the important part, and the same pattern appears again in
`ast_channel_fd_add` (`:1096-1116`) with a blunter comment:

```c
int ast_channel_fd_add(struct ast_channel *chan, int value)
{
	int pos = AST_EXTENDED_FDS;
	int gap;

	while (ast_channel_fd_isset(chan, pos)) {
		pos += 1;
	}

	/* Unused slots must not become descriptors (in particular, stdin). */
	for (gap = AST_VECTOR_SIZE(&chan->fds); gap < pos; ++gap) {
		if (AST_VECTOR_REPLACE(&chan->fds, gap, -1)) {
			return -1;
		}
	}
	if (AST_VECTOR_REPLACE(&chan->fds, pos, value)) {
		return -1;
	}

	return pos;
}
```

"Unused slots must not become descriptors (in particular, stdin)" is the whole
reason both loops exist. A grown-but-unwritten vector slot would be zero, and
`ast_waitfor` treats any non-negative value as a descriptor to poll
(`include/asterisk/channel.h:2875-2885`):

```c
/*!
 * \brief if fd is a valid descriptor, set *pfd with the descriptor
 * \return Return 1 (not -1!) if added, 0 otherwise (so we can add the
 * return value to the index into the array)
 */
static inline int ast_add_fd(struct pollfd *pfd, int fd)
{
	pfd->fd = fd;
	pfd->events = POLLIN | POLLPRI;
	return fd >= 0;
}
```

`ast_add_fd` returning 1 rather than -1 is what lets the caller write
`max += ast_add_fd(&pfds[max], fd)` and have empty slots silently vanish from
the poll set. Without the `-1` pre-fill, every channel would be polling fd 0.

Reading is bounds-checked against the vector size rather than against
`AST_MAX_FDS`, so an out-of-range query is `-1` rather than undefined
(`main/channel_internal_api.c:1053-1065`):

```c
int ast_channel_fd(const struct ast_channel *chan, int which)
{
	return (which >= AST_VECTOR_SIZE(&chan->fds)) ? -1 : AST_VECTOR_GET(&chan->fds, which);
}
int ast_channel_fd_isset(const struct ast_channel *chan, int which)
{
	return ast_channel_fd(chan, which) > -1;
}

int ast_channel_fd_count(const struct ast_channel *chan)
{
	return AST_VECTOR_SIZE(&chan->fds);
}
```

`ast_channel_internal_fd_clear` writes `-1` into a slot and
`ast_channel_internal_fd_clear_all` resets the vector to size zero (`:1041-1052`).
Both are internal; a channel driver uses `ast_channel_set_fd(chan, which, -1)`
instead, which is why that call shows up so often in the drivers.

`ast_channel_fd_count` carries `\pre chan is locked`
(`include/asterisk/channel.h:4392-4402`), as does `ast_channel_fd_add`
(`:4404-4416`). `ast_channel_fd` and `ast_channel_fd_isset` carry no
precondition at all (`:4389-4390`) — chapter 03's general point about the
accessor layer applies here too.

### Who uses which slot

In the low region the conventions are per-driver and entirely informal. Slot 0
is the primary media descriptor everywhere it is used — RTP for
`channels/chan_rtp.c:383`, the DAHDI subchannel fd for
`channels/chan_dahdi.c:9696`, a raw socket for
`channels/chan_audiosocket.c:277`. `chan_motif` uses 0 and 1 for audio RTP and
RTCP and 2 and 3 for video (`channels/chan_motif.c:846-847`, `:733-734`), and
`addons/chan_ooh323.c:4644-4646` uses 0, 1 and 5.

Above `AST_EXTENDED_FDS` the convention is different: a driver computes its own
index and must be able to invert the arithmetic later. `res_pjsip_session.c`
registers one descriptor per media read callback
(`res/res_pjsip_session.c:1239-1250`):

```c
	/* Remove all current file descriptors from the channel */
	for (i = 0; i < AST_VECTOR_SIZE(&session->active_media_state->read_callbacks); ++i) {
		ast_channel_internal_fd_clear(session->channel, i + AST_EXTENDED_FDS);
	}

	/* Add all the file descriptors from the pending media state */
	for (i = 0; i < AST_VECTOR_SIZE(&session->pending_media_state->read_callbacks); ++i) {
		struct ast_sip_session_media_read_callback_state *callback_state;

		callback_state = AST_VECTOR_GET_ADDR(&session->pending_media_state->read_callbacks, i);
		ast_channel_internal_fd_set(session->channel, i + AST_EXTENDED_FDS, callback_state->fd);
	}
```

and `chan_pjsip`'s read callback recovers the callback index by subtraction
(`channels/chan_pjsip.c:949-963`):

```c
static struct ast_frame *chan_pjsip_read_stream(struct ast_channel *ast)
{
	struct ast_sip_channel_pvt *channel = ast_channel_tech_pvt(ast);
	struct ast_sip_session *session = channel->session;
	struct ast_sip_session_media_read_callback_state *callback_state;
	struct ast_frame *f;
	int fdno = ast_channel_fdno(ast) - AST_EXTENDED_FDS;
	struct ast_frame *cur;

	if (fdno >= AST_VECTOR_SIZE(&session->active_media_state->read_callbacks)) {
		return &ast_null_frame;
	}

	callback_state = AST_VECTOR_GET_ADDR(&session->active_media_state->read_callbacks, fdno);
	f = callback_state->read_callback(session, callback_state->session);
```

That bounds check on line 958 is not decoration. It is the driver defending
itself against being called with an `fdno` that does not correspond to one of
its own slots — see §5.4.

`chan_websocket` does the same thing with named constants instead of arithmetic
(`channels/chan_websocket.c:121-126`), and its comment is a useful warning:
"These are the indexes in the channel's file descriptor array not the file
descriptors themselves." The `fds` vector holds descriptors; `fdno`,
`WS_TIMER_FDNO` and the `AST_*_FD` names are all indexes into it.

For code that does not care where its descriptor lands, `ast_channel_fd_add`
returns the slot it chose. It has exactly two non-test callers, both in
`main/audiohook.c` (`:158` for the initial attach and `:204` for the
masquerade fixup), and both store the returned slot so they can compare it
against `fdno` later (`:238`).

### Two prototypes that do not exist

`include/asterisk/channel.h:4201-4202`:

```c
int ast_channel_epfd(const struct ast_channel *chan);
void ast_channel_epfd_set(struct ast_channel *chan, int value);
```

Neither of those is defined anywhere in Asterisk. They are the last trace of
the epoll backend that `5d938045d4` (Joshua Colp, 2017-03-08, "channel: Remove
old epoll support and fixed max number of file descriptors") deleted on the way
to Asterisk 15. That commit is also what replaced the fixed
`int fds[AST_MAX_FDS]` array with the vector and introduced
`ast_channel_fd_count` and `ast_channel_fd_add`. The definitions it removed
lived inside `#ifdef HAVE_EPOLL` in `main/channel_internal_api.c`; the
declarations in the header never had the guard, so they were simply left
behind. They have been orphans since.

## 5.2 The alert pipe

The alert pipe is how a *frame queued by another thread* becomes a poll event.
It is the mechanism that makes `ast_queue_frame` work across threads at all,
and it is the reason slot 10 is occupied on every real channel.

The storage is two ints in the private struct (`main/channel_private.h:167`),
and the first thing `__ast_channel_alloc_ap` does with them is clear them to
`-1` (`main/channel.c:782`), alongside the same treatment for `timingfd`
(`:781`). The dummy-channel path does the same (`:1027-1029`). That matters for
the registration below.

### eventfd, not a pipe

`ast_alertpipe_init` prefers an `eventfd` and falls back twice
(`main/alertpipe.c:38-77`):

```c
int ast_alertpipe_init(int alert_pipe[2])
{
#ifdef HAVE_EVENTFD

	int fd = eventfd(0, EFD_NONBLOCK | EFD_SEMAPHORE);
	if (fd > -1) {
		alert_pipe[0] = alert_pipe[1] = fd;
		return 0;
	}

	ast_log(LOG_WARNING, "Failed to create alert pipe with eventfd(), falling back to pipe(): %s\n",
		strerror(errno));
	ast_alertpipe_clear(alert_pipe);

#endif

#ifdef HAVE_PIPE2

	if (pipe2(alert_pipe, O_NONBLOCK)) {
		ast_log(LOG_WARNING, "Failed to create alert pipe: %s\n", strerror(errno));
		return -1;
	}

#else

	if (pipe(alert_pipe)) {
		ast_log(LOG_WARNING, "Failed to create alert pipe: %s\n", strerror(errno));
		return -1;
	} else {
		if (ast_fd_set_flags(alert_pipe[0], O_NONBLOCK)
		   || ast_fd_set_flags(alert_pipe[1], O_NONBLOCK)) {
			ast_alertpipe_close(alert_pipe);
			return -1;
		}
	}

#endif

	return 0;
}
```

In the `eventfd` case both ends of the "pipe" are the same descriptor, which is
why `ast_alertpipe_close` has a special case for
`alert_pipe[0] == alert_pipe[1]` (`:79-100`) and why
`ast_alertpipe_readable`/`_writable` test the two array entries separately
(`include/asterisk/alertpipe.h:109-130`).

`EFD_SEMAPHORE` is the flag that makes the whole scheme work. With it, each
`read()` decrements the counter by one and returns one event, so the
one-write-per-frame / one-read-per-frame bookkeeping below stays in step. The
`pipe2`/`pipe` fallbacks get the same behaviour from writing eight bytes per
frame.

### Registration, and the `needqueue` fork

`__ast_channel_alloc_ap`, `main/channel.c:834-855`:

```c
	if ((timer = ast_timer_open())) {
		ast_channel_timer_set(tmp, timer);
		if (strcmp(ast_timer_get_name(ast_channel_timer(tmp)), "timerfd")) {
			needqueue = 0;
		}
		ast_channel_timingfd_set(tmp, ast_timer_fd(ast_channel_timer(tmp)));
	}

	if (needqueue && ast_channel_internal_alertpipe_init(tmp)) {
		/* See earlier channel creation abort comment above. */
		return ast_channel_unref(tmp);
	}

	if (!(topology = ast_stream_topology_alloc())) {
		return ast_channel_unref(tmp);
	}
	ast_channel_internal_set_stream_topology(tmp, topology);

	/* Always watch the alertpipe */
	ast_channel_set_fd(tmp, AST_ALERT_FD, ast_channel_internal_alert_readfd(tmp));
	/* And timing pipe */
	ast_channel_set_fd(tmp, AST_TIMING_FD, ast_channel_timingfd(tmp));
```

Three things happen here. The timer is opened, and then — note the `strcmp`,
which is *non-zero* when the names differ — if the opened timer is anything
other than `timerfd`, `needqueue` is forced to 0. The sense is the opposite of
what the shape of the line suggests: a channel backed by `res_timing_timerfd`
keeps its alert pipe, and a channel backed by one of the other timing modules
(`res_timing_dahdi`, `res_timing_pthread`, `res_timing_kqueue`) loses it and
falls back to continuous mode on the timing fd. Commit `12a40275f2` says so
outright: "To fix this issue the alert pipe will now be used when the timerfd
module is in use." The alert pipe is created only if `needqueue` survives.
And then *both* slots are registered unconditionally, which is safe precisely
because `:781-782` pre-set both to `-1`: a channel without an alert pipe
registers `-1` at slot 10, and `ast_add_fd` drops it.

`needqueue` is a parameter of `__ast_channel_alloc_ap` (`:746`) and therefore of
the public `ast_channel_alloc` family (`:967`, `:986`); see
[chapter 02](02-lifecycle.md).

### One write per frame, one read per frame

The write side is the tail of `__ast_queue_frame` (`main/channel.c:1151-1164`),
and it is a three-way fallback:

```c
	if (ast_channel_alert_writable(chan)) {
		/* Write to the alert pipe for each added frame */
		while (new_frames--) {
			if (ast_channel_alert_write(chan)) {
				ast_log(LOG_WARNING, "Unable to write to alert pipe on %s (qlen = %u): %s!\n",
					ast_channel_name(chan), queued_frames, strerror(errno));
				break;
			}
		}
	} else if (ast_channel_timingfd(chan) > -1) {
		ast_timer_enable_continuous(ast_channel_timer(chan));
	} else if (ast_test_flag(ast_channel_flags(chan), AST_FLAG_BLOCKING)) {
		pthread_kill(ast_channel_blocker(chan), SIGURG);
	}
```

An alert pipe if there is one; otherwise the timer in continuous mode;
otherwise `SIGURG` to whichever thread is parked in `poll`. Chapter 04 covers
that last branch and the other two `pthread_kill` sites.

The read side is in `__ast_read`, immediately after the timing-fd dispatch and
before the read queue is touched (`main/channel.c:3659-3664`):

```c
	/* Read and ignore anything on the alertpipe, but read only
	   one sizeof(blah) per frame that we send from it */
	if (ast_channel_internal_alert_read(chan) == AST_ALERT_READ_FATAL) {
		f = &ast_null_frame;
		goto done;
	}
```

The counting has to be exact in both directions, so every other path that
changes the queue length pokes the pipe to match. The queue-overload flush
reads once per discarded frame (`:1131-1133`). When `__ast_read` finds the
queue non-empty but every frame on it unacceptable — buffered DTMF it is not
allowed to return yet — it writes the pipe back so the next `poll` still fires
(`:3686-3701`).

### Masquerade

Masquerade swaps alert pipes between the two channels, which forces a
correction (`main/channel.c:7031-7055`):

```c
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

The comment is the clearest statement in the tree of what the alert pipe is: a
count that must match the readq length. Chapter 06 covers the rest of the
swap.

The destructor closes the pipe (`main/channel.c:2274`).

## 5.3 `ast_waitfor` and friends

Five public entry points, one implementation:

| Function | Defined | Call sites | Notes |
|---|---|---|---|
| `ast_waitfor_nandfds` | `main/channel.c:2992` | 14 (5 of them inside `channel.c`) | the only real implementation |
| `ast_waitfor` | `:3166` | 57 | one channel, no plain fds; what most code calls |
| `ast_waitfor_n` | `:3161` | 7 | n channels, no plain fds |
| `ast_waitfor_n_fd` | `:2984` | 4 | plain fds only, no channel |
| `ast_waitfordigit` | `:3179` | 68 | wrapper over `ast_waitfordigit_full` (`:3244`, 6 call sites) |

`ast_waitfor_n_fd` passes `c = NULL, n = 0`, so "wait on a bare fd" and "wait
on a channel" are the same code path — `bridges/bridge_softmix.c:1950` waits on
a timing fd that way. The reverse trick is also in the tree:
`res/res_agi.c:4346` passes a real channel pointer with `n = 0` to watch only
the AGI control fd while deliberately *not* watching the channel.

### Pass one: deadlines and sizing

```c
	for (x = 0; x < n; x++) {
		ast_channel_lock(c[x]);
		if (!ast_tvzero(*ast_channel_whentohangup(c[x]))) {
			if (ast_tvzero(whentohangup))
				now = ast_tvnow();
			diff = ast_tvsub(*ast_channel_whentohangup(c[x]), now);
			if (diff.tv_sec < 0 || ast_tvzero(diff)) {
				ast_test_suite_event_notify("HANGUP_TIME", "Channel: %s", ast_channel_name(c[x]));
				/* Should already be hungup */
				ast_channel_softhangup_internal_flag_add(c[x], AST_SOFTHANGUP_TIMEOUT);
				ast_channel_unlock(c[x]);
				return c[x];
			}
			if (ast_tvzero(whentohangup) || ast_tvcmp(diff, whentohangup) < 0)
				whentohangup = diff;
		}
		sz += ast_channel_fd_count(c[x]);
		ast_channel_unlock(c[x]);
	}
```

Two things to notice. First, this is already a mutation: a channel whose
`whentohangup` has passed gets `AST_SOFTHANGUP_TIMEOUT` set and is returned
immediately, before any polling happens. That is the second of the three
places in the tree that set that bit; the other two are `ast_check_hangup`
(`:456`, see chapter 04) and the post-poll pass below.

Second, `sz` is accumulated here from `ast_channel_fd_count` under each
channel's lock, and the lock is then dropped. The `pfds` and `fdmap` arrays are
`ast_alloca`'d to that size at `:3040-3041`, and pass two re-reads
`ast_channel_fd_count` without re-checking the total against `sz`. If
`sz` is zero — every channel has an empty fd vector and no plain fds were
passed — the function returns `NULL` at `:3036-3038` with `*ms` untouched,
which is how a caller polling a descriptorless channel ends up in a busy loop.
This is what the struct comment in §5.1 means by "at least one must be
non -1".

### Pass two: build the poll set and claim the channels

```c
	/*
	 * Build the pollfd array, putting the channels' fds first,
	 * followed by individual fds. Order is important because
	 * individual fd's must have priority over channel fds.
	 */
	max = 0;
	for (x = 0; x < n; x++) {
		ast_channel_lock(c[x]);
		for (y = 0; y < ast_channel_fd_count(c[x]); y++) {
			fdmap[max].fdno = y;  /* fd y is linked to this pfds */
			fdmap[max].chan = x;  /* channel x is linked to this pfds */
			max += ast_add_fd(&pfds[max], ast_channel_fd(c[x], y));
		}
		CHECK_BLOCKING(c[x]);
		ast_channel_unlock(c[x]);
	}
	/* Add the individual fds */
	for (x = 0; x < nfds; x++) {
		fdmap[max].chan = -1;
		max += ast_add_fd(&pfds[max], fds[x]);
	}
```

`CHECK_BLOCKING` is taken here, under each channel's lock, once per channel —
including channels that contributed no descriptors. Its definition is in
`include/asterisk/channel.h:2921-2945`:

```c
/*!
 * \brief Set the blocking indication on the channel.
 *
 * \details
 * Indicate that the thread handling the channel is about to do a blocking
 * operation to wait for media on the channel.  (poll, read, or write)
 *
 * Masquerading and ast_queue_frame() use this indication to wake up the thread.
 *
 * \pre The channel needs to be locked
 */
#define CHECK_BLOCKING(c) \
	do { \
		if (ast_test_flag(ast_channel_flags(c), AST_FLAG_BLOCKING)) { \
			/* This should not happen as there should only be one thread handling a channel's media at a time. */ \
			ast_log(LOG_DEBUG, "Thread LWP %d is blocking '%s', already blocked by thread LWP %d in procedure %s\n", \
				ast_get_tid(), ast_channel_name(c), \
				ast_channel_blocker_tid(c), ast_channel_blockproc(c)); \
			ast_assert(0); \
		} \
		ast_channel_blocker_tid_set((c), ast_get_tid()); \
		ast_channel_blocker_set((c), pthread_self()); \
		ast_channel_blockproc_set((c), __PRETTY_FUNCTION__); \
		ast_set_flag(ast_channel_flags(c), AST_FLAG_BLOCKING); \
	} while (0)
```

So entering `poll` records three things on the channel: the OS thread id, the
pthread handle, and `__PRETTY_FUNCTION__` of the function that blocked. The
thread id and function name exist only for diagnostics — the assertion in this
macro, the `ast_hangup` warning in §5.5, and `CLI` output via
`apps/app_dumpchan.c:169`. The pthread handle is operational: it is the target
of the three `SIGURG` sends catalogued in chapter 04.

### The poll itself

```c
	if (sizeof(int) == 4) {	/* XXX fix timeout > 600000 on linux x86-32 */
		do {
			int kbrms = rms;
			if (kbrms > 600000) {
				kbrms = 600000;
			}
			res = ast_poll(pfds, max, kbrms);
			if (!res) {
				rms -= kbrms;
			}
		} while (!res && (rms > 0));
	} else {
		res = ast_poll(pfds, max, rms);
	}
	for (x = 0; x < n; x++) {
		ast_channel_lock(c[x]);
		ast_clear_flag(ast_channel_flags(c[x]), AST_FLAG_BLOCKING);
		ast_channel_unlock(c[x]);
	}
	if (res < 0) { /* Simulate a timeout if we were interrupted */
		if (errno != EINTR) {
			*ms = -1;
		}
		return NULL;
	}
```

The poll runs with **no channel lock held** — all the locks were released at
the end of pass two. The 32-bit branch above it is a workaround for a Linux
x86-32 `poll` timeout limit, re-polling in 600-second chunks; the comment is
the only documentation it has.

Pass three clears `AST_FLAG_BLOCKING` from every channel, re-taking each lock
individually. Note that this happens before the error check, so an interrupted
poll still unblocks everything.

The `errno` handling is worth reading twice. On a hard error `*ms` is set to
`-1`; on `EINTR` the function returns `NULL` with `*ms` *unmodified*, because
the early return skips the decrement block at `:3152-3157`. So an interrupted
`ast_waitfor(c, -1)` returns 100000 — a positive, apparently-ready value — with
nothing ready and `fdno` still `-1` from the last `ast_read`. That is one of
the "certain situations" the `AST_DEVMODE` check in §5.4 is tolerant of.

### Pass four: who won

```c
	/*
	 * Then check if any channel or fd has a pending event.
	 * Remember to check channels first and fds last, as they
	 * must have priority on setting 'winner'
	 */
	for (x = 0; x < max; x++) {
		res = pfds[x].revents;
		if (res == 0) {
			continue;
		}
		if (fdmap[x].chan >= 0) {	/* this is a channel */
			winner = c[fdmap[x].chan];	/* override previous winners */
			ast_channel_lock(winner);
			if (res & POLLPRI) {
				ast_set_flag(ast_channel_flags(winner), AST_FLAG_EXCEPTION);
			} else {
				ast_clear_flag(ast_channel_flags(winner), AST_FLAG_EXCEPTION);
			}
			ast_channel_fdno_set(winner, fdmap[x].fdno);
			ast_channel_unlock(winner);
		} else {			/* this is an fd */
			if (outfd) {
				*outfd = pfds[x].fd;
			}
			if (exception) {
				*exception = (res & POLLPRI) ? -1 : 0;
			}
			winner = NULL;
		}
	}
```

Four consequences, none of them obvious from the signature:

- **Every ready channel gets `fdno` written and `AST_FLAG_EXCEPTION`
  set or cleared**, not just the one returned. The loop overrides `winner` as
  it goes; the side effects are not undone for the losers.
- **A ready plain fd beats a ready channel.** Pass two put channel descriptors
  first and plain fds last, and the fd branch assigns `winner = NULL`. Since the
  loop runs to the end of the array, a plain fd that is ready always has the
  last word. That is what the comment at `:3122-3126` means by fds having
  "priority on setting `winner`".
- **Only the last ready plain fd is reported.** `*outfd` is overwritten each
  time, so a caller watching several plain fds must be prepared to be told
  about only one of them per call.
- **`exception` and `AST_FLAG_EXCEPTION` are different channels of the same
  information.** `POLLPRI` on a plain fd goes to the `*exception` out-parameter;
  `POLLPRI` on a channel descriptor goes to the flag. The out-parameter is
  zeroed up front at `:3012-3014` and then written only in the plain-fd branch,
  so a channel event never raises it.

The post-poll deadline pass at `:3106-3117` is the third `AST_SOFTHANGUP_TIMEOUT`
site, and it sets `winner` only if nothing else has claimed it.

Finally `*ms` is decremented by the elapsed wall time at `:3152-3157`. `*ms` is
an in/out parameter on every function in this family, which is why the
idiomatic loop is `while ((ms = ast_remaining_ms(start, timeout)))` and not a
hand-rolled countdown.

### `ast_waitfor` is not "wait forever"

```c
int ast_waitfor(struct ast_channel *c, int ms)
{
	if (ms < 0) {
		do {
			ms = 100000;
			ast_waitfor_nandfds(&c, 1, NULL, 0, NULL, NULL, &ms);
		} while (!ms);
	} else {
		ast_waitfor_nandfds(&c, 1, NULL, 0, NULL, NULL, &ms);
	}
	return ms;
}
```

`ast_waitfor(c, -1)` is a loop of 100-second polls, repeated as long as each
one times out (`*ms` is set to 0 on timeout, so `!ms` is true). It is not a
single infinite `poll`, and as noted above it can exit early on `EINTR` with a
large positive return value.

### `ast_waitfordigit_full`

`ast_waitfordigit_full` (`main/channel.c:3244`) is the one wrapper worth
reading in full, because it demonstrates the intended shape of a
waitfor/read loop: a `while ((ms = ast_remaining_ms(start, timeout_ms)))`
countdown, a single `ast_waitfor_nandfds` per iteration, and a strict
`rchan`/`outfd`/neither three-way dispatch on the result. It also sets
`AST_FLAG_END_DTMF_ONLY` on entry (`:3254`) and has to clear it on every one of
its many exit paths — a good illustration of why `ast_channel_set_flag` and
`ast_channel_clear_flag` exist (chapter 04).

## 5.4 `fdno`: the handoff

```
   int fdno;   /*!< Which fd had an event detected on */
               main/channel_private.h:147
```

`fdno` is the only thing connecting the two halves. The write set is small:

| Site | What it does |
|---|---|
| `main/channel.c:3140` | `ast_waitfor_nandfds` records the winning slot |
| `main/channel.c:3616` | `__ast_read` clears it before calling a timing func with the lock dropped |
| `main/channel.c:3792` | `__ast_read` clears it on the way out |
| `main/channel.c:3236` | `ast_settimeout_full` clears it when the timing fd is being disabled |
| `main/channel.c:7163` | masquerade copies the clone's `fdno` onto the original |

Four of those five are the waitfor/read pair keeping its own books. The fifth is
not: `ast_channel_fdno_set(original, ast_channel_fdno(clonechan))` runs inside
`ast_do_masquerade`, under the comment "Clone exception becomes real one, as with
fdno", one line below the `AST_FLAG_EXCEPTION` copy at `:7162`. A thread that
returned from `ast_waitfor` and has not yet called `ast_read` can therefore have
both halves of the handoff state rewritten underneath it by a third thread, with
no API call of its own in between. That is the sharpest form of the warning this
chapter opens with, and chapter 06 is about the mechanism.

The read set is where the design shows. `__ast_read` dispatches on `fdno`
before it looks at anything else: the timing fd at `:3592`, the jitterbuffer fd
at `:3655`, and otherwise the driver's `read_stream` or `read` callback — which
reads `fdno` itself to work out which of *its* descriptors fired
(`channels/chan_pjsip.c:955`, `main/audiohook.c:238`,
`main/abstract_jb.c:969`).

The reset at `:3788-3792` says what it is for:

```c
	/*
	 * Reset the recorded file descriptor that triggered this read so that we can
	 * easily detect when ast_read() is called without properly using ast_waitfor().
	 */
	ast_channel_fdno_set(chan, -1);
```

And the detection it enables is a debug log, not an error
(`main/channel.c:3573-3587`):

```c
#ifdef AST_DEVMODE
		/*
		 * The ast_waitfor() code records which of the channel's file
		 * descriptors reported that data is available.  In theory,
		 * ast_read() should only be called after ast_waitfor() reports
		 * that a channel has data available for reading but certain
		 * situations with stasis and ARI could give a false indication.
		 * For this reason, we don't stop any processing.
		 */
		if (ast_channel_fdno(chan) == -1) {
			ast_debug(3,
				"ast_read() on chan '%s' called with no recorded file descriptor.\n",
				ast_channel_name(chan));
		}
#endif
```

This is the honest version of a rule the tree cannot enforce. `ast_read`
*should* only follow a successful `ast_waitfor` on the same channel in the same
thread. When it does not, `fdno` is either `-1` — harmless, you fall through to
the driver's `read` callback with no slot information — or stale, in which case
a driver that indexes an array by `fdno - AST_EXTENDED_FDS` is indexing with a
number that means nothing. Hence `chan_pjsip`'s bounds check, and hence
`audiohook.c:238` comparing `fdno` against the exact slot it was given rather
than assuming.

### `AST_FLAG_EXCEPTION`

The exception flag is the narrowest flag in the subsystem: eight textual
occurrences in the whole tree. One enum definition
(`include/asterisk/channel.h:1009`), one setter and one clearer in the winner
loop (`main/channel.c:3136`, `:3138`), two more clears in `__ast_read`'s
timing and jitterbuffer branches (`:3596`, `:3656`), one copy across a
masquerade (`:7162`), and exactly one place that tests it to decide anything:

```c
	} else {
		if (ast_test_flag(ast_channel_flags(chan), AST_FLAG_EXCEPTION)) {
			if (ast_channel_tech(chan)->exception)
				f = ast_channel_tech(chan)->exception(chan);
			else {
				ast_log(LOG_WARNING, "Exception flag set on '%s', but no exception handler\n", ast_channel_name(chan));
				f = &ast_null_frame;
			}
			/* Clear the exception flag */
			ast_clear_flag(ast_channel_flags(chan), AST_FLAG_EXCEPTION);
```

So `POLLPRI` on a channel descriptor becomes a call to the driver's `exception`
callback instead of its `read` callback, and a driver that registers a
descriptor which can report `POLLPRI` without implementing `exception` earns a
`LOG_WARNING` and a null frame. Note that the `else` branch here is the
not-from-the-readq branch — the exception path is only reached when the read
queue was empty.

## 5.5 `AST_FLAG_BLOCKING` and interruption

`CHECK_BLOCKING` (§5.3) and the three `SIGURG` sends (chapter 04) exist to
solve one problem: a thread parked in `poll` on a channel's descriptors cannot
notice anything that is not a descriptor event. The alert pipe handles queued
frames. `SIGURG` handles the cases where there is no alert pipe, plus hangup
and masquerade, which must interrupt the poll regardless.

The flag is also a correctness assertion. `ast_hangup` refuses to proceed
quietly if someone is still blocked on the channel
(`main/channel.c:2602-2607`):

```c
	if (ast_test_flag(ast_channel_flags(chan), AST_FLAG_BLOCKING)) {
		ast_log(LOG_WARNING, "Hard hangup called by thread LWP %d on %s, while blocked by thread LWP %d in procedure %s!  Expect a failure\n",
			ast_get_tid(), ast_channel_name(chan), ast_channel_blocker_tid(chan),
			ast_channel_blockproc(chan));
		ast_assert(0);
	}
```

"Expect a failure" is not hyperbole: the next thing `ast_hangup` does is call
the driver's `hangup` callback and drop the channel's last reference, while
another thread is inside `poll` on descriptors that callback is about to close.
`blockproc` exists to tell you which function that other thread is in.

Note the asymmetry with `CHECK_BLOCKING` itself: the macro's own
double-block check is also an `ast_assert`, so in a production build both of
these degrade to a log line and the code carries on.

One thing the name does not tell you: the flag is not the waitfor path's
private property. `CHECK_BLOCKING` has ten call sites, and only one of them is
`ast_waitfor_nandfds` (`:3068`). `ast_sendtext_data` takes it across its driver
callback (`:4787`, cleared `:4827`), `ast_write_stream` takes it seven times
around seven different driver calls (`:5286`-`:5486`, each with its own clear),
and `chan_dahdi`'s own read path takes it around the `read` syscall
(`chan_dahdi.c:9064-9066`). The macro's doxygen is accurate where the name is
not: it says the flag means the channel is blocked in "(poll, read, or write)".
So a thread holding `AST_FLAG_BLOCKING` is not necessarily in `poll`, and the
`SIGURG` sent to `ast_channel_blocker` may be interrupting a write.

## 5.6 The timing fd

Every real channel gets a timer at allocation (`main/channel.c:834-840`) and
its descriptor registered at `AST_TIMING_FD`. The timer does three jobs: it
drives generators, it substitutes for the alert pipe when there isn't one, and
it is the clock for anything that needs to be called at a fixed rate on a
channel's own thread.

### Arming it

`main/channel.c:3189-3242`:

```c
int ast_settimeout_full(struct ast_channel *c, unsigned int rate, int (*func)(const void *data), void *data, unsigned int is_ao2_obj)
{
	int res;
	unsigned int real_rate = rate, max_rate;
	struct ast_timer *timer = ast_channel_timer(c);

	ast_channel_lock(c);

	if (ast_channel_timingfd(c) == -1) {
		ast_channel_unlock(c);
		return -1;
	}

	if (!func) {
		rate = 0;
		data = NULL;
	}

	if (rate && rate > (max_rate = ast_timer_get_max_rate(timer))) {
		real_rate = max_rate;
	}

	ast_debug(3, "Scheduling timer at (%u requested / %u actual) timer ticks per second\n", rate, real_rate);

	res = ast_timer_set_rate(timer, real_rate);

	if (ast_channel_timingdata(c) && ast_test_flag(ast_channel_flags(c), AST_FLAG_TIMINGDATA_IS_AO2_OBJ)) {
		ao2_ref(ast_channel_timingdata(c), -1);
	}

	ast_channel_timingfunc_set(c, func);
	ast_channel_timingdata_set(c, data);

	if (data && is_ao2_obj) {
		ao2_ref(data, 1);
		ast_set_flag(ast_channel_flags(c), AST_FLAG_TIMINGDATA_IS_AO2_OBJ);
	} else {
		ast_clear_flag(ast_channel_flags(c), AST_FLAG_TIMINGDATA_IS_AO2_OBJ);
	}

	if (func == NULL && rate == 0 && ast_channel_fdno(c) == AST_TIMING_FD) {
		/* Clearing the timing func and setting the rate to 0
		 * means that we don't want to be reading from the timingfd
		 * any more. Setting c->fdno to -1 means we won't have any
		 * errant reads from the timingfd, meaning we won't potentially
		 * miss any important frames.
		 */
		ast_channel_fdno_set(c, -1);
	}

	ast_channel_unlock(c);

	return res;
}
```

Points worth extracting:

- It takes the channel lock itself, so unlike most of `channel.c` it is safe to
  call without one — and safe to call with one, since the channel lock is
  recursive (chapter 03).
- `ast_settimeout` (`:3184-3187`) is `ast_settimeout_full` with
  `is_ao2_obj = 0`. The `_full` form is what manages an ao2 reference on the
  callback data, and `AST_FLAG_TIMINGDATA_IS_AO2_OBJ` is the record of whether
  it did (chapter 04 catalogues that flag; this is its only writer).
- The requested rate is silently clamped to the timing module's maximum
  (`:3207-3209`). You get `ast_debug` output, not an error.
- Disarming clears `fdno` if it currently points at the timing fd
  (`:3229-3237`), with a comment explaining the race it avoids: a pending
  `ast_read` that still believes the timing fd fired would consume a timer
  event that no longer has a handler, and lose a real frame.
- A channel with no timer (`timingfd == -1`) fails the call outright
  (`:3197-3200`).

### Reading it

`__ast_read` handles the timing fd before everything else
(`main/channel.c:3590-3606`):

```c
	prestate = ast_channel_state(chan);

	if (ast_channel_timingfd(chan) > -1 && ast_channel_fdno(chan) == AST_TIMING_FD) {
		enum ast_timer_event res;
		int trigger_dtmf_emulating = should_trigger_dtmf_emulating(chan);

		ast_clear_flag(ast_channel_flags(chan), AST_FLAG_EXCEPTION);

		res = ast_timer_get_event(ast_channel_timer(chan));

		switch (res) {
		case AST_TIMING_EVENT_EXPIRED:
			if (ast_timer_ack(ast_channel_timer(chan), 1) < 0) {
				ast_log(LOG_ERROR, "Failed to acknowledge timer in ast_read\n");
				goto done;
			}

```

The expired case acknowledges the timer, then — if there is a timing function —
saves `func` and `data`, bumps the ao2 reference if the flag says to, clears
`fdno`, **drops the channel lock**, and calls out (`:3607-3621`). That unlock
is why `:3644-3645` returns `&ast_null_frame` directly instead of using
`goto done`: the exit label re-unlocks, and the lock is already gone.

The function's own header comment at `:3548-3550` promises "there is only one
return point at the end (there are only two exceptions to this)". Count them:
`__ast_read` spans `:3541-4275` and contains exactly two `return` statements,
`:3645` and the exit point itself at `:4275`. There is one exception, not two.
The comment is stale — a second early return existed once and was folded back
into the `goto done` discipline without anyone updating the count.

The continuous case is the alert-pipe substitute:

```c
		case AST_TIMING_EVENT_CONTINUOUS:
			if (AST_LIST_EMPTY(ast_channel_readq(chan)) ||
				!AST_LIST_NEXT(AST_LIST_FIRST(ast_channel_readq(chan)), frame_list)) {
				ast_timer_disable_continuous(ast_channel_timer(chan));
			}
			break;
		}

	} else if (ast_channel_fd_isset(chan, AST_JITTERBUFFER_FD) && ast_channel_fdno(chan) == AST_JITTERBUFFER_FD) {
		ast_clear_flag(ast_channel_flags(chan), AST_FLAG_EXCEPTION);
	}
```

Continuous mode is turned on by `__ast_queue_frame` when there is no alert pipe
(§5.2) and turned off here once the read queue has at most one frame left, and
again at `:3700` when the only frames left are buffered DTMF. The
jitterbuffer branch tacked onto the same `if`/`else if` chain does nothing but
clear the exception flag — the actual jitterbuffer work happens later, in a
framehook.

### Generators

The generator API is the main consumer. `ast_activate_generator` arms the timer
at 50 ticks per second — one tick per 20ms frame — and immediately prods the
channel (`main/channel.c:2972-2978`):

```c
	if (!res) {
		ast_settimeout(chan, 50, generator_force, chan);
		ast_channel_generator_set(chan, gen);
	}
	ast_channel_unlock(chan);

	ast_prod(chan);
```

`ast_prod` (`:5023-5038`) writes a single empty voice frame, and only if the
channel is not yet `AST_STATE_UP`; its `src` string is `"ast_prod"` and
`ast_write` tests for it by name. Teardown is `ast_deactivate_generator`
(`:2892-2906`), which takes the channel lock, calls
`deactivate_generator_nolock` — that is what does the `ast_settimeout(chan, 0,
NULL, NULL)` disarm at `:2888` — and then, if DTMF emulation is mid-flight,
immediately re-arms the timer at 50 ticks per second so the emulation keeps its
clock. Deactivating a generator does not necessarily stop the timing fd.

Callers that need the generator to have actually drained do the waiting
themselves; `ast_tonepair` (`:7636`) is the model, looping on
`ast_channel_generatordata(chan) && ast_waitfor(chan, 100) >= 0` and freeing
every frame `ast_read` hands back until the generator data pointer goes NULL.

## 5.7 Common bugs

| Symptom | Cause |
|---|---|
| `ast_read() on chan '...' called with no recorded file descriptor` in a dev-mode log | `ast_read` without a preceding successful `ast_waitfor` — or a preceding `ast_waitfor` that returned early on `EINTR` |
| Driver read callback indexes past the end of its own array | stale `fdno` from a previous `ast_waitfor`; bounds-check the subtraction like `chan_pjsip.c:958` does |
| 100% CPU in a thread doing `ast_waitfor` | the channel's fd vector is *empty* — `AST_VECTOR_SIZE` zero, not merely all `-1` — so `sz == 0` and the call returns instantly without touching `*ms`. A vector full of `-1`s sleeps correctly instead: `sz` counts slots, `ast_add_fd` drops the empty ones, and `poll` is called with zero descriptors and the full timeout |
| Channel polls on stdin | a vector slot left at 0 instead of `-1`; use `ast_channel_set_fd(chan, which, -1)`, never a bare write |
| Queued frames are never noticed | alert pipe absent (`needqueue == 0`) and the timer is not in continuous mode and `AST_FLAG_BLOCKING` is clear — the three-way fallback at `:1151-1164` has no fourth branch |
| Alert pipe counter drifts; `poll` fires with an empty readq, or a full readq never fires | a path that changed the readq length without a matching `ast_channel_alert_write` / `ast_channel_internal_alert_read` |
| `Hard hangup called by thread LWP N ... while blocked by thread LWP M` | `ast_hangup` raced a thread inside `poll`; `blockproc` names the function that blocked |
| `Thread LWP N is blocking '...', already blocked by thread LWP M in procedure ...` | two threads doing media I/O on one channel — the thing chapter 03's one-thread-per-channel rule forbids |
| `Exception flag set on '...', but no exception handler` | driver registered a descriptor that reported `POLLPRI` and has no `exception` callback |
| Timer events consumed with no handler; occasional lost frames | `fdno` still pointing at `AST_TIMING_FD` after the timing function was cleared; this is what `:3229-3237` guards against |

## 5.8 Rules

| Rule | Why |
|---|---|
| Call `ast_waitfor` (or a relative) before every `ast_read` on the same channel in the same thread | `ast_read` does not poll; it believes `fdno` |
| One thread per channel for media I/O | `CHECK_BLOCKING` asserts it; two threads polling one channel is undefined |
| Treat `*ms` as in/out, and loop on `ast_remaining_ms` | every function in the family rewrites it, including setting it to `-1` on error |
| Do not read `ast_waitfor`'s return value as "ready"; check the channel pointer, `*outfd` and `*ms` separately | a plain fd beats a channel, and `EINTR` can return a positive `*ms` with nothing ready |
| Write `-1`, never 0, to retire a descriptor slot | 0 is a valid descriptor and `ast_add_fd` will poll it |
| Use `ast_channel_fd_add` rather than picking your own extended slot | it finds the first free slot at or above `AST_EXTENDED_FDS` and fills the gaps with `-1` |
| Store the slot you were given and compare it against `fdno`; bounds-check any arithmetic on `fdno` | `fdno` can be stale or `-1` when your callback runs |
| Hold the channel lock for `ast_channel_fd_count` and `ast_channel_fd_add` | both carry `\pre chan is locked` |
| Never clear `AST_FLAG_BLOCKING` for a channel you did not `CHECK_BLOCKING` | whoever set it clears it, and there are ten setters — `ast_waitfor_nandfds` (`:3068`), `ast_sendtext_data` (`:4787`), seven in `ast_write_stream` (`:5286`-`:5486`) and the DAHDI driver's own read path (`chan_dahdi.c:9064`). The flag means "blocked in poll, read, or write", not "blocked in waitfor" |
| Let `ast_settimeout` take the lock | it is one of the few functions in `channel.c` that handles its own locking |
| Do not assume `AST_MAX_FDS` bounds anything | it is the vector's initial capacity and nothing more; the vector grows |
| Treat the `AST_JITTERBUFFER_FD` comment as wrong | it says "generator"; the user is `main/abstract_jb.c` |

## Next

[Chapter 06](06-masquerade.md) covers masquerade and `ast_channel_move` —
including the fd-vector copy at `main/channel.c:7098-7104` and the alert-pipe
swap in §5.2, both of which exist to make a channel's I/O survive having its
guts replaced.
