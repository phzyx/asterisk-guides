# 02 — Lifecycle: allocation, linking, hangup, destruction

A channel has four distinct phases, and most channel bugs are a confusion between
two of them:

```
  alloc        finalize + link         hangup              destructor
    │                 │                   │                     │
    ▼                 ▼                   ▼                     ▼
 ┌──────┐      ┌─────────────┐     ┌─────────────┐      ┌──────────────┐
 │ raw  │─────▶│   live      │────▶│  unlinked   │─────▶│   freed      │
 │      │      │             │     │             │      │              │
 │ ao2  │      │ finalized=1 │     │ ZOMBIE set  │      │ ao2 refcount │
 │ obj  │      │ in storage  │     │ out of      │      │ hit zero     │
 │ only │      │ publishes   │     │ storage,    │      │              │
 │      │      │ snapshots   │     │ refs may    │      │              │
 │      │      │             │     │ still exist │      │              │
 └──────┘      └─────────────┘     └─────────────┘      └──────────────┘
```

The two that get conflated are the third and the fourth. `ast_hangup()` returns
having torn the channel down and dropped *its* reference — but if anyone else
still holds one, the object is still there, with a dead tech and no storage entry.
The destructor runs later, on whichever thread drops the last reference.

## 2.1 The entry points

| Function | `main/channel.c` | Destructor installed | Linked into storage |
|---|---|---|---|
| `__ast_channel_alloc` | `:967` | `ast_channel_destructor` | yes |
| `__ast_channel_alloc_with_initializers` | `:986` | `ast_channel_destructor` | yes |
| `__ast_dummy_channel_alloc` | `:1007` | `ast_dummy_channel_destructor` | **no** |

The first two are thin `va_list` wrappers over `__ast_channel_alloc_ap`
(`:746-965`); the public names you actually call are macros at
`include/asterisk/channel.h:1299-1309` and `:1328`, which inject `__FILE__`,
`__LINE__` and the enclosing function name so `REF_DEBUG` can report the
allocation site.

Allocation happens in two stages, in two different files, and the split matters:
`channel_internal_api.c` builds the object, `channel.c` makes it a channel.

## 2.2 Stage one — the object (`channel_internal_api.c:1242`)

`__ast_channel_internal_alloc_with_initializers` is the only place in the tree
that calls `__ao2_alloc` for a channel:

```c
	tmp = __ao2_alloc(sizeof(*tmp), destructor,
		AO2_ALLOC_OPT_LOCK_MUTEX, "", file, line, function);
```

`AO2_ALLOC_OPT_LOCK_MUTEX` is why `ast_channel_lock()` works at all — the channel
lock *is* the ao2 object lock (`ast_channel_lock(chan)` is `#define`d to
`ao2_lock(chan)`, `include/asterisk/channel.h:2983`), so there is no separate
channel mutex to find.

It is a **recursive** mutex, and not because of `DEBUG_THREADS`. An
`AO2_ALLOC_OPT_LOCK_MUTEX` object is initialised with `ast_mutex_init`
(`main/astobj2.c:702`), which sets `AST_MUTEX_KIND` (`main/lock.c:170`), which is
`PTHREAD_MUTEX_RECURSIVE_NP` or `PTHREAD_MUTEX_RECURSIVE` depending on platform
(`include/asterisk/lock.h:82-88`) — unconditionally. So nested
`ast_channel_lock()` on the same thread is legal and is relied on throughout the
tree. It is not a read/write lock, so there is no shared-read mode: every reader
of a mutable field serialises against every other.

Three things get built here that the destructor will later unbuild:

- the string field pool, `ast_string_field_init(tmp, 128)` — 128 bytes to start,
  grown on demand
- `dialed_causes`, an ao2 hash container of per-destination hangup causes
- the fd vector and the ARI variable vector:

```c
	AST_VECTOR_INIT(&tmp->fds, AST_MAX_FDS);
	AST_VECTOR_INIT(&tmp->ari_report_vars, 8);

	/* Force all channel snapshot segments to be created on first use, so we don't have to check if
	 * an old snapshot exists.
	 */
	ast_set_flag(&tmp->snapshot_segment_flags, AST_FLAGS_ALL);
```

