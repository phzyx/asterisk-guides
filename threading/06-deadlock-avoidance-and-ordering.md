# 6. Deadlock avoidance and ordering

Every `file:line` citation in this guide was verified against the `24` branch.
Newer releases and other branches may differ — line numbers drift — so if you
are reading this against a different tree, treat the citations as approximate
and re-grep for the symbol name.

Everything so far has been about acquiring one lock. This chapter is about the
case where a thread already holds one lock and wants a second, and the order it
wants them in is not the order the rest of the tree agrees on. The tree's answer
is a family of macros in `lock.h` with *DEADLOCK_AVOIDANCE* in the name, and the
answer they implement is not avoidance in the sense of a protocol that makes
deadlock impossible. It is: let go of what you are holding, sleep for a moment,
and take it again.

That is a weaker guarantee than it sounds, and the whole of this chapter follows
from it. Letting go means the invariants that justified holding the lock are
void when you get it back — so every call site has to re-check its world
afterwards, and the macros cannot help with that. Sleeping for a moment means
there is no bound on how long the loop runs. And taking it again means the lock
tracking of chapter 05 sees a brand-new acquisition, which is where most of the
observable strangeness in this chapter comes from.

There are five macro names, each defined twice, and the two halves of the
`#ifdef DEBUG_THREADS` split do not merely differ in instrumentation: for four
of the five the *syntax class* differs between builds. There are 35 call sites in
the whole tree, almost all of them in channel drivers, and reading them is the
only way to recover the lock-ordering rules, because the tree does not document
them in one place.

## 6.1 Five names, ten definitions

The region is `include/asterisk/lock.h:339-499`, which spans both the `#ifdef
DEBUG_THREADS` block that opens at `:251` and the `#else` arm at `:474`. The
five names are `AO2_DEADLOCK_AVOIDANCE` (`:339` debug, `:476` production),
`CHANNEL_DEADLOCK_AVOIDANCE` (`:361`, `:481`), `DEADLOCK_AVOIDANCE` (`:383`,
`:486`), `DLA_UNLOCK` (`:417`, `:497`) and `DLA_LOCK` (`:436`, `:499`).

The debug definitions are 21, 21, 21, 6 and 15 lines. The production
definitions are 4, 4, 10, 1 and 1. That ratio is the chapter in miniature: what
the three `*DEADLOCK_AVOIDANCE` macros do in a production build is unlock, sleep
and relock, and what they do in a debug build is unlock, sleep and relock
wrapped in bookkeeping that exists for one reason, which §6.3 covers.

Take the first one whole, because the other three are variations on it:

```c
#define AO2_DEADLOCK_AVOIDANCE(obj) \
	do { \
		char __filename[80], __func[80], __mutex_name[80]; \
		int __lineno; \
		int __res = ast_find_lock_info(ao2_object_get_lockaddr(obj), __filename, sizeof(__filename), &__lineno, __func, sizeof(__func), __mutex_name, sizeof(__mutex_name)); \
		int __res2 = ao2_unlock(obj); \
		usleep(1); \
		if (__res < 0) { /* Could happen if the ao2 object does not have a mutex. */ \
			if (__res2) { \
				ast_log(LOG_WARNING, "Could not unlock ao2 object '%s': %s and no lock info found!  I will NOT try to relock.\n", #obj, strerror(__res2)); \
			} else { \
				ao2_lock(obj); \
			} \
		} else { \
			if (__res2) { \
				ast_log(LOG_WARNING, "Could not unlock ao2 object '%s': %s.  {{{Originally locked at %s line %d: (%s) '%s'}}}  I will NOT try to relock.\n", #obj, strerror(__res2), __filename, __lineno, __func, __mutex_name); \
			} else { \
				__ao2_lock(obj, AO2_LOCK_REQ_MUTEX, __filename, __func, __lineno, __mutex_name); \
			} \
		} \
	} while (0)
```

— `include/asterisk/lock.h:339-359`

Six things to notice, because each of them recurs. The macro declares three
80-byte buffers and three `int`s (`:341-344`), all with names beginning `__`,
which is why nesting two of these shadows them. `__res` records whether the lock
could be *found* in this thread's registry, before anything is unlocked
(`:343`). `__res2` records whether the unlock succeeded (`:344`). The sleep is a
literal `usleep(1)` and happens before either result is examined (`:345`). The
four-way branch on `__res` and `__res2` has exactly one arm that relocks with
preserved information (`:356`) and two arms that log and return with the lock
*not* held (`:348`, `:354`). And the comment at `:346` is the only commentary in
the expansion.

`CHANNEL_DEADLOCK_AVOIDANCE` at `:361-381` is the same macro with
`ast_channel_unlock`/`ast_channel_lock` substituted for the `ao2_` pair, the
word *channel* in the messages, and a different comment on the miss branch.
Since `ast_channel_lock` is literally `ao2_lock`:

```c
#define ast_channel_lock(chan) ao2_lock(chan)
#define ast_channel_unlock(chan) ao2_unlock(chan)
#define ast_channel_trylock(chan) ao2_trylock(chan)
```

— `include/asterisk/channel.h:2983-2985`

the two macros are the same code reached through different spellings. In
particular the relock at `:378`, `__ao2_lock(chan, AO2_LOCK_REQ_MUTEX, …)`, is
not a bypass of `ast_channel_lock` — it *is* `ast_channel_lock`, expanded by
hand so that the location arguments can be supplied explicitly instead of coming
from `__FILE__` and `__LINE__`.

## 6.2 The shape of the call sites

Most of the 35 call sites are one of two shapes. The common one is a retry
loop: hold lock A, try for lock B without blocking, and if the attempt fails,
run deadlock avoidance on A — the lock you are holding, never the one you
want.

```c
		if (!ast_channel_trylock(pvt->subs[sub_idx].owner)) {
			/* Got subchannel owner lock */
			break;
		}
		/* We must unlock the private to avoid the possibility of a deadlock */
		DEADLOCK_AVOIDANCE(&pvt->lock);
	}
```

— `channels/chan_dahdi.c:3785-3791`

That is `dahdi_lock_sub_owner`, and `ast_autochan_channel_lock` is the same
thing in a header:

```c
#define ast_autochan_channel_lock(autochan) \
	do { \
		ast_mutex_lock(&(autochan)->lock); \
		while (ast_channel_trylock((autochan)->chan)) { \
			DEADLOCK_AVOIDANCE(&(autochan)->lock); \
		} \
		ast_mutex_unlock(&(autochan)->lock); \
	} while (0)
```

