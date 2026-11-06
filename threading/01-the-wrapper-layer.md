# 1. The Wrapper Layer

Every `file:line` citation in this guide was verified against the `24` branch.
Newer releases and other branches may differ — line numbers drift — so if you
are reading this against a different tree, treat the citations as approximate
and re-grep for the symbol name.

Asterisk has its own name for every pthread synchronisation primitive, and the
renaming is not cosmetic. `ast_mutex_t` is not `pthread_mutex_t`,
`ast_mutex_lock` is not a function, and the struct behind `ast_mutex_t` changes
field names — but not its layout — depending on how the tree was configured.
None of this is visible from a call site, and all of it matters the first time
you read a lock error in a log or try to work out why `core show locks` knows
your function name.

This chapter is the shape of the layer. What the macros expand to, what the
types really are, why the tracking fields are renamed instead of removed, and
how a statically declared lock gets initialised on a platform where static
initialisation does not work.

## 1.1 One macro, five arguments, four of them free

Every mutex operation is declared twice: once as a function taking a call-site
descriptor, and once as a macro that fills the descriptor in. The functions
first:

```c
int __ast_pthread_mutex_init(int tracking, const char *filename, int lineno, const char *func, const char *mutex_name, ast_mutex_t *t);
int __ast_pthread_mutex_destroy(const char *filename, int lineno, const char *func, const char *mutex_name, ast_mutex_t *t);
int __ast_pthread_mutex_lock(const char *filename, int lineno, const char *func, const char* mutex_name, ast_mutex_t *t);
int __ast_pthread_mutex_trylock(const char *filename, int lineno, const char *func, const char* mutex_name, ast_mutex_t *t);
int __ast_pthread_mutex_unlock(const char *filename, int lineno, const char *func, const char *mutex_name, ast_mutex_t *t);
```

— `include/asterisk/lock.h:187-191`

Then the macros that nearly every caller in the tree actually uses:

```c
#define ast_mutex_init(pmutex)            __ast_pthread_mutex_init(1, __FILE__, __LINE__, __PRETTY_FUNCTION__, #pmutex, pmutex)
#define ast_mutex_init_notracking(pmutex) __ast_pthread_mutex_init(0, __FILE__, __LINE__, __PRETTY_FUNCTION__, #pmutex, pmutex)
#define ast_mutex_destroy(a)              __ast_pthread_mutex_destroy(__FILE__, __LINE__, __PRETTY_FUNCTION__, #a, a)
#define ast_mutex_lock(a)                 __ast_pthread_mutex_lock(__FILE__, __LINE__, __PRETTY_FUNCTION__, #a, a)
#define ast_mutex_unlock(a)               __ast_pthread_mutex_unlock(__FILE__, __LINE__, __PRETTY_FUNCTION__, #a, a)
#define ast_mutex_trylock(a)              __ast_pthread_mutex_trylock(__FILE__, __LINE__, __PRETTY_FUNCTION__, #a, a)
```

— `include/asterisk/lock.h:193-198`

`__FILE__`, `__LINE__` and `__PRETTY_FUNCTION__` are standard. The fourth is
the interesting one: `#pmutex` and `#a` stringify the *argument expression*, so
a call written `ast_mutex_lock(&p->lock)` arrives at the implementation with
`mutex_name` set to the literal `"&p->lock"`. That is why a lock error can name
the lock the way you wrote it rather than by address, and why the two
`ast_mutex_init` spellings exist at all: `ast_mutex_init` passes `tracking` as
1 and `ast_mutex_init_notracking` passes 0, and nothing else about them differs.

All four descriptor arguments are compile-time constants, which is what makes
this affordable. There is no runtime lookup, no symbol table walk, no registry;
the preprocessor already knew the answer at each of the 850 `ast_mutex_lock`
sites in the tree.

The consequence worth internalising early is that **`ast_mutex_lock` has no
address**. You cannot store it in a function pointer, you cannot pass it to
something expecting a `void (*)(void *)`, and it never appears in a backtrace —
a backtrace shows `__ast_pthread_mutex_lock` with your function immediately
above it, because the macro introduces no frame of its own. What the descriptor
arguments add is not the caller's identity, which the stack already has, but
the *exact* file, line and source text of the acquisition — which is what you
want when the calling function takes the same lock in four places. Code that
needs a callable lock operation has to wrap the macro in a function of its own,
and that function then becomes the thing the descriptor names.

