# Threading, Locking and Deadlock Avoidance in Asterisk — A Source-Level Guide

Every `file:line` citation in this guide was verified against the `24` branch.
Newer releases and other branches may differ — line numbers drift — so if you
are reading this against a different tree, treat the citations as approximate
and re-grep for the symbol name.

## Who this is for

A C developer working inside the Asterisk source tree who has to hold a lock
and wants to know what happens when they do. Not "should I lock this" — the
callers' guides answer that — but what `ast_mutex_lock(&p->lock)` compiles
into, where the information behind `core show locks` is stored, why five
macros in the same header are each defined twice with different semantics, and
what `DEADLOCK_AVOIDANCE` actually does to your lock.

The lock wrappers are not optional and not avoidable. Counting call sites in
`.c` files and excluding `third-party/` and the standalone `utils/extconf.c`,
there are 852 `ast_mutex_lock` sites, 1,269 `ast_mutex_unlock`, 16
`ast_mutex_trylock`, 170 rwlock acquisitions and 151 condition waits, spread
across 152 files; a further 31 headers name the lock types or operations without
calling one. There are 75 `AST_MUTEX_DEFINE_STATIC` and 15
`AST_RWLOCK_DEFINE_STATIC` declarations, and 130 thread creations — 129 through
the four `ast_pthread_create*` macros, plus one site that calls the underlying
`ast_pthread_create_stack` directly from a macro of its own
(`res/res_corosync.c:93`).

Using a raw pthread call is not merely discouraged, it is made difficult:
`lock.h` `#define`s the pthread names to deliberately unresolvable identifiers
(`:628-642`), so a stray `pthread_mutex_lock` fails to compile with an error
naming the wrapper you should have used. The files that need the real functions
take the names back with `#undef`, and the exceptions are informative rather
than sloppy — `main/lock.c` because it *is* the wrapper, `main/utils.c` because
it implements the tracking the wrapper feeds (two calls there carry the comment
`/* Intentionally not the wrapper */`, at `main/utils.c:1627` and `:1629`),
`main/threadstorage.c` because thread-local storage is the substrate tracking
is built on, `main/backtrace.c` which suppresses the header entirely with
`#define _ASTERISK_LOCK_H` (`main/backtrace.c:29-33`), and the
`AST_SPINLOCK_TYPE_PTHREAD_MUTEX` fallback in `include/asterisk/spinlock.h`
(`:404-433`), which never includes `lock.h` at all. Outside those, a raw
`pthread_mutex_lock` is a lock the core cannot see.

This guide goes *underneath* two chapters that already exist:

- [channels/03 — References and locking](../channels/03-references-and-locking.md)
  covers `CHANNEL_DEADLOCK_AVOIDANCE` from the caller's side: when to reach for
  it, its call sites, and the dangling-`else` hazard in the non-`DEBUG_THREADS`
  definition.
- [ao2/03 — Locking](../ao2/03-locking.md) covers the four ao2 lock modes, lock
  promotion, the `SCOPED_*` family and the `ast_cond_wait(cond, NULL)` trap.

Where those chapters tell you what to call, this one tells you what the call
does. It does not restate them; it links to them and keeps going down.

## The one-page mental model

There is one fact that explains almost every strange thing in this subsystem.
**Every lock operation in Asterisk is a macro that captures its own call site
and hands it to a function.** `ast_mutex_lock` is not a function you can take
the address of:

```
   your code                 include/asterisk/lock.h:196         main/lock.c:255
   ─────────                 ───────────────────────────         ───────────────

   ast_mutex_lock(&p->lock)
         │
         │  #define ast_mutex_lock(a) \
         └─▶   __ast_pthread_mutex_lock(__FILE__, __LINE__,
                                        __PRETTY_FUNCTION__, #a, a)
                    │          │          │             │     │     │
                    │          │          │             │     │     └─ the lock
                    │          │          │             │     └─ "&p->lock"
                    │          │          │             └─ "pvt_hangup"
                    │          │          └─ 4812
                    │          └─ "channels/chan_foo.c"
                    │
                    └────────────────────────────────▶ int __ast_pthread_mutex_lock(
                                                           const char *filename,
                                                           int lineno,
                                                           const char *func,
                                                           const char *mutex_name,
                                                           ast_mutex_t *t)
                                                       { ... pthread_mutex_lock(&t->mutex) ... }
```

Four of those five arguments exist only so that the lock can say where it was
taken. That is the whole reason `core show locks` can print a file, a line, a
function and the *source text* of the lock expression for every lock every
thread is holding — the preprocessor already knew all four at every one of the
852 call sites, so nothing has to be looked up at runtime.