— `include/asterisk/autochan.h:75-82`

as is `topic_lock_both`, the one and only user of `AO2_DEADLOCK_AVOIDANCE` in
the tree:

```c
#define topic_lock_both(topic1, topic2) \
	do { \
		ao2_lock(topic1); \
		while (ao2_trylock(topic2)) { \
			AO2_DEADLOCK_AVOIDANCE(topic1); \
		} \
	} while (0)
```

— `main/stasis.c:492-498`

Note that the two differ in what they leave held. `ast_autochan_channel_lock`
drops the autochan lock on the way out (`:81`) and returns holding only the
channel; `topic_lock_both` returns holding both. Both are correct for their
purpose, and neither property is visible from the macro.

The second shape, used where two locks must be dropped rather than one, is
`DLA_UNLOCK`/`DLA_LOCK`, which §6.11 covers. Three sites fit neither: two that
are not about lock order at all (§6.15), and one that is not written at the
place it runs — a `DEADLOCK_AVOIDANCE` wrapped in a function and reached through
a callback pointer from three other modules (§6.13).

The census, for orientation. `DEADLOCK_AVOIDANCE` has 27 uses: twelve in
`addons/chan_ooh323.c`, six in `channels/chan_iax2.c`, four in
`channels/chan_dahdi.c`, two in `addons/chan_mobile.c`, one each in
`channels/sig_pri.c` and `channels/sig_ss7.c`, and one inside the
`ast_autochan_channel_lock` macro in `include/asterisk/autochan.h:79`.
`CHANNEL_DEADLOCK_AVOIDANCE` has five: `main/channel.c:2555`,
`channels/chan_dahdi.c:8098` and `:8865`, `addons/chan_mobile.c:1119` and
`:1180`. `AO2_DEADLOCK_AVOIDANCE` has one, at `main/stasis.c:496`. `DLA_UNLOCK`
and `DLA_LOCK` have one each, at `channels/chan_dahdi.c:8097` and `:8102`.
Two of the 35 are in `main/` — `channel.c:2555` and `stasis.c:496` — and one is
inside a macro definition in a header (`autochan.h:79`); the remaining 32 are
channel drivers, and sixteen of those are in `addons/`, which is not built by
default.

## 6.3 What the debug arm buys: the original location

The debug arm is built around one fact that the production arm throws away, and
`lock.h` states it plainly above the function that retrieves it:

```c
/*!
 * \brief retrieve lock info for the specified mutex
 *
 * this gets called during deadlock avoidance, so that the information may
 * be preserved as to what location originally acquired the lock.
 */
int ast_find_lock_info(void *lock_addr, char *filename, size_t filename_size, int *lineno, char *func, size_t func_size, char *mutex_name, size_t mutex_name_size);
```

— `include/asterisk/lock.h:325-331`

Chapter 05 §5.8 established that `ast_find_lock_info` is the only part of the
tracking subsystem that production code reads, and that its four callers are all
macros in `lock.h`. Those four callers are four of the five macros of this
chapter; `DLA_LOCK` is the fifth, and it consumes the result `DLA_UNLOCK`
fetched rather than looking up anything itself. Nothing else in the tree calls
`ast_find_lock_info`.

What it reads is the per-thread registry: the `file`, `line_num`, `func` and
`lock_name` that were recorded when this thread acquired the lock. The macro
then passes those four values back into the relock, in place of the `__FILE__`
and `__LINE__` that every other lock macro in `lock.h` bakes in. For the mutex
flavour the relock is:

```c
				__ast_pthread_mutex_lock(__filename, __lineno, __func, __mutex_name, lock); \
```

— `include/asterisk/lock.h:398`

against a signature of:

```c
int __ast_pthread_mutex_lock(const char *filename, int lineno, const char *func, const char* mutex_name, ast_mutex_t *t);
```

— `include/asterisk/lock.h:189`

So after a deadlock-avoidance cycle, `core show locks` reports the lock as
having been acquired at the line that originally acquired it, not at the line
that relocked it. Without this, every lock that had ever been through a
retry loop would appear to have been taken inside a macro in `lock.h`, which is
true and useless. This is the one thing the 21 lines are for.

## 6.4 The relocked lock looks newly acquired

The location survives. Nothing else about the acquisition does.

The relock runs the ordinary path of chapter 02: `__ast_pthread_mutex_lock`
calls `ast_store_lock_info`, then the pthread call, then
`ast_mark_lock_acquired`. The unlock immediately before it ran
`ast_remove_lock_info`, which — per chapter 05 §5.6 — deletes the entry and
decrements `num_locks` once the outermost acquisition goes. A recursively held
lock would only have its `times_locked` decremented, but a lock being put
through deadlock avoidance is held once. So the store that follows finds no
matching address and takes the append path, which assigns fresh values:

```c
	lock_info->locks[i].times_locked = 1;
	lock_info->locks[i].times_lock_attempted = 1;
	lock_info->locks[i].last_locked = ast_tvnow();
```

— `main/utils.c:1135-1137`

Three consequences, all visible in `core show locks`.

The counters reset. `times_locked` and `times_lock_attempted` are both set to 1
on every cycle, so the `(1, 1)` that chapter 05 §5.9 showed at the end of each
stanza is what you get whether the retry loop spun once or a million times.
Nothing in the dump distinguishes a lock that was taken cleanly from one that is
being handed back and forth in a tight avoidance loop.

The hold time resets. `last_locked` is `ast_tvnow()` at `:1137`, and the printer
computes the displayed hold time from it:

```c
	held_for = ast_tvsub(now, lock_info->locks[i].last_locked);
	/* format time duration strings */
	ast_format_duration_hh_mm_ss(lock_info->locks[i].last_locked.tv_sec,
									lock_time, sizeof(lock_time));
	ast_format_duration_hh_mm_ss(lock_info->locks[i].last_unlocked.tv_sec,
									unlock_time, sizeof(unlock_time));
	ast_format_duration_hh_mm_ss(held_for.tv_sec, held_time, sizeof(held_time));
```

— `main/utils.c:1372-1378`

so `held_time` at `:1378` is measured from the relock. A lock held for an hour
across fifty avoidance cycles reports a hold time of microseconds, while the
same line of output names the source location from an hour ago. The two halves
of the stanza describe different events, and this is the mechanism — the
absolute-epoch problem of chapter 05 §5.10 is a separate defect in the same two
lines.

