# 7. Threads

Every `file:line` citation in this guide was verified against the `24` branch.
Newer releases and other branches may differ — line numbers drift — so if you
are reading this against a different tree, treat the citations as approximate
and re-grep for the symbol name.

Six chapters about locks have assumed the threads that take them. This chapter
is about where those threads come from. The surface is small enough to read in
one sitting: two functions, four macros over them, two stack-size constants,
two registration functions and a linked list. It lives in two places —
`include/asterisk/utils.h:597-642` for the declarations and macros,
`main/utils.c:1588-1730` for the implementation — plus a registry in
`main/asterisk.c:412-455` that is not in `utils.c` at all and is the reason
`core show threads` exists.

The reason this is a chapter rather than a paragraph is that the layer does
three jobs and only one of them is the job it is named for. It creates a
thread; it sets a stack size that is not the system default and is computed
rather than configured; and it wraps the caller's start routine in a function
that enrols the new thread in two separate registries, one of which is what
chapter 05's `core show locks` reads. The third job is invisible from the call
site, is skipped silently when an allocation fails, and does not exist at all
in a `LOW_MEMORY` build.

There are 129 call sites. The two most useful facts about them are that in a
default build two of the four macros are indistinguishable from the other two,
and that nothing linked into Asterisk calls `pthread_create` directly except
the one line inside the wrapper — which is a convention rather than a rule,
because the macro that would make it a rule is disabled on Linux.

## 7.1 The surface, and the guardrail that is switched off

The whole declaration block, comment and all:

```c
/*
 * Thread management support (should be moved to lock.h or a different header)
 */

#if defined(PTHREAD_STACK_MIN)
# define AST_STACKSIZE     MAX((((sizeof(void *) * 8 * 8) - 16) * 1024), PTHREAD_STACK_MIN)
# define AST_STACKSIZE_LOW MAX((((sizeof(void *) * 8 * 2) - 16) * 1024), PTHREAD_STACK_MIN)
#else
# define AST_STACKSIZE     (((sizeof(void *) * 8 * 8) - 16) * 1024)
# define AST_STACKSIZE_LOW (((sizeof(void *) * 8 * 2) - 16) * 1024)
#endif

int ast_background_stacksize(void);

#define AST_BACKGROUND_STACKSIZE ast_background_stacksize()
```

— `include/asterisk/utils.h:597-611`

The parenthetical is the tree's own. Thread creation lives in `utils.h` and
locks live in `lock.h`, which is why this chapter's region is split across two
headers where chapters 01 through 06 had one.

Then the four declarations and the four macros that are the only things any
caller uses:

```c
void ast_register_thread(char *name);
void ast_unregister_thread(void *id);

int ast_pthread_create_stack(pthread_t *thread, pthread_attr_t *attr, void *(*start_routine)(void *),
			     void *data, size_t stacksize, const char *file, const char *caller,
			     int line, const char *start_fn);

int ast_pthread_create_detached_stack(pthread_t *thread, pthread_attr_t *attr, void*(*start_routine)(void *),
				 void *data, size_t stacksize, const char *file, const char *caller,
				 int line, const char *start_fn);

#define ast_pthread_create(a, b, c, d) 				\
	ast_pthread_create_stack(a, b, c, d,			\
		0, __FILE__, __FUNCTION__, __LINE__, #c)

#define ast_pthread_create_detached(a, b, c, d)			\
	ast_pthread_create_detached_stack(a, b, c, d,		\
		0, __FILE__, __FUNCTION__, __LINE__, #c)

#define ast_pthread_create_background(a, b, c, d)		\
	ast_pthread_create_stack(a, b, c, d,			\
		AST_BACKGROUND_STACKSIZE,			\
		__FILE__, __FUNCTION__, __LINE__, #c)

#define ast_pthread_create_detached_background(a, b, c, d)	\
	ast_pthread_create_detached_stack(a, b, c, d,		\
		AST_BACKGROUND_STACKSIZE,			\
		__FILE__, __FUNCTION__, __LINE__, #c)

/* End of thread management support */
```

— `include/asterisk/utils.h:613-642`

The shape is the device of chapter 01 §1.1: a macro that captures `__FILE__`,
`__FUNCTION__` and `__LINE__` at the call site and passes them to a function
whose name the caller never types. There is a fourth captured argument that the
lock macros do not have — `#c`, the stringification of the third macro
argument, which is the start routine. §7.7 is about what that string turns out
to contain.

Note which argument is *not* captured and not defaulted: the `pthread_attr_t *`
in position `b`. All four macros pass it straight through, so a caller can hand
in an attribute object, and §7.5 covers the three that do.

The convention that all thread creation goes through this layer is enforced the
way chapter 01's lock conventions are, by poisoning the underlying name in
`lock.h`:

```c
#ifndef __CYGWIN__	/* temporary disabled for cygwin */
#define pthread_mutex_t		use_ast_mutex_t_instead_of_pthread_mutex_t
#define pthread_cond_t		use_ast_cond_t_instead_of_pthread_cond_t
#endif
#define pthread_mutex_lock	use_ast_mutex_lock_instead_of_pthread_mutex_lock
#define pthread_mutex_unlock	use_ast_mutex_unlock_instead_of_pthread_mutex_unlock
#define pthread_mutex_trylock	use_ast_mutex_trylock_instead_of_pthread_mutex_trylock
#define pthread_mutex_init	use_ast_mutex_init_instead_of_pthread_mutex_init
#define pthread_mutex_destroy	use_ast_mutex_destroy_instead_of_pthread_mutex_destroy
#define pthread_cond_init	use_ast_cond_init_instead_of_pthread_cond_init
#define pthread_cond_destroy	use_ast_cond_destroy_instead_of_pthread_cond_destroy
#define pthread_cond_signal	use_ast_cond_signal_instead_of_pthread_cond_signal
#define pthread_cond_broadcast	use_ast_cond_broadcast_instead_of_pthread_cond_broadcast
#define pthread_cond_wait	use_ast_cond_wait_instead_of_pthread_cond_wait
#define pthread_cond_timedwait	use_ast_cond_timedwait_instead_of_pthread_cond_timedwait

#define AST_MUTEX_INITIALIZER __use_AST_MUTEX_DEFINE_STATIC_rather_than_AST_MUTEX_INITIALIZER__

#define gethostbyname __gethostbyname__is__not__reentrant__use__ast_gethostbyname__instead__

#ifndef __linux__
#define pthread_create __use_ast_pthread_create_instead__
#endif
```

— `include/asterisk/lock.h:628-650`