The read-write locks follow the same pattern with nine entry points instead of
five:

```c
int __ast_rwlock_init(int tracking, const char *filename, int lineno, const char *func, const char *rwlock_name, ast_rwlock_t *t);
int __ast_rwlock_destroy(const char *filename, int lineno, const char *func, const char *rwlock_name, ast_rwlock_t *t);
int __ast_rwlock_unlock(const char *filename, int lineno, const char *func, ast_rwlock_t *t, const char *name);
int __ast_rwlock_rdlock(const char *filename, int lineno, const char *func, ast_rwlock_t *t, const char *name);
int __ast_rwlock_wrlock(const char *filename, int lineno, const char *func, ast_rwlock_t *t, const char *name);
int __ast_rwlock_timedrdlock(const char *filename, int lineno, const char *func, ast_rwlock_t *t, const char *name, const struct timespec *abs_timeout);
int __ast_rwlock_timedwrlock(const char *filename, int lineno, const char *func, ast_rwlock_t *t, const char *name, const struct timespec *abs_timeout);
int __ast_rwlock_tryrdlock(const char *filename, int lineno, const char *func, ast_rwlock_t *t, const char *name);
int __ast_rwlock_trywrlock(const char *filename, int lineno, const char *func, ast_rwlock_t *t, const char *name);
```

— `include/asterisk/lock.h:216-224`

```c
#define ast_rwlock_init(rwlock)            __ast_rwlock_init(1, __FILE__, __LINE__, __PRETTY_FUNCTION__, #rwlock, rwlock)

/*!
 * \brief wrapper for ast_rwlock_init with tracking disabled
 * \return 0 on success, non zero for error
 * \since 1.6.1
 */
#define ast_rwlock_init_notracking(rwlock) __ast_rwlock_init(0, __FILE__, __LINE__, __PRETTY_FUNCTION__, #rwlock, rwlock)

#define ast_rwlock_destroy(rwlock)         __ast_rwlock_destroy(__FILE__, __LINE__, __PRETTY_FUNCTION__, #rwlock, rwlock)
#define ast_rwlock_unlock(a)               __ast_rwlock_unlock(__FILE__, __LINE__, __PRETTY_FUNCTION__, a, #a)
#define ast_rwlock_rdlock(a)               __ast_rwlock_rdlock(__FILE__, __LINE__, __PRETTY_FUNCTION__, a, #a)
#define ast_rwlock_wrlock(a)               __ast_rwlock_wrlock(__FILE__, __LINE__, __PRETTY_FUNCTION__, a, #a)
#define ast_rwlock_tryrdlock(a)            __ast_rwlock_tryrdlock(__FILE__, __LINE__, __PRETTY_FUNCTION__, a, #a)
#define ast_rwlock_trywrlock(a)            __ast_rwlock_trywrlock(__FILE__, __LINE__, __PRETTY_FUNCTION__, a, #a)
#define ast_rwlock_timedrdlock(a, b)       __ast_rwlock_timedrdlock(__FILE__, __LINE__, __PRETTY_FUNCTION__, a, #a, b)
#define ast_rwlock_timedwrlock(a, b)       __ast_rwlock_timedwrlock(__FILE__, __LINE__, __PRETTY_FUNCTION__, a, #a, b)
```

— `include/asterisk/lock.h:231-247`

Note the argument order moved. The mutex functions take `(file, line, func,
name, lock)`; the rwlock functions take `(file, line, func, lock, name)`, with
the lock and its name swapped. It is a harmless inconsistency as long as you go
through the macros — and the tree does not always go through the macros. There
are 15 direct `__ast_*` lock calls in core files outside `main/lock.c`, 12 of
them `__ast_rwlock_*`: `main/astobj2.c:241`, `:253`, `:262`, `:304`, `:319`,
`:361`, `:373`, `:382`; `main/astobj2_global.c:43`, `:55`, `:83`, `:94`; and
`main/heap.c:283`, `:288`, `:293`.

