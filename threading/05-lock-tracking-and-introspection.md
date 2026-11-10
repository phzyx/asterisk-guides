# 5. Lock tracking and introspection

Every `file:line` citation in this guide was verified against the `24` branch.
Newer releases and other branches may differ — line numbers drift — so if you
are reading this against a different tree, treat the citations as approximate
and re-grep for the symbol name.

Chapters 02 to 04 were about `main/lock.c`: what the wrapper does around a
pthread call. This chapter is about where the wrapper puts what it learns, and
it is almost entirely a different file. `main/lock.c` calls six functions
whose names begin `ast_store`, `ast_mark`, `ast_remove`, `ast_suspend` and
`ast_restore`; a seventh, `ast_find_lock_info`, is called only by macros in
`lock.h`. Every one of them lives in `main/utils.c`, in a 609-line block
(`:978-1586`) that has nothing else to do.

The block is worth reading as a whole rather than function by function, because
the interesting things about it are relationships between functions: which one
writes a field and which one forgets to clear it, which one assumes an index
another one maintains, and which of the two ways of recording a failed
acquisition you get for any given call. `core show locks` is the output at the
end, and most of the surprises in its output are decided several functions
upstream of the code that prints it.

One framing to carry through: this machinery has two consumers with completely
different needs. `core show locks` wants a readable picture of the whole
process and is allowed to be approximate. `DEADLOCK_AVOIDANCE` (chapter 06)
wants one specific fact about one lock and is not. They read the same data
structure, and only one of them is well served by it.

## 5.1 Two records, two questions

Chapter 01 §1.4 covered `struct ast_lock_track`, which hangs off each lock and
answers *who holds this lock*. This chapter's subject is the other one, which
hangs off each thread and answers *what does this thread hold*:

```c
struct thr_lock_info {
	/*! The thread's ID */
	pthread_t thread_id;
	/*! The thread name which includes where the thread was started */
	const char *thread_name;
	/*! This is the actual container of info for what locks this thread holds */
	struct {
		const char *file;
		const char *func;
		const char *lock_name;
		void *lock_addr;
		int times_locked;
		int times_lock_attempted;
		struct timeval last_locked;
		struct timeval last_unlocked;
		int line_num;
		enum ast_lock_type type;
		/*! This thread is waiting on this lock */
		int pending:2;
		/*! A condition has suspended this lock */
		int suspended:1;
#ifdef HAVE_BKTR
		struct ast_bt *backtrace;
#endif
	} locks[AST_MAX_LOCKS];
	/*! This is the number of locks currently held by this thread.
	 *  The index (num_locks - 1) has the info on the last one in the
	 *  locks member */
	unsigned int num_locks;
	/*! The LWP id (which GDB prints) */
	int lwp;
	/*! Protects the contents of the locks member
	 * Intentionally not ast_mutex_t */
	pthread_mutex_t lock;
	AST_LIST_ENTRY(thr_lock_info) entry;
};
```

— `main/utils.c:996-1031`

The two caps are the first thing to notice, because they are two orders of
magnitude apart and both are compile-time constants. `AST_MAX_REENTRANCY` is 10
(`include/asterisk/lock.h:108`): how many times one thread may recursively take
one lock before the tracking stops recording. `AST_MAX_LOCKS` is 64
(`main/utils.c:982`): how many *distinct* locks one thread may hold before the
tracking stops recording. Overflowing the first logs `really deep reentrancy!`
and carries on with the lock held (chapter 02 §2.5). Overflowing the second is
§5.3.

The `locks` member is an anonymous struct, not a named type, which is why
nothing outside this file can be handed one. Thirteen members, of which two are
bitfields:

```
int pending:2;
int suspended:1;
```

`pending` is declared `int pending:2` and the code uses three values: `1` set by
`ast_store_lock_info` meaning the
thread is waiting, `0` set by `ast_mark_lock_acquired` meaning it has it, and
`-1` set by `ast_mark_lock_failed` meaning it asked and was refused. The five
reads are two sign tests (`:1382`, `:1383`), two equality tests against `-1`
(`:1058`, `:1121`), and one that is both a zero test and an equality test
against `-1` (`:1403`). Nothing anywhere compares against `1`.

The two-bit width only works if the compiler treats `int:2` as *signed*, which
gives the range `-2` to `1` and lets all three values round-trip. For bit-fields
the signedness of plain `int` is implementation-defined; GCC and Clang choose
signed, and `-funsigned-bitfields` makes them not. That is the kind of question
§4.9 and chapter 03 §3.3 hand back to the reader. Under an unsigned
interpretation the `== -1` tests would never match and both failure mechanisms
of §5.5 would stop working.

`suspended` is the one chapter 04 §4.5 traced from the other end. It is set and
cleared by nothing except a condition-variable wait, and it has exactly two
readers: `:1398`, which appends the literal `" - suspended"` to a stanza, and
`:1507`, which skips the entry entirely. Those two cannot both fire on the same
entry, which §5.6 comes back to.

A few more fields are worth flagging now because they come back in §5.2:
`thread_id` and `thread_name` are not written by any of the six functions
`main/lock.c` calls. Nor is `lwp`. Nor is `lock`, the `pthread_mutex_t` that
protects everything else in the struct — and whose comment, `Intentionally not
ast_mutex_t`, is the first hint that this file has to stand outside the
subsystem it implements. Tracking a mutex requires taking a mutex, so the
mutexes used by the tracking cannot themselves be tracked; `main/utils.c`
`#undef`s five pthread names at `:985-989` to get the raw ones back.

## 5.2 The registry only contains threads Asterisk started itself

`ast_dump_locks` does not enumerate threads. It walks a list:

