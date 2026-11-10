# 3. Read-write locks

Every `file:line` citation in this guide was verified against the `24` branch.
Newer releases and other branches may differ — line numbers drift — so if you
are reading this against a different tree, treat the citations as approximate
and re-grep for the symbol name.

Chapter 02 covered five mutex entry points. This chapter covers nine rwlock
ones, and most of the work is in seeing which of them are the mutex code with a
different pthread call in the middle and which are not.

Three things are genuinely different, and they are the reason to read this
chapter rather than assume it. The reentrancy array stops being a stack,
because a read lock can be held by several threads at once. Unlock loses the
ownership check that makes mutex diagnostics useful. And the one stock-build
diagnostic chapter 02 found in the mutex path — the single message a non-debug
Asterisk will tell you about a lock — has no counterpart here.

## 3.1 Nine entry points

| Entry point | Lines | Registers as | pthread call | `DETECT_DEADLOCKS` | On failure | Error logged |
| --- | --- | --- | --- | --- | --- | --- |
| `__ast_rwlock_init` | `:714-746` | — | `pthread_rwlock_init` | — | — | "already initialized" (dead, §3.3) |
| `__ast_rwlock_destroy` | `:748-793` | — | `pthread_rwlock_destroy` | — | — | "Error destroying rwlock" |
| `__ast_rwlock_unlock` | `:795-864` | — | `pthread_rwlock_unlock` | — | — | "Error releasing rwlock" |
| `__ast_rwlock_rdlock` | `:866-968` | `AST_RDLOCK` | `pthread_rwlock_rdlock` | yes | `ast_remove_lock_info` | "Error obtaining read lock" |
| `__ast_rwlock_wrlock` | `:970-1071` | `AST_WRLOCK` | `pthread_rwlock_wrlock` | yes | `ast_remove_lock_info` | "Error obtaining write lock" |
| `__ast_rwlock_timedrdlock` | `:1073-1153` | `AST_WRLOCK` | `pthread_rwlock_timedrdlock` | no | `ast_remove_lock_info` | "Error obtaining read lock" |
| `__ast_rwlock_timedwrlock` | `:1155-1235` | `AST_WRLOCK` | `pthread_rwlock_timedwrlock` | no | `ast_remove_lock_info` | "Error obtaining read lock" |
| `__ast_rwlock_tryrdlock` | `:1237-1285` | `AST_RDLOCK` | `pthread_rwlock_tryrdlock` | n/a | `ast_mark_lock_failed` | none |
| `__ast_rwlock_trywrlock` | `:1287-1335` | `AST_WRLOCK` | `pthread_rwlock_trywrlock` | n/a | `ast_mark_lock_failed` | none |

Two cells in that table do not say what you expect, and both belong to the
timed pair, in different columns: `__ast_rwlock_timedrdlock` registers as
`AST_WRLOCK`, and `__ast_rwlock_timedwrlock` reports its failures as read lock
failures. §3.9 is about why.

The nine take up `:714-1335`, which is a little over 600 lines — about 46% of
the file for a facility used, directly, less than a quarter as often as the
mutex.

## 3.2 `ast_rwlock_t` is a different struct, and the arguments are in a different order

The struct is the same device chapter 01 described for `ast_mutex_t`, with the
pthread object first and the same renamed-field arms:

```c
struct ast_rwlock_info {
	pthread_rwlock_t lock;
#if !defined(DEBUG_THREADS) && !defined(DEBUG_THREADS_LOOSE_ABI) && \
	!defined(DETECT_DEADLOCKS)
	/*!
	 * These fields are renamed to ensure they are never used when
	 * DEBUG_THREADS is not defined.
	 */
	struct ast_lock_track *_track;
	struct ast_lock_track_flags _flags;
#elif defined(DEBUG_THREADS) || defined(DETECT_DEADLOCKS)
	/*! Track which thread holds this lock */
	struct ast_lock_track *track;
	struct ast_lock_track_flags flags;
#endif
};
```

— `include/asterisk/lock.h:164-179`

Note the member name. The mutex calls its pthread object `mutex`; the rwlock
calls its `lock`. Every line of rwlock code in `main/lock.c` therefore says
`&t->lock` where the mutex code says `&t->mutex`, which is worth knowing before
you try to move a line from one to the other.

The macros have a subtler difference:

```c
#define ast_rwlock_destroy(rwlock)         __ast_rwlock_destroy(__FILE__, __LINE__, __PRETTY_FUNCTION__, #rwlock, rwlock)
#define ast_rwlock_unlock(a)               __ast_rwlock_unlock(__FILE__, __LINE__, __PRETTY_FUNCTION__, a, #a)
#define ast_rwlock_rdlock(a)               __ast_rwlock_rdlock(__FILE__, __LINE__, __PRETTY_FUNCTION__, a, #a)
#define ast_rwlock_wrlock(a)               __ast_rwlock_wrlock(__FILE__, __LINE__, __PRETTY_FUNCTION__, a, #a)
#define ast_rwlock_tryrdlock(a)            __ast_rwlock_tryrdlock(__FILE__, __LINE__, __PRETTY_FUNCTION__, a, #a)
#define ast_rwlock_trywrlock(a)            __ast_rwlock_trywrlock(__FILE__, __LINE__, __PRETTY_FUNCTION__, a, #a)
#define ast_rwlock_timedrdlock(a, b)       __ast_rwlock_timedrdlock(__FILE__, __LINE__, __PRETTY_FUNCTION__, a, #a, b)
#define ast_rwlock_timedwrlock(a, b)       __ast_rwlock_timedwrlock(__FILE__, __LINE__, __PRETTY_FUNCTION__, a, #a, b)
```

— `include/asterisk/lock.h:240-247`

The acquisition and unlock macros pass `a` *then* `#a` — the lock pointer
before the stringified name. The mutex macros pass `#a` then `a`
(`lock.h:196-198`). The last two parameters of the two families are swapped,
so a hand-written call to a `__ast_` entry point that follows the mutex habit
will pass them the wrong way round. The compiler does catch that: an
`ast_rwlock_t *` and a `const char *` are incompatible pointer types, so the
swap is a diagnostic rather than a runtime surprise — but it is a diagnostic
you have to actually read. The `init` and `destroy` macros keep the mutex
order, which is the only reason `ast_rwlock_destroy` above looks different from
the eight below it.

And one spelling difference, which matters only to grep, and which is not
even consistent within the rwlock half of the file. The seven lock and unlock
entry points spell the parameter `line` where the mutex code says `lineno` —
but `__ast_rwlock_init` and `__ast_rwlock_destroy` keep `lineno`, and use it at
`:726`, `:759` and `:772`. So a grep for `lineno` finds the mutex code, the
header prototypes and two of the nine rwlock bodies, and misses the other
seven.

## 3.3 Initialisation, and the writer-preference attribute

`__ast_rwlock_init` sets `track`, `flags.tracking` and `flags.setup` under the
same `#if defined(DEBUG_THREADS) || defined(DETECT_DEADLOCKS)` guard the mutex
uses, which matches the second arm of the struct exactly. Then it does the
attribute dance — which the mutex `init` also does, at `:169-172` — with one
line in the middle that has no mutex counterpart:

```c
#endif /* DEBUG_THREADS */

	pthread_rwlockattr_init(&attr);
#ifdef HAVE_PTHREAD_RWLOCK_PREFER_WRITER_NP
	pthread_rwlockattr_setkind_np(&attr, PTHREAD_RWLOCK_PREFER_WRITER_NP);
#endif
	res = pthread_rwlock_init(&t->lock, &attr);
	pthread_rwlockattr_destroy(&attr);
```

— `main/lock.c:736-743`

Every rwlock that actually runs `__ast_rwlock_init` is initialised with an
explicit attribute object asking for writer preference, where available —
which, as the paragraphs below work out, is not quite every `ast_rwlock_t` in
the process. The intent is the obvious one: a steady stream of readers should not be able to starve a writer
indefinitely. `HAVE_PTHREAD_RWLOCK_PREFER_WRITER_NP` is a `configure` test for
whether the constant exists in `pthread.h` (`configure.ac:1073-1077`), and on
glibc it does, so this line is live in a normal build.

Whether it has the effect the name suggests is a question about your C library,
not about Asterisk, and it is worth checking rather than assuming — the kind
constants that a given `pthread` implementation actually honours are
documented by that implementation. What the Asterisk source tells you is only
that the request is made, consistently, by `__ast_rwlock_init`.

