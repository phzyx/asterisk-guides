# 2. Mutexes

Every `file:line` citation in this guide was verified against the `24` branch.
Newer releases and other branches may differ — line numbers drift — so if you
are reading this against a different tree, treat the citations as approximate
and re-grep for the symbol name.

Chapter 01 stopped at the function signature. This chapter goes through the five
mutex entry points in `main/lock.c` statement by statement: what each one does
to the pthread object, what it does to the tracking state, which parts of it
exist in your build, and what the error messages mean when you find one in a
log.

The reason to read it is diagnostic. Almost everything in this chapter is
invisible in a stock build — and the handful of things that are *not* invisible
are the ones you will actually meet.

## 2.1 Three phases, and which of them exist in your build

Every acquisition entry point has the same shape:

1. **Before** the pthread call: find or allocate this lock's tracking record,
   capture a backtrace, and register with the current thread's lock list as
   *pending*.
2. **The pthread call itself**, which in the normal build is a single
   `pthread_mutex_lock`.
3. **After** it returns: push the call site onto the reentrancy stack and mark
   the thread's list entry as *acquired* — or, if the acquisition failed, undo
   phase 1.

Phases 1 and 3 are entirely inside `#ifdef DEBUG_THREADS`. In a stock build
`__ast_pthread_mutex_lock` is a function that takes five arguments, ignores
four of them, and calls `pthread_mutex_lock`. That is the whole function. The
compiler can see that too, which is why the wrapper is affordable at 852 call
sites.

Four build symbols decide what you get, and they are independent menuselect
options rather than levels of one setting:

| Symbol | Menuselect | Effect on this chapter |
|---|---|---|
| `DEBUG_THREADS` | Compiler Flags → "Enable Thread Debugging" | Compiles phases 1 and 3, the reentrancy stack, every `log_mutex_error` and the per-thread lock list behind `core show locks` |
| `DETECT_DEADLOCKS` | Compiler Flags - Development → "Detect Deadlocks" | Replaces the blocking acquire in phase 2 with a polling loop that reports waits (§2.6). Does *not* require `DEBUG_THREADS` |
| `THREAD_CRASH` | Compiler Flags - Development → "Crash on mutex errors" | Turns `DO_THREAD_CRASH` from a no-op into a deliberate null dereference |
| `DO_CRASH` | Compiler Flags - Development → "Crash on fatal errors" | Makes a failed tracking allocation `abort()` rather than silently degrade |

`DO_THREAD_CRASH` is worth seeing, because it appears 22 times in `main/lock.c`
and does nothing in almost every build:

```c
#ifdef DEBUG_THREADS

#ifdef THREAD_CRASH
#define DO_THREAD_CRASH do { *((int *)(0)) = 1; } while(0)
#else
#define DO_THREAD_CRASH do { } while (0)
#endif
```

— `include/asterisk/lock.h:251-257`

Two things to take from that. It is defined inside `#ifdef DEBUG_THREADS`, so
the 22 call sites are all in debug-only code anyway; and even in a debug build
it is a no-op unless you also asked for `THREAD_CRASH`. When it *is* live it
writes to address zero rather than calling `abort()`, which gets you a
`SIGSEGV` with the offending `__ast_*` frame on the stack instead of an
`abort()` frame — deliberately, because the point is the backtrace.

`DETECT_DEADLOCKS` not implying `DEBUG_THREADS` is a real configuration, not a
theoretical one. Built that way you get the wait detection in §2.6 and the
`DeadlockStart` AMI event, but no reentrancy stack — so the report tells you a
thread has been waiting 20 seconds and cannot tell you who is holding the lock.

## 2.2 `log_mutex_error`, and why `logger.c` is special

Almost every diagnostic in the file goes through one macro:

```c
#if defined(DEBUG_THREADS) || defined(DETECT_DEADLOCKS)
#define log_mutex_error(canlog, ...) \
	do { \
		if (canlog) { \
			ast_log(LOG_ERROR, __VA_ARGS__); \
		} else { \
			fprintf(stderr, __VA_ARGS__); \
		} \
	} while (0)
#endif
```

— `main/lock.c:57-66`

And `canlog` is computed the same way in every entry point:

```c
#if defined(DETECT_DEADLOCKS) || defined(DEBUG_THREADS)
	int canlog = t->flags.tracking && strcmp(filename, "logger.c");
#endif
```

— `main/lock.c:260-262`

That exact line appears 12 times in the file, with two further spellings of
the same expression elsewhere, and it carries two separate decisions.