And there is a window in which the lock is advertised as *wanted* rather than
*held*. `ast_store_lock_info` sets `pending = 1` at `:1139`, and
`ast_mark_lock_acquired` clears it only after the pthread call returns. A
concurrent `core show locks` landing in that window prints `Waiting for ` ahead
of the stanza, from `:1382`. That is accurate — the thread genuinely is blocked
on the relock — but it means a thread that was holding a lock a moment ago, and
holds it again a moment later, can be caught in the dump apparently waiting for
it.

## 6.5 Why the retry loop does not corrupt the registry

Chapter 05 §5.5 left a loose end: a failed trylock does not remove its registry
entry, it marks it with `pending = -1` and leaves it in place as a tombstone,
and the only thing that ever removes a tombstone is a sweep inside
`ast_store_lock_info`. The retry loop of §6.2 is the idiom that sweep was
written for, and it is worth tracing once, because the two pieces of machinery
fit together more exactly than either one looks.

Take `chan_dahdi.c:7415-7421`, two peer locks of the same type:

```c
	if (needlock) {
		ast_mutex_lock(&master->lock);
		if (slave) {
			while (ast_mutex_trylock(&slave->lock)) {
				DEADLOCK_AVOIDANCE(&master->lock);
			}
		}
```

— `channels/chan_dahdi.c:7415-7421`

The thread holds `master->lock`, registry index 0. The
`ast_mutex_trylock(&slave->lock)` fails, which stores an entry for `slave->lock`
at index 1 and then marks it failed, so index 1 is a tombstone and `num_locks`
is 2. `DEADLOCK_AVOIDANCE(&master->lock)` then unlocks `master->lock`, and
`ast_remove_lock_info` shifts the tombstone down to index 0 with `num_locks`
back to 1. The relock calls `ast_store_lock_info`, which finds no entry for
`master->lock` and reaches the sweep with `i == 1`:

```c
	if (i && lock_info->locks[i - 1].pending == -1) {
		/* The last lock on the list was one that this thread tried to lock but
		 * failed at doing so.  It has now moved on to something else, so remove
		 * the old lock from the list. */
		i--;
		lock_info->num_locks--;
		memset(&lock_info->locks[i], 0, sizeof(lock_info->locks[0]));
	}
```

— `main/utils.c:1121-1128`

`locks[0].pending` is `-1`, so the tombstone is erased and `master->lock` is
written into index 0. The next trip round the loop stores `slave->lock` fresh.
The registry is back to the shape it had at the top of the iteration — the same
addresses in the same slots, with only the timestamps moved on — and stays that
way however many times the loop spins, which is also why the counters in §6.4
never accumulate.

Two things keep this working, and both are unstated preconditions rather than
checks. The first is that `ast_remove_lock_info` moves the tombstone into the
position the sweep looks at; the sweep only ever examines `locks[i - 1]`, the
entry immediately before the append point. The second is that the sweep is on
the *append* path, reached only when the address being stored does not already
have an entry. The other path bumps an existing entry in place:

```c
	for (i = 0; i < lock_info->num_locks; i++) {
		if (lock_info->locks[i].lock_addr == lock_addr) {
			lock_info->locks[i].times_locked++;
			lock_info->locks[i].times_lock_attempted++;
			lock_info->locks[i].last_locked = ast_tvnow();
#ifdef HAVE_BKTR
			lock_info->locks[i].backtrace = bt;
#endif
			pthread_mutex_unlock(&lock_info->lock);
			return;
		}
	}
```

— `main/utils.c:1100-1111`

and it does not touch `pending`. So storing a lock whose own tombstone is still
present would leave `pending == -1` on an entry the thread actually holds, and
`core show locks` would print `Tried and failed to get ` for a held lock, from
`:1383`. Nothing in the tree does that, because the sweep always fires first —
but the protection is positional, not logical.

The same positional reasoning applies to the lookup the macros depend on.
`ast_find_lock_info` scans by address and consults nothing else:

```c
	for (i = lock_info->num_locks - 1; i >= 0; i--) {
		if (lock_info->locks[i].lock_addr == lock_addr)
			break;
	}
```

— `main/utils.c:1194-1197`

A tombstone matches. If a call site ever passed a lock it had just failed to
acquire, `__res` would be 0 and the macro would take its *hit* branch, reporting
the location of the failed attempt as the place the lock was acquired — and then
the unlock would fail, `__res2` would be non-zero, and the macro would log that
line as *originally locked at*. What keeps this honest is the convention of
§6.2: the macro is always applied to a lock the caller holds.

## 6.6 `DEADLOCK_AVOIDANCE` tests the same value the other way round

Three of the four macros write the four-way branch with the failure case first:
`if (__res2) { log } else { relock }`. `DEADLOCK_AVOIDANCE` writes it with the
success case first, and tests `__res2 == 0` rather than `__res2`:

```c
		if (__res < 0) { /* Shouldn't ever happen, but just in case... */ \
			if (__res2 == 0) { \
				ast_mutex_lock(lock); \
			} else { \
				ast_log(LOG_WARNING, "Could not unlock mutex '%s': %s and no lock info found!  I will NOT try to relock.\n", #lock, strerror(__res2)); \
			} \
		} else { \
			if (__res2 == 0) { \
				__ast_pthread_mutex_lock(__filename, __lineno, __func, __mutex_name, lock); \
			} else { \
				ast_log(LOG_WARNING, "Could not unlock mutex '%s': %s.  {{{Originally locked at %s line %d: (%s) '%s'}}}  I will NOT try to relock.\n", #lock, strerror(__res2), __filename, __lineno, __func, __mutex_name); \
			} \
		} \
```

— `include/asterisk/lock.h:390-402`

This is equivalent. `__res2` is the return of `ast_mutex_unlock`, zero on
success, so `if (__res2 == 0) { relock } else { log }` and `if (__res2) { log }
else { relock }` select the same arms. It is worth pointing out only because
these four macros are otherwise close enough to diff against each other, and
this is the one difference that is not a difference — a reader checking whether
the four agree has to normalise it by hand before the real asymmetries (§6.7,
§6.8, §6.10) show up.

## 6.7 Two of the five production macros are not statements

The production arm is where the macros stop being equivalent to each other in
any sense.

```c
#define AO2_DEADLOCK_AVOIDANCE(obj) \
	ao2_unlock(obj); \
	usleep(1); \
	ao2_lock(obj);

#define CHANNEL_DEADLOCK_AVOIDANCE(chan) \
	ast_channel_unlock(chan); \
	usleep(1); \
	ast_channel_lock(chan);
```

— `include/asterisk/lock.h:476-484`

