# 2. Refcounting and Destruction

**Target tree:** Asterisk, the `24` branch

Chapter 01 showed where an ao2 object lives: a private header behind the pointer
you hold, and a lock behind that. This chapter follows the one function that
decides when that allocation stops existing. `__ao2_ref` is 171 lines
(`main/astobj2.c:498-668`) and it is the whole of ao2's lifetime management —
there is no separate free, no destroy, no close. Every object in Asterisk dies
inside this function or not at all.

```
   ao2_ref(obj, delta)  →  __ao2_ref(obj, delta, NULL, __FILE__, __LINE__, func)
                                     │
        ┌────────────────────────────┤
        │                            ▼
        │                  __INTERNAL_OBJ_CHECK  ──── obj == NULL ──→ return -1
        │                            │                (:511-518, logs **invalid**)
        │                            ▼
        │                     delta == 0 ───────────→ return ref_counter
        │                            │                (:520-523, a pure read)
        │                            ▼
        │        delta < 0 && magic == AO2_MAGIC && weakptr
        │                     → ao2_lock(weakproxy)   (:525-527)
        │                            │
        │                            ▼
        │   ret = ast_atomic_fetch_add(&ref_counter, delta, __ATOMIC_RELAXED)
        │   current_value = ret + delta                (:530-531)
        │                            │
        │        ┌───────────────────┼────────────────────┐
        │        ▼                   ▼                    ▼
        │  current_value > 0   current_value < 0    current_value == 0
        │   object lives        "must never          last reference
        │   (:573-596)           happen"            (:613-667)
        │        │              (:598-611)                │
        │        │                   │                    ▼
        │        │                   │          destructor_fn(user_data)
        │        │                   │          ── WITH NO LOCK HELD ──
        │        │                   │                    │
        │        │                   │                    ▼
        │        │                   │          magic = 0      (:624)
        │        │                   │          switch on lock mode
        │        │                   │          ast_free(...)  (:628-659)
        │        ▼                   ▼                    ▼
        │   return ret          return -1           return ret
        └────────────────────────────────────────────────────
```

Three facts carry this chapter:

1. **The destructor runs with no lock held, on a still-valid object.** The call
   is at `main/astobj2.c:614-616`. The magic number is not cleared until `:624`
   and the lock is not destroyed until the switch at `:628-659`, so `ao2_lock`
   works *inside* a destructor — which is exactly why so many destructors lock
   the object they are destroying. `ast_channel_destructor` locks the channel
   three times (`main/channel.c:2207`, `:2212`, `:2229`), and the last of those
   is a lock/unlock pair with nothing between them.
2. **`ao2_cleanup` never calls `__ao2_cleanup`.** The macro at
   `include/asterisk/astobj2.h:1934` routes unconditionally to
   `__ao2_cleanup_debug`, which forwards the caller's `__FILE__` and `__LINE__`.
   `__ao2_cleanup` exists only to be taken as a function pointer: 14 occurrences
   across 8 files outside the ao2 implementation, and not one of them is a call.
   Every reference dropped through that pointer is attributed in the `refs` log
   to `astobj2.c` line 680.
3. **The refcount is touched exactly once, and relaxed.**
   `ast_atomic_fetch_add(&obj->priv_data.ref_counter, delta, __ATOMIC_RELAXED)`
   at `:530` is the only access to `ref_counter` in the function apart from the
   `delta == 0` read at `:522`. Every branch after it is taken on the stack
   locals `ret` and `current_value`, and nothing orders the refcount against the
   object's own fields.

## 2.1 The five exits

`__ao2_ref` has five `return` statements — `:517`, `:522`, `:595`, `:610` and
`:667` — plus two assertion paths that may not return at all, and a call to
arbitrary user code. The entry and the first two exits:

```c
int __ao2_ref(void *user_data, int delta,
	const char *tag, const char *file, int line, const char *func)
{
	struct astobj2 *obj = __INTERNAL_OBJ_CHECK(user_data, file, line, func);
	struct astobj2_lock *obj_mutex;
	struct astobj2_rwlock *obj_rwlock;
	struct astobj2_lockobj *obj_lockobj;
	int32_t current_value;
	int32_t ret;
	uint32_t privdataoptions;
	struct ao2_weakproxy *weakproxy = NULL;
	const char *lock_state;

	if (obj == NULL) {
		if (ref_log && user_data) {
			fprintf(ref_log, "%p,%d,%d,%s,%d,%s,**invalid**,%s\n",
				user_data, delta, ast_get_tid(), file, line, func, tag ?: "");
			fflush(ref_log);
		}
		return -1;
	}

	/* if delta is 0, just return the refcount */
	if (delta == 0) {
		return obj->priv_data.ref_counter;
	}
```