The `strcmp` is the obvious one and the important one: the logger has its own
mutexes, so reporting a lock error *from* `logger.c` through `ast_log` would
take a logger lock while diagnosing a logger lock. The test uses the captured
`__FILE__` — which chapter 01 noted is a compile-time constant at the call site
— so it costs a string compare against a literal and no lookup. Errors from the
logger's own locks go to `stderr` and are the one class of lock error you will
only see on a terminal.

The `t->flags.tracking` half is easy to misread. A lock created with
`ast_mutex_init_notracking` has `tracking` clear, and `canlog` is therefore 0
*regardless of the file*, so **every error on a notracking lock goes to
`stderr`**. That is intentional — notracking locks are the ones used by the
logging and memory-management substrate, where going through `ast_log` is
exactly the recursion being avoided — but it means "I saw nothing in the log"
is not evidence that nothing went wrong.

## 2.3 Initialisation

`__ast_pthread_mutex_init` is short, and the half that matters is the half that
is conditional:

```c
#if defined(DEBUG_THREADS) || defined(DETECT_DEADLOCKS)
	t->track = NULL;
	t->flags.tracking = tracking;
	t->flags.setup = 0;
#endif /* DEBUG_THREADS */

	pthread_mutexattr_init(&attr);
	pthread_mutexattr_settype(&attr, AST_MUTEX_KIND);
	res = pthread_mutex_init(&t->mutex, &attr);
	pthread_mutexattr_destroy(&attr);

	return res;
```

— `main/lock.c:163-174`

Note the condition: this is the arm of chapter 01's `#if` that gives the fields
their short names, and it is the same condition. In a stock build those three
assignments do not exist, which is consistent — in a stock build nothing ever
reads `_track` or `_flags` either, so leaving them at whatever the allocator
produced is harmless. It also means **a heap-allocated `ast_mutex_t` in a stock
build has uninitialised tracking fields for its entire lifetime**, and that is
fine right up until someone adds code that reads them unconditionally.

`tracking` arrives as the literal 1 or 0 from whichever of the two macro
spellings the caller used (chapter 01, §1.1), and `setup` starts clear, which
is what makes the allocation in §2.4 lazy.

There is one more block, which on most trees is not compiled at all:

```c
#if defined(DEBUG_THREADS) && defined(AST_MUTEX_INIT_W_CONSTRUCTORS) && \
	defined(CAN_COMPARE_MUTEX_TO_INIT_VALUE)
	if ((t->mutex) != ((pthread_mutex_t) PTHREAD_MUTEX_INITIALIZER)) {
		int canlog = tracking && strcmp(filename, "logger.c");

		log_mutex_error(canlog, "%s line %d (%s): NOTICE: mutex '%s' is already initialized.\n",
				   filename, lineno, func, mutex_name);
		DO_THREAD_CRASH;
		return EBUSY;
	}
#endif /* AST_MUTEX_INIT_W_CONSTRUCTORS */
```

— `main/lock.c:151-161`

`CAN_COMPARE_MUTEX_TO_INIT_VALUE` is a `configure` test for whether
`pthread_mutex_t` can be compared with `==` to `PTHREAD_MUTEX_INITIALIZER` at
all — on glibc it cannot, and `autoconfig.h` leaves the symbol undefined. The
same pair of symbols guards two more detection blocks, in `destroy` (`:186`)
and in `unlock` (`:463`), which catch destroying or unlocking an uninitialised
mutex. The third appearance of the pair, at `:234`, is not a check at all — it
guards the `lt_cleanup:` label, which only exists when those blocks do.

The other half of the attribution is `AST_MUTEX_INIT_W_CONSTRUCTORS`, and that
one is not a `configure` result at all:

```c
#if (defined(SOLARIS) || defined(BSD))
#define AST_MUTEX_INIT_W_CONSTRUCTORS
#endif /* SOLARIS || BSD */
```

— `include/asterisk/lock.h:76-78`

So on Linux *both* guards fail independently, and the double-init and
use-before-init checks in this file are dead code. Do not go looking for those
messages.

## 2.4 `ast_get_reentrancy`: one global lock for every first use

Chapter 01 said the `ast_lock_track` is allocated lazily. This is the function
that does it, and it is the most interesting forty lines in the file:

```c
static inline struct ast_lock_track *ast_get_reentrancy(struct ast_lock_track **plt,
	struct ast_lock_track_flags *flags, int no_setup)
{
	pthread_mutexattr_t reentr_attr;
	struct ast_lock_track *lt;

	if (!flags->tracking || flags->setup) {
		return *plt;
	}

	pthread_mutex_lock(&reentrancy_lock.mutex);

	if (*plt) {
		pthread_mutex_unlock(&reentrancy_lock.mutex);
		return *plt;
	}

	if (no_setup) {
		pthread_mutex_unlock(&reentrancy_lock.mutex);
		return NULL;
	}

	lt = *plt = ast_std_calloc(1, sizeof(*lt));
	if (!lt) {
		fprintf(stderr, "%s: Failed to allocate lock tracking\n", __func__);
#if defined(DO_CRASH) || defined(THREAD_CRASH)
		abort();
#else
		pthread_mutex_unlock(&reentrancy_lock.mutex);
		return NULL;
#endif
	}

	pthread_mutexattr_init(&reentr_attr);
	pthread_mutexattr_settype(&reentr_attr, AST_MUTEX_KIND);
	pthread_mutex_init(&lt->reentr_mutex, &reentr_attr);
	pthread_mutexattr_destroy(&reentr_attr);

	flags->setup = 1;
	pthread_mutex_unlock(&reentrancy_lock.mutex);
	return lt;
```

— `main/lock.c:87-127`

The whole function sits inside a single `#ifdef DEBUG_THREADS` block
(`:84-143`), together with the static lock below and its one-line counterpart
`delete_reentrancy_cs` (`:130-141`). In a stock build there is no lazy
allocation because there is no allocation at all.

Read the fast path first. `if (!flags->tracking || flags->setup) return *plt;`
is one load of a bitfield and one return, and it is the path taken by every
acquisition after the first. For a notracking lock it returns `NULL` forever,
which is why `lt` is tested before every use throughout the file — a `NULL`
`lt` is the normal state of a notracking lock, not an error.

Then look at what guards the slow path:

```c
AST_MUTEX_DEFINE_STATIC(reentrancy_lock);
```

— `main/lock.c:85`

A single file-scope mutex, taken through the **raw** `pthread_mutex_lock` on its
`.mutex` member rather than through `ast_mutex_lock`. It has to be raw: going
through the wrapper would call `ast_get_reentrancy` on `reentrancy_lock`
itself, from inside `ast_get_reentrancy`. (It is a static definition, so it is
also one more place a raw pthread call is correct — inside `main/lock.c`, which
chapter 01 §1.7 already accounted for.)

Being global and process-wide, that lock serialises the *first* tracked
operation on every mutex in the process. It is not on the hot path, because
`flags.setup` short-circuits before it, but it does mean that a debug build
starting up has a convoy point: a thousand threads each touching a fresh lock
for the first time queue behind one mutex and one `calloc` each. This is part of
why a `DEBUG_THREADS` build is slow in a way that is not uniform — it is slow
at the start of each lock's life, not throughout.

Three details that matter when reading the rest of the file:

- **The double check is a double-checked lock, and the second half tests a
  different variable.** The fast path tests `flags->setup`; under the global
  lock it tests `*plt`. That is deliberate: `setup` is set *after* the
  `reentr_mutex` is initialised, so a thread that sees `setup` set knows the
  record is fully usable. A thread that only sees `*plt` non-NULL — which
  cannot happen through this function, but could through a static initialiser
  — gets the record without waiting. The `volatile` on `setup` that chapter 01
  noted is for the unlocked read in the fast path.
- **`no_setup` means "tell me if there is one, do not make one".** Exactly one
  caller passes 1, and it is `__ast_pthread_mutex_destroy` (`:183`). Destroying
  a lock that was never tracked should not allocate a tracking record in order
  to throw it away.
- **An allocation failure degrades rather than fails.** Without `DO_CRASH` or
  `THREAD_CRASH` the function returns `NULL`, and `flags->setup` stays clear —
  so every subsequent operation on that lock retries the allocation, taking the
  global lock each time. The lock still works. It is simply untracked, and
  slower than an untracked lock, until an allocation succeeds.

The teardown half is unremarkable except for the ordering:

```c
static inline void delete_reentrancy_cs(struct ast_lock_track **plt)
{
	struct ast_lock_track *lt;

	if (*plt) {
		lt = *plt;
		*plt = NULL;

		pthread_mutex_destroy(&lt->reentr_mutex);
		ast_std_free(lt);
	}
}
```

— `main/lock.c:130-141`

`*plt` is cleared *before* the nested mutex is destroyed, so a concurrent
`ast_get_reentrancy` on a lock being destroyed sees `NULL` rather than a pointer
to a destroyed mutex. It will then happily allocate a fresh record for a mutex
that is going away, which leaks — but you got there by destroying a lock while
another thread was using it, and the leak is the least of your problems.

