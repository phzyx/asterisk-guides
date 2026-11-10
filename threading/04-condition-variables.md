# 4. Condition variables

Every `file:line` citation in this guide was verified against the `24` branch.
Newer releases and other branches may differ — line numbers drift — so if you
are reading this against a different tree, treat the citations as approximate
and re-grep for the symbol name.

Chapters 02 and 03 were about a wrapper that does a great deal around a pthread
call. This one is about a wrapper that mostly does nothing, and about the one
place where it does something no other entry point in `main/lock.c` does: it
takes a lock's tracking record, puts it aside, and puts it back.

The whole facility is 176 lines, `:537-712`, which is 13% of `main/lock.c`.
Four of its six entry points are one-line pass-throughs. The remaining two are
the same function twice.

## 4.1 `ast_cond_t` is a bare typedef

Chapter 01 quoted the three typedefs together. The third one is the odd one:

```c
typedef pthread_cond_t ast_cond_t;
```

— `include/asterisk/lock.h:185`

`ast_mutex_t` and `ast_rwlock_t` are `struct ast_mutex_info` and `struct
ast_rwlock_info`, each wrapping its pthread object alongside a `track` pointer
and a `flags` word. `ast_cond_t` is the pthread object and nothing else. There
is no `struct ast_cond_info`, no `ast_cond_track`, no `#if !defined(DEBUG_THREADS)`
arm renaming fields so they cannot be touched in the wrong build. A condition
variable carries no debug state at all, in any build.

That has a direct consequence for debugging: nothing in Asterisk records who
signalled a condition variable, who is waiting on it, or how many times. The
only reason `__FILE__`, `__LINE__` and `__PRETTY_FUNCTION__` are threaded
through the macros is that the two `wait` functions use them — and they use
them to describe the *mutex*, not the condition variable.

```c
#define ast_cond_init(cond, attr)             __ast_cond_init(__FILE__, __LINE__, __PRETTY_FUNCTION__, #cond, cond, attr)
#define ast_cond_destroy(cond)                __ast_cond_destroy(__FILE__, __LINE__, __PRETTY_FUNCTION__, #cond, cond)
#define ast_cond_signal(cond)                 __ast_cond_signal(__FILE__, __LINE__, __PRETTY_FUNCTION__, #cond, cond)
#define ast_cond_broadcast(cond)              __ast_cond_broadcast(__FILE__, __LINE__, __PRETTY_FUNCTION__, #cond, cond)
#define ast_cond_wait(cond, mutex)            __ast_cond_wait(__FILE__, __LINE__, __PRETTY_FUNCTION__, #cond, #mutex, cond, mutex)
#define ast_cond_timedwait(cond, mutex, time) __ast_cond_timedwait(__FILE__, __LINE__, __PRETTY_FUNCTION__, #cond, #mutex, cond, mutex, time)
```

— `include/asterisk/lock.h:208-213`

Two details worth noticing before moving on. The two `wait` macros stringify
*both* arguments, `#cond` and `#mutex`, so the entry points receive two names —
and `cond_name` is the one that is never used. And `ast_cond_init` passes its
`attr` argument through to a raw `pthread_condattr_t *`: this is the only place
in the lock API where a pthread attribute type appears in a public signature
(`lock.h:201`). §4.9 is about what Asterisk does with it, which is nothing.

## 4.2 Four of the six entry points do nothing

```c
int __ast_cond_init(const char *filename, int lineno, const char *func,
				  const char *cond_name, ast_cond_t *cond, pthread_condattr_t *cond_attr)
{
	return pthread_cond_init(cond, cond_attr);
}

int __ast_cond_signal(const char *filename, int lineno, const char *func,
				    const char *cond_name, ast_cond_t *cond)
{
	return pthread_cond_signal(cond);
}

int __ast_cond_broadcast(const char *filename, int lineno, const char *func,
				       const char *cond_name, ast_cond_t *cond)
{
	return pthread_cond_broadcast(cond);
}

int __ast_cond_destroy(const char *filename, int lineno, const char *func,
				     const char *cond_name, ast_cond_t *cond)
{
	return pthread_cond_destroy(cond);
}
```