```c
static AST_LIST_HEAD_NOLOCK_STATIC(lock_infos, thr_lock_info);
```

— `main/utils.c:1040`

There is exactly one `AST_LIST_INSERT_TAIL` against that list in the entire
tree, and it is here:

```c
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
```

— `main/utils.c:1611-1629`

That is `dummy_start`, the wrapper every `ast_pthread_create*` macro routes
through. So the list contains precisely the threads Asterisk created through its
own macros — 129 call sites through the four macros, plus `res/res_corosync.c:93`
going directly to `ast_pthread_create_stack`, per the overview — and nothing
else.

The thread that does not go through `dummy_start` is the one running `main()`.
It takes locks from the first module load onward, and under `--no-fork` it is
also the console thread (`consolethread = pthread_self()`,
`main/asterisk.c:4372`). It is not in `lock_infos`, so **no lock held by the
main thread appears in `core show locks`**. The same applies to any
thread a linked library creates and then calls Asterisk code from.

What makes this easy to miss is that such a thread still gets a working
`thr_lock_info`. `ast_store_lock_info` obtains its record the same way
`dummy_start` does, through `ast_threadstorage_get`, and that function allocates
on demand:

```c
	pthread_once(&ts->once, ts->key_init);
	if (!(buf = pthread_getspecific(ts->key))) {
		if (!(buf = ast_calloc(1, init_size))) {
			return NULL;
		}
```

— `include/asterisk/threadstorage.h:185-189`

The storage object is declared `AST_THREADSTORAGE_CUSTOM(thread_lock_info, NULL,
lock_info_destroy)` (`main/utils.c:1085`) — a destructor, but `NULL` for the
initialiser. So the lazily created record is `ast_calloc`'d and then used as-is.
Three consequences, in increasing order of how much they matter:

`thread_id` is 0, `thread_name` is `NULL` and `lwp` is 0. For any thread that has
a visible lock `ast_dump_locks` prints a header regardless of `lwp` (`:1511`),
using `:1512` only to choose between two formats, so a zero `lwp` would take the
one that prints `LWP:0` alongside a `NULL` thread name. Which is academic,
because:

The record is never linked into `lock_infos`, so neither `ast_dump_locks` nor
`ast_log_show_lock` can reach it. The locks are tracked correctly and reported
nowhere.

And `lock_info->lock` is never initialised. `dummy_start` builds it with
`AST_MUTEX_KIND` at `:1622-1624`; the lazy path performs no initialisation at
all, leaving the `ast_calloc`'d zero bytes that every subsequent
`pthread_mutex_lock(&lock_info->lock)` then operates on. What the source
establishes is that the two paths disagree: one builds the mutex with an
explicit kind, the other never calls `pthread_mutex_init`. Whether a zeroed
`pthread_mutex_t` is a usable mutex is a property of your pthread
implementation, not of this tree, and this guide hands that back to the reader
the same way §4.9 did with the condition-variable clock and chapter 03 §3.3 with
writer preference. POSIX requires either `pthread_mutex_init` or
`PTHREAD_MUTEX_INITIALIZER`, and the lazy path uses neither.

The honest summary is that the per-thread record is correct for every thread and
*reachable* only for threads Asterisk started. Which of the two consumers that
hurts depends on which one you are: `ast_find_lock_info` (§5.8) reads the
calling thread's own record straight out of thread-local storage and never
touches `lock_infos`, so `DEADLOCK_AVOIDANCE` works in the main thread exactly
as it does anywhere else. It is only the two functions that print that are
blind.

`lock_info_destroy` still runs for such a thread, because the threadstorage
destructor is registered regardless, and its first act is to remove the record
from a list it was never in. `AST_LIST_REMOVE` tolerates that: with an empty or
non-containing list the forward walk from `head->first` ends on a `NULL`
predecessor and the macro takes its `__elm = NULL` branch
(`include/asterisk/linkedlists.h:877-879`) without touching the head. So the one thing that would have turned this into a
crash is handled.

## 5.3 Storing a lock has two paths, and writes eleven of thirteen fields

`ast_store_lock_info` (`:1088-1147`) is called eight times from `main/lock.c`,
once at the top of each of the eight acquisition entry points, and always inside
`if (lt)` — the gate chapter 02 §2.5 introduced. That gate is load-bearing in a
way §5.9 comes back to: a lock with no tracking record never enters any thread's
list at all.

The function begins with a forward linear search for an existing entry with the
same address. If it finds one, the lock is being re-entered, and four things are
bumped before it returns: `times_locked`, `times_lock_attempted`, `last_locked`
and the backtrace (`:1102-1106`). Note what is *not* bumped: `pending` is left
alone, because the thread already holds the lock and is not waiting for it.

If the search falls through, this is a new lock for this thread, and the first
thing checked is the cap:

```c
	if (lock_info->num_locks == AST_MAX_LOCKS) {
		/* Can't use ast_log here, because it will cause infinite recursion */
		fprintf(stderr, "XXX ERROR XXX A thread holds more locks than '%d'."
			"  Increase AST_MAX_LOCKS!\n", AST_MAX_LOCKS);
		pthread_mutex_unlock(&lock_info->lock);
		return;
	}
```

— `main/utils.c:1113-1119`

The `fprintf` and its comment are the most quietly informative three lines in
the file. `ast_log` would route into the logger, which takes locks, which calls
`ast_store_lock_info`, which is already at its cap and would `fprintf` again —
so the one diagnostic this subsystem cannot emit through the logging subsystem
is the one about itself. Chapter 02 §2.2 showed the same problem solved a
different way for `log_mutex_error`, which special-cases `logger.c` by filename.
Here there is no filename to test, so the message goes to `stderr` and will not
appear in a log file, in `core show locks`, or anywhere a remote console can see
it. If a thread in your build is silently failing to record locks past the 64th,
this is the only place it says so.