Note also that nothing uses the `reentrancy_lock` here; `delete_reentrancy_cs`
races with `ast_get_reentrancy`'s slow path by construction. Destruction is
assumed to be the last operation on a lock.

## 2.5 `__ast_pthread_mutex_lock`, phase by phase

Phase 1, all of it conditional:

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
		ast_store_lock_info(AST_MUTEX, filename, lineno, func, mutex_name, t, bt);
	}
#endif /* DEBUG_THREADS */
```

— `main/lock.c:264-286`

The comment is the lesson, and the bug it refers to is instructive:
`backtrace()` takes locks of its own, so capturing into a local `tmp` first and
copying under `lt->reentr_mutex` keeps the two lock domains from nesting.
`HAVE_BKTR` is on by default where `configure` finds `execinfo.h`, which on
glibc it does.

Note what the backtrace is written to: `lt->backtrace[lt->reentrancy]`, the
slot *above* the current top of stack, before anything has been pushed. At this
point in the function that slot belongs to the acquisition that has not
happened yet. It matters in §2.6.

Then `ast_store_lock_info` registers the attempt on the current thread's lock
list, marked **pending**. Chapter 05 covers that list; the point here is that
the registration happens before the blocking call, which is the only reason
`core show locks` can show you a thread waiting rather than only threads
holding.

Phase 2 in a normal build is one line of live code and four of dead:

```c
#else /* !DETECT_DEADLOCKS || !DEBUG_THREADS */
#ifdef	HAVE_MTX_PROFILE
	ast_mark(mtx_prof, 1);
	res = pthread_mutex_trylock(&t->mutex);
	ast_mark(mtx_prof, 0);
	if (res)
#endif
	res = pthread_mutex_lock(&t->mutex);
#endif /* !DETECT_DEADLOCKS || !DEBUG_THREADS */
```

— `main/lock.c:351-359`

The `if (res)` inside the `#ifdef` and the `res = pthread_mutex_lock` outside it
are one statement split across a conditional — with `HAVE_MTX_PROFILE` the
blocking lock is only reached if the trylock failed; without it the blocking
lock is unconditional. It works, and it is the kind of construction that breaks
the moment someone adds a statement between the two lines.

Phase 3, the push:

```c
#ifdef DEBUG_THREADS
	if (lt && !res) {
		ast_reentrancy_lock(lt);
		if (lt->reentrancy < AST_MAX_REENTRANCY) {
			lt->file[lt->reentrancy] = filename;
			lt->lineno[lt->reentrancy] = lineno;
			lt->func[lt->reentrancy] = func;
			lt->thread_id[lt->reentrancy] = pthread_self();
			lt->reentrancy++;
		} else {
			log_mutex_error(canlog, "%s line %d (%s): '%s' really deep reentrancy!\n",
							   filename, lineno, func, mutex_name);
		}
		ast_reentrancy_unlock(lt);
		ast_mark_lock_acquired(t);
	} else if (lt) {
```

— `main/lock.c:361-376`

Four stores and an increment, under the nested mutex. Two of the four are
pointers — `filename` and `func` — and they are the compile-time constants from
the macro, so this is genuinely four word writes and no copying of strings; the
other two are an `int` and a `pthread_t`. `ast_mark_lock_acquired` then flips the thread's
list entry from pending to held.

The `else` branch is the cap case chapter 01 corrected: at depth 10 the
acquisition has already succeeded, the error is logged, and `reentrancy` is not
incremented. The eleventh acquisition is therefore invisible to tracking *and*
the matching unlock will decrement a count that was never incremented, which
is how one "really deep reentrancy!" message can be followed by a spurious
"freed more times than we've locked!" later. Treat the first message as the
real one.

Finally the failure path:

```c
#ifdef HAVE_BKTR
		if (lt->reentrancy) {
			ast_reentrancy_lock(lt);
			bt = &lt->backtrace[lt->reentrancy-1];
			ast_reentrancy_unlock(lt);
		} else {
			bt = NULL;
		}
#endif
		ast_remove_lock_info(t, bt);
	}
	if (res) {
		log_mutex_error(canlog, "%s line %d (%s): Error obtaining mutex: %s\n",
				   filename, lineno, func, strerror(res));
		DO_THREAD_CRASH;
	}
```

— `main/lock.c:377-392`