Those are not sloppiness, and they are the second reason the `__ast_*` functions
are public at all. Each of those sites is itself the implementation of a macro
that captured *its* caller's `file`/`line`/`func`, and it has to forward that
descriptor rather than let the preprocessor substitute `main/astobj2.c` and a
line number inside astobj2. Forwarding a descriptor is the legitimate case for
calling an `__ast_*` function directly; wanting to avoid the macro is not.

Condition variables are wrapped the same way but, as section 1.2 shows, the
wrapping buys much less.

## 1.2 `ast_mutex_t` is a struct

The three typedefs are the centre of the subsystem, and they are not symmetric:

```c
typedef struct ast_mutex_info ast_mutex_t;

typedef struct ast_rwlock_info ast_rwlock_t;

typedef pthread_cond_t ast_cond_t;
```

— `include/asterisk/lock.h:181-185`

Two of the three name a struct Asterisk defines. The third is a bare alias for
the pthread type.

`struct ast_mutex_info` puts the real mutex first and the bookkeeping after it:

```c
/*! \brief Structure for mutex and tracking information.
 *
 * We have tracking information in this structure regardless of DEBUG_THREADS being enabled.
 * The information will just be ignored in the core if a module does not request it..
 */
struct ast_mutex_info {
	pthread_mutex_t mutex;
#if !defined(DEBUG_THREADS) && !defined(DEBUG_THREADS_LOOSE_ABI) && \
	!defined(DETECT_DEADLOCKS)
	/*!
	 * These fields are renamed to ensure they are never used when
	 * DEBUG_THREADS is not defined.
	 */
	struct ast_lock_track *_track;
	struct ast_lock_track_flags _flags;
#elif defined(DEBUG_THREADS) || defined(DETECT_DEADLOCKS)
	/*! Track which thread holds this mutex. */
	struct ast_lock_track *track;
	struct ast_lock_track_flags flags;
#endif
};
```

— `include/asterisk/lock.h:137-157`

`struct ast_rwlock_info` (`:159-179`) is the same construction with
`pthread_rwlock_t lock` in place of `pthread_mutex_t mutex`. The `#if` block is
identical down to the whitespace, with one exception: the Doxygen comment on the
debug-build members reads "Track which thread holds this mutex." for the mutex
and "Track which thread holds this lock" for the rwlock. Nothing depends on
that, but it is worth knowing if you ever try to diff the two by eye and
conclude they have diverged.

Putting the pthread object first is deliberate and load-bearing. It means
`&t->mutex` is `(pthread_mutex_t *) t`, so the implementation can hand the
address straight to pthreads with no offset arithmetic, and a debugger pointed
at an `ast_mutex_t` shows you a usable `pthread_mutex_t` at offset zero.

What it also means is that `sizeof(ast_mutex_t)` is normally *larger* than
`sizeof(pthread_mutex_t)` — on x86-64 glibc, 56 bytes against 40 — and that the
extra 16 bytes are there in a stock build as much as a debug one. The tracking
fields are not compiled out by turning `DEBUG_THREADS` off. There is exactly one
way to compile them out, and the next section is about it.

## 1.3 The renamed-field ABI device

The `#if` at `include/asterisk/lock.h:144-156` is the single cleverest thing in
`lock.h`, and it is easy to misread as a mistake. Both branches declare the
same two members with the same two types in the same order, and the only
difference between the two spellings is a leading underscore.

There are three distinct outcomes, though, not two, and the third is the one
that explains the design. Measured on x86-64 glibc, where
`sizeof(pthread_mutex_t)` is 40:

| Build | Fields | `sizeof(ast_mutex_t)` |
|---|---|---|
| stock — none of the three symbols | `_track`, `_flags` | 56 |
| `DEBUG_THREADS` | `track`, `flags` | 56 |
| `DETECT_DEADLOCKS` | `track`, `flags` | 56 |
| `DEBUG_THREADS_LOOSE_ABI` alone | *none* — struct ends at `mutex` | 40 |
| `DEBUG_THREADS` **and** `DEBUG_THREADS_LOOSE_ABI` | `track`, `flags` | 56 |

The first three rows are the rename: same types, same order, same offsets, same
size. Only the names change.

The header says why in as many words, at `:137-141`:

> We have tracking information in this structure regardless of DEBUG_THREADS
> being enabled. The information will just be ignored in the core if a module
> does not request it..