Two consequences follow from this being in `init` rather than in the struct
initialiser. A statically initialised rwlock — `AST_RWLOCK_DEFINE_STATIC`
expanding to `AST_RWLOCK_INIT_VALUE`, which is `PTHREAD_RWLOCK_INITIALIZER`
plus tracking fields (`lock.h:91`, `:105`) — gets the library's default kind,
not the requested one, unless the constructor arm of `__AST_RWLOCK_DEFINE`
(`lock.h:533-545`) runs and calls `ast_rwlock_init`. And an `ast_rwlock_t` that
some structure zeroes and uses without calling `ast_rwlock_init` gets the
default too.

The dead-code story from chapter 02 §2.3 repeats here without change. The
double-init check at `:725` and the uninitialised-rwlock checks at `:758` and
`:807` are all guarded by `AST_MUTEX_INIT_W_CONSTRUCTORS &&
CAN_COMPARE_MUTEX_TO_INIT_VALUE`, and on a mainstream Linux build neither is
defined. Those three messages cannot be emitted.

## 3.4 Phase 1 is one block, shared by six functions

All six acquisition paths contain this block. In the two try variants it is
the first thing in the function; `rdlock` and `wrlock` put a three-line
`canlog` declaration ahead of it (`:871-873`, `:975-977`); the timed pair fold
that declaration into the block itself (`:1080`, `:1162`):

```c
#ifdef DEBUG_THREADS
	struct ast_lock_track *lt = ast_get_reentrancy(&t->track, &t->flags, 0);
	struct ast_bt *bt = NULL;

	if (lt) {
#ifdef HAVE_BKTR
		struct ast_bt tmp;

		/* The implementation of backtrace() may have its own locks.
		 * Capture the backtrace outside of the reentrancy lock to
		 * avoid deadlocks. See ASTERISK-22455. */
		ast_bt_get_addresses(&tmp);

		ast_reentrancy_lock(lt);
		if (lt->reentrancy < AST_MAX_REENTRANCY) {
			lt->backtrace[lt->reentrancy] = tmp;
			bt = &lt->backtrace[lt->reentrancy];
		}
		ast_reentrancy_unlock(lt);
#endif
		ast_store_lock_info(AST_RDLOCK, filename, line, func, name, t, bt);
```

— `main/lock.c:875-895`

Not "the same shape as". The same block, byte for byte, in four of the six,
with one token varying: the first argument to `ast_store_lock_info`. The timed
pair differ by exactly the `int canlog` declaration mentioned above, which sits
inside the block rather than before it and makes their version 22 lines instead
of 21. It is also the same block as the mutex's phase 1 (`:264-286`), again
differing only in that argument and in the `lineno`/`line` and
`mutex_name`/`name` parameter spellings.

The comment is worth reading once because it explains a shape you will see
repeated. `ast_bt_get_addresses` is called *before* `ast_reentrancy_lock`, into
a stack temporary, and the result is copied into the array under the lock.
`backtrace()` in libc may take locks of its own, and calling it while holding
the per-lock reentrancy mutex was a real deadlock — the bug reference in the
comment is the record of it. The cost is one `struct ast_bt` copy per
acquisition, which in a build that is already calling `backtrace()` on every
lock is not the expensive part.

Note also that the backtrace is stored at index `reentrancy` — the slot that
phase 3 is *about* to fill — and the `if` means that past the cap nothing is
stored and `bt` stays `NULL`. The slot is written before the lock is held, and
is only committed to by the increment in phase 3.

## 3.5 Phase 3 drops the branch that mutexes have

```c
#ifdef DEBUG_THREADS
	if (!res && lt) {
		ast_reentrancy_lock(lt);
		if (lt->reentrancy < AST_MAX_REENTRANCY) {
			lt->file[lt->reentrancy] = filename;
			lt->lineno[lt->reentrancy] = line;
			lt->func[lt->reentrancy] = func;
			lt->thread_id[lt->reentrancy] = pthread_self();
			lt->reentrancy++;
		}
		ast_reentrancy_unlock(lt);
		ast_mark_lock_acquired(t);
	} else if (lt) {
#ifdef HAVE_BKTR
		if (lt->reentrancy) {
```

— `main/lock.c:935-949`

Compare that `if` with the mutex's (`:364-372`). The mutex has an `else` on it
that logs `really deep reentrancy!`. The rwlock does not. Past
`AST_MAX_REENTRANCY` the rwlock path silently declines to record the
acquisition and returns success, with nothing in the log.