Then a sweep, and then the fields:

```c
	if (i && lock_info->locks[i - 1].pending == -1) {
		/* The last lock on the list was one that this thread tried to lock but
		 * failed at doing so.  It has now moved on to something else, so remove
		 * the old lock from the list. */
		i--;
		lock_info->num_locks--;
		memset(&lock_info->locks[i], 0, sizeof(lock_info->locks[0]));
	}

	lock_info->locks[i].file = filename;
	lock_info->locks[i].line_num = line_num;
	lock_info->locks[i].func = func;
	lock_info->locks[i].lock_name = lock_name;
	lock_info->locks[i].lock_addr = lock_addr;
	lock_info->locks[i].times_locked = 1;
	lock_info->locks[i].times_lock_attempted = 1;
	lock_info->locks[i].last_locked = ast_tvnow();
	lock_info->locks[i].type = type;
	lock_info->locks[i].pending = 1;
#ifdef HAVE_BKTR
	lock_info->locks[i].backtrace = bt;
#endif
	lock_info->num_locks++;
```

— `main/utils.c:1121-1143`

The sweep exists because of §5.5: a failed trylock leaves a tombstone entry
rather than removing itself, and this is where the tombstone is collected — on
the next store, not when the thread moves on. The `memset` is the only place in
the file that zeroes a slot.

Count the assignments. Ten fields, plus `backtrace` under `HAVE_BKTR`, which is
defined on a mainstream Linux build (`include/asterisk/autoconfig.h:126`), so
eleven in practice. The struct has thirteen members. The two that are not
written are `last_unlocked` — harmless, since §5.9 shows it is only meaningful
once something has unlocked — and `suspended`, which is §5.6.

## 5.4 `acquired` and `failed` both assume the lock is the last one

The two functions that run after the pthread call are near-identical and both
rest on the comment at `:1021-1023`: *the index (num_locks - 1) has the info on
the last one in the locks member*.

```c

	pthread_mutex_lock(&lock_info->lock);
	if (lock_info->locks[lock_info->num_locks - 1].lock_addr == lock_addr) {
		lock_info->locks[lock_info->num_locks - 1].pending = 0;
	}
	pthread_mutex_unlock(&lock_info->lock);
```

— `main/utils.c:1156-1161`

```c
	pthread_mutex_lock(&lock_info->lock);
	if (lock_info->locks[lock_info->num_locks - 1].lock_addr == lock_addr) {
		lock_info->locks[lock_info->num_locks - 1].pending = -1;
		lock_info->locks[lock_info->num_locks - 1].times_locked--;
		lock_info->locks[lock_info->num_locks - 1].last_unlocked = ast_tvnow();
	}
```

— `main/utils.c:1173-1178`

The guard is the saving grace. Both functions check that the last entry is
actually the lock they were told about, so when the assumption is wrong they do
nothing rather than corrupting a different lock's record. Two details follow
from that.

First, `num_locks` is `unsigned int` (`:1024`), and both functions index
`locks[num_locks - 1]` before testing anything. If `num_locks` were ever 0 the
subscript would be `locks[4294967295]`, far outside a 64-element array, and the
read would happen before the guard could reject it. In practice it is not 0:
`main/lock.c` only calls these immediately after `ast_store_lock_info`, which
leaves `num_locks` at least 1 on every path that reaches the bottom, and no
other thread can modify this thread's thread-local list.

But "in practice" is doing work there. `ast_store_lock_info` has two returns
before it writes anything. The cap return (`:1113-1119`) leaves `num_locks` at
64, which is harmless. The other is `:1095-1096`, where `ast_threadstorage_get`
fails its `ast_calloc` and the function returns having touched nothing.
`ast_mark_lock_acquired` and `ast_mark_lock_failed` then call
`ast_threadstorage_get` *again* (`:1154`, `:1170`); if that second call succeeds
it hands back a freshly zeroed record with `num_locks == 0`, and `:1158` /
`:1174` index `locks[4294967295]`. The bound rests on the assumption that the
two allocations either both fail or both succeed, which nothing guarantees and
nothing checks. The window is tiny and only opens under allocation failure,
which is the condition under which you would most want the diagnostics to
work.

Second, the assumption is genuinely violated on one path. When
`ast_store_lock_info` took its existing-entry branch — a re-entrant acquisition
— the entry it touched is at whatever index the lock already occupied, which is
the last one only if nothing else was taken after it. If the acquisition then
succeeds, `ast_mark_lock_acquired` either sets a `pending` that was already 0 or
no-ops, and nothing is lost. If it *fails*, the two recording styles of §5.5
diverge: the blocking paths call `ast_remove_lock_info`, which searches by
address and so correctly decrements the inflated `times_locked`; the trylock
paths call `ast_mark_lock_failed`, which does not search. If the re-entered lock
is the last entry, that marks a lock the thread still holds as `pending = -1`,
which §5.5 and §5.9 show has consequences beyond the display; if it is not the
last entry, the `times_locked` bump from the store is never undone and the entry
outlives the real unlock as a phantom.

How reachable that is depends on whether a thread can fail to acquire a lock it
already holds. For a recursive mutex it cannot — that is what recursive means,
and chapter 01 established every Asterisk mutex is one. For a read-write lock
it is a question about the library: `pthread_rwlock_tryrdlock` is permitted to
fail for a thread that already holds the lock for reading, and chapter 03 §3.3
declined to assert what this library does about writer preference for the same
reason. So the path exists, the guard keeps it from corrupting anything else,
and whether any build reaches it is not something the Asterisk source can tell
you.