Read the two halves of that sentence separately and the design falls out. The
*layout* is unconditional so that the ABI is stable: a module compiled without
`DEBUG_THREADS` and a core compiled with it agree on where every field of every
`ast_mutex_t` lives, so they can pass locks to each other. The *names* are
conditional so that the fields are unusable in a build that cannot support
them: writing `t->track` in a stock build is a compile error naming a field
that does not exist, rather than a silent write into memory the core will never
read.

That is why this is a rename rather than a `char _reserved[16]` or an `#ifdef`
that deletes the members. A rename gets you a compile-time error at the exact
expression that is wrong. Padding gets you a runtime mystery.

### `DEBUG_THREADS_LOOSE_ABI`, and why the default build uses it

The fourth row is worth working through by hand, because it is not an
oversight. `DEBUG_THREADS_LOOSE_ABI` appears **only** in the negated first
condition, so defining it alone makes the first arm false without making the
`#elif` true — *neither* arm is taken and the struct ends at `pthread_mutex_t
mutex`, 16 bytes shorter.

The natural guess is that this is an escape hatch for someone building the
whole tree from source with no binary compatibility to preserve. It is not.
Exactly one file in the tree defines it, and it is not an unusual build — it is
the file every ao2 object's lock lives in:

```c
/* This reduces the size of lock structures within astobj2 objects when
 * DEBUG_THREADS is not defined. */
#define DEBUG_THREADS_LOOSE_ABI
```

— `main/astobj2.c:28-30`

`struct ao2_lock_priv` embeds an `ast_mutex_t` by value
(`main/astobj2.c:115-117`), and `struct ao2_rwlock_priv` an `ast_rwlock_t`
(`:126-130`), so those 16 bytes are paid once per ao2 object, of which a busy
Asterisk has a great many. `astobj2.c` declines to pay them.

Which ought to be a disaster, since `astobj2.c` passes those locks straight to
`main/lock.c` — the `__ast_pthread_mutex_lock` call at `main/astobj2.c:241` is
handing a 40-byte struct to a function compiled against a 56-byte one. It is
safe for two reasons that have to hold together:

1. In a stock build `main/lock.c` never reads `_track` or `_flags`. Every access
   is inside `#if defined(DEBUG_THREADS) || defined(DETECT_DEADLOCKS)`, so the
   16 bytes the core believes are there are bytes it never touches. This is
   precisely the header's "will just be ignored in the core".
2. In a `DEBUG_THREADS` build the core *does* read them — and in that build
   `astobj2.c` gets the fields back, because the first arm's condition also
   requires `!defined(DEBUG_THREADS)`. That is the fifth row of the table:
   `DEBUG_THREADS` wins over `DEBUG_THREADS_LOOSE_ABI`, so the two translation
   units cannot disagree in the build where disagreeing would matter.

So "loose ABI" means loose with respect to *external modules only*. A
third-party module compiled against the stock header sees a 56-byte
`ast_mutex_t`; `astobj2.c` sees 40; and the arrangement holds because the
difference is confined to bytes nobody reads. If you ever add a field to these
structs and then read it unconditionally in `main/lock.c`, you break
`main/astobj2.c` specifically, and you break it silently.

The practical rules this hands you:

- Never touch `track` or `flags` from outside `main/lock.c`. Nothing in the
  tree does, and the names are arranged so that an attempt fails to compile in
  the stock build.
- Never add a field to `struct ast_mutex_info` or `struct ast_rwlock_info`
  without putting it under the same `#if`, and never read one unconditionally
  in `main/lock.c`. The first breaks the module ABI; the second breaks
  `main/astobj2.c`.
- Never `memcpy` an `ast_mutex_t`. It is a `pthread_mutex_t` plus a pointer to
  per-lock state with a nested `pthread_mutex_t` of its own; copying it gives
  two locks sharing one reentrancy record.

## 1.4 What tracking holds

The field the rename protects points at this:

```c
struct ast_lock_track {
	const char *file[AST_MAX_REENTRANCY];
	int lineno[AST_MAX_REENTRANCY];
	int reentrancy;
	const char *func[AST_MAX_REENTRANCY];
	pthread_t thread_id[AST_MAX_REENTRANCY];
#ifdef HAVE_BKTR
	struct ast_bt backtrace[AST_MAX_REENTRANCY];
#endif
	pthread_mutex_t reentr_mutex;
};
```

