# 6. Masquerade

**Target tree:** Asterisk, the `24` branch. Line numbers and quoted code may
differ on newer releases and on other branches; re-grep for the symbol name if
you are reading against a different tree.

A channel is bound to the thread that is running it. The PBX thread that
executes dialplan on a channel, or the bridge thread that services it, owns that
channel for as long as it lives, and there is no mechanism anywhere in Asterisk
for handing a running channel from one thread to another.

Masquerade is the workaround. Instead of moving the thread to the call, it moves
the *call* to a channel object that the destination thread already owns, field by
field, and then kills the object the call came from.

```
   before                                    after

   ┌──────────────────┐                      ┌──────────────────┐
   │ dest  "Local/..." │  ← a thread is      │ dest  "PJSIP/..." │  ← same thread,
   │  tech   Local     │    already here,    │  tech   PJSIP     │    now driving
   │  pvt    local_pvt │    or about to be   │  pvt    sip_pvt   │    the real call
   │  fds    [...]     │                     │  fds    [...]     │
   │  context/exten    │  ─────────────────▶ │  context/exten    │  ← NOT moved:
   └──────────────────┘   channel_do_        └──────────────────┘    dest keeps its
                          masquerade                                 own dialplan
   ┌──────────────────┐   (dest, source)     ┌──────────────────┐    position
   │ source "PJSIP/.."│                      │ source "Local/..."│
   │  tech   PJSIP    │                      │  tech   Local    │  ← ZOMBIE,
   │  pvt    sip_pvt  │                      │  pvt    local_pvt│    SOFTHANGUP_DEV,
   │  fds    [...]    │                      │  fds    [...]    │    about to be
   └──────────────────┘                      └──────────────────┘    hung up
                                              ↑ never cleared: the
                                                zombie's slots now
                                                alias dest's fds
```

Three facts carry the chapter:

1. **The surviving object is `dest`, but the surviving *identity* is `source`'s.**
   Name, uniqueid, linkedid, Stasis topic, snapshot and endpoint are all
   *swapped*, so after a masquerade the `dest` pointer answers to the name the
   `source` channel had. Code holding a `source` pointer is left holding a
   zombie; code holding a name or uniqueid is fine. Which of those two a
   subsystem holds decides whether it needs a fixup callback.