For a mutex, hitting the cap means one thread has recursively locked the same
mutex ten times, which is unusual enough to deserve a message. For an rwlock
the cap is shared across *all* current holders, so eleven threads each holding
one read lock is enough to reach it — and that is not a bug, it is a popular
read lock. A message there would be noise.

`AST_MAX_REENTRANCY` is 10 (`lock.h:108`), so on a busy system this is not a
hypothetical threshold. It is worth being precise about what crossing it
actually costs, because the obvious guess — that the eleventh holder becomes
invisible — is wrong, and it is wrong because two different data structures are
in play.

`core show locks` does not read `ast_lock_track`. It walks a per-thread list of
`struct thr_lock_info` (`main/utils.c:996-1031`, dumped by `handle_show_locks`
at `:1552`) whose own cap is `AST_MAX_LOCKS`, 64 (`:982`). That list is
populated by the `ast_store_lock_info` call in phase 1, which sits *outside* the
`AST_MAX_REENTRANCY` test, and the acquisition is confirmed by
`ast_mark_lock_acquired(t)` at `:946`, also outside it. So the eleventh
read-lock holder does appear in `core show locks`, with its file, line and
function, exactly like the first ten — and its unlock removes it again, because
`ast_remove_lock_info` searches the same per-thread list.

What the eleventh holder loses is the *per-lock* record. No backtrace is
captured for it — `bt` stays `NULL` in phase 1 — and nothing is written to
`lt->file`, `lt->lineno`, `lt->func` or `lt->thread_id`. That has a tidy
consequence at unlock: §3.6's backwards search will not find an entry for that
thread, so `lock_found` stays 0 and `reentrancy` is not decremented, which is
exactly right, since nothing was ever recorded. The array under-counts rather
than desynchronising. The one real loss is that the `DETECT_DEADLOCKS` report
in §3.8 cannot name an untracked holder as the thread the lock "was locked
here" by, because there is no slot holding its details.

## 3.6 Unlock is a search, not a pop

This is the structural difference the rest of the chapter hangs on. A mutex can
be held by exactly one thread, so its tracking array is a stack and unlock pops
the top. An rwlock read-locked by five threads has five live entries belonging
to five different threads, in acquisition order, and the thread calling unlock
is not necessarily the one on top.

So unlock searches:

```c
	lt = ast_get_reentrancy(&t->track, &t->flags, 0);
	if (lt) {
		ast_reentrancy_lock(lt);
		if (lt->reentrancy) {
			int i;
			pthread_t self = pthread_self();
			for (i = lt->reentrancy - 1; i >= 0; --i) {
				if (lt->thread_id[i] == self) {
					lock_found = 1;
					if (i != lt->reentrancy - 1) {
						lt->file[i] = lt->file[lt->reentrancy - 1];
						lt->lineno[i] = lt->lineno[lt->reentrancy - 1];
						lt->func[i] = lt->func[lt->reentrancy - 1];
						lt->thread_id[i] = lt->thread_id[lt->reentrancy - 1];
					}
#ifdef HAVE_BKTR
					bt = &lt->backtrace[i];
#endif
					lt->file[lt->reentrancy - 1] = NULL;
					lt->lineno[lt->reentrancy - 1] = 0;
					lt->func[lt->reentrancy - 1] = NULL;
					lt->thread_id[lt->reentrancy - 1] = AST_PTHREADT_NULL;
					break;
				}
			}
		}
```

— `main/lock.c:814-839`

Read it as three steps. Scan backwards from the top for an entry whose
`thread_id` is mine, so that a thread holding the lock recursively releases its
most recent acquisition first. If the entry found is not the top one, move the
top entry down into the hole — a swap-with-last removal, which keeps the array
dense at the cost of reordering it. Then clear the top slot and break.

The reordering matters when you are reading `core show locks` output. The array
is in acquisition order only until the first out-of-order release; after that
the order is whatever the swaps left behind. It is a set of current holders,
not a history.

The backwards scan direction is doing real work beyond recursion order. If two
threads hold read locks and the *first* one releases, the search finds its
entry at index 0, and the swap moves the second thread's entry down to index 0.
That second thread's own later unlock then finds itself at index 0 rather than
index 1 — which is correct, and is why the removal has to be a search rather
than an index remembered at acquisition time.