Read the last three lines again. Every other entry in that list is
unconditional, or guarded only for Cygwin. `pthread_create` is poisoned
`#ifndef __linux__` — that is, everywhere *except* the platform the
overwhelming majority of Asterisk builds target. On Linux a module can call
`pthread_create` directly and nothing complains.

In practice nothing does. A sweep of the tree's C sources for the bare name
finds exactly three calls: `main/utils.c:1703`, which is the one inside the
wrapper and carries a comment saying so, and two in
`contrib/utils/eagi_proxy.c:136-137`, a standalone sample program that is not
linked into Asterisk and does not include `lock.h`. (A fourth sits in the
feature-test program at `autoconf/ax_pthread.m4:329`, compiled by `configure`
and never by `make`.) The implementation file undoes the poison for itself with
a one-line `#undef` at `main/utils.c:975`, whose comment — `/* For
ast_pthread_create function only */` — is the clearest statement of the rule
anywhere in the tree. That `#undef` is itself wrapped in `#ifndef __linux__`
(`:974-976`), which it has to be: on Linux there was never any poison to
undo.

So the discipline holds, but it holds by habit. If you are reading a patch that
adds a thread, the compiler will not tell you it bypassed the layer.

## 7.2 Four macros, two behaviours

The four macros differ in exactly two bits: whether they route to the detached
entry point, and whether they pass `0` or `AST_BACKGROUND_STACKSIZE` as the
stack size. The first bit is real. The second one usually is not.

`AST_BACKGROUND_STACKSIZE` is not a constant. It is a function call, and this
is the function:

```c
int ast_background_stacksize(void)
{
#if !defined(LOW_MEMORY)
	return AST_STACKSIZE;
#else
	return AST_STACKSIZE_LOW;
#endif
}
```

— `main/utils.c:1650-1657`

In any build that is not `LOW_MEMORY`, `ast_background_stacksize()` returns
`AST_STACKSIZE` — the same value `ast_pthread_create_stack` substitutes when it
is passed `0`, as §7.4 shows. So in a default build `ast_pthread_create` and
`ast_pthread_create_background` request *identical* stacks, and so do
`ast_pthread_create_detached` and `ast_pthread_create_detached_background`. The
word "background" in the macro name means "give this thread a smaller stack
**if** this tree was configured `LOW_MEMORY`", and nothing else. 61 of the 129
call sites use a background variant; in a stock build 58 of those 61 get the
same stack as the other 68. The three exceptions are in
`addons/ooh323cDriver.c`, one of the two Asterisk-side files of `chan_ooh323`
— the other 35 are the bundled ooh323c protocol stack, listed at
`addons/Makefile:17-27` — and that file replaces the macro for its own
translation unit, §7.10.

That is worth knowing in both directions. Choosing
`ast_pthread_create_background` for a thread you expect to be cheap buys
nothing unless somebody builds `LOW_MEMORY`. And choosing `ast_pthread_create`
for a thread with deep recursion does not opt out of the `LOW_MEMORY`
reduction — it opts out of *ever* being reduced, because the plain macro passes
`0` and the implementation turns `0` into `AST_STACKSIZE` unconditionally. The
naming suggests a general/background distinction. The behaviour is a
`LOW_MEMORY`/always distinction.

Why a function rather than `#ifdef LOW_MEMORY` around a macro definition? The
header does not say. The effect, though, is the effect chapter 01 §1.3
described for the renamed tracking fields: the value is decided by the *core's*
build, not by the build of whichever module expands the macro. A module
compiled without `LOW_MEMORY` and loaded into a core compiled with it still
asks for the small stack, because the question is answered at run time by a
function that lives in the core. The lock layer reaches the same place by a
different route: `ast_mutex_lock` is an unconditional macro over a core
function (`include/asterisk/lock.h:196`), so a module's own `DEBUG_THREADS`
setting cannot change what the core does with the lock either — the header says
as much at `:139-141`. What is unusual here is only the price, a call into the
core on every thread creation to answer a question that was settled when the
core was compiled.

## 7.3 The stack size is not a constant either

```c
#if defined(PTHREAD_STACK_MIN)
# define AST_STACKSIZE     MAX((((sizeof(void *) * 8 * 8) - 16) * 1024), PTHREAD_STACK_MIN)
# define AST_STACKSIZE_LOW MAX((((sizeof(void *) * 8 * 2) - 16) * 1024), PTHREAD_STACK_MIN)
#else
# define AST_STACKSIZE     (((sizeof(void *) * 8 * 8) - 16) * 1024)
# define AST_STACKSIZE_LOW (((sizeof(void *) * 8 * 2) - 16) * 1024)
#endif
```

— `include/asterisk/utils.h:601-607`

Three things about those four definitions.

First, the arithmetic. `sizeof(void *) * 8 * 8` is the only place the word size
enters: on a 64-bit build it is 512, minus 16 is 496, times 1024 is 507,904
bytes — 496 KiB. `AST_STACKSIZE_LOW` uses `* 2` where `AST_STACKSIZE` uses
`* 8`, giving 112 KiB. On a 32-bit build the same two expressions give 240 KiB
and 48 KiB. The `- 16` has no comment and no evident purpose beyond keeping the
result a round number of kibibytes below a power of two. Note what these
numbers are *not*: a Linux thread created with a default attribute object gets
the soft `RLIMIT_STACK`, typically 8 MiB, so almost every thread Asterisk
creates has a stack roughly sixteen times smaller than one created by a bare
`pthread_create` — the exceptions being the two modules of §7.10 that ask for
more. That is the layer's most consequential side effect and it is nowhere in
the macro names.

Second, `MAX` is not an arithmetic operator here. It is this, from the same
header:

```c
#undef MAX
#define MAX(a, b) ({ typeof(a) __a = (a); typeof(b) __b = (b); ((__a < __b) ? __b : __a);})
```

— `include/asterisk/utils.h:253-254`

A GCC statement expression. So in the `PTHREAD_STACK_MIN` arm `AST_STACKSIZE`
expands to a braced block with two local variables, which means it is not an
integer constant expression: it cannot appear in a `case` label, an array
bound, a bit-field width or a static initialiser. Nothing in the tree tries.
`AST_STACKSIZE` has exactly three uses, all inside `main/utils.c` (`:1653`,
`:1685`, `:1687`), and `AST_STACKSIZE_LOW` has one (`:1655`); every other
reference in the tree goes through `AST_BACKGROUND_STACKSIZE`, which is a
function call and has no such problem — except in the one translation unit that
replaces it with an integer (§7.10).