— `include/asterisk/lock.h:118-128`

Four of the six members are arrays of `AST_MAX_REENTRANCY`, which is 10
(`:108`) — `file`, `lineno`, `func` and `thread_id`; with `HAVE_BKTR` the
`backtrace` array makes it five of seven. That is the shape of the whole design:
**this is a stack, not a record.** Asterisk mutexes are recursive (section 1.5), so one thread can hold
one mutex ten deep, and "where was this lock taken" has up to ten answers at
once. `reentrancy` is the depth, and `ROFFSET` (`:249`) is the macro the
implementation uses to index the top of the stack:

```c
#define	ROFFSET	((lt->reentrancy > 0) ? (lt->reentrancy-1) : 0)
```

— `include/asterisk/lock.h:249`

Note what `ROFFSET` does when `reentrancy` is 0: it yields 0, the same index as
a depth of 1. Reading the top of the stack on an unheld lock gives you the
stale entry from the last holder rather than an error, which is correct for
error messages — the most recent holder is exactly what you want to print — but
is not a way to ask whether a lock is held.

This is a separate allocation rather than a set of fields inside
`ast_mutex_info` for two reasons, and both are about cost. It is big: with
`AST_MAX_REENTRANCY` at 10 the four arrays plus the nested mutex come to 328
bytes on x86-64 glibc, against the 16 bytes of pointer-plus-flags that stand in
for them. And it is lazy: nothing is allocated at `init` time. The first
operation on the lock that needs tracking calls `ast_get_reentrancy`
(`main/lock.c:87-128`), which allocates behind the `setup` flag, so a lock that
is never contended and never introspected never pays for the 328 bytes at all.

The last member is still worth a note of its own. `reentr_mutex` is a real
`pthread_mutex_t` guarding the stack, because updating the tracking stack is
itself a critical section — and it cannot be guarded by the lock being tracked,
since the whole point is to record the moment *before* that lock is held.
Chapter 02 follows `ast_reentrancy_lock` (`:452-461`) through an acquisition.

One asymmetry to carry forward: everything above is in the struct in every
build, but the thing most people want to read it with is not.
`core show locks` is assembled from a *second*, per-thread structure that lives
entirely inside `#ifdef DEBUG_THREADS` and `#if !defined(LOW_MEMORY)`
(`main/utils.c:978-980`), so the CLI command does not exist in a stock build.
The per-lock storage is unconditional; the consumer of it is not. Chapter 05
takes both apart.

The flags are two bits:

```c
struct ast_lock_track_flags {
	/*! non-zero if lock tracking is enabled */
	unsigned int tracking:1;
	/*! non-zero if track is setup */
	volatile unsigned int setup:1;
};
```

— `include/asterisk/lock.h:130-135`

`tracking` is what `ast_mutex_init` versus `ast_mutex_init_notracking` sets, and
it never changes afterwards. `setup` is the lazy-initialisation latch: the
`ast_lock_track` is not allocated at `init` time but on first use, and `setup`
records whether that has happened. It is `volatile` because the test is read
outside the lock that protects the allocation. Chapter 02 covers the sequence
and what it means for a lock used before `ast_mutex_init` returns.

## 1.5 Asterisk requires recursive mutexes

This is stated in a comment and enforced nowhere, which makes it worth
repeating loudly:

```c
/* Asterisk REQUIRES recursive (not error checking) mutexes
   and will not run without them. */
#if defined(HAVE_PTHREAD_RECURSIVE_MUTEX_INITIALIZER_NP) && defined(HAVE_PTHREAD_MUTEX_RECURSIVE_NP)
#define PTHREAD_MUTEX_INIT_VALUE	PTHREAD_RECURSIVE_MUTEX_INITIALIZER_NP
#define AST_MUTEX_KIND			PTHREAD_MUTEX_RECURSIVE_NP
#else
#define PTHREAD_MUTEX_INIT_VALUE	PTHREAD_MUTEX_INITIALIZER
#define AST_MUTEX_KIND			PTHREAD_MUTEX_RECURSIVE
#endif /* PTHREAD_RECURSIVE_MUTEX_INITIALIZER_NP */
```

— `include/asterisk/lock.h:80-88`