## 3.7 What unlock does not check

Having found the thread's entry or not, unlock does this:

```c
		if (lock_found && --lt->reentrancy < 0) {
			log_mutex_error(canlog, "%s line %d (%s): rwlock '%s' freed more times than we've locked!\n",
					filename, line, func, name);
			lt->reentrancy = 0;
		}
```

— `main/lock.c:841-845`

Two things about that `if`. The second half cannot be true: `lock_found` is set
only inside `if (lt->reentrancy)`, so `reentrancy` is at least 1 when the
decrement runs, so the result is at least 0. The `freed more times than we've
locked!` message at `:842` is unreachable. The mutex's version of the same
message at `:487` is reachable, because the mutex decrements unconditionally;
this one is gated on having found an entry first.

And the first half is the missing check. If `lock_found` is 0 — you called
`ast_rwlock_unlock` on an rwlock this thread does not hold — nothing happens.
No message, no decrement, no crash hook. Compare the mutex, where the
equivalent situation produces the two-line "trying to unlock a mutex you don't
own, locked here" report that chapter 02 §2.8 called the most useful message in
the subsystem. There is no rwlock equivalent. The tracking state is simply left
alone and the `pthread_rwlock_unlock` goes ahead.

Then the unlock itself:

```c
	res = pthread_rwlock_unlock(&t->lock);

#ifdef DEBUG_THREADS
	if (res) {
		log_mutex_error(canlog, "%s line %d (%s): Error releasing rwlock: %s\n",
				filename, line, func, strerror(res));
		DO_THREAD_CRASH;
	}
```

— `main/lock.c:853-860`

`#ifdef DEBUG_THREADS`, not `#ifndef`. Chapter 02 §2.8 reached the mutex's
`#ifndef DEBUG_THREADS` block at `:512-523` — the one diagnostic that survives
into a stock build, the one reason the four descriptor arguments are worth
passing. The rwlock has no such
block. In a stock Asterisk, a failed `pthread_rwlock_unlock` returns its errno
to a caller that, in the overwhelming majority of cases, discards it, and
nothing is logged anywhere.

That is the single most important practical fact in this chapter. The set of
lock faults a non-debug Asterisk will tell you about contains exactly one
member, and it is a mutex fault.

## 3.8 `DETECT_DEADLOCKS`, minus the clamp and minus the event

`rdlock` and `wrlock` get the same trylock-and-poll treatment as the mutex:

```c
#if defined(DETECT_DEADLOCKS)
	{
		time_t seconds = time(NULL);
		time_t wait_time, reported_wait = 0;
		do {
			res = pthread_rwlock_tryrdlock(&t->lock);
			if (res == EBUSY) {
				wait_time = time(NULL) - seconds;
				if (wait_time > reported_wait && (wait_time % 5) == 0) {
					log_mutex_error(canlog, "%s line %d (%s): Deadlock? waited %d sec for readlock '%s'?\n",
						filename, line, func, (int)wait_time, name);
#ifdef DEBUG_THREADS
					if (lt) {
						ast_reentrancy_lock(lt);
#ifdef HAVE_BKTR
						__dump_backtrace(&lt->backtrace[lt->reentrancy], canlog);
#endif
						log_mutex_error(canlog, "%s line %d (%s): '%s' was locked  here.\n",
								lt->file[lt->reentrancy-1], lt->lineno[lt->reentrancy-1],
								lt->func[lt->reentrancy-1], name);
#ifdef HAVE_BKTR
						__dump_backtrace(&lt->backtrace[lt->reentrancy-1], canlog);
#endif
						ast_reentrancy_unlock(lt);
					}
#endif
					reported_wait = wait_time;
				}
				usleep(200);
			}
		} while (res == EBUSY);
	}
```

— `main/lock.c:899-930`

The polling interval, the five-second reporting grid and the
`wait_time > reported_wait` guard against `time()`'s one-second resolution are
all exactly as chapter 02 §2.6 described them. Three things are not.

There is no AMI event. The mutex version raises `DeadlockStart` once, guarded
by `if ((int) wait_time < 10)`, with a `DOCUMENTATION` block at `:322-341`. A
read or write lock that has been waited on for twenty seconds produces log
lines and nothing else — so an AMI client watching for `DeadlockStart` is
watching for mutexes only.