Third, and least obvious: `PTHREAD_STACK_MIN` is itself not guaranteed to be a
constant. On glibc from 2.34 onward, when `__USE_DYNAMIC_STACK_SIZE` is in
effect, `<bits/pthread_stack_min-dynamic.h>` defines it as a call to
`__sysconf`, evaluated every time the macro is expanded. On such a system
`AST_STACKSIZE` is a run-time expression that calls into libc, and
`ast_pthread_create_stack` evaluates it once per thread creation. This is not
an Asterisk bug and it changes no behaviour that matters; it is worth knowing
only because the name looks like a compile-time constant and in a modern Linux
build it is not one.

The signedness is worth a glance in passing. `MAX` takes the `typeof` of each
argument: the left-hand side is `size_t` arithmetic and the right-hand side on
glibc is `long int`, so the comparison converts the signed value to unsigned.
If the `sysconf` call ever returned `-1`, `MAX` would select it as the larger
operand and `pthread_attr_setstacksize` would be handed `SIZE_MAX` and fail —
which is the benign outcome, because as §7.4 shows a failed
`pthread_attr_setstacksize` is logged and then ignored.

## 7.4 `ast_pthread_create_stack`, line by line

This is the whole of it:

```c
int ast_pthread_create_stack(pthread_t *thread, pthread_attr_t *attr, void *(*start_routine)(void *),
			     void *data, size_t stacksize, const char *file, const char *caller,
			     int line, const char *start_fn)
{
#if !defined(LOW_MEMORY)
	struct thr_arg *a;
#endif

	if (!attr) {
		attr = ast_alloca(sizeof(*attr));
		pthread_attr_init(attr);
	}

#if defined(__linux__) || defined(__FreeBSD__)
	/* On Linux and FreeBSD , pthread_attr_init() defaults to PTHREAD_EXPLICIT_SCHED,
	   which is kind of useless. Change this here to
	   PTHREAD_INHERIT_SCHED; that way the -p option to set realtime
	   priority will propagate down to new threads by default.
	   This does mean that callers cannot set a different priority using
	   PTHREAD_EXPLICIT_SCHED in the attr argument; instead they must set
	   the priority afterwards with pthread_setschedparam(). */
	if ((errno = pthread_attr_setinheritsched(attr, PTHREAD_INHERIT_SCHED)))
		ast_log(LOG_WARNING, "pthread_attr_setinheritsched: %s\n", strerror(errno));
#endif

	if (!stacksize)
		stacksize = AST_STACKSIZE;

	if ((errno = pthread_attr_setstacksize(attr, stacksize ? stacksize : AST_STACKSIZE)))
		ast_log(LOG_WARNING, "pthread_attr_setstacksize: %s\n", strerror(errno));

#if !defined(LOW_MEMORY)
	if ((a = ast_malloc(sizeof(*a)))) {
		a->start_routine = start_routine;
		a->data = data;
		start_routine = dummy_start;
		if (ast_asprintf(&a->name, "%-20s started at [%5d] %s %s()",
			     start_fn, line, file, caller) < 0) {
			a->name = NULL;
		}
		data = a;
	}
#endif /* !LOW_MEMORY */

	return pthread_create(thread, attr, start_routine, data); /* We're in ast_pthread_create, so it's okay */
}
```

— `main/utils.c:1659-1704`

Seven things, in source order.

**The attribute object is borrowed or stack-allocated.** If the caller passed
`NULL`, the function takes `sizeof(pthread_attr_t)` bytes from its own frame
with `ast_alloca` (`include/asterisk/astmm.h:288` — a plain
`__builtin_alloca`) and initialises them. It never calls
`pthread_attr_destroy` on them. That is safe, because `pthread_create` copies
whatever it needs out of the attribute object before returning and because
there is nothing attached to *these* objects for a destroy to release — on
glibc an attribute object takes heap storage only from the few
`pthread_attr_set*_np` calls that need it, a CPU-affinity set chief among them,
and this layer sets only stack size, inherit-sched and detach state — but note
it for §7.5, where the detached wrapper does the same thing and *does*
destroy.

**A caller's attribute object is modified, not respected.** If the caller did
pass an `attr`, the statements that follow write to it: inherit-sched, then
stack size. A caller who set a stack size in the attribute object loses it,
because the `pthread_attr_setstacksize` at `:1687` overwrites it with
`AST_STACKSIZE` whenever the `stacksize` argument is `0` — and the four macros
in `utils.h` pass either `0` or `AST_BACKGROUND_STACKSIZE`, never a number of
the caller's own. There is no way through the macros to say "use the attribute
object's stack size".

**`errno` is used as a scratch variable.** `pthread_attr_setinheritsched` and
`pthread_attr_setstacksize` return an error number rather than setting `errno`,
so the code assigns the return value *to* `errno` and tests that, purely so
`strerror(errno)` on the following line reads the right thing. The cost is that
`errno` is clobbered on the success path too — set to `0` — which §7.9 shows is
visible to five callers.

**The inherit-sched block is Linux and FreeBSD only** (`:1672-1682`), and its
comment is the clearest statement in the tree of what the `-p` command-line
option actually does. `pthread_attr_init` defaults to
`PTHREAD_EXPLICIT_SCHED`, which would leave new threads at normal priority even
in a realtime Asterisk, so the layer switches every thread to
`PTHREAD_INHERIT_SCHED`. The comment also records the price: a caller cannot
ask for `PTHREAD_EXPLICIT_SCHED` through this layer and must call
`pthread_setschedparam` after the fact. No caller in the tree does.

**The stack-size test is written twice.** `:1684-1685` sets `stacksize` to
`AST_STACKSIZE` if it is zero; `:1687` then passes `stacksize ? stacksize :
AST_STACKSIZE`. After the preceding statement the ternary's false arm is
unreachable. It is harmless, and it is the clearest evidence in the function
that the two lines were written at different times.

**Both attribute failures are warnings, not errors.** Neither
`pthread_attr_setinheritsched` nor `pthread_attr_setstacksize` failing stops
anything: the function logs and carries on, so the thread is created with
whatever the attribute object holds — in the stack-size case the system
default of several megabytes rather than the intended 496 KiB. A build where
the stack size were consistently rejected would therefore produce a flood of
warnings and a set of unexpectedly large threads, not a failure.

**The registration wrapper is best-effort.** The `#if !defined(LOW_MEMORY)`
block at `:1690-1701` allocates a `struct thr_arg`, copies the real start
routine and its data into it, and substitutes `dummy_start` as the thread's
entry point. If `ast_malloc` fails the `if` simply does not fire:
`start_routine` and `data` stay as the caller passed them and the thread is
created *without* the wrapper. It will run correctly and it will be absent from
both registries of §7.6 and §7.8. Nothing says so: the failed `ast_malloc` logs
one generic allocation-failure line of its own (§7.11), attributed to
`utils.c:1691`, and there is no second line about a thread having gone missing.
Likewise if `ast_asprintf` fails the name is set to `NULL` and the thread is
registered anonymously; §7.7 covers what that prints.