## 5.5 There are two ways to record a failure, and they are assigned oddly

Count the calls in `main/lock.c`. Eight `ast_store_lock_info`, eight
`ast_mark_lock_acquired`, three `ast_mark_lock_failed`, seven
`ast_remove_lock_info`. Five of those seven removals are not unlocks — only
`:506` (mutex unlock) and `:849` (rwlock unlock) are. The other five are failure
handling, and they line up like this:

| Entry point | Failure handled by | At |
| --- | --- | --- |
| `__ast_pthread_mutex_lock` | `ast_remove_lock_info` | `:386` |
| `__ast_pthread_mutex_trylock` | `ast_mark_lock_failed` | `:446` |
| `__ast_rwlock_rdlock` | `ast_remove_lock_info` | `:957` |
| `__ast_rwlock_wrlock` | `ast_remove_lock_info` | `:1061` |
| `__ast_rwlock_timedrdlock` | `ast_remove_lock_info` | `:1143` |
| `__ast_rwlock_timedwrlock` | `ast_remove_lock_info` | `:1225` |
| `__ast_rwlock_tryrdlock` | `ast_mark_lock_failed` | `:1280` |
| `__ast_rwlock_trywrlock` | `ast_mark_lock_failed` | `:1330` |

Three trylocks mark; five blocking acquisitions erase. The split is deliberate —
the header says so, naming the caller rather than describing the effect:

```c
/*!
 * \brief Mark the last lock as failed (trylock)
 */
void ast_mark_lock_failed(void *lock_addr);
```

— `include/asterisk/lock.h:283-286`

The practical consequence is a reporting asymmetry, and it is the opposite of
what the two kinds of failure deserve. A trylock that returns `EBUSY` leaves a
visible record which `core show locks` renders as `Tried and failed to get`, and
which survives until the thread's next new lock sweeps it away (§5.3). A
`timedwrlock` that reaches its deadline — the closest thing in the API to a
trylock, and a far more interesting event, because it means a lock was contended
for a measurable length of time — erases its own entry and leaves nothing. The
16 `ast_mutex_trylock` sites in the tree get a tombstone; the timed rwlock
acquisitions get silence.

There is one more thing the erasing paths quietly get right. Chapter 02 §2.5
showed `__ast_pthread_mutex_lock`'s `else if (lt)` arm recovering a backtrace
from `lt->backtrace[lt->reentrancy-1]` before calling `ast_remove_lock_info`,
which looks like pointless work on a path where the entry is about to be deleted.
It is not pointless: `ast_remove_lock_info` has four exits, and the
`times_locked > 1` exit at `:1298-1306` keeps the entry and stores that
backtrace. A re-entrant acquisition that fails goes down that path.

## 5.6 Removal leaves the slot dirty

`ast_remove_lock_info` (`:1276-1318`) searches backwards by address, returns
silently on a miss (§4.5 traced the same shape), decrements and returns if
`times_locked > 1`, and otherwise closes the gap:

```c
	if (i < lock_info->num_locks - 1) {
		/* Not the last one ... *should* be rare! */
		memmove(&lock_info->locks[i], &lock_info->locks[i + 1],
			(lock_info->num_locks - (i + 1)) * sizeof(lock_info->locks[0]));
	}

	lock_info->num_locks--;
```

— `main/utils.c:1308-1314`

Nothing clears `locks[num_locks]` after the decrement. The slot that just fell
outside the live range keeps every byte it had, and in the common case — the
removed entry was the last one, so the `memmove` does not even run — those bytes
are the removed lock's own record, intact.

That would be harmless if the next `ast_store_lock_info` overwrote the slot
completely. §5.3 showed it does not: it writes eleven of thirteen fields, and one
of the two it skips is `suspended`.

So the question is whether a slot can go out of the live range with `suspended`
set. The bit is set only by `ast_suspend_lock_info` and cleared only by
`ast_restore_lock_info`, and chapter 04 §4.7 is precisely about the path where
the second one does not run:

```c
	if (res) {
		log_mutex_error(canlog, "%s line %d (%s): Error waiting on condition mutex '%s'\n",
				   filename, lineno, func, strerror(res));
		DO_THREAD_CRASH;
	} else if (lt) {
		restore_lock_tracking(lt, &lt_orig);
		ast_restore_lock_info(t);
	}
```

— `main/lock.c:636-643`

If `pthread_cond_wait` returns an error, the mutex's `suspended` bit stays set.
The thread continues — `DO_THREAD_CRASH` is a no-op without `THREAD_CRASH`
(chapter 04 §4.4) — and eventually unlocks the mutex, which runs the code above
and leaves a slot outside the live range carrying `suspended = 1`. The next new
lock this thread takes lands on that slot, inherits the bit, and is then skipped
before it can ever be printed:

```c
		for (i = 0; str && i < lock_info->num_locks; i++) {
			/* Don't show suspended locks */
			if (lock_info->locks[i].suspended) {
				continue;
			}
```

— `main/utils.c:1505-1509`

An ordinary, correctly held, fully tracked lock, invisible in `core show locks`.
Note also that the `:1398` reader is unreachable in that output: `:1507` has
already `continue`d past the entry, so nothing ever prints the `" - suspended"`
suffix for it. The bit hides the lock and announces nothing.

How long the slot stays poisoned is worth being careful about, because the
obvious answer is wrong in both directions.