Neither of those is wrapped in `do { … } while (0)`. Each expands to three
statements, and each ends in a semicolon of its own, so a call site written
`CHANNEL_DEADLOCK_AVOIDANCE(chan);` emits a fourth, empty statement. The
consequences are the textbook ones. Used as the body of an unbraced `if`, only
the unlock is conditional and the relock runs unconditionally. Followed by an
`else`, the code does not compile. Used as the body of an unbraced `while`, the
loop body is the unlock alone — so the loop spins calling `ast_channel_unlock`
on a channel it no longer holds, and never reaches the relock.

The third one *is* wrapped, and is also the only one of the five that checks
whether the unlock succeeded:

```c
#define DEADLOCK_AVOIDANCE(lock) \
	do { \
		int __res; \
		if (!(__res = ast_mutex_unlock(lock))) { \
			usleep(1); \
			ast_mutex_lock(lock); \
		} else { \
			ast_log(LOG_WARNING, "Failed to unlock mutex '%s' (%s).  I will NOT try to relock. {{{ THIS IS A BUG. }}}\n", #lock, strerror(__res)); \
		} \
	} while (0)
```

— `include/asterisk/lock.h:486-495`

So in a production build, `DEADLOCK_AVOIDANCE` is a well-formed statement that
degrades loudly, and `AO2_DEADLOCK_AVOIDANCE` and `CHANNEL_DEADLOCK_AVOIDANCE`
are statement sequences that will relock regardless of whether the unlock
worked. In a debug build all three are `do { … } while (0)` and all three check.
The syntax class of two of these three depends on a build flag; §6.11 shows
that the `DLA` pair's does too.

All six call sites of the two unbraced macros brace the block the macro sits
in, so the hazard is latent rather than live: `main/channel.c:2554-2556`,
`channels/chan_dahdi.c:8095-8107` and `:8864-8865`,
`addons/chan_mobile.c:1118-1120` and `:1179-1181`, and
`main/stasis.c:495-497`. It is latent only in production
builds, which is the awkward part — a new call site written without braces
compiles and behaves correctly under `DEBUG_THREADS` and silently does something
else in the build that ships.

## 6.8 `usleep(1)` is the entire back-off

There is no back-off policy, no retry counter and no escalation anywhere in this
family. The delay between dropping a lock and taking it again is `usleep(1)` in
three of the five debug definitions and in three of the five production ones —
the three `*DEADLOCK_AVOIDANCE` macros in each arm (`lock.h:345`, `:367`,
`:389`, `:478`, `:483`, `:490`), and neither half of the `DLA` pair in either
arm, which §6.11 returns to.

What that delay actually is, is a platform question. POSIX requires `usleep` to
suspend the thread for *at least* the requested interval, and the realised sleep
is whatever the kernel's timer granularity rounds a one-microsecond request up
to; it is not a one-microsecond sleep on any system the author has measured. The
guide's position on this is the same as chapter 04 §4.9's on the
condition-variable clock: the source asks for one microsecond, and what you get
is yours to measure.

The one place the sleep is conditional is the production `DEADLOCK_AVOIDANCE`
quoted in §6.7, where `usleep(1)` sits at `:490`, inside the branch taken when
the unlock succeeded. If the unlock fails, that macro logs and returns with no
sleep at all — and since its call sites are retry loops, the loop then spins at
full speed, still holding the lock, logging on every iteration. The debug arm
sleeps at `:389` before examining either result, so it does not have that
property.

One incidental note on the sleep: `lock.h` calls `usleep` without including
`<unistd.h>`. It arrives through the tree's convention that every source file
includes `asterisk.h` first, which includes `asterisk/compat.h` at
`include/asterisk.h:22`, which includes `<unistd.h>` at
`include/asterisk/compat.h:34`. The header is not self-sufficient on this point.

## 6.9 When the "shouldn't ever happen" branch happens

Three of the macros comment their `__res < 0` branch as *Shouldn't ever happen,
but just in case…* (`lock.h:368`, `:390`, and `:437` in `DLA_LOCK`).
`AO2_DEADLOCK_AVOIDANCE` is more specific and says *Could happen if the ao2
object does not have a mutex* (`:346`). There are five ways to reach it, and
they are not equally rare.

The first is a build combination. Chapter 05 §5.13 established that under
`DEBUG_THREADS` with `LOW_MEMORY`, `ast_find_lock_info` is compiled down to:

```c
#else /* if defined(LOW_MEMORY) */
	return -1;
#endif
```

— `main/utils.c:1213-1215`

`__res` is then `-1` on every call, so all four lookups fail and every macro
takes the miss branch permanently, and the preserved-location machinery that
§6.3 showed is the sole purpose of the debug arm is dead while the 240 bytes of
buffers and the lookup call remain. `LOW_MEMORY` is an *extended* menuselect
option and `DEBUG_THREADS` a *core* one, so nothing stops them being selected
together.

The second is specific to the two `ao2_` flavours, and is what the comment at
`:346` means. The address they look up comes from `ao2_object_get_lockaddr`,
which handles one of the four ao2 locking options:

```c
	switch (obj->priv_data.options & AO2_ALLOC_OPT_LOCK_MASK) {
	case AO2_ALLOC_OPT_LOCK_MUTEX:
		obj_mutex = INTERNAL_OBJ_MUTEX(user_data);
		return &obj_mutex->mutex.lock;
	default:
		break;
	}

	return NULL;
```

— `main/astobj2.c:487-495`

An object allocated `AO2_ALLOC_OPT_LOCK_RWLOCK`, `_NOLOCK` or `_OBJ` yields
`NULL`, and `ast_find_lock_info(NULL, …)` finds nothing. The `_OBJ` case is the
sharp one, because `__ao2_lock` *does* handle it, by recursing into the
delegate's lock at `main/astobj2.c:277`, under the `case` label at `:275`. So
a lockobj-backed object locks and unlocks normally and is simply never findable
by address, and `AO2_DEADLOCK_AVOIDANCE` on one of those will take the miss
branch every time while appearing to work. Only two kinds of object reach the
two `ao2_` flavours between them, and both are safe: channels are allocated
`AO2_ALLOC_OPT_LOCK_MUTEX` at `main/channel_internal_api.c:1248`, and stasis
topics through plain `ao2_t_alloc` at `main/stasis.c:654`, which is the mutex
default.