The last statement is the only `pthread_create` in Asterisk, and the comment
tacked onto it — `/* We're in ast_pthread_create, so it's okay */` — is
addressed to the reader who just came from `lock.h:649`.

## 7.5 The detached variant, and the three callers that hand-roll it

```c
int ast_pthread_create_detached_stack(pthread_t *thread, pthread_attr_t *attr, void *(*start_routine)(void *),
			     void *data, size_t stacksize, const char *file, const char *caller,
			     int line, const char *start_fn)
{
	unsigned char attr_destroy = 0;
	int res;

	if (!attr) {
		attr = ast_alloca(sizeof(*attr));
		pthread_attr_init(attr);
		attr_destroy = 1;
	}

	if ((errno = pthread_attr_setdetachstate(attr, PTHREAD_CREATE_DETACHED)))
		ast_log(LOG_WARNING, "pthread_attr_setdetachstate: %s\n", strerror(errno));

	res = ast_pthread_create_stack(thread, attr, start_routine, data,
	                               stacksize, file, caller, line, start_fn);

	if (attr_destroy)
		pthread_attr_destroy(attr);

	return res;
}
```

— `main/utils.c:1707-1730`

It is a one-attribute wrapper around §7.4. Two details separate it from its
sibling.

It tracks ownership of the attribute object with `attr_destroy` and calls
`pthread_attr_destroy` only on an object it created itself — the correct
behaviour, and the opposite of what `ast_pthread_create_stack` does with the
object *it* creates. One of the two functions is wrong and it does not matter
which, because `ast_alloca` memory needs no release and, for an object holding
only the three settings this layer writes, `pthread_attr_destroy` has nothing
to free. The asymmetry is simply unexplained.

And when the caller does pass an attribute object, the
`pthread_attr_setdetachstate` at `:1720` writes `PTHREAD_CREATE_DETACHED` into
it and leaves it that way: the caller's object comes back changed. No call site
in the tree is affected, because no call site passes an attribute object to a
detached variant — which brings us to the three that pass one at all.

All three build an attribute object for exactly one purpose, setting the detach
state, which is the purpose the `_detached` macros exist to serve:

```c
								pthread_attr_t attr;
								pthread_t threadid;
								struct mwi_thread_data *mtd;

								pthread_attr_init(&attr);
								pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_DETACHED);

								ast_debug(1, "Maybe some MWI on port %d!\n", i->channel);
								if ((mtd = ast_calloc(1, sizeof(*mtd)))) {
									mtd->pvt = i;
									memcpy(mtd->buf, buf, res);
									mtd->len = res;
									i->mwimonitoractive = 1;
									if (ast_pthread_create_background(&threadid, &attr, mwi_thread, mtd)) {
										ast_log(LOG_WARNING, "Unable to start mwi thread on channel %d\n", i->channel);
										i->mwimonitoractive = 0;
										ast_free(mtd);
									}
```

— `channels/chan_dahdi.c:12141-12158`

That is `ast_pthread_create_background` with a detached attribute object —
`ast_pthread_create_detached_background` spelled out longhand, plus an
attribute object nobody destroys.

```c
	} else {
		pthread_attr_init(&attr);
		pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_JOINABLE);
		/* Start a new monitor */
		if (ast_pthread_create(&monitor_thread, &attr, do_monitor, NULL) < 0) {
			ast_mutex_unlock(&monlock);
```

— `channels/chan_unistim.c:5946-5951`

`PTHREAD_CREATE_JOINABLE` is the state of a freshly initialised attribute
object, so the `pthread_attr_setdetachstate` at `:5948` is a no-op — the
`pthread_attr_init` above it is not, since `&attr` is what gets passed. And
`chan_unistim` really does join this thread (`channels/chan_unistim.c:7174`),
so the intent is right even though the second line says nothing.

The third one is not redundant:

```c
		pthread_attr_init(&attr);
		pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_DETACHED);
		/* Start a new monitor */
		if (ast_pthread_create(&monitor_thread, &attr, do_monitor, NULL) < 0) {
			ast_mutex_unlock(&monlock);
```

— `addons/chan_ooh323.c:4102-4106`

That creates the monitor thread **detached** and keeps its `pthread_t` in a
file-static. At unload:

```c
	if (monitor_thread != AST_PTHREADT_NULL) {
		if (!ast_mutex_lock(&monlock)) {
			if (monitor_thread && (monitor_thread != AST_PTHREADT_STOP)) {
				pthread_cancel(monitor_thread);
				pthread_kill(monitor_thread, SIGURG);
				pthread_join(monitor_thread, NULL);
			}
```

— `addons/chan_ooh323.c:4335-4341`

`pthread_join` on a detached thread is undefined — `EINVAL` if you are lucky,
and if the thread has already exited the `pthread_t` may have been reused, in
which case the `pthread_cancel` and `pthread_kill` two lines above are aimed at
whatever thread inherited the identifier. `chan_unistim.c:7172-7174` is the
same three calls against a thread that was created joinable — `:7171` is the
guard above them. The two modules are near-copies of each other; one of them
changed its attribute object and did not change its teardown.

Where the detached/joinable choice *is* load-bearing, it is made explicitly.
`main/bridge.c` picks the macro from a flag:

```c
		if ((flags & AST_BRIDGE_IMPART_CHAN_MASK) == AST_BRIDGE_IMPART_CHAN_INDEPENDENT) {
			res = ast_pthread_create_detached(&bridge_channel->thread, NULL,
				bridge_channel_ind_thread, bridge_channel);
		} else {
			res = ast_pthread_create(&bridge_channel->thread, NULL,
				bridge_channel_depart_thread, bridge_channel);
		}
```

— `main/bridge.c:2043-2049`

The independent case is detached and nothing ever waits for it; the departable
case is joinable because `ast_bridge_depart` reaches
`pthread_join(bridge_channel->thread, NULL)` at `main/bridge.c:2139`. Of the
129 call sites, 52 use a detached variant.

## 7.6 `dummy_start`: what the wrapper buys, and what it costs

The substituted entry point, with the struct it receives:

```c
#if !defined(LOW_MEMORY)
/*
 * support for 'show threads'. The start routine is wrapped by
 * dummy_start(), so that ast_register_thread() and
 * ast_unregister_thread() know the thread identifier.
 */
struct thr_arg {
	void *(*start_routine)(void *);
	void *data;
	char *name;
};
```