Three consequences carry most of the weight:

1. **`ast_mutex_t` is a struct, not a typedef of `pthread_mutex_t`.** It is
   `struct ast_mutex_info` (`include/asterisk/lock.h:142-157`), whose first
   member is the real `pthread_mutex_t` and whose remaining members are
   tracking state. `ast_rwlock_t` is the same shape
   (`:164-179`). `ast_cond_t`, by contrast, *is* a bare typedef —
   `typedef pthread_cond_t ast_cond_t` (`:185`) — so condition variables carry
   no tracking at all. Chapter 01 and chapter 04.
2. **The tracking fields are present whether or not you built with
   `DEBUG_THREADS`, but they are *renamed* when you did not.** The `#if` at
   `:144-156` gives you `_track`/`_flags` in a normal build and `track`/`flags`
   in a debug build — same types, same order, same offsets, same `sizeof`. The
   rename is deliberate: it makes any accidental use of the fields a compile
   error without changing the ABI, so a module built one way can still hand a
   lock to a core built the other. Defining `DEBUG_THREADS_LOOSE_ABI` takes
   *neither* arm, shrinking the struct to a bare `pthread_mutex_t`; exactly one
   file in the tree does so, and it is not an exotic build but the default one.
   Chapter 01.
3. **Five macros in `lock.h` are defined twice, and the pairs are not
   equivalent.** The header splits at `#ifdef DEBUG_THREADS` (`:251`) and
   rejoins at `#else /* !DEBUG_THREADS */` (`:474`), and
   `AO2_DEADLOCK_AVOIDANCE` (`:339`/`:476`), `CHANNEL_DEADLOCK_AVOIDANCE`
   (`:361`/`:481`), `DEADLOCK_AVOIDANCE` (`:383`/`:486`), `DLA_UNLOCK`
   (`:417`/`:497`) and `DLA_LOCK` (`:436`/`:499`) each appear once on either
   side. Note which macros those are: the lock *operations* —
   `ast_mutex_lock` and friends at `:193-198`, the cond macros at `:208-213`,
   the rwlock macros at `:231-247` — sit outside the split and have one
   definition each. Four of the five pairs differ in syntax, not just
   behaviour: the non-debug `AO2_DEADLOCK_AVOIDANCE` (`:476-479`) and
   `CHANNEL_DEADLOCK_AVOIDANCE` (`:481-484`) are three bare statements with no
   `do { } while (0)` wrapper, and `DLA_UNLOCK`/`DLA_LOCK` are a *split macro*
   in the debug build — `DLA_UNLOCK` opens a `do {` that only `DLA_LOCK`
   closes — against two independent one-line expressions without it.
   Chapter 06.

Where the tracking state actually lives is worth knowing before you start:
nothing is stored in a global lock registry. Each lock points at a
`struct ast_lock_track` (`:118-128`) holding up to `AST_MAX_REENTRANCY` (10)
stacked file/line/function/thread tuples, because Asterisk mutexes are
recursive and the tenth re-entry needs to remember the ninth. Separately, each
*thread* keeps a `struct thr_lock_info` (`main/utils.c:996`) with room for
`AST_MAX_LOCKS` (64) locks, in a thread-local, and that is what
`ast_dump_locks()` (`main/utils.c:1473`) walks to produce `core show locks`.
Two structures, two directions: one asks "who holds this lock", the other asks
"what does this thread hold".

The asymmetry to hold on to is that only the *first* of those survives a normal
build. `thr_lock_info`, `AST_MAX_LOCKS`, `ast_dump_locks()` and the CLI command
itself are all inside `#ifdef DEBUG_THREADS` and `#if !defined(LOW_MEMORY)`
(`main/utils.c:978` and `:980`, closing at `:1586` and `:1585` respectively),
so `core show locks` does not
exist in a stock build — whereas the per-lock `ast_lock_track` pointer, as
consequence 2 says, is in the struct regardless. The storage is
unconditional; the consumer is not. Chapter 05.

## The guide