— `main/lock.c:537-559`

That is the complete text of four of the six entry points. Set it against the
mutex and rwlock acquisition paths, which run to roughly a hundred lines each:
there is no `#ifdef DEBUG_THREADS` anywhere in these twenty-three lines, which
means all four are byte-identical in a debug build and a stock build. And
`filename`, `lineno`, `func` and `cond_name` are unused in all four — not
"unused in a stock build", unused always.

So every `ast_cond_signal` in Asterisk costs exactly one `pthread_cond_signal`
plus four arguments pushed and discarded. If you are reading a profile and
wondering whether the debug wrapper is in the way, for these four it is not.

The practical consequence is the one that matters when something goes wrong: a
condition variable that is never signalled, or signalled before anyone waits,
produces no diagnostic from this layer. There is nothing here that could
produce one. Whatever you learn about a stuck `ast_cond_wait` you will learn
from the *mutex* side, which §4.5 is about, or from a backtrace.

## 4.3 Why nothing escapes the wrapper

`main/lock.c` is allowed to call the pthread functions because it `#undef`s
them at the top of the file — chapter 01 quoted that block (`:43-55`). Here is
what it is undoing:

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
```

— `include/asterisk/lock.h:628-642`

Rename-to-garbage is a blunt instrument, and it is worth knowing it is here
because the resulting compiler error names a symbol that does not exist
anywhere, which is confusing the first time you see it. It is also worth
knowing how well it works. Grep the tree for `pthread_cond_` outside
`third-party/` and the standalone `utils/extconf.c`, and the only two files
that match are `main/lock.c` and `include/asterisk/lock.h` itself. Sixty-eight
`.c` files use the `ast_cond_*` family; not one of them reaches past it. For a
convention enforced by nothing but a preprocessor prank, that is close to a
clean sweep.

Close, not clean. The list renames the type and the functions; it does not
rename `PTHREAD_COND_INITIALIZER`, which is not mentioned in `lock.h` at all.
So `apps/app_voicemail.c:1135` compiles:

```c
static ast_cond_t poll_cond = PTHREAD_COND_INITIALIZER;
```

— `apps/app_voicemail.c:1135`

That is the only occurrence of the macro in the tree outside `third-party/`,
and `app_voicemail.c` calls `ast_cond_init` zero times. The variable is used at
`:14036` (`ast_cond_timedwait`) and `:14159` (`ast_cond_signal`), so it is a
working condition variable initialised by a route the wrapper layer never sees.
The gap is harmless here — §4.2 showed `__ast_cond_init` is a one-line
passthrough to `pthread_cond_init` that discards all four of its tracking
arguments, so static initialisation reaches the same state by a shorter route —
but it is a hole, and the interesting part is that it is not an oversight of
kind. Two lines past the end of the block above, `lock.h:644` poisons an
initialiser macro by exactly the same trick:

```c
#define AST_MUTEX_INITIALIZER __use_AST_MUTEX_DEFINE_STATIC_rather_than_AST_MUTEX_INITIALIZER__
```

— `include/asterisk/lock.h:644`

So whoever wrote this block did think about static initialisers, and did close
the mutex door. There is no condition-variable equivalent, and that is the one
`app_voicemail.c` walks through.

Note that the typedef poisoning at `:629-630` is inside `#ifndef __CYGWIN__`
while the eleven function poisons at `:632-642` are unconditional, and that
`lock.h:185` can typedef `ast_cond_t` from `pthread_cond_t` only because it
appears 445 lines *before* the `#define` that renames it. The undo block in
`main/lock.c` is one short of the thirteen names defined here: it `#undef`s
`pthread_mutex_t` but not `pthread_cond_t`, which costs nothing, because the
file refers to the type as `ast_cond_t` throughout. The rwlock family is not
poisoned at all — there is no `pthread_rwlock_*` entry in this list, which is
why chapter 03 had nothing to say about it.