— `main/utils.c:1588-1598`

```c
/*
 * on OS/X, pthread_cleanup_push() and pthread_cleanup_pop()
 * are odd macros which start and end a block, so they _must_ be
 * used in pairs (the latter with a '1' argument to call the
 * handler on exit.
 * On BSD we don't need this, but we keep it for compatibility.
 */
static void *dummy_start(void *data)
{
	void *ret;
	struct thr_arg a = *((struct thr_arg *) data);	/* make a local copy */
#ifdef DEBUG_THREADS
	struct thr_lock_info *lock_info;
	pthread_mutexattr_t mutex_attr;

	if (!(lock_info = ast_threadstorage_get(&thread_lock_info, sizeof(*lock_info))))
		return NULL;

	lock_info->thread_id = pthread_self();
	lock_info->lwp = ast_get_tid();
	lock_info->thread_name = ast_strdup(a.name);

	pthread_mutexattr_init(&mutex_attr);
	pthread_mutexattr_settype(&mutex_attr, AST_MUTEX_KIND);
	pthread_mutex_init(&lock_info->lock, &mutex_attr);
	pthread_mutexattr_destroy(&mutex_attr);

	pthread_mutex_lock(&lock_infos_lock.mutex); /* Intentionally not the wrapper */
	AST_LIST_INSERT_TAIL(&lock_infos, lock_info, entry);
	pthread_mutex_unlock(&lock_infos_lock.mutex); /* Intentionally not the wrapper */
#endif /* DEBUG_THREADS */

	/* note that even though data->name is a pointer to allocated memory,
	   we are not freeing it here because ast_register_thread is going to
	   keep a copy of the pointer and then ast_unregister_thread will
	   free the memory
	*/
	ast_free(data);
	ast_register_thread(a.name);
	pthread_cleanup_push(ast_unregister_thread, (void *) pthread_self());

	ret = a.start_routine(a.data);

	pthread_cleanup_pop(1);

	return ret;
}
```

— `main/utils.c:1600-1646`

The first statement copies the heap `struct thr_arg` into a local before
anything can free it, and `:1637` frees the heap copy — but not `a.name`, and
the comment above it explains why: the name pointer is handed to
`ast_register_thread`, which takes ownership, and `ast_unregister_thread`
releases it. §7.8 covers the case where that hand-off does not happen.

The `DEBUG_THREADS` block at `:1611-1630` is the other half of chapter 05
§5.2. It contains the **only** insertion into the `lock_infos` list anywhere in
the tree: one `AST_LIST_INSERT_TAIL` at `:1628`, and a grep for the list name
finds no other. Everything chapter 05 said about which threads `core show
locks` can see follows from that single line. A thread reaches the list if and
only if it was created through `ast_pthread_create_stack` or
`ast_pthread_create_detached_stack` — by one of the four macros, or by the
private macro of §7.10 that calls the same function — in a build that is
neither `LOW_MEMORY` (which removes `dummy_start` entirely — the guards are at
`:1588` and `:1648`) nor missing `DEBUG_THREADS` (which removes the block), and
in which the `ast_malloc` of §7.4 succeeded. The thread running `main()` fails
the first of those conditions: nothing wrapped it, so nothing enrolled it,
which is why it is invisible there.

Three details in that block repay attention. The `pthread_mutex_lock` on the
list is marked `/* Intentionally not the wrapper */` — taking `lock_infos_lock`
through `ast_mutex_lock` would recurse into the tracking code the list exists
to serve, which is the reasoning chapter 02 §2.2 gave for `logger.c`. The
`thread_name` at `:1620` is a *second* copy of the same string, owned by the
`thr_lock_info` record and freed by `lock_info_destroy` at `:1076-1078`, so one
thread's name exists twice on the heap under two different owners. And the
early exit at `:1615-1616` is the sharp edge: if threadstorage allocation
fails, `dummy_start` returns `NULL` **before** `ast_free(data)` and before
`a.start_routine` is ever called. The thread exits immediately, the `thr_arg`
and its name leak, and the caller's `pthread_create` already returned `0`, so
the caller believes its thread is running. The only trace is the generic
allocation-failure line of §7.11, attributed to `threadstorage.h` and saying
nothing about threads. It is a debug-build-only path behind an allocation
failure, and it is the one place in the layer where a successful return means
nothing at all.

`pthread_cleanup_push` at `:1639` paired with `pthread_cleanup_pop(1)` at
`:1643` is how deregistration survives every exit path: returning from the
start routine reaches the `pop(1)`, and `pthread_exit` or cancellation runs the
handler on the way out. That pairing is what the OS/X comment at `:1600-1606`
is defending. The argument is `pthread_self()` cast to `void *`, which is why
`ast_unregister_thread` takes a `void *` and compares it with a cast rather
than taking a `pthread_t`.

Two consequences follow from registration happening *inside* the new thread
rather than in its creator. The obvious one is a window: between
`pthread_create` returning to the caller and `dummy_start` reaching `:1638` the
thread exists and is not in `thread_list`, so a `core show threads` issued in
that window is short by one. `lock_infos` has its own, shorter window, closing
ten lines earlier at `:1628`. The subtler one is that the wrapper adds a frame
to every thread in Asterisk, so every thread's backtrace bottoms out in
`dummy_start` rather than in the function the call site named.

## 7.7 The thread name is whatever the preprocessor saw

The name is built in one `ast_asprintf`:

```c
		if (ast_asprintf(&a->name, "%-20s started at [%5d] %s %s()",
			     start_fn, line, file, caller) < 0) {
			a->name = NULL;
		}
```

— `main/utils.c:1695-1698`

Four fields: the stringified start-routine argument, the line, the file and the
calling function, in that order — so the file comes *after* the line number it
belongs to, and the function named is the one that *created* the thread, not
the one the thread runs. A typical result reads `do_monitor           started
at [ 5950] chan_unistim.c restart_monitor()`. `__FILE__` is whatever path the
compiler was given, which in the tree's own build is the name relative to the
directory `make` ran in — `chan_unistim.c` for `channels/chan_unistim.c`, but
`ari/ari_websockets.c` for the files `res/Makefile:66` builds out of a
subdirectory.

The `%-20s` assumes the start routine's name fits in twenty columns. `#c` is
the stringification of a *macro argument*, though, not of a function name, and
the tree puts expressions there. Three examples, each the only text that will
ever identify that thread:

```c
		if (ast_pthread_create(&cal->thread, NULL, cal->tech->load_calendar, cal)) {
```

— `res/res_calendar.c:518-518`