`__INTERNAL_OBJ_CHECK` is the validating form from chapter 01: it does the
pointer arithmetic back to the header, tests the magic number, and calls
`log_bad_ao2` before returning NULL if the check fails. So the `obj == NULL`
branch at `:511` covers two quite different accidents — a literal NULL
`user_data`, and a non-NULL pointer that is not an ao2 object. The `ref_log`
line distinguishes them only by whether it is written: the guard is
`if (ref_log && user_data)`, so a NULL `user_data` produces no record at all
while a bad pointer produces an `**invalid**` line. Chapter 07 explains what
`refcounter.py` does with those lines, and why it does less than you would
hope.

The second exit is the pure read at `:521-523`. `ao2_ref(obj, 0)` takes no lock,
performs no atomic, and returns the raw `int32_t` — `ref_counter` is not even
declared `volatile` (`main/astobj2.c:63`). The form `ao2_ref(x, 0)` appears at 40
sites in the tree, none of them inside the ao2 implementation: 24 are log or
trace arguments, 13 are assertions in the unit tests, one is dead, and **two**
put the count into live control flow. Those two are worth comparing, because one
is defensible and the other is not.

```c
static void wait_wrapper_removal(struct wait_bridge_wrapper *wrapper)
{
	if (!wrapper) {
		return;
	}

	ao2_lock(wait_bridge_wrappers);
	if (ao2_ref(wrapper, 0) == 2) {
		/* Either we have the only real reference or else wrapper isn't in the container anyway. */
		ao2_unlink(wait_bridge_wrappers, wrapper);
	}
	ao2_unlock(wait_bridge_wrappers);

	ao2_cleanup(wrapper);
}
```

This is the defensible use. The count is read with
`wait_bridge_wrappers` locked, and the only way the count can be 2 is that the
container holds one reference and the caller holds the other — so nobody else
can be racing toward `ao2_unlink`. The comment at `apps/app_bridgewait.c:394`
spells out that the author knows this is what the number means. Read the same
count without the container lock and it is a number that was true once.

The other one is `apps/app_meetme.c:3295`, and it is the same comparison against
2 with the lock given back first — `ast_mutex_unlock(&conf->announcelistlock)` is
at `:3293`, two lines above. It spins, sleeping a second at a time, until the
announce thread drops the reference that `:3290` added. It usually works, and
"usually" is the problem: nothing stops a third party from taking a reference
between the read and the next iteration, and the loop has no way to notice.

The dead site is `main/bridge.c:621`,
`int in_destructor = !ao2_ref(bridge, 0);`, inside an `#if 0` block
(`main/bridge.c:620-622`) with the explanatory comment *"In case we need to know
when the destructor is calling us."* It is commented-out diagnostics, but what
it documents is worth knowing: during destruction the refcount really is zero,
because the atomic at `:530` has already been applied. A function called from a
destructor can tell that it is in one.

## 2.2 The relaxed atomic

```c
	if (delta < 0 && obj->priv_data.magic == AO2_MAGIC && (weakproxy = obj->priv_data.weakptr)) {
		ao2_lock(weakproxy);
	}

	/* we modify with an atomic operation the reference counter */
	ret = ast_atomic_fetch_add(&obj->priv_data.ref_counter, delta, __ATOMIC_RELAXED);
	current_value = ret + delta;
```

Two things happen here that look unremarkable and are not.

The weak-proxy test at `:525` is conditional on `magic == AO2_MAGIC`, not on
`is_ao2_object`. Chapter 01 noted that the magic check accepts both
`AO2_MAGIC` and `AO2_WEAK` because `IS_AO2_MAGIC_BAD` ORs the value with 1.
Here the comparison is exact, and it has to be, because `weakptr` is an
overloaded field. `__ao2_weakproxy_set_object` (`main/astobj2.c:818`) writes it
in both directions —
`weakproxy_internal->priv_data.weakptr = obj` at `main/astobj2.c:844` and
`obj_internal->priv_data.weakptr = weakproxy` at `:845`. On a target object the
field points at the proxy; on the proxy it points back at the target. So the
magic number is the only thing that says which of those two meanings the field
currently has. Loosen the test to accept `AO2_WEAK` and `ao2_ref(proxy, -1)`
would assign the local `weakproxy` the *target*, lock the target at `:526`, and
then read `weakproxy->destroyed_cb` at `:551-552` out of a struct that is not a
`struct ao2_weakproxy` at all. The failure is type confusion, not recursion — recursion
is prevented separately, by the unlink at `:547-548`, which the comment at `:554`
calls out. Chapter 06 covers what the block at `:537-571` does with the lock
once it has it.