If `pthread_mutex_lock` returned an error, phase 1's registration has to be
undone — hence `ast_remove_lock_info`. The symmetry is the thing to remember:
`ast_store_lock_info` happens on every attempt, so exactly one of
`ast_mark_lock_acquired` or `ast_remove_lock_info` must happen on every return.

A blocking `pthread_mutex_lock` on a recursive mutex has very few ways to fail,
so "Error obtaining mutex" in a log usually means the `ast_mutex_t` is not a
valid mutex at all: freed memory, a `memcpy`'d lock, or a heap allocation never
passed to `ast_mutex_init`.

## 2.6 `DETECT_DEADLOCKS` replaces the blocking acquire

With `DETECT_DEADLOCKS` defined, phase 2 is not a `pthread_mutex_lock` at all.
It is a poll:

```c
#if defined(DETECT_DEADLOCKS)
	{
		time_t seconds = time(NULL);
		time_t wait_time, reported_wait = 0;
		do {
#ifdef	HAVE_MTX_PROFILE
			ast_mark(mtx_prof, 1);
#endif
			res = pthread_mutex_trylock(&t->mutex);
#ifdef	HAVE_MTX_PROFILE
			ast_mark(mtx_prof, 0);
#endif
			if (res == EBUSY) {
				wait_time = time(NULL) - seconds;
				if (wait_time > reported_wait && (wait_time % 5) == 0) {
					log_mutex_error(canlog, "%s line %d (%s): Deadlock? waited %d sec for mutex '%s'?\n",
							   filename, lineno, func, (int) wait_time, mutex_name);
```

— `main/lock.c:288-304`

and it ends:

```c
						manager_event(EVENT_FLAG_SYSTEM, "DeadlockStart",
							"Mutex: %s\r\n",
							mutex_name);
					}
				}
				usleep(200);
			}
		} while (res == EBUSY);
	}
```

— `main/lock.c:342-350`

Four properties of this loop are worth knowing before you turn the flag on.

It spins with a 200-microsecond sleep, so a contended lock costs a syscall and
a context switch per 200µs of waiting instead of one futex wait. On a busy
system that is a large and *uneven* slowdown: uncontended locks are unaffected,
contended ones get much worse, which changes the timing of exactly the race you
were trying to catch.

It reports on a five-second grid, not after five seconds of waiting. The test
is `wait_time > reported_wait && (wait_time % 5) == 0` against a one-second
`time()` resolution, so a wait that begins late in a second can cross a
multiple of five in under a second and report almost immediately, and a wait
that is unlucky in the other direction can go nine seconds unreported.
`reported_wait` prevents the same multiple being reported twice; it does not
make the grid relative to the start of the wait.

It emits the `DeadlockStart` AMI event once per wait, guarded by
`if ((int) wait_time < 10)` with the comment "Only emit an event when a deadlock
starts, not every 5 seconds". Combined with the grid, that means the event fires
on the 5-second report and not the 10-, 15- or 20-second ones. The event's own
documentation (inline at `:322-341`) says delivery is attempted but not
guaranteed, "for example if the manager itself is deadlocked", which is honest
about the circumstances.

And the diagnostic it prints names *two* places, in an order that is easy to get
backwards. Under `DEBUG_THREADS` it dumps `lt->backtrace[lt->reentrancy]`
followed by `lt->file[ROFFSET]` and `lt->backtrace[ROFFSET]` (`:305-319`). The
first is the slot phase 1 wrote — **your** backtrace, the waiter's. `ROFFSET`
is the top of the stack — the **holder's** file, line and function. So the
report reads "here is where I am stuck" then "'name' was locked here", and the
second is the one you want. Without `DEBUG_THREADS` there is no `lt` and you
get neither.

## 2.7 `trylock`, and the one place it diverges

`__ast_pthread_mutex_trylock` is `__ast_pthread_mutex_lock` with phase 2
replaced by a bare `pthread_mutex_trylock` (`:428`) and no `DETECT_DEADLOCKS`
variant — there is nothing to detect, since it does not wait. Phase 1 is
the same block but for one extra line, the `int canlog` declaration at `:405`.
Phase 3 differs in exactly one branch:

```c
#ifdef DEBUG_THREADS
	if (lt && !res) {
		ast_reentrancy_lock(lt);
		if (lt->reentrancy < AST_MAX_REENTRANCY) {
			lt->file[lt->reentrancy] = filename;
			lt->lineno[lt->reentrancy] = lineno;
			lt->func[lt->reentrancy] = func;
			lt->thread_id[lt->reentrancy] = pthread_self();
			lt->reentrancy++;
		} else {
			log_mutex_error(canlog, "%s line %d (%s): '%s' really deep reentrancy!\n",
					   filename, lineno, func, mutex_name);
		}
		ast_reentrancy_unlock(lt);
		ast_mark_lock_acquired(t);
	} else if (lt) {
		ast_mark_lock_failed(t);
	}
```