```c
	if (ast_pthread_create_background(&desc->master, NULL, desc->accept_fn, desc)) {
```

— `main/tcptls.c:912-912`

```c
		if (ast_pthread_create(&thread->thread, NULL, idx < thrash->update_threads ?
			sorcery_memory_cache_thrash_update : sorcery_memory_cache_thrash_retrieve, thread)) {
```

— `tests/test_sorcery_memory_cache_thrash.c:354-355`

The first two name a function-pointer field, so `core show threads` reports the
thread as `cal->tech->load_calendar` and `desc->accept_fn` — the mechanism
rather than the implementation, which in the `accept_fn` case is the one thing
that would distinguish the TCP listener thread from the TLS one and does not.
The third stringifies a conditional expression. The argument spans two source
lines with no backslash — it is a newline inside a macro invocation, which is
just white space — and the preprocessor collapses each run of white space in an
argument to a single space, so that thread's name
is the 104-character string `idx < thrash->update_threads ?
sorcery_memory_cache_thrash_update : sorcery_memory_cache_thrash_retrieve` —
both operands of a choice whose outcome the name cannot tell you.

If `ast_asprintf` fails, `a->name` is `NULL` (`:1697`), `ast_register_thread`
stores the null pointer, and `core show threads` prints it through `%s`. Glibc
renders that as `(null)`; the C standard requires nothing in particular.

Finally, and worth stating because it surprises people debugging a live system:
**Asterisk never tells the operating system any of this.** There is no
`pthread_setname_np` and no `prctl(PR_SET_NAME)` anywhere in the tree. Every
thread in `top -H`, in `ps -L` and in `/proc/*/task/*/comm` is called
`asterisk`. The names in this section exist only inside two of Asterisk's own
data structures, reachable only through `core show threads` and `core show
locks`, and the LWP id printed beside them — `ast_get_tid()`,
`main/utils.c:2786-2806` — is the only field that lets you match a line of CLI
output to a line of `top`.

## 7.8 The registry, and its two consumers

The list `core show threads` reads is not in `utils.c`. It is in
`main/asterisk.c`, inside that file's large `#if !defined(LOW_MEMORY)` region
(`:412` through `:806`):

```c
#if !defined(LOW_MEMORY)
struct thread_list_t {
	AST_RWLIST_ENTRY(thread_list_t) list;
	char *name;
	pthread_t id;
	int lwp;
};

static AST_RWLIST_HEAD_STATIC(thread_list, thread_list_t);
```

— `main/asterisk.c:412-420`

Three data fields: the name `dummy_start` built, the `pthread_t`, and the LWP
id — the same pair of identifiers `thr_lock_info` keeps, for the same reason.

```c
void ast_register_thread(char *name)
{
	struct thread_list_t *new = ast_calloc(1, sizeof(*new));

	if (!new)
		return;

	ast_assert(multi_thread_safe);
	new->id = pthread_self();
	new->lwp = ast_get_tid();
	new->name = name; /* steal the allocated memory for the thread name */
	AST_RWLIST_WRLOCK(&thread_list);
	AST_RWLIST_INSERT_HEAD(&thread_list, new, list);
	AST_RWLIST_UNLOCK(&thread_list);
}
```

— `main/asterisk.c:422-436`

`ast_assert(multi_thread_safe)` at `:429` is the only assertion in this
chapter's whole region, and in a stock build it is not an assertion at all:
outside `AST_DEVMODE`, `ast_assert` is defined as nothing
(`include/asterisk/utils.h:778-779`). The flag is set twice, and neither place
is where you would guess. `main()` (`main/asterisk.c:3621-4117`) sets it at
`:4082`, on the branch it takes when an Asterisk is already running and this
process is only a remote console. Everything else about startup happens in
`asterisk_daemon()` (`:4132` onward), which `main()` calls at `:4112` and which
sets the flag at `:4165`, immediately after the `daemon()` call at `:4151`.
Every `ast_pthread_create*` in the file runs after that: two in
`asterisk_daemon` itself (`:4204`, `:4407`), one in `ast_makesocket`, which
`asterisk_daemon` calls at `:4287` (`:1654`), and one in `listener`'s accept
loop at run time (`:1588`). The check exists to catch a thread created before
the fork, which would not survive it.

`new->name = name`, with its comment, is where the ownership established in
§7.6 lands. The failure mode is three lines above it: if `ast_calloc` fails,
`ast_register_thread` returns without storing the pointer, so the name string
that `ast_asprintf` allocated in §7.4 is leaked, nothing is registered, and the
matching `ast_unregister_thread` later finds no entry and frees nothing.

```c
void ast_unregister_thread(void *id)
{
	struct thread_list_t *x;

	AST_RWLIST_WRLOCK(&thread_list);
	AST_RWLIST_TRAVERSE_SAFE_BEGIN(&thread_list, x, list) {
		if ((void *) x->id == id) {
			AST_RWLIST_REMOVE_CURRENT(list);
			break;
		}
	}
	AST_RWLIST_TRAVERSE_SAFE_END;
	AST_RWLIST_UNLOCK(&thread_list);
	if (x) {
		ast_free(x->name);
```

— `main/asterisk.c:438-452`

A linear search by `pthread_t`, compared with `==`. The tree does this nearly
everywhere: `AST_PTHREADT_NULL` and `AST_PTHREADT_STOP` are `(pthread_t) -1`
and `(pthread_t) -2` (`include/asterisk/lock.h:73-74`), which works only if
`pthread_t` is a scalar type — as it is on glibc and as POSIX does not require.
`pthread_equal`, the portable comparison, is used six times in four files
(`main/taskprocessor.c:296` and `:1519`, `main/bridge_channel.c:262` and
`:949`, `main/autoservice.c:312`, `res/res_xmpp.c:3657`), and two of the six
sit on the same line as an `!= AST_PTHREADT_NULL` test of the same value, so
the portable form and the form it exists to replace appear together.

The registry has exactly two readers. The first is the CLI command the comment
in `utils.c` named:

```c
	AST_RWLIST_RDLOCK(&thread_list);
	AST_RWLIST_TRAVERSE(&thread_list, cur, list) {
		ast_cli(a->fd, "%p %d %s\n", (void *)cur->id, cur->lwp, cur->name);
		count++;
	}
	AST_RWLIST_UNLOCK(&thread_list);
	ast_cli(a->fd, "%d threads listed.\n", count);
```

— `main/asterisk.c:631-637`

Three columns, no header and no sorting, newest first because
`ast_register_thread` inserts at the head. The `%s` is the whole composite name
of §7.7, so each line is wide and self-describing, and the `%p` is a
`pthread_t` printed as a pointer.