The `memmove` above does not help. Its destination indices are `i` through
`num_locks - 2`, so it never writes a slot outside the live range and can never
reach the poisoned one there. And when a poisoned slot *is* inside the live
range and something below it is released, the shift copies that entry — bit and
all — down one index. The bit is relocated, not cleared, and the lock it hides
stays hidden at its new index.

The one thing that really clears it is the `memset` at `:1127`. Whatever slot
`ast_store_lock_info` writes next is, at that moment, the trailing live entry,
which is exactly the slot `ast_mark_lock_failed` writes (`locks[num_locks - 1]`).
So if the lock that inherits the poisoned slot arrives through a trylock entry
point and that trylock fails, the slot becomes a `pending == -1` tombstone, and
the next store sweeps and zeroes it. That is a narrow escape hatch, not a
routine one: an ordinary successful acquisition followed by an ordinary release
puts the slot back out of range still poisoned.

So the right claim is that the slot stays poisoned until a failed trylock lands
on it, and every lock that occupies it meanwhile is hidden.

Two honest qualifications. The chain needs the suspended entry to be the
trailing one at unlock time — otherwise the `memmove` overwrites it with the
entry above and the bit is destroyed rather than orphaned. And it needs a
`pthread_cond_wait` that fails, which makes this a second-order defect sitting
on top of a first-order one; chapter 04 §4.4 declined to assert how this library
responds to a wait on an unowned recursive mutex, and that is the most likely way
to get there. Neither changes the shape of the finding:
`ast_store_lock_info` writes every field that identifies a lock and not the one
field that decides whether anyone will see it.

## 5.7 Suspend and restore are the same function twice

`ast_suspend_lock_info` is `:1218-1245` and `ast_restore_lock_info` is
`:1247-1273` — 28 lines and 27. Same locals, same `ast_threadstorage_get`, same
backwards search, same `/* Lock not found :( */` comment on the same early
return. They differ in three regions and nothing else, as chapter 04 §4.5 also
noted: the name in the signature, the brace style on the early-return guard
(`:1224-1226` braced against `:1253-1254` braceless), and `suspended = 1` at
`:1241` against `suspended = 0` at `:1269`.

Four call sites, all in `main/lock.c`, all in the two condition-wait functions:
`:629` and `:642` in `__ast_cond_wait`, `:694` and `:707` in
`__ast_cond_timedwait`. Chapter 04 covered what they mean from the waiting
thread's side, including the case where the search misses because the caller
does not hold the lock it is waiting on. Nothing else in the tree sets or clears
`suspended`.

That is the whole of the suspension mechanism: 55 lines of near-duplicated code,
one bit, four call sites, and two readers (`:1398`, `:1507`) of which only the
second can fire in `core show locks`. The `:1398` read, which appends the
`" - suspended"` suffix, is reachable only through `ast_log_show_lock`, and
§5.12 shows that has no callers.

## 5.8 `ast_find_lock_info` is the only part production code reads

Six of the nine functions in this family are write-only from the rest of the
tree's point of view: they populate a structure that two of the remaining
three — `ast_log_show_lock` and `ast_dump_locks` — consume. The ninth is
different.

`ast_find_lock_info` (`:1183-1216`) does the same backwards search and then
copies four facts out — file, line, function, and the lock's source text — into
caller-supplied buffers, returning 0 on a hit and -1 on a miss. It has four
callers, and all four are macros in `lock.h`:

```c
		int __res = ast_find_lock_info(lock, __filename, sizeof(__filename), &__lineno, __func, sizeof(__func), __mutex_name, sizeof(__mutex_name)); \
```

— `include/asterisk/lock.h:387`

`AO2_DEADLOCK_AVOIDANCE` (`:343`), `CHANNEL_DEADLOCK_AVOIDANCE` (`:365`),
`DEADLOCK_AVOIDANCE` (`:387`) and `DLA_UNLOCK` (`:421`). Chapter 06 covers what
they do with the answer; the relevant part here is *why they ask*. Deadlock
avoidance drops a lock and retakes it, and retaking it through the normal macro
would stamp the re-lock's own file and line into the tracking, losing the place
the lock was originally taken. So these macros read the old call site out of the
per-thread list before unlocking and pass it back to
`__ast_pthread_mutex_lock` explicitly, which is why the lock's record survives a
round of deadlock avoidance pointing at the code that actually wanted it.

Three things about that are worth keeping. The buffers are fixed at 80 bytes in
all four macros and `ast_copy_string` truncates, so a long enough function name
or lock expression is silently cut — in the record, not in the lock. The miss
return is -1 and every macro branches on it, with comments ranging from
`/* Could happen if the ao2 object does not have a mutex. */` (`:346`) to
`/* Shouldn't ever happen, but just in case... */` (`:368`, `:390`); §5.13 shows
one build where it happens every time. And this is the function that makes §5.2's
distinction matter: it reads thread-local storage, not `lock_infos`, so it works
for every thread, including the ones the printers cannot see.

## 5.9 Reading `core show locks`

`ast_dump_locks` (`:1473-1549`) takes `lock_infos_lock`, walks the list, takes
each thread's own `lock` mutex, and emits a header plus, per unsuspended entry, a
two-line stanza and — under `HAVE_BKTR`, which §5.3 showed is on for a mainstream
Linux build — a backtrace. `handle_show_locks` (`:1552-1580`) is a thin CLI wrapper
around it, registered at `:2655-2657` and notable only for calling
`ast_cli_allow_at_shutdown(e)` — this is a command you are expected to want when
nothing else is working.

The output begins with its own legend:

```c
	               "=== <pending> <lock#> (<file>): <lock type> <line num> <function> <lock name> <lock addr>\n"
	               "=== <locked at>, <failed at>, <held for> (attempts, times locked)\n"
```