Setting every invalidation flag at birth is the trick that lets the snapshot code
in §1.5 skip a null check: the first snapshot build always finds every segment
stale, so there is never a "no previous snapshot" special case.

Note the failure style throughout both stages:

```c
	if ((ast_string_field_init(tmp, 128))) {
		return ast_channel_unref(tmp);
	}
```

An allocation failure does not `ast_free` anything — it drops the reference and
lets the destructor clean up whatever was built so far. This is why the destructor
has to tolerate a half-built channel, and why `ast_channel_destructor` checks
`ast_channel_internal_is_finalized()` twice before touching anything that assumes
a complete channel.

### Where `uniqueid` comes from

```c
	/* set the creation time in the uniqueid */
	tmp->uniqueid.creation_time = time(NULL);
	tmp->uniqueid.creation_unique = ast_atomic_fetchadd_int(&uniqueint, 1);
```

The printable id is then either the caller's assigned one, or
`"%li.%d"` of time and counter, or `"%s-%li.%d"` prefixed with
`ast_config_AST_SYSTEM_NAME` when one is configured (`:1282-1294`). The
`creation_time`/`creation_unique` pair is kept as structured data rather than
being parsed back out of the string, because linkedid propagation compares it:
`ast_channel_internal_oldest_linkedid` (`:1326-1341`) compares `creation_time`,
then `creation_unique`, and returns the older channel.

`linkedid` is inherited from the requestor if there is one, and otherwise seeded
from this channel's own uniqueid:

```c
	/* copy linked id from parent channel if known */
	if (requestor) {
		tmp->linkedid = requestor->linkedid;
	} else {
		tmp->linkedid = tmp->uniqueid;
	}
```

Tenant id is copied *after* that, with a comment explaining exactly why
(`:1303-1307`) — it lives inside `struct ast_channel_id`, so inheriting the
linkedid would otherwise clobber it.

### The initializers ABI check

`__ast_channel_alloc_with_initializers` exists so a driver can pass fields that
have to be set before linkedid inheritance. Because the struct crosses the module
boundary, it is version-stamped, and a mismatch is a hard failure rather than a
silent misread (`:1264-1276`): `version == 0` and
`version != AST_CHANNEL_INITIALIZERS_VERSION` both log an error and unref.

## 2.3 Stage two — the channel (`channel.c:746`)

Four things are worth reading closely in `__ast_channel_alloc_ap`.

**It refuses to run during shutdown.** `ast_shutting_down()` is checked before
anything is allocated (`:763-766`), so a module teardown racing a new call gets
`NULL` rather than a channel nobody will ever hang up.

**The whole function is bracketed by a snapshot stage.**
`ast_channel_stage_snapshot(tmp)` at `:775` and
`ast_channel_stage_snapshot_done(tmp)` at `:960`. Those two are trivially small
(`main/stasis_channels.c:1066-1075`):

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

So the thirty-odd setter calls in between — any of which might publish, per §1.5 —
collapse into exactly one published snapshot at the end. This pattern is worth
copying anywhere you set several channel fields in a row.

**File descriptors are initialised to a sentinel, not zero.**

```c
	/*
	 * Init file descriptors to unopened state so
	 * the destructor can know not to close them.
	 */
	ast_channel_timingfd_set(tmp, -1);
	ast_channel_internal_alertpipe_clear(tmp);
	ast_channel_internal_fd_clear_all(tmp);
```

A zeroed `alertpipe[2]` would be fd 0, and `ast_alertpipe_close` would close
stdin. Note that the three calls do different things:
`ast_channel_internal_alertpipe_clear` and `ast_channel_internal_fd_clear` write
`-1`, but `ast_channel_internal_fd_clear_all`
(`main/channel_internal_api.c:1049-1052`) just resets the vector to length zero —
the generic fds are never closed by the destructor at all, only the alertpipe and
the timer are.