2. **There is no transaction.** `channel_do_masquerade` is 488 lines of field
   assignments (`main/channel.c:6881-7368`) interleaved with calls out to driver
   code, datastore callbacks, framehook and audiohook movers, and the bridging
   layer. Any of those can fail; none of them can roll back. The function
   returns `void`. Nor is the whole run atomic: both channel locks are held only
   from `:6966` to `:7262-7263` and again briefly at `:7351-7355`, so the 85
   lines before and the 88 lines after the main locked region — which include
   both `AST_CONTROL_MASQUERADE_NOTIFY` pairs and the call into the bridging
   layer — execute with no channel lock held at all. See [§6.3](#63-the-container-and-locking-dance).
3. **A driver that caches its owner channel pointer must implement `.fixup`,**
   and `channels/chan_websocket.c` — added in April 2025 — caches one and does
   not. See [§6.5](#65-the-fixup-protocol).

See [chapter 02](02-lifecycle.md) for `AST_FLAG_ZOMBIE` and the hangup path,
[chapter 03](03-references-and-locking.md) for `ast_channel_lock_both` and
`CHANNEL_DEADLOCK_AVOIDANCE`, and [chapter 05](05-io-and-waiting.md) for the fd
vector and alert pipe that this chapter rearranges.

## 6.1 Why it exists

The clearest statement of the problem is the fork at the top of
`ast_async_goto` (`main/pbx.c:5252-5268`):

```c
{
	struct ast_channel *newchan;

	ast_channel_lock(chan);
	/* Channels in a bridge or running a PBX can be sent directly to the specified destination */
	if (ast_channel_is_bridged(chan) || ast_channel_pbx(chan)) {
		if (ast_test_flag(ast_channel_flags(chan), AST_FLAG_IN_AUTOLOOP)) {
			priority += 1;
		}
		ast_explicit_goto(chan, context, exten, priority);
		ast_softhangup_nolock(chan, AST_SOFTHANGUP_ASYNCGOTO);
		ast_channel_unlock(chan);
		return 0;
	}
	ast_channel_unlock(chan);

	/* Otherwise, we need to gain control of the channel first */
```

If the channel is bridged or has a PBX, *someone else's thread is already
driving it*, and redirecting it is a matter of setting the dialplan location and
raising a softhangup flag — the owning thread notices and goes there. That is
cheap and it is not a masquerade.

If neither is true, nobody is driving the channel, and the caller cannot simply
start a PBX thread on it: the channel may be in the middle of an `ast_read` from
some other thread's point of view, its driver may hold references to it, and
`ast_pbx_start` on a channel that something else owns is how you get two threads
reading one channel. So instead `ast_channel_yank` allocates a *fresh* channel —
one that demonstrably nobody is driving, because it was created a microsecond
ago — and moves the call into it:

```c
	/* Do not hold any channel locks while calling channel_alloc() since the function
	 * locks the channel container when linking the new channel in. */
	if (!(yanked_chan = ast_channel_alloc(0, AST_STATE_DOWN, 0, 0, my_vars.accountcode,
					my_vars.exten, my_vars.context, NULL, yankee, my_vars.amaflags,
					"Surrogate/%s", my_vars.name))) {
		ao2_cleanup(my_vars.writeformat);
		ao2_cleanup(my_vars.readformat);
		return NULL;
	}

	/* Make formats okay */
	ast_channel_set_readformat(yanked_chan, my_vars.readformat);
	ast_channel_set_writeformat(yanked_chan, my_vars.writeformat);
	ao2_cleanup(my_vars.readformat);
	ao2_cleanup(my_vars.writeformat);
	ast_channel_priority_set(yanked_chan, my_vars.priority);

	ast_channel_unlock(yanked_chan);

	if (ast_channel_move(yanked_chan, yankee)) {
		ast_hangup(yanked_chan);
		return NULL;
	}

	return yanked_chan;
```

The new channel is named `Surrogate/<original name>`, is allocated with
`ast_channel_alloc`, inherits the yankee's formats and dialplan position, and is
then the target of `ast_channel_move`. When `ast_channel_yank` returns, the
caller holds a channel carrying the real call that no other thread has ever
touched, and `ast_pbx_start` on it is safe. The yankee is a zombie.

The priority arithmetic at `:10794-10802` is worth reading twice. A channel
executing dialplan has `priority` set to where it *is*; the yanked channel needs
`priority` set to where it should *go*. The `AST_FLAG_IN_AUTOLOOP` test
distinguishes the two cases, and the header documents the result as a contract:
"The dialplan location on the returned channel is where the channel should be
started in the dialplan if it is returned to it."

`ast_channel_yank` has two callers: `ast_async_goto` above, and
`main/bridge.c:2716`. `ast_channel_move` has five, which §6.7 catalogues.

## 6.2 Entry points and the hangup hold-off

`ast_channel_move` is the only public way in. `channel_do_masquerade` is
`static` and is called from exactly one place — `main/channel.c:10880`, the
statement before `ast_channel_move`'s `return 0`:

```c
int ast_channel_move(struct ast_channel *dest, struct ast_channel *source)
{
	RAII_VAR(struct ast_json *, blob, NULL, ast_json_unref);
	SCOPED_MUTEX(lock, &channel_move_lock);

	if (dest == source) {
		ast_log(LOG_WARNING, "Can't move channel '%s' into itself!\n",
			ast_channel_name(dest));
		return -1;
	}

	ast_channel_lock_both(dest, source);

	if (ast_test_flag(ast_channel_flags(dest), AST_FLAG_ZOMBIE)
		|| ast_test_flag(ast_channel_flags(source), AST_FLAG_ZOMBIE)) {
		/* Zombies! Run! */
		ast_log(LOG_WARNING,
			"Can't move channel. One or both is dead (%s <-- %s)\n",
			ast_channel_name(dest), ast_channel_name(source));
		ast_channel_unlock(source);
		ast_channel_unlock(dest);
		return -1;
	}

	ast_channel_masq_set(dest, source);
	ast_channel_masqr_set(source, dest);

	blob = ast_json_pack("{s: s}",
			"newchanneluniqueid", ast_channel_uniqueid(dest));
	ast_channel_publish_blob(source, ast_channel_masquerade_type(), blob);

	ast_channel_unlock(dest);
	ast_channel_unlock(source);

	channel_do_masquerade(dest, source);
	return 0;
}
```

Four things are set up here, and each is load-bearing.

**The global mutex.** Masquerade cannot hold both channel locks for its whole
duration, so the `masq`/`masqr` pointers have to survive periods where neither
channel is locked. A process-wide mutex is the blunt instrument that keeps two
concurrent moves from interleaving:

```c
/*!
 * Mutex that prevents multiple ast_channel_move() operations
 * from occurring simultaneously. This is necessary since the
 * involved channels have to be locked and unlocked throughout
 * the move operation.
 *
 * The most important data being protected are the masq and masqr
 * data on channels. We don't want them getting criss-crossed due
 * to multiple moves mucking with them.
 */
AST_MUTEX_DEFINE_STATIC(channel_move_lock);
```

This serialises *every* masquerade in the system. A busy transfer-heavy
installation performs them one at a time.

**The zombie check.** A channel already marked `AST_FLAG_ZOMBIE` is one that
`ast_hangup` has committed to destroying, and moving a call into or out of it
would race the destructor. The check at `:10859-10868` is the only thing
rejecting that, and it runs under both locks.

**`masq` and `masqr`.** Two pointers in the struct
(`main/channel_private.h:54-55`), with the terse comments "Channel that will
masquerade as us" and "Who we are masquerading as". They are set here, before
any content moves, and cleared at `:7352-7353`, after everything has moved.
Between those two points they mean "a masquerade is in flight on this channel".
Two places act on that. The first is the top of `ast_hangup`
(`main/channel.c:2552-2561`):

```c
	ast_channel_lock(chan);

	while (ast_channel_masq(chan) || ast_channel_masqr(chan))  {
		CHANNEL_DEADLOCK_AVOIDANCE(chan);
	}

	/* Mark as a zombie so a masquerade cannot be setup on this channel. */
	ast_set_flag(ast_channel_flags(chan), AST_FLAG_ZOMBIE);

	ast_channel_unlock(chan);
```

That loop is the entire interlock between masquerade and hangup. A thread that
calls `ast_hangup` on a channel mid-masquerade spins in
`CHANNEL_DEADLOCK_AVOIDANCE` — dropping and retaking the channel lock, which is
what lets the masquerading thread make progress — until the masquerade clears
the pointers. Only then does it set `AST_FLAG_ZOMBIE` and proceed. Set the flag
before the masquerade finishes and you would be destroying a channel whose
fields are half-swapped.

Note the asymmetry with the check in `ast_channel_move`: `ast_hangup` waits for
an in-flight masquerade, but `ast_channel_move` refuses outright if the flag is
already set. One blocks, the other fails.

The second place is `ast_can_pickup` (`main/pickup.c:82`), which tests
`!ast_channel_masq(chan)` alongside `!ast_channel_pbx(chan)` and the zombie
flag: a channel that something is already masquerading *into* is not offered as
a pickup target. That is a public predicate (`include/asterisk/pickup.h:39`)
with seven call sites across `main/pickup.c` and `apps/app_directed_pickup.c`.
Everything else that reads the pair is diagnostic — `main/features.c:387-391`
logs it, `res/snmp/agent.c:316-324` reports it.

**The Stasis message.** `ast_channel_publish_blob(source, ast_channel_masquerade_type(), ...)`
is published *before* the move, carrying `newchanneluniqueid` — the dest's
uniqueid as it is *right now*, which is the uniqueid the dest is about to lose.
ARI consumers correlate on this.

## 6.3 The container and locking dance

`channel_do_masquerade` opens with a precondition that is a locking-order
statement, not a convenience:

```c
/*!
 * \brief Masquerade a channel
 *
 * \note Assumes _NO_ channels and _NO_ channel pvt's are locked.  If a channel is locked while calling
 *       this function, it invalidates our channel container locking order.  All channels
 *       must be unlocked before it is permissible to lock the channels' ao2 container.
 */
static void channel_do_masquerade(struct ast_channel *original, struct ast_channel *clonechan)
```

Channel names and uniqueids are keys in the channel storage container
(see [chapter 02](02-lifecycle.md)), and masquerade changes both. So the
channels have to come out of the container, be rewritten, and go back in — and
taking the container lock while holding a channel lock is the inversion that
chapter 03 warns about. Hence: unlink first, with no channel locks held.

```c
	/*
	 * The container lock is necessary for proper locking order
	 * because the channels must be unlinked to change their
	 * names.
	 *
	 * The original and clonechan locks must be held while the
	 * channel contents are shuffled around for the masquerade.
	 *
	 * The masq and masqr pointers need to be left alone until the masquerade
	 * has restabilized the channels to hold off ast_hangup() and until
	 * AST_FLAG_ZOMBIE can be set on the clonechan.
	 */
	CHANNELSTORAGE_API(current_channel_storage_instance, wrlock);

	/* Bump the refs to ensure that they won't disappear on us. */
	ast_channel_ref(original);
	ast_channel_ref(clonechan);

	/*
	 * Since channel name and unique id will change, and both could be keys
	 * in the channel storage backend, we need to remove them from the backend.
	 * We'll add them back in after the changes are compete.
	 */
	CHANNELSTORAGE_API(current_channel_storage_instance, remove, original, 0);
	CHANNELSTORAGE_API(current_channel_storage_instance, remove, clonechan, 0);

	CHANNELSTORAGE_API(current_channel_storage_instance, unlock);

```

The ordering here is strict. Take the container write lock; bump both refs so
that neither channel can be destroyed while unlinked (being unlinked means the
container no longer holds a reference); remove both; release the container lock.
The channels now exist but cannot be found by name or uniqueid by anything in
the system. `core show channels` will not list them. The reinsert does not
happen until `:7357-7360`, about 400 lines and several out-calls later.

The enforcement is an assert in the accessor:

```c
void ast_channel_internal_swap_uniqueid_and_linkedid(struct ast_channel *a, struct ast_channel *b)
{
	struct ast_channel_id temp;

	/* This operation is used as part of masquerading and so does not invalidate the peer
	 * segment. This is due to the masquerade process invalidating all segments.
	 */

	/*
	 * Since unique ids can be a key in the channel storage backend,
	 * ensure that neither channel is linked in or the keys will be
	 * invalid.
	 */
	ast_assert(!a->linked_in_container && !b->linked_in_container);

	temp = a->uniqueid;
	a->uniqueid = b->uniqueid;
	b->uniqueid = temp;

	temp = a->linkedid;
	a->linkedid = b->linkedid;
	b->linkedid = temp;
}

```

`ast_assert(!a->linked_in_container && !b->linked_in_container)` is a real
guard against a real mistake — swapping the keys of a hashed object while it is
still in the hash table. It is also devmode-only: `include/asterisk/utils.h:779`
defines `ast_assert(a)` as nothing outside `AST_DEVMODE`, so in a production
build the condition is not even evaluated and the container corrupts in
silence — no FRACK, no log line, no backtrace.

Only after the unlink does the function take the channel locks
(`ast_channel_lock_both(original, clonechan)` at `:6966`), and the first thing
it does with them is mark every snapshot segment dirty on both channels
(`:6975-6976`) — because it is about to invalidate essentially all of them and
tracking which would be pointless.

Three out-calls deliberately happen before the locked region: the
`MASQUERADE_NOTIFY` pair at `:6919-6920`, the MOH stop at `:6950-6954`, and the
`ast_indicate(original, -1)` at `:6963`. Each reaches into driver or application
code that may lock other things. The capture that precedes that last one is the
exception that proves the rule — reading `ast_channel_visible_indication` is a
field read, not an out-call, so it takes the channel lock for exactly two lines
(`:6960-6962`) and drops it again before indicating.

The function is not atomic end to end, either. Both locks are held only from
`:6966` to `:7262-7263`, and then again for four lines at `:7351-7355` to clear
`masq`/`masqr` and relink. The 85 lines before the main region and the 88 lines
after it run with no channel lock held — and the after-region is where the
bridging layer is notified (`ast_bridge_notify_masquerade`, `:7278`), where the
second `MASQUERADE_NOTIFY` pair is delivered (`:7275-7276`), where DTMF is ended
(`:7290`) and where MOH is restarted (`:7329`). A reader who assumes "both
channels are locked throughout `channel_do_masquerade`" will mis-reason about
every one of those.

## 6.4 What moves, and what does not

This is the substance of the function. The distinction that matters is not
swap-versus-copy but **what the surviving `original` object ends up with**.

The `original` object takes on the clone's **identity**:

| Field | Operation | Site |
|---|---|---|
| `uniqueid`, `linkedid` | swap | `:6996` |
| Stasis `topic`, `channel_forward` | swap | `:6999` |
| `endpoint_forward` | swap | `:7004` |
| `snapshot` | swap | `:7007` |
| `endpoint` | swap | `:7010` |
| `name` | swap, via `ast_channel_name_set` to avoid rename events | `:7015-7017` |

its **media plane**:

| Field | Operation | Site |
|---|---|---|
| `tech`, `tech_pvt` | swap | `:7020-7026` |
| `alertpipe` | swap | `:7029` |
| `readq` | *merged*, clone's frames first — see below | `:7042-7055` |
| `rawreadformat`, `rawwriteformat` | swap | `:7058-7066` |
| fd vector | cleared, then copied slot-for-slot from clone | `:7099-7104` |
| `fdno`, `AST_FLAG_EXCEPTION`, `AST_FLAG_OUTGOING` | copied one way | `:7162-7163` |
| `nativeformats` | copied one way, via a temporary cap | `:7210-7215` |
| `stream_topology` | swap | `:7250` |
| timing fd | *re-registered from the original's own timer* | `:7207` |

and its **call state**:

| Field | Operation | Site |
|---|---|---|
| `state` | swap | `:7074-7076` |
| `AST_FLAG_DISABLE_DEVSTATE_CACHE` | swap, conditionally | `:7080-7090` |
| `language`, `parkinglot`, `musicclass`, `accountcode`, `userfield`, `adsicpe` | copied one way | `:7093`, `:7096`, `:7227`, `:7230`, `:7233`, `:7156` |
| hangup handler list | swap | `:7109-7111` |
| `dialed`, `caller`, `connected`, `redirecting` party structs | swap, whole-structure | `:7172-7202` |
| datastores | clone's list *appended* to original's | `:7142-7144` |
| framehooks, audiohooks, autochans | moved by their own subsystems | `:7147-7152` |
| channel variables | appended and then duplicated back — see below | `:6846-6860` |

What the `original` object **keeps as its own**:

| Field | Why it matters |
|---|---|
| `context`, `exten`, `priority` | this is the whole point — the dest's dialplan position is what the call is being moved *to* |
| `pbx` | the thread driving it stays driving it |
| `app`, `appl`, `data` | "Application and data remain the same" (`:7160`) |
| `readformat`, `writeformat` | saved at `:6982-6983`, re-applied at `:7221-7224` through `ast_set_write_format`/`ast_set_read_format`, which is what rebuilds the translation path |
| bridge, CDR, sched context, streams | per the comment block below |
| `AST_FLAG_BLOCKING`, `blocker`, `blockproc`, softhangup flags | *not handled at all* |

That last row is the tree's own admission:

```c
	/* Bridge remains the same */
	/* CDR fields remain the same */
	/* XXX What about blocking, softhangup, blocker, and lock and blockproc? XXX */
	/* Application and data remain the same */
	/* Clone exception  becomes real one, as with fdno */
	ast_set_flag(ast_channel_flags(original), ast_test_flag(ast_channel_flags(clonechan), AST_FLAG_EXCEPTION | AST_FLAG_OUTGOING));
	ast_channel_fdno_set(original, ast_channel_fdno(clonechan));
	/* Schedule context remains the same */
	/* Stream stuff stays the same */
	/* Keep the original state.  The fixup code will need to work with it most likely */
```

`/* XXX What about blocking, softhangup, blocker, and lock and blockproc? XXX */`
has been in the file for a very long time and is a fair question. The `blocker`
thread handle on the surviving channel still refers to whichever thread was
blocked on the *dest* before the masquerade, and the `SIGURG` at `:7336` is sent
to it. Chapter 05 §5.5 covers what that flag is supposed to mean.

**The readq merge is not a swap.** It is the one content operation with a
documented three-part invariant, and the reason is the alert pipe:

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

Because the *alertpipe* was swapped at `:7029` but the dest's pre-existing
frames were not written to that pipe, every one of them has to be re-poked.
Chapter 05 §5.2 explains why the count has to come out exact. This block is the
clearest statement in the tree of the pipe-as-counter contract.

**The fd copy is destructive, and then one slot is put back.** `:7099` calls
`ast_channel_internal_fd_clear_all(original)`, which is an `AST_VECTOR_RESET` —
the vector becomes *empty*, size zero, not a vector of `-1`s. The loop at
`:7102-7104` then copies the clone's slots across:

```c
	/* Clear all existing file descriptors */
	ast_channel_internal_fd_clear_all(original);

	/* Copy all file descriptors present on clonechan to original */
	for (x = 0; x < ast_channel_fd_count(clonechan); x++) {
		ast_channel_set_fd(original, x, ast_channel_fd(clonechan, x));
	}
```

Which wipes `AST_TIMING_FD`, because the clone's slot 9 holds the *clone's*
timer fd, and the clone's timer is not moving — `timer` and `timingfd` are not
in any of the tables above. Hence the correction 100 lines later at `:7207`,
`ast_channel_set_fd(original, AST_TIMING_FD, ast_channel_timingfd(original))`,
whose comment reads "Restore original timing file descriptor". Between `:7099`
and `:7207` the surviving channel's timing slot holds a descriptor belonging to
a channel that is about to be hung up.

Note also what is *not* done: this is the only `fd_clear_all` in the function,
and it is called on `original`. The clone's vector is never touched, so the
zombie walks away holding a full set of slots whose descriptors are now also
installed on the surviving channel — plus an `AST_ALERT_FD` slot pointing at the
pipe that was swapped out from under it at `:7029`. Nothing polls a zombie, so
this is harmless in practice, but it means you cannot use "the fd vector is
empty" as a test for whether a channel has been masqueraded away.

**Channel variables are appended and then copied back**
(`main/channel.c:6846-6860`):

```c
static void clone_variables(struct ast_channel *original, struct ast_channel *clonechan)
{
	struct ast_var_t *current, *newvar;
	/* Append variables from clone channel into original channel */
	/* XXX Is this always correct?  We have to in order to keep MACROS working XXX */
	AST_LIST_APPEND_LIST(ast_channel_varshead(original), ast_channel_varshead(clonechan), entries);

	/* then, dup the varshead list into the clone */

	AST_LIST_TRAVERSE(ast_channel_varshead(original), current, entries) {
		newvar = ast_var_assign(current->name, current->value);
		if (newvar)
			AST_LIST_INSERT_TAIL(ast_channel_varshead(clonechan), newvar, entries);
	}
}
```

The clone's variables are appended to the original's list, and then the whole
merged list is *duplicated back* onto the clone. The `XXX Is this always
correct? We have to in order to keep MACROS working XXX` is the tree's own
assessment. The reason the zombie needs variables at all is that its hangup
handlers and `h` extension can still run.

## 6.5 The fixup protocol

Everything above moves data that `channel.c` knows about. Everything that
*other* subsystems know about has to be fixed up by those subsystems, through
four separate mechanisms, called in a specific order.

### `tech->fixup`

```c

	/*! \brief Fix up a channel:  If a channel is consumed, this is called.  Basically update any ->owner links */
	int (* const fixup)(struct ast_channel *oldchan, struct ast_channel *newchan);
```

"Basically update any `->owner` links" is the whole job. A channel driver's
private structure almost always holds a back-pointer to the channel that owns
it; after the tech_pvt swap at `:7024-7026`, that pointer is wrong. Both
directions get fixed, and the two calls are mirror images — each driver is
called as `fixup(the other channel, the one now holding its pvt)`:

```c
	/* Fixup the original clonechan's physical side */
	if (ast_channel_tech(original)->fixup && ast_channel_tech(original)->fixup(clonechan, original)) {
		ast_log(LOG_WARNING, "Channel type '%s' could not fixup channel %s, strange things may happen. (clonechan)\n",
			ast_channel_tech(original)->type, ast_channel_name(original));
	}

	/* Fixup the original original's physical side */
	if (ast_channel_tech(clonechan)->fixup && ast_channel_tech(clonechan)->fixup(original, clonechan)) {
		ast_log(LOG_WARNING, "Channel type '%s' could not fixup channel %s, strange things may happen. (original)\n",
			ast_channel_tech(clonechan)->type, ast_channel_name(clonechan));
	}
```

Read the pairing carefully. `ast_channel_tech(original)` is, by this point, the
*clone's* original tech — the swap happened 200 lines earlier. So the first call
hands the clone's driver `(oldchan = clonechan, newchan = original)`: your call
leg now lives in `original`. The second hands the dest's old driver
`(oldchan = original, newchan = clonechan)`: your call leg now lives in the
zombie. Both drivers are told, and a failure in either is a `LOG_WARNING` with
the words "strange things may happen" and nothing else.

`chan_pjsip`'s is representative of a correct one
(`channels/chan_pjsip.c:1156-1174`):

```c
static int chan_pjsip_fixup(struct ast_channel *oldchan, struct ast_channel *newchan)
{
	struct ast_sip_channel_pvt *channel = ast_channel_tech_pvt(newchan);

	if (channel->session->channel != oldchan) {
		return -1;
	}

	/*
	 * The masquerade has suspended the channel's session
	 * serializer so we can safely change it outside of
	 * the serializer thread.
	 */
	channel->session->channel = newchan;

	set_channel_on_rtp_instance(channel->session, ast_channel_uniqueid(newchan));

	return 0;
}
```

Three things: it validates that `oldchan` is the channel it thinks it owns and
returns `-1` if not; it repoints `session->channel`; and it refreshes the
channel id recorded on the RTP instance, because the uniqueid changed at
`:6996`. The comment — "The masquerade has suspended the channel's session
serializer so we can safely change it outside of the serializer thread" — is
`MASQUERADE_NOTIFY` doing its job; §6.6.

Fourteen `.fixup` assignments exist in the tree, naming eleven distinct
functions; `ast_unreal_fixup` is used four times (by `core_local.c`,
`chan_bridge_media.c` twice, and `conf_chan_announce.c`). It is the most
intricate of them, because an unreal channel has *two* owner pointers and a
masquerade must not leave the pair bridged to itself
(`main/core_unreal.c:369-397`):

```c
int ast_unreal_fixup(struct ast_channel *oldchan, struct ast_channel *newchan)
{
	struct ast_unreal_pvt *p = ast_channel_tech_pvt(newchan);
	struct ast_bridge *bridge_owner;
	struct ast_bridge *bridge_chan;

	if (!p) {
		return -1;
	}

	ao2_lock(p);

	if ((p->owner != oldchan) && (p->chan != oldchan)) {
		ast_log(LOG_WARNING, "Old channel %p wasn't %p or %p\n", oldchan, p->owner, p->chan);
		ao2_unlock(p);
		return -1;
	}
	if (p->owner == oldchan) {
		p->owner = newchan;
	} else {
		p->chan = newchan;
	}

	if (ast_check_hangup(newchan) || !p->owner || !p->chan) {
		ao2_unlock(p);
		return 0;
	}

	/* Do not let a masquerade cause an unreal channel to be bridged to itself! */
```

**Three channel drivers in the tree have no `.fixup` at all:**
`channels/chan_rtp.c`, `channels/chan_audiosocket.c` and
`channels/chan_websocket.c`. For two of them that is correct —
`chan_audiosocket`'s `struct audiosocket_instance`
(`channels/chan_audiosocket.c:48-52`) holds a file descriptor, a server
address and a UUID and no channel pointer, and `chan_rtp` uses the
`ast_rtp_instance` itself as its tech_pvt. (`chan_rtp` does leave a stale
`channel_id` string on the RTP instance after a masquerade, set at `:222` and
`:381` and never refreshed — the exact thing `chan_pjsip_fixup` bothers to
correct. It affects RTP statistics reporting, not call handling.)

`chan_websocket` is a different matter. Its private structure holds a
reference-counted channel back-pointer:

```c
struct websocket_pvt {
	enum ast_websocket_type type;
	struct ast_websocket_client *client;
	struct ast_websocket *websocket;
	struct ast_format *native_format;
	struct ast_codec *native_codec;
	struct ast_channel *channel;
	struct ast_timer *timer;
```

`instance->channel` is set at `:1822` with `ao2_bump` and released in the
destructor at `:1410-1413`. It is read in roughly seventy places, and not all of
them are diagnostics: `ast_queue_frame(instance->channel, &ast_null_frame)` at
`:939`, `:951` and `:965`, `ast_queue_control(instance->channel, AST_CONTROL_ANSWER)`
at `:791` and `:1256`, `ast_queue_hangup_with_cause(instance->channel, ...)` at
`:1276`, and two writes into the channel's fd vector —
`ast_channel_set_fd(instance->channel, WS_WEBSOCKET_FDNO, ...)` at `:1237` and
`ast_channel_set_fd(instance->channel, WS_TIMER_FDNO, ...)` at `:1571`. The tech
table has no `.fixup` entry:

```c
static struct ast_channel_tech websocket_tech = {
	.type = "WebSocket",
	.description = "Media over WebSocket Channel Driver",
	.requester = webchan_request,
	.call = webchan_call,
	.read = webchan_read,
	.write = webchan_write,
	.hangup = webchan_hangup,
	.send_digit_end = webchan_send_dtmf_text,
};
```

After a masquerade moves a WebSocket call into another channel object,
`instance->channel` still points at the object that became the zombie. Media and
hangups queued by the WebSocket read thread go onto a channel nobody is reading,
and `set_channel_timer` installs a timing fd on a channel that is being torn
down. The driver was added in `5963e624e2` (2025-04-28, "Media over Websocket
Channel Driver") and has 23 commits; `git log -S'.fixup' -- channels/chan_websocket.c`
returns nothing, so the callback has never been present. Any path that
masquerades a WebSocket channel — call pickup, a blind transfer out of a bridge,
an attended transfer through a `Local` channel, a PJSIP `REFER`, or
`ast_async_goto` on an unbridged channel — will hit this. (Unreal/`Local`
*optimisation* is not one of them: `ast_bridge_unreal_optimize_out`
(`main/bridge.c:3129`) moves bridge channels around with
`try_swap_optimize_out`/`try_merge_optimize_out` and never calls
`ast_channel_move`. The five entry points are exactly the ones
[§6.7](#67-the-five-callers) lists.)

### Datastore callbacks

Two callbacks, distinguished only by which channel's datastore list they are
invoked for:

```c
	/*!
	 * \brief Fix up channel references on the masquerading channel
	 *
	 * \arg data The datastore data
	 * \arg old_chan The old channel owning the datastore
	 * \arg new_chan The new channel owning the datastore
	 *
	 * This is exactly like the fixup callback of the channel technology interface.
	 * It allows a datastore to fix any pointers it saved to the owning channel
	 * in case that the owning channel has changed.  Generally, this would happen
	 * when the datastore is set to be inherited, and a masquerade occurs.
	 */
	void (*chan_fixup)(void *data, struct ast_channel *old_chan, struct ast_channel *new_chan);

	/*!
	 * \brief Fix up channel references on the channel being masqueraded into
	 *
	 * \arg data The datastore data
	 * \arg old_chan The old channel owning the datastore
	 * \arg new_chan The new channel owning the datastore
	 *
	 * This is the same as the above callback, except it is called for the channel
	 * being masqueraded into instead of the channel that is masquerading.
	 */
	void (*chan_breakdown)(void *data, struct ast_channel *old_chan, struct ast_channel *new_chan);
```

```c
	/* Call fixup handlers for the clone chan */
	if (AST_LIST_FIRST(ast_channel_datastores(clonechan))) {
		struct ast_datastore *ds;
		/* We use a safe traversal here because some fixup routines actually
		 * remove the datastore from the list and free them.
		 */
		AST_LIST_TRAVERSE_SAFE_BEGIN(ast_channel_datastores(clonechan), ds, entry) {
			if (ds->info->chan_fixup) {
				ds->info->chan_fixup(ds->data, clonechan, original);
			}
		}
		AST_LIST_TRAVERSE_SAFE_END;
	}

	/* Call breakdown handlers for the original chan */
	if (AST_LIST_FIRST(ast_channel_datastores(original))) {
		struct ast_datastore *ds;
		/* We use a safe traversal here because some breakdown routines may
		 * remove the datastore from the list and free them.
		 */
		AST_LIST_TRAVERSE_SAFE_BEGIN(ast_channel_datastores(original), ds, entry) {
			if (ds->info->chan_breakdown) {
				ds->info->chan_breakdown(ds->data, clonechan, original);
			}
		}
		AST_LIST_TRAVERSE_SAFE_END;
	}
```

`chan_fixup` runs over the **clone's** datastores — the ones that are about to be
moved onto the surviving channel — and tells each one where it is going.
`chan_breakdown` runs over the **original's** datastores — the ones that are
about to be orphaned, because the clone's list is appended to theirs at
`:7142-7144` and the surviving channel ends up with both. Both callbacks receive
the same `(clonechan, original)` argument pair regardless of which list is being
traversed, which makes the parameter names `old_chan` and `new_chan` misleading
for `chan_breakdown`: for a breakdown datastore, the channel it is attached to is
`new_chan`, not `old_chan`.

The one implementation with a comment spells this out
(`channels/pjsip/channel_info.c:67-81`):

```c
static void channel_info_breakdown(void *data, struct ast_channel *old_chan,
	struct ast_channel *new_chan)
{
	struct pjsip_channel_info *info = data;
	struct ast_datastore *datastore = info->datastore;

	/*
	 * Core calls breakdown on the destination's old datastores before moving
	 * the source's datastores over. Both channels are locked. The moved
	 * datastore needs no fixup because it contains no channel pointer.
	 */
	if (!ast_channel_datastore_remove(new_chan, datastore)) {
		ast_datastore_free(datastore);
	}
}
```

"Core calls breakdown on the destination's old datastores before moving the
source's datastores over" — and the response is to remove and free itself,
because the surviving channel is about to receive the clone's equivalent
datastore and should not have two.

Both traversals use `AST_LIST_TRAVERSE_SAFE_BEGIN` specifically because
callbacks do this. There are nine `.chan_fixup` and four `.chan_breakdown`
registrations in the tree, naming eight and three distinct functions —
`main/stasis_channels.c:2196-2204` registers both on two
datastore types, which is how dial events survive a masquerade; `res_stasis.c`'s
pair (`channel_stolen_cb` at `:1200`, `channel_replaced_cb` at `:1231`) is how
ARI tracks a channel into and out of `Stasis()`, and both carry long comments
explaining which pointer is which, because the naming does not.

### Framehooks and audiohooks

`ast_framehook_list_fixup(clonechan, original)` at `:7147` and
`ast_audiohook_move_all(clonechan, original)` at `:7150`. Framehooks have their
own per-hook `chan_fixup_cb` / `chan_breakdown_cb` pair, driven from
`main/framehook.c:235` and `:264` — used by `main/audiohook.c:322-323`,
`res/res_pjsip_t38.c:665-666` and `main/channel.c:10966`. Audiohooks must move
because, as chapter 05 noted, `main/audiohook.c` is the only non-test caller of
`ast_channel_fd_add` and stores the slot index it was given.

## 6.6 `AST_CONTROL_MASQUERADE_NOTIFY`

A control frame that is never a frame. `include/asterisk/frame.h:311` is
unambiguous: "A masquerade is about to begin/end. (Never sent as a frame but
directly with `ast_indicate_data()`.)" It is delivered four times per
masquerade — to both channels before (`:6919-6920`, payload `1`) and both after
(`:7275-7276`, payload `0`).

Two things make it unlike every other indication. First, it is the sole
exemption from `ast_indicate_data`'s zombie guard
(`main/channel.c:4685-4690`), which is why the post-masquerade notify can be
delivered to a channel that was marked `AST_FLAG_ZOMBIE` at `:7259`. Second,
`main/bridge_channel.c:2320-2322` asserts if it ever arrives as a queued frame:
`/* Should never happen. */ ast_assert(0);`.

Its real purpose is to let a driver quiesce its own threading before its
tech_pvt is moved. `chan_pjsip` is the motivating case
(`channels/chan_pjsip.c:1875-1890`):

```c
		ast_assert(datalen == sizeof(int));
		if (*(int *) data) {
			/*
			 * Masquerade is beginning:
			 * Wait for session serializer to get suspended.
			 */
			ast_channel_unlock(ast);
			ast_sip_session_suspend(channel->session);
			ast_channel_lock(ast);
		} else {
			/*
			 * Masquerade is complete:
			 * Unsuspend the session serializer.
			 */
			ast_sip_session_unsuspend(channel->session);
		}
```

On begin, it *drops the channel lock*, blocks until the session serializer has
suspended, and retakes it. That is why `chan_pjsip_fixup` can rewrite
`session->channel` from the masquerading thread without racing the PJSIP
serializer thread. On end, it unsuspends. Unreal channels block the indication
entirely (`main/core_unreal.c:637-642`, "Always block this because this is the
channel being masqueraded; not anything down the chain") — forwarding it down an
unreal pair would notify the wrong channel.

Most drivers ignore it. `main/channel.c:4636` lists it among the conditions with
"Nothing left to do for these", so a driver without an `indicate` callback, or
one that doesn't handle the condition, gets silence rather than the
"Unable to handle indication" warning.

## 6.7 The five callers

| Caller | What it is doing |
|---|---|
| `main/pickup.c:372` | call pickup. Note the direction: `ast_channel_move(target, chan)` makes the *ringing* channel's object the survivor and the channel that ran `Pickup` the zombie, which is the opposite of how the feature reads from the outside |
| `main/bridge_channel.c:1901` | blind transfer: an after-bridge callback moves the departing channel into a target |
| `main/core_local.c:778` | `Local` channel masquerade action, below |
| `res/res_pjsip_refer.c:1937` | SIP `REFER` with `Replaces` |
| `main/channel.c:10826` | `ast_channel_yank`, itself called from `ast_async_goto` and `main/bridge.c:2716` |

The `Local` channel case is the one worth reading, because it shows the
intended shape of a masquerade's aftermath
(`main/core_local.c:774-783`):

```c
		break;
	case LOCAL_CALL_ACTION_MASQUERADE:
		publish_local_bridge_message(p);
		ast_answer(chan);
		res = ast_channel_move(p->action.masq, chan);
		if (!res) {
			/* Chan is now an orphaned zombie.  Destroy it. */
			ast_hangup(chan);
		}
		p->action.masq = ast_channel_unref(p->action.masq);
```

`ast_channel_move(p->action.masq, chan)` followed immediately by
`ast_hangup(chan)`, under the comment `/* Chan is now an orphaned zombie.
Destroy it. */`. Nothing else holds a reference, nothing is going to read it, and
leaving it linked would leave a dead channel visible in `core show channels`.
Compare `main/pickup.c:372` and `main/bridge_channel.c:1901`, which both move
and then let the ordinary hangup path collect the zombie — and
`main/bridge_channel.c:1901-1905`, which on *failure* softhangs up the
*target*, because a failed move can leave the target unusable too.

The clone is nudged toward its own destruction from inside the masquerade:

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

	ast_channel_unlock(original);
	ast_channel_unlock(clonechan);
```

`AST_SOFTHANGUP_DEV` was already set at `:7068`, so `ast_check_hangup` is
already true for it. `AST_FLAG_ZOMBIE` is set *before* the unlock, and the
comment says why: "This must be done before we unlock clonechan to prevent
setting up another masquerade on the clonechan" — it closes the window against
`ast_channel_move`'s own zombie check at `:10859`. The `ast_queue_frame(clonechan,
&ast_null_frame)` is the wake-up for anything blocked reading it.

The tail of the function then reverses the opening:

```c
	/* Now that the operation is complete, we can clear the masq
	 * and masqr fields of both channels.
	 */
	ast_channel_lock_both(original, clonechan);
	ast_channel_masq_set(original, NULL);
	ast_channel_masqr_set(clonechan, NULL);
	ast_channel_unlock(original);
	ast_channel_unlock(clonechan);

	CHANNELSTORAGE_API(current_channel_storage_instance, wrlock);
	CHANNELSTORAGE_API(current_channel_storage_instance, insert, clonechan, 0, 0);
	CHANNELSTORAGE_API(current_channel_storage_instance, insert, original, 0, 0);
	CHANNELSTORAGE_API(current_channel_storage_instance, unlock);

	/* Release our held safety references. */
	ast_channel_unref(original);
	ast_channel_unref(clonechan);
```

`masq`/`masqr` cleared under both locks — releasing any `ast_hangup` spinning in
`CHANNEL_DEADLOCK_AVOIDANCE` — then both channels reinserted into the container
under the container write lock, then the safety references dropped. The
reinsertion is what makes the swapped names and uniqueids visible to the rest of
the system. Note that `ast_channel_unref(clonechan)` at `:7364` is dropping the
*safety* reference taken at `:6938`, not the last one; the zombie survives until
its real owner hangs it up.

Between `:7262` (unlock) and `:7332` (relock) the function also notifies the
bridging layer (`ast_bridge_notify_masquerade` at `:7278`, which reconfigures the
bridge and carries its own `XXX ASTERISK-22366 this needs more work` about format
renegotiation), re-simulates an `UNHOLD` if the clone was on hold, completes a
half-sent DTMF digit through `ast_channel_end_dtmf(..., "masquerade")`, restores
the visible indication captured back at `:6961`, and restarts MOH. Those are all
things that are *state in the driver* rather than state in the struct, and so
have to be replayed rather than copied.

## 6.8 Common bugs

| Symptom | Cause |
|---|---|
| Driver works until the first transfer or pickup, then media stops | `tech->fixup` missing or not repointing the owner pointer. The call leg moved; the driver is still queueing to the zombie |
| `Channel type 'X' could not fixup channel Y, strange things may happen` | the driver's fixup returned non-zero, usually its `oldchan != p->owner` validation. The masquerade continued anyway |
| Crash in driver code just after a transfer, on a freed channel | driver cached the channel pointer without a reference, or with a reference but no fixup |
| Stale uniqueid in CDR/CEL/RTP statistics after a transfer | something cached `ast_channel_uniqueid` rather than re-reading it; it was swapped at `:6996` |
| A channel briefly invisible to `core show channels` / AMI / CLI | it is between `:6945` (container remove) and `:7358` (reinsert) |
| `ast_assert` on `linked_in_container` | something tried to swap uniqueids on a linked channel; the container's hash is now wrong |
| Hangup appears to hang | an `ast_hangup` is spinning in the `masq`/`masqr` loop at `:2554`, waiting for a masquerade to finish. Look for a stuck driver fixup or a `MASQUERADE_NOTIFY` handler that never returns |
| Transfers serialise system-wide under load | `channel_move_lock` (`:10844`) is process-global |
| Deadlock involving a driver's own lock during transfer | a fixup callback takes the driver lock while the masquerading thread holds both channel locks. `chan_pjsip` avoids it by suspending its serializer in `MASQUERADE_NOTIFY` instead |
| Frames queued before a transfer arrive after frames queued later | by design — the readq merge at `:7046-7047` puts the clone's frames first |
| A timing-fd generator misbehaves immediately after a transfer | between `:7099` and `:7207` the surviving channel's `AST_TIMING_FD` slot holds the clone's timer fd |

## 6.9 Rules

| Rule | Why |
|---|---|
| If your tech_pvt holds a channel pointer, implement `.fixup` | nothing else will repoint it, and the tech_pvt is swapped at `:7024-7026` |
| In `.fixup`, validate that `oldchan` is the channel you think you own before repointing | every correct implementation in the tree does; see `chan_pjsip.c:1159-1161` and `core_unreal.c:381-385` |
| Never hold a channel lock when calling `ast_channel_move` | documented on both `ast_channel_move` and `ast_channel_yank`: "absolutely _NO_ channel locks should be held" |
| Never use the source channel after a successful `ast_channel_move` | "the source channel is in a state where its continued use is unreliable". Use the dest |
| Hang up the source yourself if nothing else will | `core_local.c:779-781` is the model |
| Do not use `ast_channel_move` on bridged channels | use `ast_bridge_join`, `ast_bridge_impart` or `ast_bridge_move`; the header says so twice |
| Cache uniqueids and names only if you have a fixup | both are swapped. Caching the *pointer* is worse |
| Prefer a datastore with `chan_fixup` to a raw pointer | it is the only mechanism that gets called for inherited state |
| Use `MASQUERADE_NOTIFY`, not `.fixup`, to quiesce a driver thread | `.fixup` runs with both channel locks held; the notify runs without them and may drop the lock |
| Do not assume a masquerade succeeded | `channel_do_masquerade` returns `void` and logs its failures |
| Expect to be masqueraded | any channel reachable by pickup, blind or attended transfer, `REFER` or `ast_async_goto` can be, at any time, from another thread |

## Next

[Chapter 07](README.md) covers `struct ast_channel_tech` in full — what a driver
must implement, and the `ast_request`/`ast_call`/`ast_answer` path that creates
the channels this chapter has been shuffling. `.fixup` is one entry in a table of
thirty-odd, and it is not the only one with a non-obvious contract.