— `main/lock.c:430-447`

Where `lock` calls `ast_remove_lock_info` on failure, `trylock` calls
`ast_mark_lock_failed`. The difference is that a failed `trylock` is not an
error and the information is worth keeping: the entry stays on the thread's
lock list, flagged as a failed attempt, so `core show locks` can show you that a
thread tried and gave up. A failed blocking lock, by contrast, means something
is broken, and the entry is removed.

There is also no `if (res)` error log at the end. `EBUSY` from a trylock is the
expected outcome, not a fault, so there is nothing to report — which means a
trylock that fails for a *real* reason, like an invalid mutex, fails silently
with only its return value to go on. Sixteen call sites in the tree use
`ast_mutex_trylock`; all of them have to check it.

## 2.8 `unlock` is the only entry point that checks you

Phase 1 of unlock is not registration, it is validation, and it is the richest
error handling in the file:

```c
	lt = ast_get_reentrancy(&t->track, &t->flags, 0);
	if (lt) {
		ast_reentrancy_lock(lt);
		if (lt->reentrancy && (lt->thread_id[ROFFSET] != pthread_self())) {
			log_mutex_error(canlog, "%s line %d (%s): attempted unlock mutex '%s' without owning it!\n",
					   filename, lineno, func, mutex_name);
			log_mutex_error(canlog, "%s line %d (%s): '%s' was locked here.\n",
					   lt->file[ROFFSET], lt->lineno[ROFFSET], lt->func[ROFFSET], mutex_name);
#ifdef HAVE_BKTR
			__dump_backtrace(&lt->backtrace[ROFFSET], canlog);
#endif
			DO_THREAD_CRASH;
		}
```

— `main/lock.c:472-484`

This is the single most useful message in the subsystem. A recursive
`pthread_mutex_t` returns `EPERM` from an unlock by a non-owner and tells you
nothing else; the tracking stack turns that into two lines naming both the
unlock site and the lock site. The `lt->reentrancy &&` guard matters for a
different reason than it looks: `ROFFSET` at depth 0 does not fail, it silently
indexes slot 0 — the same slot depth 1 uses. Without the guard, every unlock of
an unheld lock would read whatever slot 0 currently holds and report it as the
lock site. In practice that is `(null)` and `0`, because the unlock path below
clears the slot it pops, but the point is that the expression has no
out-of-range case to trip on.

Note that this check does not stop the unlock. It logs, optionally crashes, and
falls through. If `THREAD_CRASH` is off you get the message *and* the
`pthread_mutex_unlock` attempt.

Then the underflow check:

```c
		if (--lt->reentrancy < 0) {
			log_mutex_error(canlog, "%s line %d (%s): mutex '%s' freed more times than we've locked!\n",
					   filename, lineno, func, mutex_name);
			lt->reentrancy = 0;
		}
```

— `main/lock.c:486-490`

The decrement happens first and unconditionally, so the counter is already
negative when the test runs and is clamped back to 0. "Freed more times than
we've locked" means the tracking stack went negative — which is usually a real
unbalanced unlock, but as §2.5 noted can also be the delayed consequence of a
`really deep reentrancy!` ten acquisitions earlier.

And the pop:

```c
		if (lt->reentrancy < AST_MAX_REENTRANCY) {
			lt->file[lt->reentrancy] = NULL;
			lt->lineno[lt->reentrancy] = 0;
			lt->func[lt->reentrancy] = NULL;
			lt->thread_id[lt->reentrancy] = 0;
		}

#ifdef HAVE_BKTR
		if (lt->reentrancy) {
			bt = &lt->backtrace[lt->reentrancy - 1];
		}
#endif
		ast_reentrancy_unlock(lt);

		ast_remove_lock_info(t, bt);
```

— `main/lock.c:492-506`

The slot is cleared rather than merely abandoned, which is why `ROFFSET` on a
fully released lock reads `NULL` and `0` rather than the last holder. And the
`AST_MAX_REENTRANCY` guard on the pop cannot fail: `reentrancy` has just been
decremented, so it is at most 9 against a cap of 10. There is no mutex unlock
path that pops without clearing. The `bt` handed to
`ast_remove_lock_info` is the *new* top of stack: if you are unlocking from
depth 3 to depth 2, the thread still holds the lock and the entry that remains
should point at the depth-2 acquisition.