## 4.4 `wait` checks ownership, with the check `unlock` has

Everything interesting in this chapter is in `__ast_cond_wait` (`:584-647`) and
its twin. The first half is a familiar shape:

```c
	lt = ast_get_reentrancy(&t->track, &t->flags, 0);
	if (lt) {
		ast_reentrancy_lock(lt);
		if (lt->reentrancy && (lt->thread_id[ROFFSET] != pthread_self())) {
			log_mutex_error(canlog, "%s line %d (%s): attempted wait using mutex '%s' without owning it!\n",
					   filename, lineno, func, mutex_name);
			log_mutex_error(canlog, "%s line %d (%s): '%s' was locked here.\n",
					   lt->file[ROFFSET], lt->lineno[ROFFSET], lt->func[ROFFSET], mutex_name);
#ifdef HAVE_BKTR
			__dump_backtrace(&lt->backtrace[ROFFSET], canlog);
#endif
			DO_THREAD_CRASH;
		} else if (lt->reentrancy <= 0) {
			log_mutex_error(canlog, "%s line %d (%s): attempted wait using an unlocked mutex '%s'\n",
					   filename, lineno, func, mutex_name);
			DO_THREAD_CRASH;
		}

		/* Waiting on a condition completely suspends a recursive mutex,
		 * even if it's been recursively locked multiple times. Make a
		 * copy of the lock tracking, and reset reentrancy to zero */
		lt_orig = *lt;
		lt->reentrancy = 0;
		ast_reentrancy_unlock(lt);

		ast_suspend_lock_info(t);
	}
```

— `main/lock.c:604-630`

The `if` is the same test chapter 02 §2.8 found in `unlock`: compare
`thread_id[ROFFSET]` against `pthread_self()`, and on a mismatch print the
offender's line, the holder's line, and the holder's backtrace. That is the
report chapter 02 called the most useful message in the subsystem. The string
`without owning it` occurs at exactly three sites in `main/lock.c` — `:476` in
`unlock`, and `:608` and `:673` here — so outside `unlock` it is produced only
by these two functions, which are copies of each other. Chapter 03 noted that rwlock
unlock has no such check; `ast_cond_wait` does, because waiting on a condition
variable with a mutex you do not own is as wrong as unlocking one you do not
own, and for the same reason.

The `else if (lt->reentrancy <= 0)` arm catches the other way to get it wrong:
nobody holds the mutex, including you. It is `<= 0` rather than `== 0`, which
is defensive rather than meaningful — chapter 02 established that `reentrancy`
reaching a negative value is itself diagnosed elsewhere.

Now the part that matters. **Neither arm returns.** Both end in
`DO_THREAD_CRASH` and nothing else, so unless you built with `THREAD_CRASH`
(chapter 02 §2.1), the macro expands to `do { } while (0)` and execution falls
straight through into the save-and-suspend block below, and then into the real
`pthread_cond_wait`. You get the message *and* the wait. This is the same
fall-through that chapter 02 §2.8 described for `unlock`, and in a stock build
it is also moot, because the whole block is inside `#ifdef DEBUG_THREADS`.

Eight of the twenty-two `DO_THREAD_CRASH` sites in `main/lock.c` are in these
176 lines. Proportionally, the condition-variable code is the most
crash-on-error part of the file by a wide margin, and all eight are no-ops in
any build you are likely to be running.

### What the fall-through costs

The unowned case is worth following as far as the source goes, because the
fall-through means a diagnostic path keeps running after it has diagnosed a
fault. Thread W calls `ast_cond_wait` on a mutex that thread H holds. The first
arm fires and logs both lines. Then, before any `pthread` call, the quoted block
above reaches across into H's bookkeeping twice. `lt_orig = *lt` saves H's
entry and `lt->reentrancy = 0` erases the live copy of it — both on `t->track`,
which belongs to the lock rather than to either thread, which is exactly why W
can reach it. That much is settled by the code on the page, and it is already
enough to make `core show locks` misdescribe who holds what.