The third is an untracked lock. Chapter 02 showed that `main/lock.c` calls
`ast_store_lock_info` only inside `if (lt)`, and chapter 01 §1.6 with chapter 02
§2.4 showed that `AST_MUTEX_DEFINE_STATIC_NOTRACKING` leaves `lt` NULL. Such a
lock is held but never registered, so it cannot be found — though the tree has
exactly one such lock, `main/astmm.c:180`, and nothing in this chapter goes near
it, so this route is open only to new code. The fourth is thread-local storage
allocation failure at `main/utils.c:1189-1190`, which chapter 05 treated as the
same class of problem. The fifth is a thread that already holds `AST_MAX_LOCKS`
locks: `ast_store_lock_info` writes a complaint to `stderr` and returns without
storing (`main/utils.c:1113-1119`), so every lock taken past the cap is held and
unregistered in the same way.

The behaviour of the miss branch is worth stating plainly, because it is not a
degradation to nothing. It relocks — `ao2_lock(obj)`, `ast_channel_lock(chan)`,
`ast_mutex_lock(lock)` — using the call site's own `__FILE__` and `__LINE__`.
Deadlock avoidance still works; what is lost is the original location, so the
lock reappears in `core show locks` attributed to a line inside `lock.h`. Under
`DEBUG_THREADS` with `LOW_MEMORY` that is the only behaviour there is.

## 6.10 `AO2_DEADLOCK_AVOIDANCE` always asks for exclusive

`ao2_unlock` releases whichever lock an ao2 object has. Both of
`AO2_DEADLOCK_AVOIDANCE`'s relocks request `AO2_LOCK_REQ_MUTEX` — explicitly at
`:356`, and implicitly at `:350` because `ao2_lock` is
`__ao2_lock(a, AO2_LOCK_REQ_MUTEX, …)` per `astobj2.h:717`. For an rwlock-backed
object, `__ao2_lock` maps `AO2_LOCK_REQ_MUTEX` and `AO2_LOCK_REQ_WRLOCK` to the
same arm:

```c
	case AO2_ALLOC_OPT_LOCK_RWLOCK:
		obj_rwlock = INTERNAL_OBJ_RWLOCK(user_data);
		switch (lock_how) {
		case AO2_LOCK_REQ_MUTEX:
		case AO2_LOCK_REQ_WRLOCK:
			res = __ast_rwlock_wrlock(file, line, func, &obj_rwlock->rwlock.lock, var);
```

— `main/astobj2.c:248-253`

So an rwlock-backed object that was *read*-locked and put through
`AO2_DEADLOCK_AVOIDANCE` comes back *write*-locked, and both arms of the macro
do it: the hit branch asks for `AO2_LOCK_REQ_MUTEX` explicitly, and the miss
branch asks for it through `ao2_lock`. The hit branch is in fact unreachable on
such an object, because `ao2_object_get_lockaddr` returns NULL for every
allocation option but `_MUTEX` (§6.9) — but that only routes rwlock objects to
the miss branch, which upgrades too. What is missing is not a path through the
macro, it is a call site: the one user in the tree is a stasis topic, allocated
with a mutex. The macro has no way to express *put it back the way it was*, and
with one call site in the tree there has been no pressure on it to.

## 6.11 `DLA_UNLOCK` and `DLA_LOCK` are one macro cut in half

Where two locks have to be dropped, the tree supplies a pair of macros that are
not two macros. `DLA_UNLOCK` opens a block and does not close it:

```c
#define DLA_UNLOCK(lock) \
	do { \
		char __filename[80], __func[80], __mutex_name[80]; \
		int __lineno; \
		int __res = ast_find_lock_info(lock, __filename, sizeof(__filename), &__lineno, __func, sizeof(__func), __mutex_name, sizeof(__mutex_name)); \
		int __res2 = ast_mutex_unlock(lock);
```

— `include/asterisk/lock.h:417-422`

The expansion ends mid-block, with all six locals in scope and no closing
brace. `DLA_LOCK` supplies the rest:

```c
#define DLA_LOCK(lock) \
		if (__res < 0) { /* Shouldn't ever happen, but just in case... */ \
			if (__res2) { \
				ast_log(LOG_WARNING, "Could not unlock mutex '%s': %s and no lock info found!  I will NOT try to relock.\n", #lock, strerror(__res2)); \
			} else { \
				ast_mutex_lock(lock); \
			} \
		} else { \
			if (__res2) { \
				ast_log(LOG_WARNING, "Could not unlock mutex '%s': %s.  {{{Originally locked at %s line %d: (%s) '%s'}}}  I will NOT try to relock.\n", #lock, strerror(__res2), __filename, __lineno, __func, __mutex_name); \
			} else { \
				__ast_pthread_mutex_lock(__filename, __lineno, __func, __mutex_name, lock); \
			} \
		} \
	} while (0)
```

— `include/asterisk/lock.h:436-450`

`DLA_LOCK` opens no block of its own; its first token is `if`, it refers to
`__res` and `__res2` which it never declares, and it closes with the `} while
(0)` that terminates `DLA_UNLOCK`'s `do {`. The two are one `do { … } while (0)`
split across two macro definitions, with another lock's worth of code expected
in the gap. Hence the doc comment's emphasis:

```c
/*!
 * \brief Deadlock avoidance unlock
 *
 * In certain deadlock avoidance scenarios, there is more than one lock to be
 * unlocked and relocked.  Therefore, this pair of macros is provided for that
 * purpose.  Note that every DLA_UNLOCK _MUST_ be paired with a matching
 * DLA_LOCK.  The intent of this pair of macros is to be used around another
 * set of deadlock avoidance code, mainly CHANNEL_DEADLOCK_AVOIDANCE, as the
 * locking order specifies that we may safely lock a channel, followed by its
 * pvt, with no worries about a deadlock.  In any other scenario, this macro
 * may not be safe to use.
 */
```

— `include/asterisk/lock.h:405-416`

The comment at `:424-435` is the same twelve lines again with the title changed
from *Deadlock avoidance unlock* to *Deadlock avoidance lock*; nothing else
differs.

Three properties follow from the split that the comment does not mention.

The pair contains no `usleep(1)`. All three `DEADLOCK_AVOIDANCE` macros sleep
between the unlock and the relock; `DLA_UNLOCK` does not, and `DLA_LOCK` does
not. The pair relies entirely on whatever sits between them to yield — which,
given the stated intent, is the `usleep(1)` inside the
`CHANNEL_DEADLOCK_AVOIDANCE` it is meant to wrap. A `DLA_UNLOCK`/`DLA_LOCK`
pair used around anything else has no back-off at all.

The brace structure differs between builds. In production the pair is:

```c
#define DLA_UNLOCK(lock) ast_mutex_unlock(lock)

#define DLA_LOCK(lock) ast_mutex_lock(lock)
```