| File | Covers | Status |
|---|---|---|
| [01-the-wrapper-layer.md](01-the-wrapper-layer.md) | the macro-to-function indirection and the four captured arguments, `ast_mutex_t`/`ast_rwlock_t` as structs, the renamed-field ABI device and `DEBUG_THREADS_LOOSE_ABI`, why Asterisk requires recursive mutexes, static initializers, `AST_MUTEX_DEFINE_STATIC` and the `AST_MUTEX_INIT_W_CONSTRUCTORS` variant | planned |
| [02-mutexes.md](02-mutexes.md) | `__ast_pthread_mutex_init` through `__ast_pthread_mutex_unlock` statement by statement, `log_mutex_error` and the `logger.c` exception, lazy tracking setup and the global `reentrancy_lock`, the reentrancy stack, `DETECT_DEADLOCKS`, and a table of every mutex error message | written |
| [03-read-write-locks.md](03-read-write-locks.md) | the nine rwlock entry points, the member-name and argument-order differences from the mutex family, the writer-preference attribute, why the reentrancy array is a search rather than a stack, the ownership check and stock-build diagnostic that rwlocks do not have, the timed and try variants, and a table of every rwlock error message | written |
| [`04-condition-variables.md`](04-condition-variables.md) | `ast_cond_t` as a bare typedef, the four entry points that do nothing, how a condition wait suspends a mutex in two separate records, `restore_lock_tracking`, and the clock nobody sets | written |
| [`05-lock-tracking-and-introspection.md`](05-lock-tracking-and-introspection.md) | the two tracking records and the two caps, which threads the registry can and cannot see, the eleven-of-thirteen field write and the slot `ast_remove_lock_info` leaves dirty, the two incompatible ways a failed acquisition is recorded, `core show locks` read line by line, the three timestamps that are not durations, the seven checks that cannot fire, and what each of `DEBUG_THREADS`, `LOW_MEMORY`, `DETECT_DEADLOCKS` and `THREAD_CRASH` actually buys | written |
| `06-deadlock-avoidance-and-ordering.md` | the `#ifdef DEBUG_THREADS` split and why the two definitions differ, `DEADLOCK_AVOIDANCE` and its 27 call sites (26 in `.c` files, all in channel drivers, plus one buried inside `ast_autochan_channel_lock`), `DLA_UNLOCK`/`DLA_LOCK` pairing, the lock-ordering rules the tree actually follows | planned |
| `07-threads.md` | `ast_pthread_create_stack` and `ast_pthread_create_detached_stack` and the four macros over them, `AST_STACKSIZE` and `AST_BACKGROUND_STACKSIZE`, thread registration and naming, what the 130 call sites are for | planned |
| `08-atomics.md` | `ast_atomic_*` over `__atomic` and the `__sync` fallback, the `memorder` argument that silently disappears, `ast_atomic_fetchadd_int` and `ast_atomic_dec_and_test`, the disabled compare-exchange | planned |

## Canonical source files

| Path | Role |
|---|---|
| `include/asterisk/lock.h` | The entire public surface: the lock types, every macro in both `DEBUG_THREADS` flavours, the scoped-lock helpers and the atomics (782 lines) |
| `main/lock.c` | The implementation every macro calls: 20 `__ast_*` entry points over pthreads, plus the reentrancy bookkeeping (1,335 lines) |
| `main/utils.c` | Per-thread lock tracking (`thr_lock_info`, `ast_store_lock_info`, `ast_dump_locks`, `core show locks`) and the `ast_pthread_create*` implementation |
| `include/asterisk/utils.h` | The four `ast_pthread_create*` macros and the stack-size constants (`:601-640`) |
| `include/asterisk/inline_api.h` | `AST_INLINE_API`, which is how the atomic helpers get a single definition |

## Reading order

Read [01-the-wrapper-layer.md](01-the-wrapper-layer.md) first and do not skip
it — the macro indirection and the `DEBUG_THREADS` rename are the two facts
that make the rest of the subsystem legible, and both are invisible from the
call site. Then [02-mutexes.md](02-mutexes.md), which is the only chapter whose
content you are likely to need while reading a backtrace.

[03-read-write-locks.md](03-read-write-locks.md) reads as a diff against 02 and
assumes it. Read it if the lock you are chasing turns out to be an
`AST_RWLIST_HEAD` or a locked vector, which is where most read-write locking in
the tree lives, or if you need to know which of the mutex diagnostics have no
rwlock counterpart. The short answer is: nearly all of them.

Chapters 05 and 06 are the practical pair: the first tells you how to see what
is held, the second tells you what the tree does about it. Read them together
when you have a hang to explain.

[04-condition-variables.md](04-condition-variables.md) is short and mostly
independent of 02 and 03. Read it if you have a thread parked in a wait and
want to know why `core show locks` shows it holding nothing, or before you
trust a `timedwait` deadline. Threads and atomics are reference material. Read
them when you need them.

## Next

[01-the-wrapper-layer.md](01-the-wrapper-layer.md) — what
`ast_mutex_lock(&p->lock)` actually expands to, and why `ast_mutex_t` is a
struct.