The third call in the block, `ast_suspend_lock_info(t)`, does *not* reach H.
It resolves its list through `ast_threadstorage_get` (§4.5), so it sees only
W's own per-thread list, and `t` is not in it — W never locked `t`. The
backwards search misses, `i` lands on `-1`, and the function returns at
`main/utils.c:1238` having written nothing. The suspension half of §4.5's two
mechanisms simply does not happen on this path.

What happens at the `pthread_cond_wait` two lines further on is not settled by
the source, and this is where the trace has to stop. Asterisk mutexes are
recursive, by two routes that agree. `AST_MUTEX_KIND` is
`PTHREAD_MUTEX_RECURSIVE_NP` or `PTHREAD_MUTEX_RECURSIVE`
(`include/asterisk/lock.h:84`, `:87`) and is applied at `main/lock.c:170` to
every mutex that goes through `__ast_pthread_mutex_init`; the 76
`AST_MUTEX_DEFINE_STATIC` sites never reach that function on Linux, because
`AST_MUTEX_INIT_W_CONSTRUCTORS` is Solaris/BSD only (`lock.h:76-78`), and they
get the same recursive kind from `PTHREAD_MUTEX_INIT_VALUE` at `lock.h:83`
instead. Either way the mutex is recursive, and POSIX leaves it undefined what
`pthread_cond_wait` does when the calling thread does not own the mutex it is
handed. Whether W blocks, or returns an error without ever blocking, is your
library's business, exactly like the writer preference in chapter 03 §3.3 and
the condition-variable clock in §4.9, and this chapter will not assert it. Note
only that the two possibilities diverge sharply: an error return goes down
§4.7's error branch, which restores nothing, while a successful wait eventually
runs `restore_lock_tracking` and writes H's stale entry back over whatever is
there by then.

The part that does not depend on any of that is the advice. Once you see
`attempted wait using mutex ... without owning it!`, the tracking for that mutex
has already been rewritten by a thread that had no business touching it, and
nothing downstream puts it right. Stop trusting what the lock tracking tells you
about that mutex, and go read the two logged lines instead — they name both the
caller and the real holder, which is the one piece of information here that is
reliable.

## 4.5 Suspending a mutex means two records, by two different mechanisms

The comment at `:622-624` explains the problem in one sentence: a condition
wait completely suspends a recursive mutex, however many times it was locked.
A thread that has locked `t` three times and then waits is holding it zero
times until it wakes. Both of the structures chapter 03 §3.5 distinguished have
to be told, and they are told differently.

The per-lock `struct ast_lock_track` is handled by hand, in the two lines at
`:625-626`: copy the whole struct to a stack local, then zero `reentrancy`.
Zeroing the count is all that is needed, because every reader of the array
tests `reentrancy` before indexing it — which is exactly the property chapter
01 §1.4 described for `ROFFSET`. The stale `file`, `lineno`, `func` and
`thread_id` slots are left in place and are simply unreachable until the count
comes back.

The per-thread `struct thr_lock_info` is handled by a dedicated pair of
functions, and it is not zeroed — it gets a flag:

```c
	pthread_mutex_lock(&lock_info->lock);

	for (i = lock_info->num_locks - 1; i >= 0; i--) {
		if (lock_info->locks[i].lock_addr == lock_addr)
			break;
	}

	if (i == -1) {
		/* Lock not found :( */
		pthread_mutex_unlock(&lock_info->lock);
		return;
	}

	lock_info->locks[i].suspended = 1;

	pthread_mutex_unlock(&lock_info->lock);
```

— `main/utils.c:1228-1243`

That is the body of `ast_suspend_lock_info` (`:1218-1245`);
`ast_restore_lock_info` (`:1247-1273`) is the same function, differing in three
regions and nothing else: the name in the signature, the brace style on the
early-return guard, and `= 0` in place of `= 1`. Both search the calling thread's own list backwards by address and
return silently if they do not find it. That is the same backwards-linear-search
shape chapter 03 §3.6 traced in rwlock unlock — keyed on `thread_id` there
(`main/lock.c:821`) and on the lock's address here, over a per-lock array there
and a per-thread list here — and §3.7 showed that one also treats a miss as
nothing to do rather than as an error. Chapter 03 counted that silence a defect,
because unlocking a lock you do not hold is a bug worth naming. Here it is
merely the right thing to do, since the two functions are bookkeeping rather
than enforcement.