The two arms are *not* equivalent, and the difference is easy to miss.
`AST_MUTEX_KIND` is the recursive type in both — `PTHREAD_MUTEX_RECURSIVE_NP`
when `configure` found the `_NP` ("non-portable") glibc names, plain
`PTHREAD_MUTEX_RECURSIVE` otherwise. `PTHREAD_MUTEX_INIT_VALUE` is not.
In the first arm it is `PTHREAD_RECURSIVE_MUTEX_INITIALIZER_NP` (`:83`), a
static initialiser for a recursive mutex; in the `#else` it is plain
`PTHREAD_MUTEX_INITIALIZER` (`:86`), which is *not* recursive. So the runtime
path is recursive on every platform and the static path is recursive only where
the `_NP` symbols exist. Section 1.6 is where that matters.

Every mutex `__ast_pthread_mutex_init` creates gets `AST_MUTEX_KIND`:

```c
	pthread_mutexattr_init(&attr);
	pthread_mutexattr_settype(&attr, AST_MUTEX_KIND);
	res = pthread_mutex_init(&t->mutex, &attr);
	pthread_mutexattr_destroy(&attr);
```

— `main/lock.c:169-172`

An attribute object is built, set to `AST_MUTEX_KIND`, used once and destroyed,
for every mutex in the process. Every mutex that goes through this function is
recursive; the gap, as above, is the one that never does — a statically
initialised lock on a platform taking the `#else` arm without
`AST_MUTEX_INIT_W_CONSTRUCTORS`. On glibc the gap is not live, since
`configure` finds both `_NP` symbols and defines them in `autoconfig.h`, but it
is the reason the constructor form in section 1.6 exists.

Two things follow, and they pull in opposite directions. A thread
double-locking the same `ast_mutex_t` **does not deadlock** — it increments a
count, and must unlock the same number of times. That is what makes code like
`ast_channel_lock` safe to call from a function that may or may not already be
holding the channel. But it also means the usual beginner's deadlock does not
announce itself. Instead of an immediate hang you get pthreads' own recursion
counter climbing, with an implementation-defined limit and no diagnostic, and
the failure surfaces later and somewhere else as an unlock that did not
actually release anything.

`AST_MAX_REENTRANCY` is not the backstop it looks like. It bounds the *tracking*
stack, not the lock, and the push that respects it is inside
`#ifdef DEBUG_THREADS` (`main/lock.c:361-392`) — so in a stock build nothing
counts toward it, and in a debug build exceeding it is not quiet either:
the acquisition still succeeds, and the tenth re-entry simply logs
`'%s' really deep reentrancy!` (`main/lock.c:370`) and goes unrecorded.

The comment's parenthesis — "not error checking" — rules out the alternative
explicitly. `PTHREAD_MUTEX_ERRORCHECK` would turn the double lock into an
`EDEADLK` return, which would be more informative and would break a large
amount of the tree.

## 1.6 Static locks, and two ways to initialise them

A file-scope lock is declared with a macro, not with `ast_mutex_t foo =
...`. There are 75 `AST_MUTEX_DEFINE_STATIC` and 15
`AST_RWLOCK_DEFINE_STATIC` declarations in the tree, and both expand through a
layer that exists for one platform problem.

The usual case is a plain static initialiser:

```c
#define AST_MUTEX_INIT_VALUE { PTHREAD_MUTEX_INIT_VALUE, NULL, {1, 0} }
#define AST_MUTEX_INIT_VALUE_NOTRACKING { PTHREAD_MUTEX_INIT_VALUE, NULL, {0, 0} }

#define AST_RWLOCK_INIT_VALUE { __AST_RWLOCK_INIT_VALUE, NULL, {1, 0} }
#define AST_RWLOCK_INIT_VALUE_NOTRACKING { __AST_RWLOCK_INIT_VALUE, NULL, {0, 0} }
```

— `include/asterisk/lock.h:102-106`

Three initialisers for a struct that may have one or three members, and it is
correct in every build because of §1.3: in a stock build the fields are
`_track` and `_flags`, in a debug build they are `track` and `flags`, and in
both cases there are exactly two of them in the same order. `NULL` is the
not-yet-allocated `ast_lock_track`, and `{1, 0}` is `tracking` on with `setup`
off — which is to say, every statically defined lock starts out lazily tracked.
The `_NOTRACKING` spellings differ in exactly that bit. (Under
`DEBUG_THREADS_LOOSE_ABI`, where the struct has only its `pthread_mutex_t`, the
extra initialisers are surplus and the compiler will say so; that build is
expected to be configured deliberately.)