The second reader is not about introspection at all:

```c
	}
	ast_verb(1, "Setting %s thread priority on all threads\n", policy_str);
	AST_RWLIST_RDLOCK(&thread_list);
	AST_RWLIST_TRAVERSE(&thread_list, cur, list) {
		/* Don't care about the return value. It should work. */
		sched_setscheduler(cur->lwp, policy, &sched);
	}
	AST_RWLIST_UNLOCK(&thread_list);
```

— `main/asterisk.c:1838-1845`

That is the tail of `set_priority_all` (`main/asterisk.c:1811-1848`), which is
what the `-p` option and a runtime priority change reach. It walks the same
list to call `sched_setscheduler` on each thread's **LWP id**, not its
`pthread_t` — the only consumer of the `lwp` field outside diagnostics, and the
reason the field is stored at all. Note the two arms above it
(`main/asterisk.c:1813-1818`): on a non-Linux build the function sets the
process priority and returns, and on a `LOW_MEMORY` build it logs `Unable to
enumerate all threads to update priority` and does the same. Only the second of
those is a build without a registry. The list is guarded by `#if
!defined(LOW_MEMORY)` and by nothing else, so a non-Linux Asterisk has a
perfectly good `core show threads` and merely has no portable way to reach each
thread's scheduler; it is the `LOW_MEMORY` warning, and only that, that reports
the registry's absence to the operator.

Which is the summary of this section. Under `LOW_MEMORY`, `struct
thread_list_t`, `ast_register_thread`, `ast_unregister_thread`,
`handle_show_threads` and its `AST_CLI_DEFINE` (`main/asterisk.c:2651`)
are all compiled out, and so is the `dummy_start` that would have called them.
The declarations at `utils.h:613-614` remain, declaring two functions that do
not exist; nothing references them, so nothing fails to link. But a
`LOW_MEMORY` Asterisk has no thread list, no `core show threads`, no
per-thread priority setting and — because `dummy_start` is the only writer of
`lock_infos` — no `core show locks` content either.

## 7.9 What `errno` says after a failed thread creation

`pthread_create` reports failure by return value and is not required to touch
`errno`. Five call sites print `strerror(errno)` anyway:

```c
		if (ast_pthread_create_detached_background(&launched, NULL, handle_tcptls_connection, tcptls_session)) {
			ast_log(LOG_ERROR, "TCP/TLS unable to launch helper thread for peer '%s': %s\n",
				ast_sockaddr_stringify(&tcptls_session->remote_address),
				strerror(errno));
			ao2_ref(tcptls_session, -1);
```

— `main/tcptls.c:360-364`

The other four are `main/tcptls.c:912-916` — the same call site §7.7 quoted for
a different reason — `main/asterisk.c:1588-1590`, `channels/sig_pri.c:9131-9139`
and `res/res_aeap/aeap.c:352-355`.

Follow what §7.4 did to `errno` on the way in. On Linux the last two writes to
it before `pthread_create` runs are the assignments at `:1680` and `:1687`,
both of which store the return value of a `pthread_attr_set*` call that
normally succeeds — that is, `0`. For the detached variants there is a third
such assignment at `:1720`, with the same result. `pthread_create` then fails
and returns `EAGAIN` or `EINVAL` to its caller without, as far as this code
path is concerned, writing `errno` at all.

So the likeliest rendering of all five messages is `strerror(0)`: `TCP/TLS
unable to launch helper thread for peer '10.0.0.1:5060': Success`. Whatever
libc happens to have left in `errno` internally may make it something else,
which is worse rather than better — the one thing it will not reliably be is
the reason the thread was not created. That value is the return code these
five sites test and discard. The layer gives them no other way to get it:
`ast_pthread_create_stack` returns `pthread_create`'s code unchanged, so the
fix at each site is to keep the return value rather than spend it on an `if`.

## 7.10 The 129 call sites

By macro: 44 `ast_pthread_create_background`, 35
`ast_pthread_create_detached`, 33 `ast_pthread_create` and 17
`ast_pthread_create_detached_background`. By directory: 30 in `main`, 28 in
`channels`, 24 in `res`, 20 in `tests`, 9 in `pbx`, 8 each in `apps` and
`addons`, and one each in `funcs` and `bridges`. The 20 in `tests` are test
modules, so 109 create threads in shipping code.

What they are for falls into four groups, and the macro choice tracks the group
loosely rather than reliably.

Long-lived singletons, one per subsystem, started at load and running until
shutdown: `logger_thread` (`main/logger.c:2208`), `sched_run`
(`main/sched.c:216`), `do_devstate_changes` (`main/devicestate.c:624`),
`do_cdr` (`main/cdr.c:4582`), `do_timing` (`res/res_timing_pthread.c:428`),
`listener` (`main/asterisk.c:1654`). These mostly use `ast_pthread_create` or
`ast_pthread_create_background` and are either joined at shutdown or never
stopped.

Per-driver monitor threads, one per loaded channel driver, polling hardware or
a socket. `do_monitor` is the start routine at three call sites in three
different modules (`channels/chan_dahdi.c:12284`,
`channels/chan_unistim.c:5950`, `addons/chan_ooh323.c:4105`), and
`chan_mobile` has two more under per-transport names. `chan_dahdi`'s
`pri_dchannel` and `ss7_linkset` are the same pattern at per-link granularity.

Per-call threads, created and destroyed at call rates: `__analog_ss_thread`
(six call sites in `channels/sig_analog.c` alone), `analog_ss_thread`,
`pri_ss_thread`, `pbx_thread`, `bridge_channel_ind_thread`,
`handle_tcptls_connection`, `ari_channel_thread`. Almost all of these use a
detached variant, which is the one group where the macro choice is consistently
meaningful: nobody joins a per-call thread.

And one-shot workers: `fast_originate`, `mixmonitor_thread`, `recordthread`,
`snoop_stasis_thread`, `canary_thread`. `main/asterisk.c:4203-4204` is the
purest example — a `pthread_t dont_care;` and a detached creation, a thread
identifier the code declares only because the API demands one.

Two modules decide the layer's stack size is not enough for them, and they do
it two different ways. The first writes a fifth macro of the same shape:

```c
/*! \brief Version of pthread_create to ensure stack is large enough */
#define corosync_pthread_create_background(a, b, c, d)				\
	ast_pthread_create_stack(a, b, c, d,					\
		(AST_BACKGROUND_STACKSIZE + (3 * COROSYNC_IPC_BUFFER_SIZE)),	\
		__FILE__, __FUNCTION__, __LINE__, #c)
```

— `res/res_corosync.c:91-95`