`suspended` is one bit in the per-lock-per-thread record (`:1016`). It matters
because of what reads it:

```c
		for (i = 0; str && i < lock_info->num_locks; i++) {
			/* Don't show suspended locks */
			if (lock_info->locks[i].suspended) {
				continue;
			}
```

— `main/utils.c:1505-1509`

That is `ast_dump_locks` (`:1473`), which is what `core show locks` prints. A
thread parked in `ast_cond_wait` is not shown as holding the mutex, which is
correct and is the whole point: it is not holding it. If you are reading
`core show locks` while hunting a hang, this is the reason a thread you know is
blocked on a condition variable shows up holding fewer locks than you expected,
or shows up with no locks and therefore no thread header at all.

There is one loose end. `append_lock_information` (`:1363`) has a field for
this — it appends the literal `" - suspended"` at `:1398` when the bit is set.
But the loop that calls it at `:1522` has already `continue`d past every
suspended entry, at `:1507`, and the function's only other caller,
`ast_log_show_lock` (`:1440`), has no callers anywhere in the tree. So
`" - suspended"` cannot be printed. The bit is write-only as far as any output
you can reach is concerned; its observable effect is an absence.

## 4.6 `restore_lock_tracking` is a hand-maintained copy of a struct definition

```c
static void restore_lock_tracking(struct ast_lock_track *lt, struct ast_lock_track *lt_saved)
{
	ast_reentrancy_lock(lt);

	/*
	 * The following code must match the struct ast_lock_track
	 * definition with the explicit exception of the reentr_mutex
	 * member.
	 */
	memcpy(lt->file, lt_saved->file, sizeof(lt->file));
	memcpy(lt->lineno, lt_saved->lineno, sizeof(lt->lineno));
	lt->reentrancy = lt_saved->reentrancy;
	memcpy(lt->func, lt_saved->func, sizeof(lt->func));
	memcpy(lt->thread_id, lt_saved->thread_id, sizeof(lt->thread_id));
#ifdef HAVE_BKTR
	memcpy(lt->backtrace, lt_saved->backtrace, sizeof(lt->backtrace));
#endif

	ast_reentrancy_unlock(lt);
}
```

— `main/lock.c:562-581`

This is the only function in `main/lock.c` whose correctness depends on a
comment being obeyed. `struct ast_lock_track` (`lock.h:118-128`, quoted in
chapter 01) has seven members: `file`, `lineno`, `reentrancy`, `func`,
`thread_id`, `backtrace` under `HAVE_BKTR`, and `reentr_mutex`. The six lines
above restore the first six in declaration order. In this tree they match, so
the function is correct — but nothing checks that, and a new member added to
the struct would be silently dropped on every condition wait rather than
failing to compile.

The asymmetry is the interesting part. The *save* is a single struct
assignment, `lt_orig = *lt` at `:625`, which copies all seven members including
`reentr_mutex` — a `pthread_mutex_t` copied by value, while it is locked, since
`:606` took `ast_reentrancy_lock(lt)` first. Copying a pthread object like that
means nothing; the copy is never used as a mutex, only as a source of the six
field reads above, so it is harmless. But it is why the restore has to be written out
member by member instead of being another struct assignment: writing the whole
struct back would overwrite the live `reentr_mutex` with a snapshot of itself
taken in a locked state, and that one would matter.

## 4.7 The error path does not restore

```c
	res = pthread_cond_wait(cond, &t->mutex);

#ifdef DEBUG_THREADS
	if (res) {
		log_mutex_error(canlog, "%s line %d (%s): Error waiting on condition mutex '%s'\n",
				   filename, lineno, func, strerror(res));
		DO_THREAD_CRASH;
	} else if (lt) {
		restore_lock_tracking(lt, &lt_orig);
		ast_restore_lock_info(t);
	}
#endif /* DEBUG_THREADS */

	return res;
```