**The tech is looked up from the name.** This is the surprising one
(`:872-928`): the channel's name is built from the caller's format string, the
leading `tech/` component is split off it, and the result is handed to
`ast_get_channel_tech()` (`:593`). If the first component does not resolve and
the name had a second `/`, the second component is tried too (`:883-889` splits
both, `:921-923` retries). If neither matches, the channel gets `&null_tech`
(`:703-706`), whose description is "Null channel (should not see this)". So the
channel driver a channel ends up pointing at is a *consequence of its name* — a
typo in the name format gives you a channel with no working tech rather than a
compile or runtime error.

### The linking sequence

The ordering at the end of allocation is deliberate and is the model for how to
publish a new object safely:

```c
	/* You might scream "locking inversion" at seeing this but it is actually perfectly fine.
	 * Since the channel was just created nothing can know about it yet or even acquire it.
	 */
	ast_channel_lock(tmp);

	CHANNELSTORAGE_API(current_channel_storage_instance, wrlock);

	if (do_ids_conflict(assignedids, 0)) {
		ast_channel_internal_errno_set(AST_CHANNEL_ERROR_ID_EXISTS);
		ast_channel_unlock(tmp);
		CHANNELSTORAGE_API(current_channel_storage_instance, unlock);
		/* See earlier channel creation abort comment above. */
		return ast_channel_unref(tmp);
	}

	/* Finalize and link into the channels container. */
	ast_channel_internal_finalize(tmp);
	ast_atomic_fetchadd_int(&chancount, +1);
	CHANNELSTORAGE_API(current_channel_storage_instance, insert, tmp, OBJ_NOLOCK, 0);
	CHANNELSTORAGE_API(current_channel_storage_instance, unlock);
```

Channel lock before container lock is the *opposite* of the ordering the rest of
the tree must observe (chapter 03), which is why the comment is there. The
uniqueid conflict check and the insert happen under one continuous hold of the
container write lock, so two threads cannot both pass the check and then both
insert the same assigned id.

### Allocation returns the channel **locked**

This is the single easiest thing to get wrong in this chapter. Read the block
above again: `ast_channel_lock(tmp)` at `:933` is released on the error path at
`:939`, and *nowhere else*. The success path at `:964` returns with the channel
lock still held.

```c
	ast_debug(1, "Channel %p '%s' allocated\n", tmp, ast_channel_name(tmp));

	return tmp;
}
```

It is documented, in the header rather than at the implementation —
`include/asterisk/channel.h:1256` and `:1297` both carry the note *"Since 12.0.0
this function returns with the newly created channel locked."* Every caller of
`ast_channel_alloc` owes an `ast_channel_unlock`, and a driver's `requester`
callback that returns the channel to `ast_request` is handing back a locked
channel by design. If you write a new channel driver and your first call deadlocks
against itself on an rwlock-shaped assumption, this is why — though since the
channel lock is recursive (§2.2), a nested lock will not deadlock, it will just
leak a lock level.

The dummy allocator is the exception: `__ast_dummy_channel_alloc` never takes the
lock, so it returns unlocked.

Note also that `chancount` (`:97`) is incremented here and decremented in the
destructor at `:2320` — it counts *allocated* channels, which is why
`ast_undestroyed_channels()` (`:505-507`) reads it while `ast_active_channels()`
(`:500-503`) asks the storage container instead. A gap between the two numbers
means channels have been unlinked but not yet freed. A *transient* gap is normal:
`ast_hangup` unlinks at `:2573` and does not unref until `:2617`, so every hangup
in flight contributes one, as does every `RAII_VAR` still holding an unlinked
channel. A gap that does not drain is the signature of a leaked reference.

## 2.4 `finalized` is not a lock, it is a publication gate

```c
void ast_channel_internal_finalize(struct ast_channel *chan)
{
	chan->finalized = 1;
}

int ast_channel_internal_is_finalized(struct ast_channel *chan)
{
	return chan->finalized;
}
```

`main/channel_internal_api.c:1449-1457`. Two independent gates guard snapshot
publication, and they mean different things:

| Gate | Meaning | Set/cleared |
|---|---|---|
| `finalized` | "this channel is real; the world may be told about it" | set once at `channel.c:946`, never cleared |
| `AST_FLAG_SNAPSHOT_STAGE` | "I am mid-edit; batch my publishes" | set/cleared around any multi-field edit |