Then the counter itself. `__ATOMIC_RELAXED` is passed through to
`__atomic_fetch_add` on the `HAVE_C_ATOMICS` path
(`include/asterisk/lock.h:676`), where it buys atomicity of the
read-modify-write and nothing else: no acquire, no release, no fence. That is
sufficient for the refcount *as a count* — the decrement that reaches zero is
unique, so exactly one thread runs the destructor. It is not sufficient for
anything you may have written to the object before handing the last reference
away. If thread A initialises a field and then drops its reference, and thread B
holds the only other reference and later reads that field, the ordering comes
from whatever synchronisation handed the object between them — the container
lock, the channel lock, a taskprocessor queue — never from `ao2_ref`. ao2
refcounting is not a memory barrier, and nothing in `astobj2.h` claims it is.

There is one wrinkle that makes this easier to get wrong rather than harder.
`ast_atomic_fetch_add` has a second definition at `include/asterisk/lock.h:709`,
used when the compiler offers `__sync` builtins but not C11 atomics, and it
*discards* the `memorder` argument: `__sync_fetch_and_add` is a full barrier.
So on that build the ordering is there, by accident, and code that depends on it
will work until someone builds on a toolchain that takes the other branch at
`:674`. Write as if `:676` is the definition in play.

Both `ret` and `current_value` are locals. From `:531` onward the function never
re-reads `ref_counter`, which is what makes the rest of it safe to reason about:
the value it decides on is the value its own atomic produced, not whatever the
counter happens to hold by the time the branch is taken.

## 2.3 The object lives

```c
	if (0 < current_value) {
		/* The object still lives. */
#define EXCESSIVE_REF_COUNT		100000

		if (EXCESSIVE_REF_COUNT <= current_value && ret < EXCESSIVE_REF_COUNT) {
			char excessive_ref_buf[100];

			/* We just reached or went over the excessive ref count trigger */
			snprintf(excessive_ref_buf, sizeof(excessive_ref_buf),
				"Excessive refcount %d reached on ao2 object %p",
				(int)current_value, user_data);
			ast_log(__LOG_ERROR, file, line, func, "%s\n", excessive_ref_buf);

			__ast_assert_failed(0, excessive_ref_buf, file, line, func);
		}

		if (ref_log && !(obj->priv_data.options & AO2_ALLOC_OPT_NO_REF_DEBUG)) {
			fprintf(ref_log, "%p,%s%d,%d,%s,%d,%s,%d,%s\n", user_data,
				(delta < 0 ? "" : "+"), delta, ast_get_tid(),
				file, line, func, (int)ret, tag ?: "");
			fflush(ref_log);
		}
		return ret;
	}
```

`EXCESSIVE_REF_COUNT` is defined at `:575` and tested at `:577`, and those are
the only two places the name appears anywhere in the tree. The `#define` sits
inside the function body, which is cosmetic rather than scoping — a macro is not
function-scoped, and this one is never `#undef`'d, so it is live for the rest of
the translation unit. The guard is deliberately one-shot:
`EXCESSIVE_REF_COUNT <= current_value && ret < EXCESSIVE_REF_COUNT` fires only on
the single increment that crosses 100,000, so a leak that keeps climbing
produces one error and one assertion rather than one per reference.

The assertion is worth a second look, because it is `__ast_assert_failed` at
`:586` and not `ast_assert`. `ast_assert` is `#define ast_assert(a)` — nothing at
all — outside `AST_DEVMODE` (`include/asterisk/utils.h:779`). Calling
`__ast_assert_failed` directly bypasses that, so the excessive-refcount tripwire
is present in a production build. It still only *crashes* under `DO_CRASH`, but
not because of the `noreturn` attribute — that is a consequence. The abort comes
from the last line of `__ast_assert_failed`, which calls `ast_do_crash`
(`main/utils.c:2870`), and `ast_do_crash`'s entire body is inside
`#if defined(DO_CRASH)` (`main/utils.c:2838-2848`). Without that flag you get a
`FRACK!` on stderr, the same message through the logger, and a full backtrace
from `ast_log_backtrace` (`main/utils.c:2856-2862`) — then execution continues.