— `main/lock.c:633-646`

`else if`, not a second `if`. A failed `pthread_cond_wait` returns to the
caller with the mutex's `reentrancy` still zero and its per-thread entry still
flagged `suspended`, and relocking the mutex does not clear the flag. In all of
`main/utils.c` the bit has exactly two writers: `:1241`, the `= 1` quoted in
§4.5, and `:1269`, the `= 0` in `ast_restore_lock_info`. `ast_store_lock_info`
is not one of them. When it finds an existing entry for the address it bumps
`times_locked`, `times_lock_attempted`, `last_locked` and the backtrace, then
returns (`:1100-1111`); when it writes a fresh entry it sets eleven named
fields and `suspended` is not among them (`:1130-1142`). So the one routine
that would undo this is the one the `else if` has just skipped. The window matters less than it sounds — this is
`EINVAL`-class failure, a mutex and condition variable that disagree or a
second thread waiting on the same condition with a different mutex, not
something that happens under load — but if you ever do see the message, the
tracking state for that mutex is finished, and a `core show locks` that looks
suspiciously empty is the expected follow-on.

The message itself has a defect worth knowing if you grep for it. The format
string says `condition mutex '%s'` and the argument supplied is
`strerror(res)`. The `mutex_name` that every other message in these two
functions prints is not passed. So the line you will actually see reads
`Error waiting on condition mutex 'Invalid argument'`, naming the errno where
it looks like it is naming the lock. Both copies do this, at `:637` and `:702`.
Match on `Error waiting on condition` and nothing after it.

## 4.8 `timedwait` is the same function, five lines different

`__ast_cond_timedwait` is `:649-712`, the same 64 lines as `__ast_cond_wait`.
A line-by-line diff of the two reports exactly three changed regions, five
lines in total: the three-line signature, the pthread call, and this:

```c
	res = pthread_cond_timedwait(cond, &t->mutex, abstime);

#ifdef DEBUG_THREADS
	if (res && (res != ETIMEDOUT)) {
		log_mutex_error(canlog, "%s line %d (%s): Error waiting on condition mutex '%s'\n",
				   filename, lineno, func, strerror(res));
		DO_THREAD_CRASH;
	} else if (lt) {
		restore_lock_tracking(lt, &lt_orig);
		ast_restore_lock_info(t);
	}
```

— `main/lock.c:698-708`

`if (res && (res != ETIMEDOUT))` where the untimed version has `if (res)`. That
is the whole difference in behaviour, and it is the right difference: a timeout
is a normal return, the mutex has been re-acquired, and the tracking must be
restored. Routing `ETIMEDOUT` down the `else if` is what makes
`ast_cond_timedwait` usable in a polling loop without leaking a suspended entry
on every iteration.

`main/lock.c` contains four of these near-duplicate pairs: `rdlock`/`wrlock`,
`tryrdlock`/`trywrlock`, the rwlock timed pair, and this one. Chapter 03 found
the first two faithful to each other and the third not — it registers the wrong
lock type at `:1099` and logs the wrong lock kind at `:1228`. This pair is the
one where the difference between the two functions is not an oversight but the
point.

## 4.9 The clock nobody sets

`pthread_condattr_t` appears exactly twice in the tree outside `third-party/`:
in the prototype at `lock.h:201` and in the definition at `main/lock.c:538`.
There is no `pthread_condattr_init` anywhere, no
`pthread_condattr_setclock`, and no variable of that type. Of the 86
`ast_cond_init` calls in the tree, 85 pass `NULL` and one passes a literal `0`
(`res/stasis/command.c:69`), which is the same thing.

So every condition variable in Asterisk is created with default attributes,
and its clock is whichever one the C library defaults to. The source does not
assert which that is, and neither will this chapter — it is a property of your
`pthread` implementation, the same caveat chapter 03 §3.3 attached to writer
preference.