`COROSYNC_IPC_BUFFER_SIZE` is `8192 * 128` (`res/res_corosync.c:89`), so that
asks for `AST_BACKGROUND_STACKSIZE` plus 3 MiB — about 3.5 MiB on a 64-bit
build, seven times the layer's default. It is the only place outside
`main/utils.c` that names `ast_pthread_create_stack` or
`ast_pthread_create_detached_stack`, and it is the model to copy if a thread of
yours needs a different stack: write the macro, keep the four captured
arguments, and the registration of §7.6 still happens.

The second way is the one to avoid. `addons/ooh323cDriver.c` replaces the
core's macro outright:

```c
#undef AST_BACKGROUND_STACKSIZE
#define AST_BACKGROUND_STACKSIZE 768 * 1024
```

— `addons/ooh323cDriver.c:25-26`

Those two lines come after the `asterisk/pbx.h` include that reaches `utils.h`,
so they take effect, and the three background-variant call sites in that file
(`:169`, `:201`, `:206`) ask for 768 KiB instead of calling
`ast_background_stacksize()`. They are the only three threads in the tree whose
stack size is fixed by the preprocessor in the *module's* translation unit,
which is to say the only three on which `LOW_MEMORY` has no effect. The macro
still has "background" in its name and the call sites still read exactly like
the other 58.

## 7.11 Messages

The layer emits three `ast_log` strings and `main/asterisk.c` three more. None
of them is an error.

| Message | Origin | Means |
|---|---|---|
| `pthread_attr_setinheritsched: %s` | `main/utils.c:1681` | Could not switch the new thread to `PTHREAD_INHERIT_SCHED`. Logged and ignored, so a realtime Asterisk silently gains a thread at default priority (§7.4) |
| `pthread_attr_setstacksize: %s` | `main/utils.c:1688` | Could not set the stack size. Logged and ignored, so the thread gets the system default — typically 8 MiB rather than 496 KiB (§7.3, §7.4) |
| `pthread_attr_setdetachstate: %s` | `main/utils.c:1721` | Could not request detachment. Logged and ignored, so the thread is created *joinable* and nobody will join it — a thread descriptor held for the life of the process (§7.5) |
| `Unable to enumerate all threads to update priority` | `main/asterisk.c:1817` | Linux `LOW_MEMORY` build: there is no registry to walk, so only the process priority changes. The only operator-visible notice that the thread list does not exist (§7.8) |
| `Unable to set %s thread priority on main thread` | `main/asterisk.c:1836` | `sched_setscheduler` on the process failed; `set_priority_all` returns `-1` without touching the registry (§7.8) |
| `Setting %s thread priority on all threads` | `main/asterisk.c:1839` | `ast_verb(1, …)` rather than a log line, and the only positive confirmation that the registry walk happened (§7.8) |

What is *not* in that table is the point of it. Four distinct failure modes
leave `ast_pthread_create*` returning `0`: the `ast_malloc` at
`main/utils.c:1691` failing, so a thread is created unregistered; the
`ast_asprintf` at `:1695` failing, so a thread is named `(null)`; the
`ast_calloc` at `main/asterisk.c:424` failing, so the name is leaked and
nothing is registered; and `dummy_start` returning early at `:1616`, so the
start routine never runs at all.

Each of those four is an allocation failure, and every allocator behind the
`ast_malloc` family reports one — `Memory Allocation Failure in function %s at
line %d of %s`, from the `MALLOC_FAILURE_MSG` macro at `main/astmm.c:70-71`,
through `ast_log_safe`. So the events are not quite silent. But that line
names the allocation and not the consequence: read literally it says a
`calloc` of a few dozen bytes failed inside `ast_register_thread`, where what
it means is that a thread is now running which `core show threads` will never
list and `-p` will never reach. No message anywhere in the tree says the
second thing.

## 7.12 Rules

| Rule | Why |
|---|---|
| Never call `pthread_create` directly | The thread would be absent from both registries, `core show threads` and `core show locks` could not see it, `-p` would not reach it, and it would get an 8 MiB stack instead of 496 KiB. The poison that would stop you is disabled on Linux (§7.1, §7.3) |
| Do not read `_background` as "cheap" or "low priority" | It changes nothing in a build that is not `LOW_MEMORY`, where it asks for 112 KiB instead of 496 KiB. It has no effect on scheduling priority at all (§7.2) |
| Assume roughly 496 KiB of stack, not megabytes | Threads created through this layer are sized by `AST_STACKSIZE`, computed from the pointer size and about a sixteenth of the Linux default — less under `LOW_MEMORY`, more only in the two modules that override it (§7.10). Deep recursion and large stack buffers are the two ways to discover this the hard way (§7.3) |
| Pass an attribute object only for things this layer does not set | It overwrites the stack size unconditionally and inherit-sched on Linux and FreeBSD, and the detached variant also overwrites detach state and hands your object back changed (§7.4, §7.5) |
| Use `ast_pthread_create_detached*` rather than building a detached attribute object | The three call sites that hand-roll it are the three with a defect: one duplicates the macro, one sets a state that is already the default, and one detaches a thread it later joins (§7.5) |
| Keep the return value of `ast_pthread_create*`; do not print `strerror(errno)` | The error is the return value. `errno` was last written by a `pthread_attr_set*` call that succeeded, so the message is likely to read `Success` (§7.9) |
| A `0` return does not prove the start routine ran | Under `DEBUG_THREADS` a threadstorage allocation failure makes `dummy_start` return before calling it, and the only log line is a generic allocation failure (§7.6, §7.11) |
| Expect every thread's backtrace to pass through `dummy_start` | It is the real entry point of every thread the layer creates in a normal build; the function the call site named is one frame up (§7.6) |
| Do not expect the operating system to know your thread's name | Nothing in the tree calls `pthread_setname_np`. Match CLI output to `top -H` by the LWP id, which both print (§7.7) |
| Do not expect `#c` to be a function name | It is a stringified macro argument. Three call sites put a struct member or a conditional expression there, and that text is the thread's only name (§7.7) |
| Treat `LOW_MEMORY` as removing thread introspection entirely | No `thread_list`, no `core show threads`, no per-thread priority, and — because `dummy_start` is the only writer of `lock_infos` — no `core show locks` content either (§7.6, §7.8) |

## Next

Chapter 08 covers the atomics: `ast_atomic_*` as a thin layer over the
compiler's `__atomic` built-ins with a `__sync` fallback, the `memorder`
argument the fallback erases in the preprocessor, the two older helpers
`ast_atomic_fetchadd_int` and `ast_atomic_dec_and_test` that most of the tree
still uses, and the compare-exchange that is defined and then disabled.