Then the one thing in this entire chapter that happens in a stock build:

```c
	res = pthread_mutex_unlock(&t->mutex);

#ifndef DEBUG_THREADS
	/*
	 * An unlock failure (e.g. EPERM) means the mutex may stay locked.
	 * The strcmp only runs when the unlock failed, so there is no
	 * performance impact on normal unlocks. It avoids recursion if the
	 * logger's own mutex fails to unlock.
	 */
	if (res && strcmp(filename, "logger.c")) {
		ast_log(LOG_ERROR, "Failed to unlock mutex '%s' at %s:%d (%s): %s\n",
			mutex_name, filename, lineno, func, strerror(res));
	}
#endif
```

— `main/lock.c:510-523`

`#ifndef DEBUG_THREADS` — the only one in the file. In a stock build, a failed
unlock is the single mutex fault Asterisk will tell you about, and it tells you
using the call-site descriptor that chapter 01 said was otherwise unused in
such a build. The comment explains the ordering: the `strcmp` is after `res`
in the `&&` so it is only evaluated on failure.

If you have ever wondered why the four descriptor arguments are worth passing
at 1,269 unlock sites in a build that mostly ignores them, this is the answer.
This one message is what they are for.

## 2.9 Destruction

`__ast_pthread_mutex_destroy` is the only entry point that asks for the
tracking record without creating one:

```c
#ifdef DEBUG_THREADS
	struct ast_lock_track *lt = ast_get_reentrancy(&t->track, &t->flags, 1);
	int canlog = t->flags.tracking && strcmp(filename, "logger.c");
```

— `main/lock.c:182-184`

The `1` is `no_setup`. One other caller passes 1 — `__ast_rwlock_destroy`
(`:753`), for the same reason — and everything else in the file passes 0.

Before destroying, it checks whether anyone holds the lock, by trying to take
it:

```c
	res = pthread_mutex_trylock(&t->mutex);
	switch (res) {
	case 0:
		pthread_mutex_unlock(&t->mutex);
		break;
	case EINVAL:
		log_mutex_error(canlog, "%s line %d (%s): Error: attempt to destroy invalid mutex '%s'.\n",
				  filename, lineno, func, mutex_name);
		break;
	case EBUSY:
		log_mutex_error(canlog, "%s line %d (%s): Error: attempt to destroy locked mutex '%s'.\n",
				   filename, lineno, func, mutex_name);
		if (lt) {
			ast_reentrancy_lock(lt);
			log_mutex_error(canlog, "%s line %d (%s): Error: '%s' was locked here.\n",
				    lt->file[ROFFSET], lt->lineno[ROFFSET], lt->func[ROFFSET], mutex_name);
#ifdef HAVE_BKTR
			__dump_backtrace(&lt->backtrace[ROFFSET], canlog);
#endif
			ast_reentrancy_unlock(lt);
		}
		break;
	}
#endif /* DEBUG_THREADS */
```

— `main/lock.c:202-225`

The `case 0` is the acquire-and-release probe: if the trylock succeeded, nobody
held it, and the lock is immediately given back so that
`pthread_mutex_destroy` sees an unheld mutex. `EBUSY` gets the "was locked
here" treatment, which again is the whole payoff of the reentrancy stack — the
`ROFFSET` entry names the acquisition that is still outstanding.

Note the gap. Because mutexes are recursive, a thread destroying a lock *it
holds itself* gets `0` from the trylock, not `EBUSY`. The probe succeeds, the
matching unlock drops it back to the depth it was at, and the "attempt to
destroy locked mutex" message never appears. Self-held destruction is the one
case this check cannot see.

Then, after the `pthread_mutex_destroy`, the cleanup:

```c
#endif
	if (lt) {
		ast_reentrancy_lock(lt);
		lt->file[0] = filename;
		lt->lineno[0] = lineno;
		lt->func[0] = func;
		lt->reentrancy = 0;
		lt->thread_id[0] = 0;
#ifdef HAVE_BKTR
		memset(&lt->backtrace[0], 0, sizeof(lt->backtrace[0]));
#endif
		ast_reentrancy_unlock(lt);
		delete_reentrancy_cs(&t->track);
	}
```

— `main/lock.c:236-249`

Those six statements — five assignments and a `memset` of the backtrace slot
(`:245`) — write the destroy site into slot 0 of a record that
`delete_reentrancy_cs` frees three lines later (`:248`). They are not useless — they are what a debugger or a
concurrent reader sees in the window between the reset and the free, and
writing the destroy site there means that window shows "destroyed here" rather
than a stale acquisition. But they are, strictly, stores to about-to-be-freed
memory, and reading them as meaningful bookkeeping will mislead you.