Static initialisation is not available everywhere, and `lock.h` decides by
platform:

```c
#if (defined(SOLARIS) || defined(BSD))
#define AST_MUTEX_INIT_W_CONSTRUCTORS
#endif /* SOLARIS || BSD */
```

— `include/asterisk/lock.h:76-78`

This is the other half of the asymmetry section 1.5 ended on. Where the `_NP`
symbols are missing, `PTHREAD_MUTEX_INIT_VALUE` is plain
`PTHREAD_MUTEX_INITIALIZER` and a statically initialised lock would come out
non-recursive — which Asterisk cannot run on. So on the platforms where that is
the case, the definition macro stops relying on static initialisation and grows
a constructor and a destructor instead, routing every static lock through
`ast_mutex_init` at load time:

```c
#define __AST_MUTEX_DEFINE(scope, mutex, init_val, track)	\
	scope ast_mutex_t mutex = init_val;			\
static void  __attribute__((constructor)) init_##mutex(void)	\
{								\
	if (track)						\
		ast_mutex_init(&mutex);				\
	else							\
		ast_mutex_init_notracking(&mutex);		\
}								\
								\
static void  __attribute__((destructor)) fini_##mutex(void)	\
{								\
	ast_mutex_destroy(&mutex);				\
}
```

— `include/asterisk/lock.h:508-521`

Everywhere else it is the one-liner:

```c
/* By default, use static initialization of mutexes. */
#define __AST_MUTEX_DEFINE(scope, mutex, init_val, track) scope ast_mutex_t mutex = init_val
```

— `include/asterisk/lock.h:523-524`

and the two public macros pick the initial value and the tracking bit:

```c
#define AST_MUTEX_DEFINE_STATIC(mutex) __AST_MUTEX_DEFINE(static, mutex, AST_MUTEX_INIT_VALUE, 1)
#define AST_MUTEX_DEFINE_STATIC_NOTRACKING(mutex) __AST_MUTEX_DEFINE(static, mutex, AST_MUTEX_INIT_VALUE_NOTRACKING, 0)
```

— `include/asterisk/lock.h:527-528`

`__AST_RWLOCK_DEFINE` (`:533-547`) and its two public forms (`:550-551`) are
the same construction for rwlocks.

The `if (track)` inside the constructor is worth a second look: `track` is a
macro parameter that is always the literal `1` or `0`, so the branch is
resolved at compile time and one of the two `ast_mutex_init` calls is
discarded. It reads like a runtime test and is not one.

Two consequences of the constructor form, both of which have bitten people:

- The order in which constructors run across translation units is unspecified.
  A statically defined lock is guaranteed initialised before `main`, and *not*
  guaranteed initialised before another translation unit's constructor. Code
  that locks from a constructor works on Linux (static initialisation, so the
  lock is ready at load time) and is a coin flip on BSD.
- The destructor calls `ast_mutex_destroy` at process exit, after which the
  lock is unusable. Anything running in an `atexit` handler or a later
  destructor that touches a static lock is in undefined territory on those
  platforms and fine on Linux.

Which is to say: if you are only ever going to build on Linux, this entire
section is a no-op, and that is precisely why the hazards in it are easy to
ship.

## 1.7 How the implementation escapes its own wrappers

`main/lock.c` has a problem most files do not. It includes `asterisk/lock.h`,
which means `pthread_mutex_lock` is itself a macro by the time the
implementation needs the real function — so it takes the names back:

```c
/* Allow direct use of pthread_mutex_* / pthread_cond_* */
#undef pthread_mutex_init
#undef pthread_mutex_destroy
#undef pthread_mutex_lock
#undef pthread_mutex_trylock
#undef pthread_mutex_t
#undef pthread_mutex_unlock
#undef pthread_cond_init
#undef pthread_cond_signal
#undef pthread_cond_broadcast
#undef pthread_cond_destroy
#undef pthread_cond_wait
#undef pthread_cond_timedwait
```

— `main/lock.c:43-55`