The ordinary `refs` line at `:589-594` is the one you will read most. It is
suppressed for objects allocated with `AO2_ALLOC_OPT_NO_REF_DEBUG`, which
chapter 01 showed has exactly two users — the hash and rbtree container nodes —
and it records `(int)ret`, the count *before* the operation. A line reading
`+1,...,3` means the count went from 3 to 4.

One thing the line does not record is who holds the reference. The tag is a
string the caller wrote, and `ao2_ref` passes NULL for it
(`include/asterisk/astobj2.h:459`). `ao2_t_ref` is the tagging form, and counting
occurrences outside the five implementation files — `main/astobj2.c`,
`astobj2_container.c`, `astobj2_hash.c`, `astobj2_rbtree.c` and
`include/asterisk/astobj2.h` — it has 88 against `ao2_ref`'s 2,963. The ratio is
worse than that looks: 36 of the 88 are in ao2's own unit tests
(`tests/test_astobj2.c` and `tests/test_astobj2_weaken.c`), leaving 52 tagged
call sites in production code. The refcount is a count, not a registry; this is
why a leaked object is diagnosed by diffing a log rather than by interrogating
the object.

Worth correcting a common belief here: none of this is gated on the `REF_DEBUG`
compile flag. `ref_log` is a static `FILE *` (`main/astobj2.c:44`) opened in
`astobj2_init` when `ast_opt_ref_debug` is set (`:1173-1179`), and that flag is
set by `refdebug = yes` in `asterisk.conf` (`main/options.c:325-326`). The
comment at `main/options.c:230-231` is explicit: *"The REF_DEBUG compiler flag is
now only used to enable refdebug by default. Support for debugging reference
counts is always compiled in."* You do not need a rebuild to get a `refs` file —
only a restart, since `ref_log` is opened once in `astobj2_init`.

## 2.4 The branch that must never happen

```c
	/* this case must never happen */
	if (current_value < 0) {
		ast_log(__LOG_ERROR, file, line, func,
			"Invalid refcount %d on ao2 object %p\n", (int)current_value, user_data);
		if (ref_log) {
			/* Log to ref_log even if AO2_ALLOC_OPT_NO_REF_DEBUG */
			fprintf(ref_log, "%p,%d,%d,%s,%d,%s,**invalid**,%s\n",
				user_data, delta, ast_get_tid(), file, line, func, tag ?: "");
			fflush(ref_log);
		}
		ast_assert(0);
		/* stop here even if assert doesn't DO_CRASH */
		return -1;
	}
```

A negative count means someone released a reference they did not hold. There is
nothing to be done about it — the object may already have been freed by the
thread that took the count to zero, and this function is reading through a
pointer into it — so the path logs and gets out.

Three details. The `ref_log` guard at `:602` has no `NO_REF_DEBUG` test, and the
comment at `:603` says why: a container node going negative is a bug worth
recording even though its ordinary traffic is suppressed. The assertion at `:608`
*is* `ast_assert`, so unlike the excessive-refcount tripwire it vanishes outside
`AST_DEVMODE` — hence the comment at `:609` and the explicit `return -1`, which
is the only thing stopping the function from falling through into the destructor
path with a negative count. And the two `**invalid**` flavours, this one and the
bad-pointer one at `:513`, print the identical format string. Nothing in the
`refs` file tells you which happened.

## 2.5 The destructor runs unlocked

```c
	/* last reference, destroy the object */
	if (obj->priv_data.destructor_fn != NULL) {
		obj->priv_data.destructor_fn(user_data);
	}

#ifdef AO2_DEBUG
	ast_atomic_fetchadd_int(&ao2.total_mem, - obj->priv_data.data_size);
	ast_atomic_fetchadd_int(&ao2.total_objects, -1);
#endif

	/* In case someone uses an object after it's been freed */
	obj->priv_data.magic = 0;
	/* Save the options locally so the ref_log print at the end doesn't access freed data */
	privdataoptions = obj->priv_data.options;
```