— `include/asterisk/lock.h:497-499`

No block is opened, so the code between the two macros is in the enclosing
scope. Under `DEBUG_THREADS` it is inside a `do { … } while (0)`. A `break` or
`continue` written between a `DLA_UNLOCK` and its `DLA_LOCK` skips the relock in
both builds, because the relock lives inside `DLA_LOCK`, past the point the
keyword jumps from. What differs is what gets exited. Under `DEBUG_THREADS` the
keyword binds to the macro's own `do`-`while`, so control resumes in the
caller's loop body with the lock dropped and the caller's loop not exited at
all; in production it binds to the caller's enclosing loop. A `return` or `goto`
escapes both, skipping the relock either way.

And nesting shadows. `DLA_UNLOCK` declares `__filename`, `__func`,
`__mutex_name`, `__lineno`, `__res` and `__res2`; a
`CHANNEL_DEADLOCK_AVOIDANCE` placed between the pair opens its own block and
declares all six again. The inner names shadow the outer ones for the duration,
and `DLA_LOCK`'s reference to `__res` resolves to `DLA_UNLOCK`'s copy because
the inner block has closed by then. This is correct, and it is 240 bytes of
character buffers shadowing another 240 that the compiler would warn about:
`-Wshadow` is enabled in the tree only through `AST_DEVMODE_STRICT`
(`Makefile:220`), which is off by default.

## 6.12 The tree's only use of the pair

`DLA_UNLOCK` and `DLA_LOCK` have one call site each, and they are five lines
apart:

```c
					while (p->subs[SUB_THREEWAY].owner && ast_channel_trylock(p->subs[SUB_THREEWAY].owner)) {
						/* Yuck, didn't get the lock on the 3-way, gotta release everything and re-grab! */
						DLA_UNLOCK(&p->lock);
						CHANNEL_DEADLOCK_AVOIDANCE(ast);
						/* We can grab ast and p in that order, without worry.  We should make sure
						   nothing seriously bad has happened though like some sort of bizarre double
						   masquerade! */
						DLA_LOCK(&p->lock);
						if (p->owner != ast) {
							ast_log(LOG_WARNING, "This isn't good...\n");
							return NULL;
						}
					}
```

— `channels/chan_dahdi.c:8095-8107`

This is the pattern the doc comment describes, realised exactly once. The thread
holds the Asterisk channel `ast` and the DAHDI private `p->lock`, and wants the
three-way subchannel's owner. The trylock at `:8095` fails, so both held locks
must go: `DLA_UNLOCK` drops the private at `:8097`, the
`CHANNEL_DEADLOCK_AVOIDANCE` at `:8098` drops and retakes the channel with its
own `usleep(1)` in the middle, and `DLA_LOCK` retakes the private at `:8102`.
Reacquisition is channel first, private second — which is the order
`lock.h:413-414` names as safe.

Nothing executable sits between `:8098` and `:8102`; `:8099-8101` is a comment.
So the brace divergence of §6.11 is not exercised here, though in a debug build
the shadowing is — the `CHANNEL_DEADLOCK_AVOIDANCE` at `:8098` is exactly the
nested declaration of all six `__` locals that §6.11 describes. The `return NULL`
at `:8105` is after `DLA_LOCK`, inside the `while` and outside the macro pair.
The comment at `:8093-8094` — *not especially easy or clean* — is the tree's own
assessment, and the fact that this construction exists in exactly one place is
the rest of it.

## 6.13 The macro survives being put in a function, and three drivers do that

Almost every macro in `lock.h` captures `__FILE__` and `__LINE__`, which makes
wrapping it in a function destroy the information it is capturing. The
deadlock-avoidance macros take their location from the runtime registry instead
(§6.3), so they are the exception: wrapping one in a function costs nothing on
the hit branch of a debug build, which is the only branch that reads the
registry. On the miss branch, and in any production build, the relock supplies
the macro's own `__FILE__` and `__LINE__` — so once wrapped, every caller that
reaches it is attributed to the single line inside the wrapper.

`chan_dahdi.c` does exactly that:

```c
static void my_deadlock_avoidance_private(void *pvt)
{
	struct dahdi_pvt *p = pvt;

	DEADLOCK_AVOIDANCE(&p->lock);
}
```

— `channels/chan_dahdi.c:2037-2042`

and installs it as a callback in three different signalling interfaces, at
`:3372`, `:3568` and `:3696`. The field is declared three times with the same
shape, in `channels/sig_analog.h:149`, `channels/sig_pri.h:182` and
`channels/sig_ss7.h:159`. So the one `DEADLOCK_AVOIDANCE` at
`channels/chan_dahdi.c:2041` is reached through a function pointer from three
protocol modules, and counting macro uses understates its reach considerably.

Each of the three modules wraps the callback in a function with a fallback for
the case where the callback is not installed, and the three fallbacks do not
agree. Analog:

```c
static void analog_deadlock_avoidance_private(struct analog_pvt *p)
{
	if (analog_callbacks.deadlock_avoidance_private) {
		analog_callbacks.deadlock_avoidance_private(p->chan_pvt);
	} else {
		/* Fallback to manual avoidance if callback not present. */
		analog_unlock_private(p);
		usleep(1);
		analog_lock_private(p);
	}
}
```

— `channels/sig_analog.c:584-594`

PRI:

```c
static void sig_pri_deadlock_avoidance_private(struct sig_pri_chan *p)
{
	if (sig_pri_callbacks.deadlock_avoidance_private) {
		sig_pri_callbacks.deadlock_avoidance_private(p->chan_pvt);
	} else {
		/* Fallback to the old way if callback not present. */
		sig_pri_unlock_private(p);
		sched_yield();
		sig_pri_lock_private(p);
	}
}
```

— `channels/sig_pri.c:400-410`

and `sig_ss7.c:92-102` is the PRI version with the names changed. Two of the
three yield with `sched_yield()` and one with `usleep(1)`, matching the macros;
and all three fallbacks drop the location preservation, check nothing, and are
therefore closer to the production arm than to the debug one regardless of how
the tree was built. The comment calls this *the old way* in two of the three.

The hand-rolled form appears without a callback too, in both of the modules
that also install one. `sig_pri_lock_owner` (`channels/sig_pri.c:1254-1257`)
and `sig_ss7_lock_owner` are line-for-line the same construction: a
`DEADLOCK_AVOIDANCE` with a manual unlock and relock wrapped around it on the
lines either side. The SS7 copy:

```c
	for (;;) {
		if (!ss7->pvts[chanpos]->owner) {
			/* There is no owner lock to get. */
			break;
		}
		if (!ast_channel_trylock(ss7->pvts[chanpos]->owner)) {
			/* We got the lock */
			break;
		}

		/* Avoid deadlock */
		sig_ss7_unlock_private(ss7->pvts[chanpos]);
		DEADLOCK_AVOIDANCE(&ss7->lock);
		sig_ss7_lock_private(ss7->pvts[chanpos]);
	}
```

— `channels/sig_ss7.c:349-363`

`:360-362` is what `DLA_UNLOCK`/`DLA_LOCK` exists to express, written out by
hand — two locks dropped, the outer one by macro and the inner one by plain
calls; the function's own preconditions at `:344-345` state that `ss7->lock` is
already held and the private is taken under it. It gives up location
preservation for the private lock in exchange for being readable, which is a
defensible trade and the opposite of the one `chan_dahdi.c:8097-8102` makes.

## 6.14 The ordering rules the tree actually follows

There is no document. The tree has no file that states a global lock order; the
rules exist as comments next to the code that depends on them, and the only way
to assemble them is to read the comments. Collected, the ones the tree states
explicitly are:

Three places write an order down as an ordered list. The bridging stack is
one:

```c
 * In general, anywhere else it is unsafe to use this API. Care should be
 * taken when using this API to ensure that the locking order remains
 * correct. The locking order must be:
 *  \arg The \ref ast_bridge "\c ast_bridge"
 *  \arg The \ref ast_bridge_channel "\c ast_bridge_channel"
 *  \arg The \ref ast_channel "\c ast_channel"
```

— `include/asterisk/bridge_channel.h:33-38`

The second is `stasis_topic_pool_get_topic`:

```c
	/*
	* Lock ordering:
	*
	*   pool->pool_container (AO2 lock)
	*       → entry->init_lock
	*           → topic locks (inside stasis_topic_create() /
	*                            stasis_forward_all())
	*
	* We intentionally do NOT hold the pool container lock while calling
	* stasis_topic_create() or stasis_forward_all() to avoid deadlocks with
	* other code that may take topic locks first and then need the pool lock.
	*/
```

— `main/stasis.c:1998-2009`

That one is a comment inside a function body rather than API documentation, so
it is reachable only by reading the implementation that obeys it. The third is
the oldest and the most emphatic, at the top of a 12,000-line application:

```c
/*!
 * \par Please read before modifying this file.
 * There are three locks which are regularly used
 * throughout this file, the queue list lock, the lock
 * for each individual queue, and the interface list lock.
 * Please be extra careful to always lock in the following order
 * 1) queue list lock
 * 2) individual queue lock
 * 3) interface list lock
 * This order has sort of "evolved" over the lifetime of this
 * application, but it is now in place this way, so please adhere
 * to this order!
 */
```

— `apps/app_queue.c:111-123`

It is the only one of the three that names the locks as a numbered sequence,
and the only one that says in so many words how the order came to be what it
is.

A channel outranks its driver private. This is stated twice in `lock.h` itself,
in the paragraph §6.11 quotes and again verbatim in its twin: *the locking order
specifies that we may safely lock a channel, followed by its pvt* (`:413-414`,
`:432-433`). It is the rule `chan_dahdi.c:8098-8102` implements and the one
`chan_motif.c:1211-1213` implies.

A channel outranks an autochan: *the channel lock is superior to the autochan
lock in locking order* (`include/asterisk/autochan.h:72-73`). This one is stated
precisely because `ast_autochan_channel_lock` has to violate it — it holds the
autochan lock and reaches for the channel.

A channel outranks an agent: *Defined locking order is channel lock then agent
lock* (`apps/app_agent_pool.c:727`).

A parent RTP instance outranks its children: *we guarantee that the locking
order is always parent->child or that the child lock is not held when acquiring
the parent lock* (`res/res_rtp_asterisk.c:8529-8530`, and the same assertion
reflowed at `:7085-7086`).

The channel container outranks every channel in it, stated twice in
`main/channel.c` — once in the comment above `channel_do_masquerade` (`:6881`),
*Assumes _NO_ channels and _NO_ channel pvt's are locked. If a channel is
locked while calling this function, it invalidates our channel container
locking order* (`:6877-6878`), and once at the point of use (`:6923`).

Two observations about this list. Every rule in it is local to one subsystem
and its own parts, and no two of these subsystems are ranked against each other
— there is no statement anywhere about, say, a stasis topic against a channel.
And the one general principle that *is* stated, channel-before-pvt, is stated
in the documentation of the macro that exists to work around it. The rules are
discovered at the sites that need to break them.

Which is why the retry loop of §6.2 appears where the order is already correct.
`chan_dahdi.c:8864-8865` holds a channel and reaches for its private — the right
order — and still needs avoidance, because a trylock in the right direction can
still lose to a thread travelling in the wrong one. Ordering rules prevent
deadlock only when every participant obeys them; deadlock avoidance is what the
obedient thread does when it cannot assume that.

The alternative strategy, which newer code prefers, is to take a reference,
drop the lock, and acquire in the documented order:

```c
		/* We had to hold the pvt lock while getting a ref to the owner channel
		 * but now we have to let this lock go in order to preserve proper
		 * locking order when grabbing the channel lock */
		ao2_unlock(pvt);

		/* Look, no deadlock avoidance, hooray! */
		ast_channel_lock(chan);
		ao2_lock(pvt);
```

— `channels/chan_motif.c:1211-1218`

The comment at `:1216` is the tree's opinion of the subject of this chapter.
This works wherever a reference can stand in for a lock, which is the point of
reference counting and not available at all in the drivers that use deadlock
avoidance most, whose private structures predate ao2. The ao2 guide's own
answer to a lock cycle is a third one again — `ao2_trylock` and a yield, in its
§3.8 and the last rule of its §3.10 — so the tree has three strategies for the
same problem and no statement anywhere about which to reach for.

## 6.15 The two uses that are not about ordering

Two of the 35 call sites hold no second lock and have nothing to do with lock
order. `ast_hangup` is one:

```c
	ast_channel_lock(chan);

	while (ast_channel_masq(chan) || ast_channel_masqr(chan))  {
		CHANNEL_DEADLOCK_AVOIDANCE(chan);
	}
```

— `main/channel.c:2552-2556`