The subscript is unclamped. The mutex uses `ROFFSET` (`lock.h:249`), which
yields 0 when `reentrancy` is 0; these lines use `lt->reentrancy-1` directly.
With `reentrancy` at 0 that is `lt->file[-1]`, a read before the start of the
array. Reaching it requires something to be holding the lock while the tracking
array is empty, which chapter 02's cap discussion and §3.5 above both describe
routes to. It is a narrow path, but it is a narrow path into undefined
behaviour rather than into a misleading message.

And the two `__dump_backtrace` calls disagree with each other on purpose or by
accident — the first uses `[lt->reentrancy]`, the pending slot this thread just
wrote in phase 1, and the second uses `[lt->reentrancy-1]`, the holder's. The
mutex does the same thing (`:309`, `:315`), so if you are reading the output,
the first backtrace is the waiter and the second is the holder.

The doubled space in `was locked  here.` is in the source, in both the `rdlock`
copy at `:916` and the `wrlock` copy at `:1020`. The mutex's version at `:311`
has one space. If you are grepping logs, match on `was locked` and not on the
whole phrase.

## 3.9 The timed variants, and what copy-paste did to them

`timedrdlock` (`:1073-1153`) and `timedwrlock` (`:1155-1235`) differ from each
other in exactly three lines: the function signature, and the two
`pthread_rwlock_*` calls inside the body. Everything else is identical.

That is the problem, because two of those identical lines should not be.

```c
		ast_store_lock_info(AST_WRLOCK, filename, line, func, name, t, bt);
```

— `main/lock.c:1099`

That is `__ast_rwlock_timedrdlock` registering a *write* lock. A successful
`ast_rwlock_timedrdlock` appears in `core show locks` as `WRLOCK`, which will
make a reader look like a writer in exactly the situation — reconstructing who
was waiting for what — where the distinction decides whether the picture makes
sense. The two untimed paths and the two try paths all pass the right constant;
this one does not.

The same applies to the error message. Both timed functions log
`Error obtaining read lock` (`:1146` and `:1228`), so a failed
`ast_rwlock_timedwrlock` reports itself as a read lock failure.

Neither has much blast radius, because the timed entry points are barely used.
Nothing in the tree calls `ast_rwlock_timedrdlock` or `ast_rwlock_timedwrlock`
directly. They are reachable through four macros —
`AST_VECTOR_RW_RDLOCK_TIMED` and `AST_VECTOR_RW_WRLOCK_TIMED`
(`include/asterisk/vector.h:956`, `:967`) and `AST_RWLIST_TIMEDWRLOCK` and
`AST_RWLIST_TIMEDRDLOCK` (`include/asterisk/linkedlists.h:67`, `:93`) — and the
only callers of any of the four are three lines in `tests/test_vector.c`
(`:504-507`). The two list macros have no callers at all.

The other thing worth knowing about the timed variants is the fallback:

```c
#ifdef HAVE_PTHREAD_RWLOCK_TIMEDWRLOCK
	res = pthread_rwlock_timedrdlock(&t->lock, abs_timeout);
#else
	do {
		struct timeval _now;
		for (;;) {
			if (!(res = pthread_rwlock_tryrdlock(&t->lock))) {
				break;
			}
			_now = ast_tvnow();
			if (_now.tv_sec > abs_timeout->tv_sec || (_now.tv_sec == abs_timeout->tv_sec && _now.tv_usec * 1000 > abs_timeout->tv_nsec)) {
				break;
			}
			usleep(1);
		}
	} while (0);
#endif
```

— `main/lock.c:1103-1119`

Three notes. The guard symbol is `HAVE_PTHREAD_RWLOCK_TIMEDWRLOCK` — the
*write* one — and it gates the read call too; `configure` tests only for the
write variant and both functions take the answer. On the fallback path the
timeout exit leaves `res` holding whatever the last `pthread_rwlock_tryrdlock`
returned on a lock it could not get — not `ETIMEDOUT`, which is what the first
arm would return. So caller code that distinguishes a timeout from a contention
failure gets different answers on different platforms. And `usleep(1)` is a
1µs request, which in practice means the scheduler's minimum — a far tighter
spin than `DETECT_DEADLOCKS`' 200µs.

On glibc `HAVE_PTHREAD_RWLOCK_TIMEDWRLOCK` is defined, so the first arm is what
you get and the `#else` is dead.