— `main/utils.c:1493-1494`

Compare the second line against what the body actually prints:

```c
				   lock_info->locks[i].times_locked,
				   lock_info->locks[i].times_lock_attempted,
```

— `main/utils.c:1396-1397`

The legend says `(attempts, times locked)`. The arguments are `times_locked`
then `times_lock_attempted`. The two numbers are printed in the opposite order
from the one the header announces. For a lock taken once and never contended
they are both 1 and nobody notices; for a mutex that has been re-entered five
times and attempted seven, `(5, 7)` reads under the published legend as five
attempts and seven acquisitions, which is not a state that can exist.

The rest of the legend is accurate but under-describes two fields. `<pending>`
is a prefix string rather than a column — `Waiting for ` when `pending > 0`,
`Tried and failed to get ` when `pending < 0`, and nothing at all when the lock
is held (`:1382-1383`), which is why a held lock's stanza begins `Lock #` with no
adjective. And `<failed at>` is `last_unlocked`, which `ast_mark_lock_failed`
stamps on a failure (`:1177`) but `ast_remove_lock_info` also stamps on an
ordinary re-entrant unlock (`:1300`). For a lock that has been recursively
locked and unlocked without anything failing, the column labelled `failed at`
holds the time of the last successful unlock.

Below each stanza, mutexes get their reentrancy stack expanded — but only
mutexes, and only contended ones:

```c
	if (!lock_info->locks[i].pending || lock_info->locks[i].pending == -1)
		return;

	/* We only have further details for mutexes right now */
	if (lock_info->locks[i].type != AST_MUTEX)
		return;

	lock = lock_info->locks[i].lock_addr;
	lt = lock->track;
	ast_reentrancy_lock(lt);
	for (j = 0; *str && j < lt->reentrancy; j++) {
```

— `main/utils.c:1403-1413`

Two gates and then an unguarded dereference. `lt = lock->track` with no `NULL`
check, handed straight to `ast_reentrancy_lock`, which chapter 04 §4.5 showed
dereferences without checking either (`include/asterisk/lock.h:452-462`). What
keeps it safe is §5.3's observation about the `if (lt)` gate: all eight
`ast_store_lock_info` calls in `main/lock.c` sit inside it, so a lock whose
`track` is `NULL` never enters any thread's list and can never be reached from
here. There are two ways to be such a lock: declared
`AST_MUTEX_DEFINE_STATIC_NOTRACKING`, of which the tree has exactly one,
`reglock` in `main/astmm.c:180`; or one whose tracking allocation failed, which
returns `NULL` in a build without `DO_CRASH` or `THREAD_CRASH`
(`main/lock.c:110-117`). The gate covers both, which is why the safety here does
not depend on the census.

The invariant is real and nothing states it. The only comment at `:1406`
explains the `AST_MUTEX` gate, which is the gate that is not load-bearing.

That one `NOTRACKING` declaration is also the tidiest illustration of why this
whole file has to be careful. Its comment is `Tracking this mutex will cause
infinite recursion, as the mutex tracking code allocates memory` — the memory
allocator's own lock cannot be tracked, because tracking allocates. The same
constraint produced the `fprintf` in §5.3 and the raw `pthread_mutex_t` in §5.1.

## 5.10 Three of the four times are not durations

Every timestamp in the output goes through one formatter:

```c
void ast_format_duration_hh_mm_ss(int duration, char *buf, size_t length)
{
	int durh, durm, durs;
	durh = duration / 3600;
	durm = (duration % 3600) / 60;
	durs = duration % 60;
	snprintf(buf, length, "%02d:%02d:%02d", durh, durm, durs);
}
```

— `main/utils.c:2331-2338`

It takes a count of seconds and splits it into hours, minutes and seconds. It
has seven callers in the tree. Five pass a difference between two times —
`main/stasis.c:2674`, `main/bridge.c:5272` and `:5338`, and
`channels/pjsip/cli_commands.c:292` and `:404`, all of the form
`ast_tvnow().tv_sec - something->creationtime.tv_sec`. Those are durations and
the function is the right one.

The other two callers are in this chapter's code, and between them they make
four calls:

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

One of those four is a duration. `held_for` is an `ast_tvsub` of now and
`last_locked`, so `held_time` is correct. The other two are `last_locked` and
`last_unlocked` straight out of the struct, and §5.3 showed where they come from:
`ast_tvnow()`, which is `gettimeofday` (`include/asterisk/time.h:153-158`) — an
absolute epoch time. Running one through a duration formatter divides it by 3600
and prints the result as an hour count. A moment in late 2026 comes out as
roughly `498408:00:00`. The fourth call is `ast_dump_locks`'s own header at
`:1485`, which formats `now.tv_sec` under the label `Currently Held Locks at
Time:` — a label that makes the intent unambiguous and the function choice
unambiguously wrong.

So in a stanza like

```
=== ---> Lock #0 (channel.c): MUTEX 2451 ast_channel_lock &chan->lock 0x7f...
===      498408:12:34.567890, 00:00:00.000000, 00:00:03.141592 (1, 1)
```

the first field is not a time of day, the second is zero because nothing has
unlocked yet, and only the third means what the legend says. The two useless
ones are the two you would reach for first when correlating against a log.

A second, smaller problem in the same signature: the parameter is `int` and
`time_t` is 64-bit, so the conversion narrows. For the five correct callers a
duration that overflows `int` is not a real concern. For the three in this file,
feeding an epoch means the value crosses `INT_MAX` on 19 January 2038, after
which the three fields become implementation-defined rather than merely
unhelpful.

## 5.11 Seven checks that cannot fire