What the source does tell you is the other half of the pair. The deadlines
handed to `ast_cond_timedwait` — 40 files call it, at the 93 sites §4.10
counts — are built from the wall clock by one of two routes, and both land on
the same clock. The common one is
`ast_tvnow()`, which is `gettimeofday` (`include/asterisk/time.h:152-159`),
plus an offset, poured into a `struct timespec`. The other is `ast_tsnow`,
which names `CLOCK_REALTIME` outright (`time.h:165-173`) and returns a
`struct timespec` ready to use:

```c
	timeout = ast_tsnow();
	timeout.tv_sec += 5;
```

— `tests/test_dns.c:679-680`

`ast_tsnow` appears at eight sites, all in three DNS test files, and seven of
the eight produce a deadline for an immediately following `ast_cond_timedwait`
— six in exactly the shape above, and `tests/test_dns_recurring.c:213` the same
thing written out field by field. The eighth, `:232`, is the only one that is
not a deadline: it reads the clock again to measure how long the wait took. It
is not a widely used helper, but where it is used it is the deadline source
rather than an aside. If your library's
default condition-variable clock is also the realtime clock, the two halves
agree; if it is not, every timed wait in Asterisk is wrong by the offset
between the two clocks. Either way, deadlines computed this way move when the
system clock is stepped, which is worth remembering before you conclude that a
`timedwait` returned early.

## 4.10 Where condition variables come from

There is no `AST_COND_DEFINE_STATIC` and no `AST_COND_INIT_VALUE`. A condition
variable is always a plain member — `ast_cond_t cond;` in a struct, or a
file-scope `static ast_cond_t` — paired by convention with a mutex and a
predicate, and initialised by an `ast_cond_init` call everywhere but the one
static `PTHREAD_COND_INITIALIZER` in §4.3. Excluding `third-party/` and the standalone
`utils/extconf.c`, but counting `main/lock.c` and `lock.h` themselves, the type
name appears 106 times across the `.c` and `.h` files — 108 with extconf's two,
and 100 if you also drop `main/lock.c`, which the function counts below do.

Counting occurrences in `.c` files, excluding `third-party/` and the standalone
`utils/extconf.c`, the way chapters 02 and 03 counted: `ast_cond_signal` 124,
`ast_cond_timedwait` 93, `ast_cond_init` 86, `ast_cond_destroy` 75,
`ast_cond_wait` 58, `ast_cond_broadcast` 17. Across 68 files — `main/lock.c`
is excluded along with the other two, since its one occurrence of each name is
the wrapper definition rather than a use.

Two of those ratios are worth a sentence. `timedwait` outnumbers `wait` 93 to
58, which is unusual for a C codebase and says something about the house style:
a thread that can wait forever is treated as a thread that can hang forever,
so most waits carry a deadline. And `signal` outnumbers `broadcast` 124 to 17,
which is the conventional preference — wake one worker rather than wake
everyone and have all but one go back to sleep — but it is also the pairing
that goes wrong quietly, because a `signal` to a condition variable with
several waiters wakes an arbitrary one, and if the predicates differ the wrong
thread can consume the wakeup.

Where they cluster is less tidy than the thread-pool story would suggest. By
raw `ast_cond_*` count the leaders are `tests/test_threadpool.c` (27),
`tests/test_taskpool.c` (25), then `tests/test_taskprocessor.c` and
`apps/app_sla.c` level at 18, `channels/chan_iax2.c` (17),
`main/stdtime/localtime.c` (16) and `main/stasis.c` (15). So the top of the
list is the test suite, which exercises condition variables deliberately rather
than using them to get work done, and the heaviest non-test user is a
conference application. `main/taskprocessor.c`, the file you would nominate
first, uses three.

That ordering is a warning about the metric, not a finding about the
architecture. A count measures how many distinct condition variables a file
declares and how many separate places it pokes them, not how much traffic flows
through them: one `ast_cond_signal` inside a hot queue push outweighs twenty
scattered through a test fixture. Each of these files is better read as part of
its own subsystem than as part of this one.