Twelve `#undef`s — six mutex and six cond — including one for a *type*
(`pthread_mutex_t`), after which everything below that line in `main/lock.c` is
talking to pthreads directly.

They are not defensive housekeeping. `lock.h` *poisons* those names
deliberately, at `:628-642`:

```c
#define pthread_mutex_lock	use_ast_mutex_lock_instead_of_pthread_mutex_lock
```

— `include/asterisk/lock.h:632`

Eleven more follow in the same shape, plus `AST_MUTEX_INITIALIZER` at `:644`,
which is redefined to
`__use_AST_MUTEX_DEFINE_STATIC_rather_than_AST_MUTEX_INITIALIZER__`. None of
those identifiers exist, so a stray raw `pthread_mutex_lock` anywhere in the
tree fails to compile with an error message that names the wrapper the author
should have used. (`pthread_mutex_t` and `pthread_cond_t` are poisoned at
`:629-630`, inside a `#ifndef __CYGWIN__`.) The `#undef`s are how a file opts
back out — and `main/lock.c` does not `#undef pthread_cond_t`, because it never
needs to name that type.

This is also the reason the "never use raw pthread calls" rule has the
exceptions it has, and the list is short enough to memorise:

- `main/lock.c` itself, which *is* the wrapper (12 `#undef`s, `:44-55`).
- `main/utils.c`, because it implements the tracking the wrappers feed, and a
  wrapper that called a wrapper to record that a wrapper was called would not
  terminate. It takes back five names (`:985-989`) plus `pthread_create`
  (`:975`), and labels two call sites `/* Intentionally not the wrapper */`
  (`:1627` and `:1629`) so that nobody tidies them.
- `main/threadstorage.c`, five `#undef`s at `:60-64`, since thread-local
  storage is the substrate the per-thread lock list is built on.
- `main/backtrace.c`, which does not `#undef` anything because it suppresses
  the header outright: `#define _ASTERISK_LOCK_H` before any include
  (`:29-33`), with a comment saying it does not want the overhead.
- `include/asterisk/spinlock.h:404-433`, the `AST_SPINLOCK_TYPE_PTHREAD_MUTEX`
  fallback, which gets away with raw calls by never including `lock.h` at all.

Outside those five, a raw `pthread_mutex_lock` is a lock the core cannot see —
invisible in exactly the situation you would want to see it.

## 1.8 Rules

| Rule | Why |
|---|---|
| Go through the macros unless you are forwarding someone else's descriptor | The macros are the only thing that fills in the call-site descriptor. Calling an `__ast_*` function directly is legitimate exactly when your own caller already captured one and you are passing it down — which is what the 15 sites in `main/astobj2.c`, `main/astobj2_global.c` and `main/heap.c` are doing. Inventing the four arguments by hand is not, and getting them wrong means a plausible-looking lie in `core show locks`. |
| Never take the address of a lock operation | They are macros (`include/asterisk/lock.h:193-198`). Wrap one in a function of your own if you need a pointer. |
| Treat `ast_mutex_t` as opaque and never copy it | It is a struct with a pointer to per-lock state containing its own mutex (`:142-157`); a copy shares that state. |
| Never name `track` or `flags` outside `main/lock.c` | The `#if` at `:144-156` renames them precisely so that the attempt fails to compile in a stock build. |
| Add fields to the lock structs only inside that same `#if` | An unconditional field changes `sizeof(ast_mutex_t)` in every build, which is the ABI break the rename exists to avoid. |
| Do not rely on recursion to paper over unclear ownership | It works — every mutex is `AST_MUTEX_KIND` (`main/lock.c:170`) — but it converts an immediate deadlock into a count that fails later and elsewhere. |
| Declare file-scope locks with `AST_MUTEX_DEFINE_STATIC` | It is the only form that is correct under both the static-initialiser and constructor builds (`:508-528`). |
| Do not lock a static lock from a constructor or an exit handler | Under `AST_MUTEX_INIT_W_CONSTRUCTORS` (Solaris, BSD) the lock may not be initialised yet, or may already be destroyed (`:76-78`, `:508-521`). |

## Next

Chapter 02 will follow `__ast_pthread_mutex_lock` from the macro
expansion to the `pthread_mutex_lock` call and back, including the lazy
tracking setup, the reentrancy stack and the paths that log rather than return.