`ast_dump_locks` tests `str` for `NULL` five times after creating it — `:1496`,
the loop condition at `:1505`, `:1525`, `:1532` and `:1538` — and returns or
breaks each time. `ast_log_show_lock` has the same loop condition at `:1456`,
and `append_lock_information` tests `*str` at `:1413`. Seven tests, all
downstream of `ast_str_append`, all on the theory that a failed append nulls the
pointer.

It does not. The only reallocation path is `_ast_str_make_space`, and on failure
it puts the old pointer back:

```c
	if ((*buf)->__AST_STR_TS == DS_ALLOCA || (*buf)->__AST_STR_TS == DS_STATIC)
		return -1;	/* cannot extend */
	*buf = (struct ast_str *)__ast_realloc(*buf, new_len + sizeof(struct ast_str), file, lineno, function);
	if (*buf == NULL) {
		*buf = old_buf;
		return -1;
	}
```

— `include/asterisk/strings.h:812-818`

`__ast_str_helper` propagates the failure as a return value
(`AST_DYNSTR_BUILD_FAILED`, `main/strings.c:113-114`) and then unconditionally
re-terminates the buffer it still has (`:118-124`). `*buf` is never left `NULL`, so after
`ast_str_create` succeeds once, `str` stays non-`NULL` for the rest of all three
of the functions that check it.

This is dead code rather than a bug, and worth knowing for one reason: it looks
like the function degrades gracefully under memory pressure and it does not. A
`core show locks` that fails to grow its 4 KB buffer mid-walk will keep
appending into a truncated string, silently, and hand back whatever fit. The
checks that would have caught it are testing the wrong thing.

## 5.12 `ast_log_show_lock` has no callers

`ast_log_show_lock` (`:1440-1470`) is called from nowhere in the tree, and that
is by design. It is declared at `lock.h:308` and carries two comments, both of
which tell you to insert a call by hand before a lock you are suspicious of. The
10-line one at `lock.h:298-307` is the summary; the 19-line one above the
definition, at `main/utils.c:1421-1439`, is the longer version and the one that
adds the gdb recipe for breaking on the `ast_log` inside it. A tool left in the
drawer, not dead code.

Two things to know before you reach for it. Its loop comment says `ONLY show
info about this particular lock, if it's acquired...` and the header comment
says it `doesn't do anything if the lock is not acquired`; both are wrong, since
`append_lock_information` is called on an address match alone and will happily
describe a lock the thread is still waiting for. And it builds one `ast_str`
outside the thread walk, appends into it per matching thread, and logs the whole
buffer each time without resetting it:

```c
	AST_LIST_TRAVERSE(&lock_infos, lock_info, entry) {
		int i;
		pthread_mutex_lock(&lock_info->lock);
		for (i = 0; str && i < lock_info->num_locks; i++) {
			/* ONLY show info about this particular lock, if
			   it's acquired... */
			if (lock_info->locks[i].lock_addr == this_lock_addr) {
				append_lock_information(&str, lock_info, i);
				ast_log(LOG_NOTICE, "%s", ast_str_buffer(str));
				break;
			}
		}
		pthread_mutex_unlock(&lock_info->lock);
	}
```

— `main/utils.c:1453-1466`

For a mutex, which has one holder, that is one match and one tidy log line. For
a read-write lock held by three readers it is three log lines, the second
repeating the first and the third repeating both. `ast_str_reset(str)` after the
`ast_log` would fix it.

## 5.13 What each build flag actually buys

Four flags decide how much of this chapter exists in a given binary. Only the
first is listed as `core` support level in `menuselect-tree` (`:2046`); the
others are `extended` (`:2064`, `:2214`, `:2223`).

| Flag | Effect on this subsystem |
| --- | --- |
| `DEBUG_THREADS` | Everything in this chapter. `main/utils.c:978-1586` is one `#ifdef`, and every function `main/lock.c` calls into this file exists only inside it |
| `LOW_MEMORY` | Guts it from the inside, see below |
| `DETECT_DEADLOCKS` | Nothing here at all, see below |
| `THREAD_CRASH` | Nothing here directly; it is what makes `DO_THREAD_CRASH` a real `*((int *)(0)) = 1` rather than `do { } while (0)` (`include/asterisk/lock.h:253-257`), which is what lets §5.6's chain start |

`LOW_MEMORY` is the interesting one, because it does not simply remove the
feature. The outer `#if !defined(LOW_MEMORY)` at `:980` closes at `:1086`,
taking out `AST_MAX_LOCKS`, `struct thr_lock_info`, `lock_infos`,
`lock_info_destroy` and the threadstorage object. Each public function then has
its *own* `LOW_MEMORY` guard around its body, so the symbols survive as stubs
and `main/lock.c` still links: the six mutators and `ast_log_show_lock` become
empty functions, `ast_find_lock_info` returns -1 (`:1213-1215`) and
`ast_dump_locks` returns `NULL` (`:1546-1548`). `locktype2str`,
`append_backtrace_information`, `append_lock_information`, `handle_show_locks`
and the CLI entry are compiled out entirely.

The consequence worth noticing is §5.8's: with `DEBUG_THREADS` *and*
`LOW_MEMORY`, `ast_find_lock_info` returns -1 unconditionally, so all four
deadlock-avoidance macros permanently take the branch their comments describe as
something that should never happen. For `DEADLOCK_AVOIDANCE` that branch is a
plain `ast_mutex_lock(lock)` instead of `__ast_pthread_mutex_lock` with the
original call site, which is behaviourally fine and loses the provenance the
macro existed to preserve.