## 3.10 The try variants

`tryrdlock` and `trywrlock` differ from each other in three lines — signature,
lock type, pthread call — and from the blocking pair in the ways you would
expect: no `canlog` at all, no `DETECT_DEADLOCKS` arm, and no error logging.

```c
#ifdef DEBUG_THREADS
	if (!res && lt) {
		ast_reentrancy_lock(lt);
		if (lt->reentrancy < AST_MAX_REENTRANCY) {
			lt->file[lt->reentrancy] = filename;
			lt->lineno[lt->reentrancy] = line;
			lt->func[lt->reentrancy] = func;
			lt->thread_id[lt->reentrancy] = pthread_self();
			lt->reentrancy++;
		}
		ast_reentrancy_unlock(lt);
		ast_mark_lock_acquired(t);
	} else if (lt) {
		ast_mark_lock_failed(t);
	}
```

— `main/lock.c:1267-1281`

The failure arm is `ast_mark_lock_failed(t)` rather than the blocking paths'
`ast_remove_lock_info(t, bt)`, which is the same choice the mutex trylock makes
(chapter 02 §2.7): a failed try is a fact worth keeping in the thread's lock
list, not an entry to retract. As with the mutex, the return value is the only
signal that a try failed, and these two functions have no `canlog` variable
because there is nothing for them to log.

Direct callers are few: 9 occurrences of `ast_rwlock_tryrdlock` and 3 of
`ast_rwlock_trywrlock` across `.c` files, excluding `third-party/` and the
standalone `utils/extconf.c`.

## 3.11 Where rwlocks actually come from

Counting the same way, direct calls in `.c` files come to 93
`ast_rwlock_rdlock`, 77 `ast_rwlock_wrlock` and 220 `ast_rwlock_unlock`, with
15 `AST_RWLOCK_DEFINE_STATIC`. That badly understates the subsystem, because
most rwlock use in Asterisk does not name an rwlock function.

Counting the same files the same way, but for the wrapper macros:
`AST_RWLIST_RDLOCK` 168, `AST_RWLIST_WRLOCK` 161, `AST_RWLIST_UNLOCK` 427,
`AST_VECTOR_RW_RDLOCK` 19, `AST_VECTOR_RW_WRLOCK` 15, `AST_VECTOR_RW_UNLOCK`
54. That is 363 acquisitions through the two macro families against 170
direct ones — so fewer than a third of the read-write acquisitions in Asterisk
name an rwlock function, and read-write locking is mostly a property of two
data structures, the read-write linked list and the locked vector, rather than
a thing modules reach for directly. If you are looking for the rwlock in a
subsystem, look for `AST_RWLIST_HEAD` and the vector types first.

There are also the scoped forms, which are worth knowing because they change
where the unlock is:

```c
#define SCOPED_RDLOCK(varname, lock) SCOPED_LOCK(varname, (lock), ast_rwlock_rdlock, ast_rwlock_unlock)
```

— `include/asterisk/lock.h:601`

with `SCOPED_WRLOCK` at `:606`. Four uses each. The unlock happens in a
cleanup attribute at scope exit, so a `SCOPED_RDLOCK` has no visible
`ast_rwlock_unlock` to grep for and no line number you can put in a bug report
— which is also true of `SCOPED_MUTEX` and `SCOPED_AO2LOCK`, and is the reason
a `core show locks` entry sometimes points at a function with no apparent
unlock.

One macro family that has no rwlock member: deadlock avoidance.
`DEADLOCK_AVOIDANCE` (`lock.h:383`, and the simpler non-`DEBUG_THREADS`
definition at `:486`) unlocks and relocks with `ast_mutex_unlock` and
`ast_mutex_lock`. It has two siblings, and each is tied to its own lock type
rather than to rwlocks: `AO2_DEADLOCK_AVOIDANCE` (`lock.h:339`, `:476`) goes
through `ao2_unlock` and `ao2_lock`, and `CHANNEL_DEADLOCK_AVOIDANCE`
(`lock.h:361`, `:481`) through `ast_channel_unlock` and `ast_channel_lock`.
There is no `RWLOCK_DEADLOCK_AVOIDANCE`. If you
need to back out of a lock-order problem while holding a read or write lock,
there is no helper for it, and chapter 06 will come back to why that shapes the
places rwlocks get used.

## 3.12 Reading an rwlock error message