There is no trylock here and no second lock. The loop drops and retakes the
channel lock, repeatedly, purely so that whichever thread is setting up a
masquerade on this channel can make progress — the predicate at `:2554` is state
that only another thread can clear, and it can only clear it while holding the
channel lock. The macro is being used as a yield, and the `usleep(1)` of §6.8 is
the whole mechanism. This is also the only `CHANNEL_DEADLOCK_AVOIDANCE` in
`main/`.

The second is `wait_for_peercallno` in `chan_iax2.c`, which is the same idea
with a bound on it:

```c
		int count = 10;
		while (count-- && pvt && !pvt->peercallno) {
			DEADLOCK_AVOIDANCE(&iaxsl[callno]);
			pvt = iaxs[callno];
		}
```

— `channels/chan_iax2.c:5484-5488`

The private lock is dropped and retaken up to ten times while waiting for
another thread to fill in `pvt->peercallno`. There is no trylock and no second
lock here either. Two things make this the most careful call site in the
chapter: `pvt` is re-read from `iaxs[callno]` on every iteration (`:5487`),
because dropping the lock lets another thread destroy the private underneath the
loop, which is the re-validation §6.17's first rule asks for written out
explicitly; and `count = 10` is the only retry limit anywhere in this family.

Everything else is about order, including the largest concentration in the tree.
`addons/chan_ooh323.c` accounts for twelve of the 27 `DEADLOCK_AVOIDANCE` uses,
in an unsupported add-on module, and all twelve are the retry-loop shape of §6.2
against the owner channel:

```c
		while (p->owner && ast_channel_trylock(p->owner)) {
			ast_debug(1,"Failed to grab lock, trying again\n");
			DEADLOCK_AVOIDANCE(&p->lock);
		}
```

— `addons/chan_ooh323.c:1580-1583`

## 6.16 Messages

Seven distinct `LOG_WARNING` strings come out of this region, from nine
`ast_log` calls. All of them mean the same thing — *the unlock failed, so I am
not going to take the lock back* — and the detail is in which half of the pair
you got.

| Message | Origin | Means |
|---|---|---|
| `Could not unlock ao2 object '%s': %s and no lock info found!  I will NOT try to relock.` | `lock.h:348` | `AO2_DEADLOCK_AVOIDANCE`, both lookups failed; the object probably has no mutex (§6.9) |
| `Could not unlock ao2 object '%s': %s.  {{{Originally locked at %s line %d: (%s) '%s'}}}  I will NOT try to relock.` | `lock.h:354` | Same, with the registry's record of where the lock was taken |
| `Could not unlock channel '%s': %s and no lock info found!  I will NOT try to relock.` | `lock.h:370` | `CHANNEL_DEADLOCK_AVOIDANCE`; channels are mutex-backed, so for a channel the lookup can only fail through the `LOW_MEMORY` build, threadstorage failure or the `AST_MAX_LOCKS` cap (§6.9) |
| `Could not unlock channel '%s': %s.  {{{Originally locked at %s line %d: (%s) '%s'}}}  I will NOT try to relock.` | `lock.h:376` | Same, with the original location |
| `Could not unlock mutex '%s': %s and no lock info found!  I will NOT try to relock.` | `lock.h:394`, `lock.h:439` | `DEADLOCK_AVOIDANCE` and `DLA_LOCK` share the wording; the two are indistinguishable in the log |
| `Could not unlock mutex '%s': %s.  {{{Originally locked at %s line %d: (%s) '%s'}}}  I will NOT try to relock.` | `lock.h:400`, `lock.h:445` | Same pairing |
| `Failed to unlock mutex '%s' (%s).  I will NOT try to relock. {{{ THIS IS A BUG. }}}` | `lock.h:493` | Production `DEADLOCK_AVOIDANCE` only; the only one of the nine that appears in a shipping build, and the only one that says it is a bug |

The `{{{…}}}` bracketing appears in no other log line in the tree: outside the
Mustache templates under `rest-api-templates/`, where it is that language's
own syntax, the only occurrences in C or header files are these five strings
and one piece of incidental brace nesting at `lock.h:97`. It is a local idiom,
not a convention. `DO_THREAD_CRASH` at `lock.h:253-257` is not involved in any
of these either — none of the nine calls it.

Two gaps are worth knowing about. Nothing is logged when deadlock avoidance
*succeeds*, so a loop that spins ten thousand times is silent, and §6.4 showed
`core show locks` cannot see it either. And nothing counts iterations, so there
is no way to distinguish contention from livelock from either the log or the CLI.

## 6.17 Rules

| Rule | Why |
|---|---|
| Re-validate everything after any of these macros | They drop the lock. Every invariant that held before it is void afterwards, and the macro cannot know what yours were (intro, §6.15) |
| Apply the macro to the lock you hold, never the one you want | It unlocks its argument. Applied to a lock you failed to acquire, the registry lookup matches a tombstone and the diagnostics lie (§6.5) |
| Brace the loop body, always | In a production build two of the five macros are statement sequences, not statements; an unbraced `while` body executes the unlock alone (§6.7) |
| Do not put a `break` or `continue` between `DLA_UNLOCK` and `DLA_LOCK` | The relock is skipped either way, because it lives inside `DLA_LOCK`; under `DEBUG_THREADS` the keyword also binds to the macro's own `do`-`while`, so it does not even exit the loop you meant (§6.11) |
| Do not use `DLA_UNLOCK`/`DLA_LOCK` around anything but `CHANNEL_DEADLOCK_AVOIDANCE` | The pair contains no `usleep(1)`; the back-off comes from what you put between them. The doc comment says the same thing less specifically (§6.11) |
| Do not read the hold time or the `(1, 1)` counters in `core show locks` as cumulative | A relocked lock is a fresh registry entry. The location is preserved; the timestamps and counters are not (§6.4) |
| Treat `DEBUG_THREADS` with `LOW_MEMORY` as having no location preservation | `ast_find_lock_info` returns `-1` unconditionally, so every lookup fails and every macro takes the miss branch for ever (§6.9, chapter 05 §5.13) |
| Prefer a reference and an ordered reacquisition where the object is ao2 | `chan_motif.c:1216` is the tree's own comment on the alternative (§6.14) |
| Do not expect the ordering rules to be written down anywhere | They are comments next to the code that breaks them. There are eight, each local to one subsystem, and none ranks two subsystems against each other (§6.14) |

## Next

Chapter 07 will cover thread creation: `ast_pthread_create_stack` and
`ast_pthread_create_detached_stack`, the four macros over them, `AST_STACKSIZE`
and `AST_BACKGROUND_STACKSIZE`, how a thread gets registered and named — which
is the other half of chapter 05 §5.2's question about which threads the registry
can see — and what the 130 call sites are for.