A setter publishes only if `finalized` is set (§1.5) and the stage flag is clear.
The destructor relies on both: it asserts the stage flag is *not* set before
publishing the final snapshot, because a channel being destroyed mid-stage means
somebody returned without calling `ast_channel_stage_snapshot_done`.

The two gates are not enforced in the same place, and this matters. The
`AST_FLAG_SNAPSHOT_STAGE` check lives inside `ast_channel_publish_snapshot`
itself (`main/stasis_channels.c:1092-1094`), so it applies to every caller. The
`finalized` check lives only in the *setter macros*
(`main/channel_internal_api.c:91`, `:98`), so it only suppresses
*setter-driven* publication. Code that calls `ast_channel_publish_snapshot` or
`ast_channel_publish_blob` directly is not gated by `finalized` at all.

## 2.5 Channel storage holds a reference

The global channel container is behind a driver interface rather than being a
bare `ao2_container`. `main/channelstorage.h:72-73` is the whole dispatch
mechanism:

```c
#define CHANNELSTORAGE_API(_instance, _func, ...) \
	(_instance)->_func((_instance), ##__VA_ARGS__)
```

Drivers self-register from a constructor at load:

```c
static struct ast_channelstorage_driver driver_type = {
	.driver_name = "ao2_legacy",
	.open_instance = get_instance,
};

static void __attribute__((constructor)) __startup(void)
{
	ast_channelstorage_register_driver(&driver_type);
}
```

`main/channelstorage_ao2_legacy.c:387-395`. The default is `"ao2_legacy"`
(`main/channelstorage.h:30`), and a second backend,
`main/channelstorage_cpp_map_name_id.cc`, is available but off by default — it
needs a C++11 compiler and is selected through menuselect
(`main/channelstorage_makeopts.xml`). The driver cannot be changed after startup:
`internal_channel_set_current_storage_driver` (`channel.c:8155`) refuses once
`ast_fully_booted` is set, and an unknown name falls back to the default with a
warning rather than failing the boot.

For the default backend, linking is one `ao2_link_flags` plus one flag:

```c
static int insert_channel(struct ast_channelstorage_instance *driver,
	struct ast_channel *chan, int flags, int lock)
{
	int ret = ao2_link_flags(getdb(driver), chan, flags);
	if (ret == 1) {
		chan->linked_in_container = 1;
	}
	return ret ? 0 : -1;
}
```

`ao2_link_flags` takes a reference. **The container is a reference holder**, and
that is the reference `ast_channel_unlink` drops. This is the single most
important fact in this chapter: as long as a channel is in storage, it cannot be
freed, no matter how many threads have finished with it.

`linked_in_container` is what makes the name and uniqueid assertions in §1.6 and
in `ast_channel_internal_swap_uniqueid_and_linkedid` (`:1368`) possible —
anything that would change a storage key asserts that the channel is not
currently keyed on it.

## 2.6 Who holds a reference

A live channel typically has several, and they are not symmetric:

| Holder | Released by |
|---|---|
| The channel storage container | `ast_channel_unlink()` — called from `ast_hangup` |
| The thread that allocated it (the PBX thread, a driver's monitor thread, `app_dial`) | `ast_hangup()`, which unrefs at the end |
| `RAII_VAR`/`ast_channel_cleanup` locals all over the tree | scope exit |
| Datastores holding a channel pointer (`app_chanspy`, autochans) | datastore destruction |
| An iterator mid-traversal | `ast_channel_iterator_destroy` |

A bridge is **not** on that list, which surprises people.
`bridge_channel->chan` is a borrowed pointer: it is a plain assignment with no
`ao2_bump` (`main/bridge.c:1860` on join, `:2018` on impart) and it is nulled
without an unref. `ast_bridge_depart` drops a `bridge_channel` reference
(`:2146`), not a channel reference. What actually happens on an independent
impart is that the *imparting thread's* reference is transferred to the bridge
thread, which ends by calling `ast_hangup` itself
(`bridge_channel_ind_thread`, `:1941-1975`). So the holder is a thread, not the
bridge object.

The destructor comments on the datastore case directly — it locks and unlocks the
channel for no other reason than to wait out a holder:

```c
	/* Lock and unlock the channel just to be sure nobody has it locked still
	   due to a reference that was stored in a datastore. (i.e. app_chanspy) */
	ast_channel_lock(chan);
	ast_channel_unlock(chan);
```

## 2.7 Soft hangup is a request; hard hangup is the teardown

These are completely different operations and the names do not help.

`ast_softhangup_nolock` (`:2449-2459`) does not tear anything down. It sets a
flag and makes sure the owning thread notices:

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

Three steps, each one necessary: set the reason, queue a null frame so a reader
wakes up, and signal `SIGURG` to the blocking thread so a thread stuck in `poll`
returns `EINTR`. `blocker` and the `AST_FLAG_BLOCKING` flag from §1.3 are what
make that last step possible.

The channel's own thread then discovers the request through `ast_check_hangup`
(`:446-458`), which also folds in the scheduled-hangup case:

```c
int ast_check_hangup(struct ast_channel *chan)
{
	if (ast_channel_softhangup_internal_flag(chan))		/* yes if soft hangup flag set */
		return 1;
	if (ast_tvzero(*ast_channel_whentohangup(chan)))	/* no if no hangup scheduled */
		return 0;
	if (ast_tvdiff_ms(*ast_channel_whentohangup(chan), ast_tvnow()) > 0)		/* no if hangup time has not come yet. */
		return 0;
```

Note what that check *does*: it converts an expired deadline into an
`AST_SOFTHANGUP_TIMEOUT` flag. The deadline is not a scheduler entry. There are
two independent places that notice it, and `ast_check_hangup` is the lazier of
them — the primary enforcer is the wait path, `ast_waitfor_nandfds`:

- a pre-poll sweep over the channels being waited on, which returns an
  already-expired channel immediately after flagging it (`:3016-3034`)
- the poll timeout itself, clamped down to the nearest `whentohangup`
  (`:3046-3054`)
- a post-poll sweep that flags every channel whose deadline passed (`:3106-3117`)

And `ast_channel_setwhentohangup_tv` queues a null frame when it sets the
deadline (`:519`), so a thread sitting in `ast_waitfor` wakes up and recomputes
its timeout rather than sleeping past the new deadline. So the deadline is
actively enforced for any channel somebody is waiting on; `ast_check_hangup` is
the backstop for channels nobody is.

The rule of thumb is nonetheless: **soft hangup from any thread; hard hangup only
from the thread that owns the channel.** That is a convention rather than
something the source states — `ast_hangup`'s own doxygen
(`include/asterisk/channel.h:1630-1638`) says only that no channel locks may be
held and that `chan` is invalid afterwards — but §2.8 step 7 is the diagnostic
that fires when it is broken.

The flag catalog (`AST_SOFTHANGUP_DEV` at `include/asterisk/channel.h:1141`
through `AST_SOFTHANGUP_HANGUP_EXEC` at `:1174`, plus `AST_SOFTHANGUP_ALL` at
`:1181`) is chapter 04's subject. Worth noticing in passing that `(1 << 6)` is
unused — the bits are not contiguous, so do not write a loop over them.

## 2.8 `ast_hangup` step by step

`main/channel.c:2540-2618`. In order:

1. **NULL-tolerant return** — explicitly, so it can be used as an `RAII_VAR`
   destructor.
2. **`ast_autoservice_stop(chan)`** — before taking the lock. A channel in
   autoservice is being read by the autoservice thread; it has to stop owning it
   first.
3. **Wait out any masquerade:**

   ```c
   	while (ast_channel_masq(chan) || ast_channel_masqr(chan))  {
   		CHANNEL_DEADLOCK_AVOIDANCE(chan);
   	}
   ```

   You cannot hang up a channel that is half-swapped with another. This loop
   drops and retakes the lock each pass (chapter 03), which is also why it is a
   loop and not an `if`.
4. **Set `AST_FLAG_ZOMBIE`** — with the comment "Mark as a zombie so a masquerade
   cannot be setup on this channel." Zombie is a one-way door; it is what makes
   step 3 terminate for everyone else.
5. **Run hangup handlers, then `ast_channel_unlink(chan)`** — *outside* the
   channel lock. From this point the channel is no longer findable by name or
   uniqueid, and the container's reference is gone.
6. **Retake the lock and tear down:** `destroy_hooks` (audiohooks detached,
   framehook list destroyed), `free_translation`, close the audio and video
   `ast_filestream`s, destroy the scheduler context, release any generator.
7. **Complain loudly if the channel is still blocked:**

   ```c
   	if (ast_test_flag(ast_channel_flags(chan), AST_FLAG_BLOCKING)) {
   		ast_log(LOG_WARNING, "Hard hangup called by thread LWP %d on %s, while blocked by thread LWP %d in procedure %s!  Expect a failure\n",
   			ast_get_tid(), ast_channel_name(chan), ast_channel_blocker_tid(chan),
   			ast_channel_blockproc(chan));
   		ast_assert(0);
   	}
   ```

   This is the diagnostic for breaking the rule in §2.7, and one of the three
   consumers of `blockproc` (§1.3; the others are `CHECK_BLOCKING`'s own
   double-block warning at `include/asterisk/channel.h:2936-2938` and
   `app_dumpchan`). Strictly it detects that *some* thread is mid-blocking-call
   on this channel, which in practice means `ast_hangup` was called from a thread
   that did not own the channel — but that is an inference, not what the check
   proves.
8. **`ast_channel_tech(chan)->hangup(chan)`** — the driver's only chance to clean
   up `tech_pvt`. Called with the channel locked.
9. **Unlock, `ast_cc_offer(chan)`, `ast_channel_unref(chan)`.**

Step 9 is the one to remember. `ast_hangup` consumes the caller's reference. The
pattern `ast_hangup(chan); ast_channel_unref(chan);` is a double free, and
`chan` is not safe to touch after the call.

## 2.9 The destructor

`ast_channel_destructor` (`:2181-2320`) is entered with **no channel lock held**
— the refcount reached zero — and takes the lock itself three times. Two of those
(`:2207` and `:2212`) are because the code they wrap expects to run locked. The
third is not a lock at all but a barrier, and the comment says so (`:2227-2230`,
quoted in §2.6): a datastore holder may still have the lock even though its
reference is gone, so the destructor acquires and immediately releases it to wait
that out.

The order is load-bearing in two places. First:

```c
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

`AST_FLAG_DEAD` is set *before* the final snapshot, and that ordering is the
whole point: the flag travels *in* the snapshot, and it is how the rest of
Asterisk learns the channel is gone. CDR, CEL and AMI all detect hangup by
testing `AST_FLAG_DEAD` on the new snapshot of a channel snapshot update —
`main/cdr.c:2466` finalises and dispatches every CDR on it, `main/cel.c:956-975`
raises `AST_CEL_HANGUP` and `AST_CEL_CHANNEL_END` on the
not-hungup→hungup transition, and `main/manager_channels.c:645-666` raises the
AMI `Hangup` event the same way. So the destructor's final snapshot is not a
courtesy; it is the trigger for every terminal record in the system.

It is also the last message on the channel's own topic — the forwards are
cancelled shortly afterwards in `ast_channel_internal_cleanup`. It is not quite
the last Stasis traffic the destructor causes: the device-state event below
reaches `ast_publish_device_state` on the device-state topic.

Second, the warnings. The destructor is where leaks are reported, not prevented:

- `tech_pvt` still set → `"Channel '%s' may not have been hung up properly"`, and
  then it `ast_free`s the pointer, which is a guess about how it was allocated
- `pbx` still set → `"PBX may not have been terminated properly on '%s'"`

Both mean a step in §2.8 was skipped — usually because the last reference was
dropped by something other than `ast_hangup`.

The rest is the mirror image of allocation: datastores freed, callid cleaned up,
scheduler destroyed, translators freed, all five format references dropped, the
five party structures freed, the alertpipe closed (unconditionally at the call
site — the `-1` sentinel from §2.3 is checked inside `ast_alertpipe_close`,
`main/alertpipe.c:83-99`) and the timer closed if non-NULL, the read queue
drained with `ast_frfree`, channel
variables deleted, group memberships discarded, jitterbuffer destroyed, CDR freed,
tone zone unreffed, then `ast_channel_internal_cleanup` for the pieces stage one
built.

One subtlety: the device name for the final device-state event is extracted
*early*, by truncating the channel name at the last `-`, and only if the channel
was finalized. The event itself is queued near the end, after
`ast_channel_internal_cleanup` has already freed the string fields — so the name
had to be copied out while it still existed. The state queued is
`AST_DEVICE_UNKNOWN`, deliberately, with a comment explaining that this instance
is dead but the device's other instances are not this code's business.

### `ast_channel_internal_cleanup`

`channel_internal_api.c:1426-1447`: drops `dialed_causes`, frees the string field
pool, **cancels both Stasis forwards**, releases the topic, clears the stream
topology, and frees the two vectors. The forward cancellation is the part that
matters to anyone writing a subsystem: a channel's topic stops forwarding to the
all-channels topic here, so a subscriber that has not seen the final snapshot by
now never will.

## 2.10 Dummy channels

`__ast_dummy_channel_alloc` (`:1007-1039`) exists for one purpose: somewhere to
evaluate channel variables and dialplan functions when there is no call. It is a
real ao2 channel object with a different destructor, and it is deliberately
missing almost everything:

- no formats, no translators, no scheduler, no timer, no stream topology
- **not linked into channel storage**, so `ast_channel_get_by_name` will never
  find it and `finalized` is never set
- fds initialised to the unopened sentinel anyway, with the comment "just in case
  autoservice is called on the channel or something tries to read a frame from
  it"

`ast_dummy_channel_destructor` (`:2324-2357`) is correspondingly short.

It is tempting to conclude that a dummy channel is Stasis-silent, and that is
not true. `finalized` gates only the *setter* macros (§2.4), so no field
assignment on a dummy channel publishes a snapshot — but a direct
`ast_channel_publish_*` call is not gated at all. The obvious case is the one
dummy channels exist for: setting a channel variable reaches
`ast_channel_publish_varset` for any non-NULL channel
(`main/pbx_variables.c:1278`, `:1281`), which publishes a VarSet blob. So a dummy
channel emits no snapshots and no lifecycle events, but it is not silent.

## 2.11 `ast_channel_release` vs `ast_hangup`

```c
struct ast_channel *ast_channel_release(struct ast_channel *chan)
{
	/* Safe, even if already unlinked. */
	ast_channel_unlink(chan);
	return ast_channel_unref(chan);
}
```

`:1571-1576`. This is `ast_hangup` with everything interesting removed: no
autoservice stop, no masquerade wait, no zombie flag, no hook destruction, no
`tech->hangup`. It is for channels that were allocated but never became calls —
you built one, something failed, and no driver state was ever attached. Using it
on a live channel skips the driver's teardown entirely, which is how you get the
`"may not have been hung up properly"` warning.

`ast_channel_unlink` itself is one line (`:10709-10712`): the storage `remove`
call, which drops the container's reference and clears `linked_in_container`.

## 2.12 Rules

| Rule | Why |
|---|---|
| Unlock the channel that `ast_channel_alloc` handed you | It returns locked, and only the header says so (§2.3) |
| After `ast_hangup(chan)`, treat `chan` as freed | `ast_hangup` consumes your reference (§2.8 step 9) |
| Soft-hangup across threads; hard-hangup only from the owning thread | The hard path asserts when another thread is blocking on the channel (§2.7, §2.8 step 7) |
| Bracket multi-field edits in `ast_channel_stage_snapshot`/`_done` | One snapshot instead of N, and the destructor asserts if you forget the `_done` (§2.3, §2.4) |
| Never `ast_free` a channel, even on an allocation failure path | Unref it; the destructor is written to handle a half-built channel (§2.2) |
| Treat a *persistent* gap between `ast_undestroyed_channels()` and `ast_active_channels()` as a leaked reference | A transient gap is just hangups in flight (§2.3) |
| Do not use `ast_channel_release` on a channel that ever had a tech | It skips `tech->hangup` (§2.11) |
| Do not rely on `finalized` to keep a dummy channel off the bus | It gates setters only; direct publishes still go out (§2.4, §2.10) |
| Do not assume a `whentohangup` deadline needs polling to take effect | `ast_waitfor_nandfds` enforces it; `ast_check_hangup` is the backstop (§2.7) |