Every message the rwlock half of `main/lock.c` can produce, in file order.
"Dead" means the enclosing `#if` cannot be true on a mainstream Linux build;
"unreachable" means the `if` cannot be true.

| Message | Line | What it means |
| --- | --- | --- |
| `Warning: rwlock '%s' is already initialized.` | `:725` | Dead (§3.3) |
| `Warning: rwlock '%s' is uninitialized.` | `:758`, `:807` | Dead (§3.3) — in `destroy` and `unlock` respectively |
| `Error destroying rwlock %s: %s` | `:771` | `pthread_rwlock_destroy` failed, almost always `EBUSY`: someone still holds it |
| `rwlock '%s' freed more times than we've locked!` | `:842` | Unreachable (§3.7) |
| `Error releasing rwlock: %s` | `:857` | `pthread_rwlock_unlock` failed. `DEBUG_THREADS` only — silent in a stock build (§3.7) |
| `Deadlock? waited %d sec for readlock '%s'?` | `:908` | `DETECT_DEADLOCKS` only. Repeats on a five-second grid; no AMI event (§3.8) |
| `'%s' was locked  here.` | `:916`, `:1020` | Needs `DETECT_DEADLOCKS` *and* `DEBUG_THREADS`: the `#ifdef DEBUG_THREADS` at `:910` is nested inside the `#if defined(DETECT_DEADLOCKS)` at `:899`. Follows the previous message. Two spaces. The holder's acquisition site, or garbage if tracking is empty (§3.8) |
| `Error obtaining read lock: %s` | `:961`, `:1146`, `:1228` | `DEBUG_THREADS` only. `:1228` is in `timedwrlock` and is mislabelled (§3.9) |
| `Deadlock? waited %d sec for writelock '%s'?` | `:1012` | As `:908`, for writers |
| `Error obtaining write lock: %s` | `:1064` | `DEBUG_THREADS` only, from `wrlock` |

Of the ten, two are dead and one is unreachable. Of the seven left, two need
`DETECT_DEADLOCKS`, one — `'%s' was locked  here.` — needs both
`DETECT_DEADLOCKS` and `DEBUG_THREADS`, and four need `DEBUG_THREADS`,
including `Error destroying rwlock`, whose `if (res)` is at `:770-773` inside
the `#ifdef DEBUG_THREADS` that opens at `:769`. A stock build emits none of
them.

## 3.13 Rules

| Rule | Why |
| --- | --- |
| Do not expect any log line from a stock build | The mutex's one stock-build unlock diagnostic has no rwlock counterpart (§3.7) |
| Always check the return of an unlock you care about | It is the only channel. In a non-`DEBUG_THREADS` build a failed `pthread_rwlock_unlock` is returned and nothing else (§3.7) |
| Never unlock an rwlock from a thread that did not lock it | There is no ownership check and no message; the tracking array silently stops matching reality (§3.7) |
| Read `core show locks` rwlock entries as a set, not a sequence | Out-of-order releases swap the top entry down, so the array is dense but unordered (§3.6) |
| Treat a `WRLOCK` entry from a timed acquisition with suspicion | `__ast_rwlock_timedrdlock` registers `AST_WRLOCK` (§3.9) |
| Do not read `core show locks` as the per-lock tracking array | They are two structures with two caps: 64 locks per thread, 10 holders per lock. Past ten holders the per-lock record stops growing while `core show locks` stays complete (§3.5) |
| Do not rely on writer preference you have not verified | `init` requests `PTHREAD_RWLOCK_PREFER_WRITER_NP`; whether your `pthread` honours it is a library question, and static rwlocks that skip `init` never request it at all (§3.3) |
| Do not expect `DeadlockStart` for a read or write lock | The AMI event exists only in the mutex path (§3.8) |
| Do not reach for a deadlock-avoidance macro while holding an rwlock | There isn't one; all three are mutex-based (§3.11) |
| Mind the argument order if you ever call a `__ast_rwlock_*` function directly | The lock pointer comes before the name, the opposite of the mutex family. The compiler will flag a swap, but only if you read the warning (§3.2) |

## Next

[Chapter 04](04-condition-variables.md) covers condition variables: a bare
typedef with no tracking of its own, four entry points that are pure
pass-throughs, and the one place in `main/lock.c` that takes a mutex's tracking
record away and gives it back.