## 4.11 Reading a condition-variable error message

Every message either `wait` function can produce, in file order. All ten sites
are inside `#ifdef DEBUG_THREADS`, so the count for a stock build is zero.
Chapter 03 reached the same count for rwlocks, but by a messier route: two of
its ten were dead, one unreachable, and three involved `DETECT_DEADLOCKS` — two
of them out of `DEBUG_THREADS`' reach entirely, since chapter 02 §2.1 noted
`DETECT_DEADLOCKS` can be defined without it, and the third needing both. Here
the gate is uniform.

| Message | Lines | What it means |
| --- | --- | --- |
| `Error: mutex '%s' is uninitialized.` | `:597`, `:662` | Dead. Needs `AST_MUTEX_INIT_W_CONSTRUCTORS` *and* `CAN_COMPARE_MUTEX_TO_INIT_VALUE`, and chapter 02 §2.3 showed neither is defined on a mainstream Linux build |
| `attempted wait using mutex '%s' without owning it!` | `:608`, `:673` | A real fault, and the most informative message in this chapter. Followed by the holder's line and backtrace. Does *not* return (§4.4) |
| `'%s' was locked here.` | `:610`, `:675` | Follows the previous message, naming the mutex's holder. One space, unlike the rwlock copies at `:916` and `:1020` |
| `attempted wait using an unlocked mutex '%s'` | `:617`, `:682` | Nobody held the mutex. No holder line, because there is no holder to name. Does *not* return (§4.4) |
| `Error waiting on condition mutex '%s'` | `:637`, `:702` | `pthread_cond_wait` itself failed. The `'%s'` is `strerror(res)`, not the mutex name (§4.7). After this, the mutex's tracking is not restored |

Five distinct messages at ten sites, because the two functions are copies. Note
what is not in the table: nothing about the condition variable. No message in
`main/lock.c` names a `cond_name`, in any build, on any path.

## 4.12 Rules

| Rule | Why |
| --- | --- |
| Do not expect this layer to tell you anything about a condition variable | `ast_cond_t` carries no tracking state and `cond_name` is never used. Everything these functions report is about the mutex (§4.1) |
| Treat `ast_cond_signal`, `broadcast`, `init` and `destroy` as the raw pthread calls | They are, in every build, with no `DEBUG_THREADS` arm at all (§4.2) |
| Always hold the mutex when you wait, and expect the wait to proceed if you don't | The ownership check logs and falls through; `DO_THREAD_CRASH` is a no-op without `THREAD_CRASH` (§4.4) |
| After `attempted wait using mutex ... without owning it!`, distrust the tracking | The caller has already zeroed the real holder's `reentrancy` before any `pthread` call, and nothing downstream puts it back correctly (§4.4) |
| Expect a thread blocked in a wait to be missing from `core show locks` | `ast_dump_locks` skips entries flagged `suspended`, which is what the wait sets (§4.5) |
| Read `Error waiting on condition mutex 'X'` as an errno, not a lock name | The format string takes `strerror(res)` where it looks like it takes `mutex_name` (§4.7) |
| Do not assume a `timedwait` deadline survives a clock step | Deadlines are built from `ast_tvnow()` or `ast_tsnow()`, both wall-clock, and no code ever sets a condition attribute to choose a clock (§4.9) |
| Prefer `signal` only when every waiter runs the same predicate | `signal` wakes an arbitrary waiter; the tree uses it 124 times against 17 `broadcast` (§4.10) |
| Add a member to `struct ast_lock_track` and you must edit `restore_lock_tracking` | It is a hand-written copy of the struct definition, enforced by a comment (§4.6) |

## Next

Chapter 05 will cover lock tracking and introspection as a subsystem in its own
right: `ast_lock_track` against `thr_lock_info`, the two caps, how
`ast_store_lock_info` and `ast_mark_lock_acquired` populate the per-thread list,
and `core show locks` read end to end.