`DETECT_DEADLOCKS` contributes nothing to this chapter, which is worth stating
because its name suggests otherwise. Its three blocks in `main/lock.c`
(`:288`,
`:899`, `:1003`) are guarded on `DETECT_DEADLOCKS` alone and are not nested
inside `DEBUG_THREADS`; chapter 02 §2.1 noted it can be defined without it, and
`build_tools/cflags-devmode.xml:11` offers it as an independent switch. In that
configuration `t->track` and `t->flags` still exist — the `#elif` at
`include/asterisk/lock.h:152` keeps them — and the `track`/`flags`
initialisation at `main/lock.c:163-167` still runs, which is what the logging
decision is derived from: `canlog` is a local recomputed per entry point as
`t->flags.tracking && strcmp(filename, "logger.c")` (`:261`), so without that
initialisation the deadlock message would go to `stderr` rather than the logger.
Note that `ast_get_reentrancy` itself is inside `#ifdef DEBUG_THREADS`
(`main/lock.c:84-143`) and so does not exist in this build at all. You get the
`Deadlock? waited %d sec for
mutex '%s'?` log and the `DeadlockStart` AMI event. You do not get
`ast_store_lock_info`, which is only called under `DEBUG_THREADS`, and you do
not get `core show locks`, because `:2655-2657` registers the CLI command under
`DEBUG_THREADS` too. A `DETECT_DEADLOCKS`-only build tells you a deadlock is
happening and gives you no way to ask who is in it.

## 5.14 Messages

Three messages originate in this file, plus two backtrace placeholders
(`No backtrace to print` at `:1342`, `Couldn't retrieve backtrace symbols` at
`:1358`). One more that you will see in `core show locks` output — the last row
below — comes from somewhere else. Most of them you will probably never see.

| Message | Site | Notes |
| --- | --- | --- |
| `Thread '%s' still has a lock! - '%s' (%p) from '%s' in %s:%d!` | `:1064` | Emitted per leftover entry when a thread exits. `lock_info_destroy` is the threadstorage destructor, so unlike the two printers it also reaches threads that were never linked into `lock_infos` — a library thread that took a tracked lock and then exited. Not the main thread, though: thread-specific-data destructors run on thread *exit*, and the initial thread returns from `main()` rather than calling `pthread_exit` (the tree's only one is `main/asterisk.c:3530`, in `canary_thread`). The loop `break`s rather than `continue`s on a `pending == -1` entry (`:1061`) — harmless, since §5.3's sweep keeps tombstones last, but it is a `break` guarding an invariant the comment does not mention |
| `XXX ERROR XXX A thread holds more locks than '%d'.  Increase AST_MAX_LOCKS!` | `:1115` | Goes to `stderr`, not the logger, deliberately (§5.3). Invisible to log files and remote consoles |
| `Could not create str` | `:1447` | `ast_log_show_lock` failing to allocate 4 KB. The function has no callers (§5.12) |
| `failed to extend from %d to %d` | `main/strings.c:110` | Not this file's, but it is what a truncated `core show locks` emits instead of the `NULL` the checks in §5.11 are looking for |

Note what is absent: nothing in this file reports a corrupted or inconsistent
list, a `suspended` bit that outlived its wait, or a `times_locked` that never
came back down. The tracking has no self-check, which is consistent with it
being bookkeeping rather than enforcement — the same judgement chapter 03 §3.7
reached about rwlock unlock, and the same one §4.5's missing searches invite.

## 5.15 Rules

| Rule | Why |
| --- | --- |
| Do not conclude from an empty `core show locks` that no locks are held | The main thread, and any thread not created through `ast_pthread_create*`, is absent from `lock_infos` entirely (§5.2) |
| Read the two trailing numbers as `(times locked, attempts)`, against the legend | The legend's order is reversed relative to the arguments (§5.9) |
| Ignore the first two time fields | They are epoch timestamps run through a duration formatter; only `<held for>` is a duration (§5.10) |
| Treat `<failed at>` as "last unlocked" | `ast_remove_lock_info` stamps it on an ordinary re-entrant unlock, not just on failure (§5.9) |
| Expect a timed-out `rwlock` acquisition to leave no trace | Only the three trylock entry points mark a failure; the five blocking ones erase the entry (§5.5) |
| If a lock you know is held is missing and the thread is not in a wait, suspect a poisoned slot | `ast_store_lock_info` does not clear `suspended`, and `ast_remove_lock_info` does not clear the slot it vacates (§5.6) |
| Do not call these functions from code that allocates or logs | The allocator's own lock is `NOTRACKING` and the cap diagnostic is an `fprintf`, both for the same reason (§5.3, §5.9) |
| Do not rely on `core show locks` degrading gracefully under memory pressure | Its seven `NULL` checks test something `ast_str_append` never does; a failed grow truncates silently (§5.11) |
| Use `ast_find_lock_info`, not the printers, if you need one fact in code | It reads thread-local storage directly and works for every thread (§5.8) |
| `DETECT_DEADLOCKS` without `DEBUG_THREADS` gives you the alarm and not the evidence | The deadlock logging is independent; `ast_store_lock_info` and the CLI command are not (§5.13) |
| Raise `AST_MAX_LOCKS` rather than ignoring the `stderr` message | Past 64 distinct locks a thread's acquisitions are simply not recorded, and nothing in the log says so (§5.3) |

## Next

[Chapter 06](06-deadlock-avoidance-and-ordering.md) covers deadlock avoidance
and lock ordering: why five macros in `lock.h` are each defined twice and how
the pairs differ, what `DEADLOCK_AVOIDANCE` does to a lock at its 27 call
sites, the `DLA_UNLOCK`/`DLA_LOCK` split macro, and the ordering rules the tree
actually follows.