The `lt_cleanup:` label just above (`:235`) is the target of the `goto` in the
uninitialised-mutex check, and it exists only when
`CAN_COMPARE_MUTEX_TO_INIT_VALUE` does — which, per §2.3, is not on glibc.

## 2.10 Reading a mutex error message

Every message in this chapter has the same prefix, and it is worth being able
to parse it at a glance. `%s line %d (%s)` is `filename`, `lineno`, `func` —
the descriptor from the *call site that is reporting*, not the location of the
code printing it. A message that says `chan_sip.c line 4812 (pvt_hangup)` was
produced inside `main/lock.c`, about a call made in `chan_sip.c`.

When a message names two locations, the first is always the reporting call site
and the second, introduced by "was locked here", is always the `ROFFSET` entry
— the top of the tracking stack, which is the holder.

The messages, and what each one actually means:

| Message | Where | What it means |
|---|---|---|
| `attempted unlock mutex '%s' without owning it!` | `:476` | The top of the tracking stack belongs to another thread. The second line names the real holder. Nearly always a genuine bug |
| `mutex '%s' freed more times than we've locked!` | `:487` | The tracking depth went negative. Usually an unbalanced unlock; can be the echo of an earlier `really deep reentrancy!` |
| `'%s' really deep reentrancy!` | `:371`, `:440` | Eleventh recursive acquisition of one mutex by one thread. The lock *was* taken; tracking just stopped following it |
| `Error obtaining mutex: %s` | `:389` | `pthread_mutex_lock` itself failed. Suspect an invalid or freed `ast_mutex_t` |
| `Error releasing mutex: %s` | `:527` | Debug-build counterpart of the stock `Failed to unlock mutex` |
| `Failed to unlock mutex '%s' at %s:%d (%s): %s` | `:520` | The stock build's only mutex diagnostic. The mutex may still be held |
| `attempt to destroy locked mutex '%s'.` | `:212` | Someone else holds it. Does not fire if *you* hold it |
| `attempt to destroy invalid mutex '%s'.` | `:208` | `trylock` returned `EINVAL`; the memory is probably not a mutex |
| `Deadlock? waited %d sec for mutex '%s'?` | `:303` | `DETECT_DEADLOCKS` only. A five-second grid, not a five-second timer |
| `Failed to allocate lock tracking` | `:111` | `calloc` failed in `ast_get_reentrancy`. Goes to `stderr`, never the log |
| `ast_reentrancy_lock failed: '%s' (%d)` | `include/asterisk/lock.h:456` | The nested tracking mutex failed. Goes to `stderr` |

Two entries on that list bypass `log_mutex_error` entirely and
`fprintf(stderr, ...)` directly, because they are reporting that the tracking
machinery itself is broken and cannot be trusted to log.

## 2.11 Rules

| Rule | Why |
|---|---|
| Check the return of `ast_mutex_trylock`, always | A failed trylock logs nothing at all (§2.7), so the return value is the only signal — including when it failed for a reason other than contention |
| Do not treat "nothing in the log" as "no lock errors" | Notracking locks send every error to `stderr` rather than the logger, as does anything reported from `logger.c` (§2.2) |
| Expect `really deep reentrancy!` to be followed by nonsense | Past depth 10 the push is skipped but the pop is not, so subsequent accounting is wrong. The first message is the real one (§2.5) |
| Never destroy a lock you hold | The `EBUSY` probe in destroy cannot see self-held locks, because the mutex is recursive (§2.9) |
| Do not leave `DETECT_DEADLOCKS` on in production | Phase 2 becomes a 200µs poll, which slows contended locks specifically and changes the timing of the race you are chasing (§2.6) |
| Do not add a read of `track` or `flags` outside a `DEBUG_THREADS` guard | In a stock build `__ast_pthread_mutex_init` never initialises them (§2.3), and `main/astobj2.c` does not even allocate space for them (chapter 01, §1.3) |
| Keep `ast_store_lock_info` paired | Every attempt registers; exactly one of `ast_mark_lock_acquired`, `ast_mark_lock_failed` or `ast_remove_lock_info` must run before every return (§2.5, §2.7) |

## Next

[Chapter 03](03-read-write-locks.md) covers the nine read-write lock entry
points: how much of this chapter they reuse verbatim, why the reentrancy array
stops being a stack, and which of the diagnostics above have no rwlock
counterpart.