`obj->priv_data.destructor_fn(user_data)` is called without `__ao2_ref` holding
the object's own lock, because `__ao2_ref` never took it. The only lock this
function ever acquires is the weak proxy's, at `:526`, and that is released at
`:558` before the destructor path is reached. There is no version of this that
locks: the lock is about to be destroyed by the switch twelve lines below. (Not
holding it is not the same as it being free — a caller who ignores the contract
below can have the lock held when the destructor runs, which is a deadlock
waiting for the destructor's first `ao2_lock`.)

The consequence is not that destructors must avoid locking — it is the opposite.
The object is fully valid at `:614`: `magic` still holds `AO2_MAGIC` (it is
cleared at `:624`), the lock still exists, and `ao2_lock` will happily take it.
So a destructor that touches state another thread could be touching *should*
lock, and `ast_channel_destructor` does:

```c
	if (ast_channel_internal_is_finalized(chan)) {
		/* A channel snapshot should not be in the process of being staged now. */
		ast_assert(!ast_test_flag(ast_channel_flags(chan), AST_FLAG_SNAPSHOT_STAGE));

		ast_channel_lock(chan);
		ast_channel_publish_final_snapshot(chan);
		ast_channel_unlock(chan);
	}

	ast_channel_lock(chan);

	ast_channel_endpoint_set(chan, NULL);

	/* Get rid of each of the data stores on the channel */
	while ((datastore = AST_LIST_REMOVE_HEAD(ast_channel_datastores(chan), entry)))
		/* Free the data store */
		ast_datastore_free(datastore);

	/* While the channel is locked, take the reference to its callid while we tear down the call. */
	callid = ast_channel_callid(chan);
	ast_channel_callid_cleanup(chan);

	ast_channel_unlock(chan);

	/* Lock and unlock the channel just to be sure nobody has it locked still
	   due to a reference that was stored in a datastore. (i.e. app_chanspy) */
	ast_channel_lock(chan);
	ast_channel_unlock(chan);
```

Three locked regions in twenty-eight lines, and the third one
(`main/channel.c:2229-2230`) has no body. The comment above it
(`:2227-2228`) explains it: *"Lock and unlock the channel just to be sure nobody
has it locked still due to a reference that was stored in a datastore. (i.e.
app_chanspy)"* That construct only makes sense if a thread can be holding the
channel lock at the moment the refcount reaches zero, and the comment names the
route: a *pointer* parked in a datastore, reachable without a reference of its
own. Something followed that pointer and took the lock. The datastores
themselves are not freed until `:2217-2219`, inside the second locked region, so
the window is real. Acquiring the lock here waits for that thread to let go;
releasing it immediately is fine, because by construction nobody can acquire it
again — there are no references left from which to reach the object legitimately.

The mirror image of that is a crash. Chapter 03 covers locking properly, but the
rule belongs here because it is a rule about references, and `ao2_ref`'s own doc
comment states it more strictly than you might expect
(`include/asterisk/astobj2.h:449-450`): *"The object must not be locked by the
caller of this function, as it is invalid to try to unlock it after releasing the
reference."* Not "across the last reference" — across **any** call. The reason
given is the narrow one, but the prohibition is broad, and in practice the thing
that bites is the last one: the unlock lands on a mutex that `:632` has already
destroyed and `:634` has already freed.

## 2.6 Teardown, by lock mode

```c
	switch (obj->priv_data.options & AO2_ALLOC_OPT_LOCK_MASK) {
	case AO2_ALLOC_OPT_LOCK_MUTEX:
		obj_mutex = INTERNAL_OBJ_MUTEX(user_data);
		lock_state = obj->priv_data.lockused ? "used" : "unused";
		ast_mutex_destroy(&obj_mutex->mutex.lock);

		ast_free(obj_mutex);
		break;
	case AO2_ALLOC_OPT_LOCK_RWLOCK:
		obj_rwlock = INTERNAL_OBJ_RWLOCK(user_data);
		lock_state = obj->priv_data.lockused ? "used" : "unused";
		ast_rwlock_destroy(&obj_rwlock->rwlock.lock);

		ast_free(obj_rwlock);
		break;
	case AO2_ALLOC_OPT_LOCK_NOLOCK:
		lock_state = "none";
		ast_free(obj);
		break;
	case AO2_ALLOC_OPT_LOCK_OBJ:
		obj_lockobj = INTERNAL_OBJ_LOCKOBJ(user_data);
		lock_state = obj->priv_data.lockused ? "used" : "unused";
		ao2_t_ref(obj_lockobj->lockobj.lock, -1, "release lockobj");

		ast_free(obj_lockobj);
		break;
	default:
		ast_log(__LOG_ERROR, file, line, func,
			"Invalid lock option on ao2 object %p\n", user_data);
		lock_state = "invalid";
		break;
	}
```

The switch mirrors the allocation switch from chapter 01 exactly, and each arm
has to free the pointer that `internal_ao2_alloc` actually got from `ast_calloc`
— not `obj`. Three of the four arms re-derive it with the matching
`INTERNAL_OBJ_*` macro. `AO2_ALLOC_OPT_LOCK_NOLOCK` is the one that frees `obj`
directly (`:645`), and that is correct rather than sloppy: with no lock in front
of the header, `INTERNAL_OBJ(user_data)` *is* the allocation base.

Two arms do more than free. The mutex and rwlock arms destroy the lock
primitive before freeing its storage (`:632`, `:639`); skipping that would leak
whatever the platform's pthread implementation attaches to a mutex. The
`AO2_ALLOC_OPT_LOCK_OBJ` arm drops a reference on another object (`:650`,
tagged `"release lockobj"`), which is a recursive entry into `__ao2_ref` from
inside the teardown of an object whose magic has already been zeroed. It is safe
because the two objects are distinct, but it means one `ao2_cleanup` can cascade
into an arbitrary chain of destructors.

The `default:` arm at `:654-658` is unreachable. Chapter 01 established why:
`AO2_ALLOC_OPT_LOCK_MASK` is two bits and all four values are named, so the
switch is exhaustive over its own domain. It is also the only assignment of
`lock_state = "invalid"`, which means the `refs` log can never print
`lock-state:invalid`. If you ever see that string, the bug is not the one the
message describes. Note too that it `break`s without freeing anything — on the
impossible path the object leaks rather than being freed wrongly, which is the
right choice for a branch nobody can reach.

`lock_state` itself comes from `obj->priv_data.lockused`, read in three of the
four arms after `magic` has been cleared but before the memory is freed. That
bit has exactly two writers, `__ao2_lock` at `main/astobj2.c:235` and
`__ao2_trylock` at `:355`, and both are `if (ref_log)`-guarded — so the
`used`/`unused` distinction in the destructor line is meaningful precisely in
the run where you can see it:

```c
	if (ref_log && !(privdataoptions & AO2_ALLOC_OPT_NO_REF_DEBUG)) {
		fprintf(ref_log, "%p,%d,%d,%s,%d,%s,**destructor**lock-state:%s**,%s\n",
			user_data, delta, ast_get_tid(), file, line, func, lock_state, tag ?: "");
		fflush(ref_log);
	}

	return ret;
```

`privdataoptions` is the reason this works. It was copied at `:626`, before the
free, with the comment at `:625` saying so — the object is gone by the time the
`NO_REF_DEBUG` bit is tested.

## 2.7 The convenience layer

Almost nobody calls `ao2_ref` with an explicit `-1` on a pointer that might be
NULL. The three wrappers that cover the common shapes are all macros, and all
three have sharp edges.

```c
void __ao2_cleanup_debug(void *obj, const char *tag, const char *file, int line, const char *function)
{
	if (obj) {
		__ao2_ref(obj, -1, tag, file, line, function);
	}
}

void __ao2_cleanup(void *obj)
{
	if (obj) {
		ao2_ref(obj, -1);
	}
}
```

```c
/*! gcc __attribute__(cleanup()) functions
 * \note they must be able to handle NULL parameters because most of the
 * allocation/find functions can fail and we don't want to try to tear
 * down a NULL */
void __ao2_cleanup(void *obj);
void __ao2_cleanup_debug(void *obj, const char *tag, const char *file, int line, const char *function);
#define ao2_cleanup(obj) __ao2_cleanup_debug((obj), NULL, __FILE__, __LINE__, __PRETTY_FUNCTION__)
#define ao2_t_cleanup(obj, tag) __ao2_cleanup_debug((obj), (tag), __FILE__, __LINE__, __PRETTY_FUNCTION__)
```

`ao2_cleanup` is NULL-safe and discards the return value. Note which function
the macro picks: `__ao2_cleanup_debug`, always, with no `#if` around it. So the
plain `__ao2_cleanup` at `:677-682` is never reached through the macro, and
the `ao2_ref(obj, -1)` inside it at `:680` expands with `__FILE__` and
`__LINE__` of *that* line. Its purpose is to be a function pointer —
bridge-hook destructors, `stasis_app_send_command_async` data destructors,
`ast_speech_unregister_engines` — 14 occurrences across 8 files outside the ao2
implementation, every one of them passing the symbol rather than calling it.
When you are reading a `refs` file and a drop is attributed to
`astobj2.c,680,__ao2_cleanup`, that is the whole class: something handed the
symbol to a callback slot, and the real owner is whichever subsystem invoked the
callback. (The file field in a `refs` line is raw `__FILE__`, and `main/` is
built from inside its own directory, so the token is `astobj2.c` with no
directory on it. The documented example lines at
`include/asterisk/astobj2.h:152-162` show the same thing for `chan_sip.c`.)

```c
#define ao2_bump(obj) \
	ao2_t_bump((obj), NULL)

#define ao2_t_bump(obj, tag)						\
	({							\
		typeof(obj) __obj_ ## __LINE__ = (obj);		\
		if (__obj_ ## __LINE__) {			\
			ao2_t_ref(__obj_ ## __LINE__, +1, (tag));	\
		}						\
		__obj_ ## __LINE__;				\
	})
```

`ao2_bump` is NULL-safe, returns the object, and is a GCC statement expression,
so it composes: `foo->bar = ao2_bump(baz)` is a ref bump and an assignment in
one. The temporary is named `__obj_ ## __LINE__`, which looks like it expands to
`__obj_1234`. It does not. `##` suppresses macro expansion of its operands, so
`__LINE__` is pasted literally and the variable is called `__obj___LINE__` on
every line of every file. The hygiene the name advertises is not there. Nesting
still works, because each statement expression opens a scope and the inner one
shadows — but it shadows rather than being distinct, so `-Wshadow` has opinions,
and a careless edit that pulls the inner expression out of its braces produces a
redefinition rather than a new variable. This is not an `ao2_bump` quirk: the
same idiom appears in `ao2_t_replace` as `__dst_`/`__src_`
(`include/asterisk/astobj2.h:506-515`), in `__INTERNAL_OBJ_CHECK`
(`main/astobj2.c:171-181`) and in `main/astobj2_private.h:49-59`, so the
literal-paste behaviour is tree-wide.

```c
#define ao2_replace(dst, src) \
	ao2_t_replace((dst), (src), NULL)

#define ao2_t_replace(dst, src, tag) \
	{\
		typeof(dst) *__dst_ ## __LINE__ = &dst; \
		typeof(src) __src_ ## __LINE__ = src; \
		if (__src_ ## __LINE__ != *__dst_ ## __LINE__) { \
			if (__src_ ## __LINE__) {\
				ao2_t_ref(__src_ ## __LINE__, +1, (tag)); \
			} \
			if (*__dst_ ## __LINE__) {\
				ao2_t_ref(*__dst_ ## __LINE__, -1, (tag)); \
			} \
			*__dst_ ## __LINE__ = __src_ ## __LINE__; \
		} \
	}
```

`ao2_replace` does the three-step dance correctly: bump the source first, then
drop the destination, then assign. Bumping first is what makes
`ao2_replace(x, x)` safe even without the `!=` guard at `:508`, and the guard
makes it free. Dropping the destination before the assignment is safe because
`*__dst` is still the old pointer at that point.

The edge here is syntactic. The macro expands to a brace block, not a statement
expression — note the `{` at `:505` where `ao2_bump` has `({`. So
`ao2_replace(a, b);` becomes `{ ... };`, a compound statement followed by an
empty one, and that breaks the one construct where a trailing semicolon matters:

<!-- illustrative -->
```c
/* WRONG — does not compile */
if (want_new)
	ao2_replace(cur, new);
else
	ao2_cleanup(cur);
```

The `else` now follows two statements. `ao2_cleanup` has no such problem because
it expands to a function call. Braces around the arms fix it, which is why you
rarely see this in the tree.

## 2.8 RAII_VAR

`RAII_VAR` is the second most common thing in this guide after `ao2_ref` itself:
2,723 invocations outside the ao2 implementation and `utils.h`'s own definition,
1,570 of which name `ao2_cleanup` as the destructor. It is not an ao2 construct
at all:

```c
#elif defined(__GNUC__)

#define RAII_VAR(vartype, varname, initval, dtor)                              \
    auto void _dtor_ ## varname (vartype * v);                                 \
    void _dtor_ ## varname (vartype * v) { dtor(*v); }                         \
    vartype varname __attribute__((cleanup(_dtor_ ## varname))) = (initval)
```

A nested function whose address is attached to the variable with GCC's
`cleanup` attribute, so it runs on every exit from the enclosing scope —
`return`, `break`, `goto`, falling off the end. There is a separate
clang implementation using blocks at `include/asterisk/utils.h:974-977`, and
`#error` for anything else at `:987`.

Three consequences follow from the expansion rather than from the intent.

The destructor runs on *scope* exit, not function exit. A `RAII_VAR` inside a
loop body drops its reference on every iteration, which is usually what you
want and occasionally a surprise if you declared it there to hold something
across iterations.

The initialiser runs exactly once, at the declaration, and the cleanup is armed
immediately. So `RAII_VAR(struct foo *, f, ao2_find(c, k, OBJ_KEY), ao2_cleanup)`
is correct when the find fails, because `ao2_cleanup(NULL)` is a no-op — which
is precisely what the comment at `include/asterisk/astobj2.h:1929-1931` says the
cleanup functions exist to tolerate.

And reassigning a `RAII_VAR` without releasing what it held leaks, while
reassigning it to something you do not own double-frees. `ao2_replace` on the
variable is the correct move; it is not a coincidence that the two macros read
as a pair.

## 2.9 Common bugs

| Symptom | Cause |
|---|---|
| Crash in `pthread_mutex_unlock` immediately after an `ao2_cleanup` | the lock was held across the drop of the last reference; the mutex was destroyed and freed by the switch at `main/astobj2.c:628-659` before the unlock ran. `include/asterisk/astobj2.h:449-450` forbids it |
| Destructor reads a field another thread is writing | the destructor runs with no lock held (`main/astobj2.c:614-616`). Lock inside it — the lock is still alive — or arrange that the last reference is undroppable while anyone can reach the object |
| `ao2_lock` inside a destructor asserts with "bad magic number" | the destructor did not run from `__ao2_ref`; something called it directly. From `__ao2_ref` the magic is intact until `:624` |
| Refcount logic read with `ao2_ref(obj, 0)` behaves differently under load | the count is a snapshot. It is only usable as a decision when a lock makes it unable to change — see `apps/app_bridgewait.c:393` for the pattern done right |
| One "Excessive refcount" error in the log and then silence | the tripwire is one-shot by construction (`main/astobj2.c:577` tests `ret < EXCESSIVE_REF_COUNT`). The leak did not stop; the reporting did |
| "Invalid refcount" in a production build with no crash and no assertion | `ast_assert(0)` at `main/astobj2.c:608` compiles to nothing outside `AST_DEVMODE` (`include/asterisk/utils.h:779`). The `return -1` at `:610` is what stops the fall-through |
| Fields written before the last reference was handed over are stale in the destructor | `__ATOMIC_RELAXED` at `:530` orders nothing. Whatever handed the object between threads supplies the ordering |
| Every leaked reference in the `refs` file points at `astobj2.c` line 680 | they were dropped through the `__ao2_cleanup` function pointer, not from a call site. Find the subsystem that registered the callback |
| `refs` log shows `**invalid**` and the object is fine | a non-ao2 pointer reached `ao2_ref`; the same format string covers that and a genuine negative count |
| `else` after `ao2_replace` fails to compile | the macro is a brace block (`include/asterisk/astobj2.h:505`), not a statement expression. Brace the arms |

## 2.10 Rules

| Rule | Source |
|---|---|
| Never hold an object's lock across the drop of your own last reference | `include/asterisk/astobj2.h:449-450` |
| A destructor may lock the object it is destroying, and should if other threads can reach its fields | `main/astobj2.c:614-616`, `:624`; `main/channel.c:2212` |
| A refcount read with `ao2_ref(obj, 0)` is a decision only under a lock that pins it | `main/astobj2.c:520-523`; `apps/app_bridgewait.c:392-397` |
| `ao2_ref` is not a memory barrier; the ordering comes from whatever transferred the object | `main/astobj2.c:530` |
| Use `ao2_bump` to take a reference inline and `ao2_replace` to swap one out; both are NULL-safe | `include/asterisk/astobj2.h:480-490`, `:501-517` |
| Pass `__ao2_cleanup` where a destructor callback is wanted; never call it | `include/asterisk/astobj2.h:1932-1934`; 14 sites, 0 calls |
| `RAII_VAR` releases on scope exit, not function exit | `include/asterisk/utils.h:981-984` |
| Ref logging needs `refdebug = yes`, not a rebuild | `main/options.c:230-231`, `:325-326`; `main/astobj2.c:1173-1179` |

## Next

[Chapter 03](03-locking.md) takes the lock the destructor path has just
destroyed and works backwards: the four lock modes as a caller sees them, what
`ao2_lock` does to an object that has an rwlock, and the promotion
`__adjust_lock` performs.
