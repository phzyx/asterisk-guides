# 3. Locking

**Target tree:** Asterisk, the `24` branch

Chapter 01 put a lock in front of the private header and chapter 02 destroyed
it. This chapter is about the 253 lines in between: `__ao2_lock`
(`main/astobj2.c:222-286`), `__ao2_unlock` (`:288-340`), `__ao2_trylock`
(`:342-406`) and the private `__adjust_lock` (`:425-474`). All four are the same
switch on the same two option bits, and all four are easier to read as one
table than as four functions.

The thing to understand first is that ao2's locking API is *wider than its
implementation*. There are three lock macros, three trylock macros and three
scoped-lock macros — nine entry points — and they all funnel into a `lock_how`
argument that two of the four lock modes ignore completely.

```
   ao2_lock(a)      →  __ao2_lock(a, AO2_LOCK_REQ_MUTEX,  ...)
   ao2_rdlock(a)    →  __ao2_lock(a, AO2_LOCK_REQ_RDLOCK, ...)
   ao2_wrlock(a)    →  __ao2_lock(a, AO2_LOCK_REQ_WRLOCK, ...)
                                      │
                                      ▼
            switch (options & AO2_ALLOC_OPT_LOCK_MASK)
                                      │
    ┌──────────────┬──────────────────┼──────────────────┬──────────────┐
    ▼              ▼                  ▼                  ▼              ▼
 MUTEX (0)     RWLOCK (1)         NOLOCK (2)         LOCK_OBJ (3)   default:
    │              │                  │                  │              │
 ignores        REQ_MUTEX ─┐       does nothing      recurse into    ast_log
 lock_how       REQ_WRLOCK ┴→ wrlock   return 0      the borrowed    return -1
 entirely       REQ_RDLOCK  → rdlock                 object's lock   (unreachable)
    │              │                  │                  │
 recursive      write lock is     no-op             whatever the
 (pthread       NOT recursive     success           lender has
  RECURSIVE)    (pthread rwlock)
    │              │
    │              └── also maintains rwlock.num_lockers:
    │                  -1 == a writer holds it, >0 == that many readers
    │                  (main/astobj2.c:128-129)
    ▼
 MUTEX:  562 of 783 sites          NOLOCK:   194 sites
 RWLOCK:  26 of 783 sites          LOCK_OBJ:   1 site
```

Three facts carry the chapter:

1. **`ao2_lock` on an object allocated `AO2_ALLOC_OPT_LOCK_RWLOCK` takes the
   *write* lock.** `AO2_LOCK_REQ_MUTEX` and `AO2_LOCK_REQ_WRLOCK` share a case
   label at `main/astobj2.c:251-252`. So the plain `ao2_lock` — 626 mentions
   outside the ao2 implementation, three of them wrapper macros rather than
   calls, so 623 direct calls plus 800 more through `ast_channel_lock` and
   `ast_tone_zone_lock` — means "mutex" on a mutex object and
   "exclusive" on an rwlock object, and the two behave differently when nested.
   The mutex is recursive (`AST_MUTEX_KIND` is `PTHREAD_MUTEX_RECURSIVE`,
   `main/lock.c:121` and `:170`), so the same `ao2_lock` nests harmlessly on 562
   allocation sites. On the 26 rwlock sites POSIX makes a nested acquisition
   *undefined*, and `main/lock.c` does nothing to tame it: `__ast_rwlock_wrlock`
   calls `pthread_rwlock_wrlock` directly (`:1036`) and returns whatever it
   returns. Under `DEBUG_THREADS` a failure is logged and `DO_THREAD_CRASH` fires
   (`:1063-1067`); in a production build the return value reaches `ao2_lock`'s
   caller, which — as §3.9 notes — almost never checks it.
2. **`ao2_lock` on a lockless object returns success.** The `NOLOCK` case is an
   empty `break` with a comment (`:272-274`), and `res` is still 0. There are 194
   allocation sites that ask for no lock, and every `ao2_lock`, `ao2_rdlock`,
   `ao2_trylock` and `SCOPED_AO2LOCK` applied to one of them silently protects
   nothing and reports that it worked. Nothing warns, at compile time or run
   time.
3. **Lock promotion is not atomic, and it is the only way the rwlock mode is
   reachable from the containers.** `__adjust_lock` promotes a read lock to a
   write lock by *releasing it* — `ao2_unlock(user_data); ao2_wrlock(user_data);`
   at `:447-448`. Every invariant the caller established under the read lock is
   void across those two statements. All 16 `__adjust_lock` call sites are inside
   the ao2 implementation, but only seven of them are on an `OBJ_NOLOCK` path —
   §3.4 breaks down the other nine, which include two in node destructors that
   run on ordinary locked paths.

## 3.1 The public surface

Nine macros, one enum and two inline functions. The enum first:

```c
/*! \brief Which lock to request. */
enum ao2_lock_req {
	/*! Request the mutex lock be acquired. */
	AO2_LOCK_REQ_MUTEX,
	/*! Request the read lock be acquired. */
	AO2_LOCK_REQ_RDLOCK,
	/*! Request the write lock be acquired. */
	AO2_LOCK_REQ_WRLOCK,
};
```

`AO2_LOCK_REQ_MUTEX` is 0, so it is also what you get by default — and, as
chapter 01 noted of `AO2_ALLOC_OPT_LOCK_MUTEX`, being zero is load-bearing
rather than incidental. The three lock macros differ only in which enumerator
they pass:

```c
int __ao2_lock(void *a, enum ao2_lock_req lock_how, const char *file, const char *func, int line, const char *var);
#define ao2_lock(a) __ao2_lock(a, AO2_LOCK_REQ_MUTEX, __FILE__, __PRETTY_FUNCTION__, __LINE__, #a)
#define ao2_rdlock(a) __ao2_lock(a, AO2_LOCK_REQ_RDLOCK, __FILE__, __PRETTY_FUNCTION__, __LINE__, #a)
#define ao2_wrlock(a) __ao2_lock(a, AO2_LOCK_REQ_WRLOCK, __FILE__, __PRETTY_FUNCTION__, __LINE__, #a)
```

Note what is captured and what is not. `__FILE__`, `__PRETTY_FUNCTION__` and
`__LINE__` go through unconditionally — unlike the `refs` logging of chapter 02,
there is no debug build or runtime flag involved, because these are the same
parameters `main/lock.c` wants for `DEBUG_THREADS` lock tracking. The sixth and
last argument is `#a`: the *stringified source expression*, so a lock taken as
`ao2_lock(chan->bridge)` records the text `chan->bridge`. That string is what
`core show locks` prints as the lock's name.

`ao2_unlock` takes no `lock_how`, which is the first hint of something important:
the unlock path has to work out for itself whether it is releasing a read lock or
a write lock.

```c
int __ao2_unlock(void *a, const char *file, const char *func, int line, const char *var);
#define ao2_unlock(a) __ao2_unlock(a, __FILE__, __PRETTY_FUNCTION__, __LINE__, #a)
```

The trylock family is the same shape with a different target:

```c
int __ao2_trylock(void *a, enum ao2_lock_req lock_how, const char *file, const char *func, int line, const char *var);
#define ao2_trylock(a) __ao2_trylock(a, AO2_LOCK_REQ_MUTEX, __FILE__, __PRETTY_FUNCTION__, __LINE__, #a)
#define ao2_tryrdlock(a) __ao2_trylock(a, AO2_LOCK_REQ_RDLOCK, __FILE__, __PRETTY_FUNCTION__, __LINE__, #a)
#define ao2_trywrlock(a) __ao2_trylock(a, AO2_LOCK_REQ_WRLOCK, __FILE__, __PRETTY_FUNCTION__, __LINE__, #a)
```

Of those three, `ao2_tryrdlock` has **zero** call sites in the entire tree and
`ao2_trywrlock` has exactly one, `res/res_sorcery_memory_cache.c:528`.
`ao2_trylock` names ten sites outside the ao2 implementation, and two of those
are not calls but further macro definitions — `ast_channel_trylock`
(`include/asterisk/channel.h:2985`) and `ast_tone_zone_trylock`
(`include/asterisk/indications.h:198`). So eight direct calls, plus whatever the
wrappers attract: `ast_channel_trylock` has 32 call sites of its own and
`ast_tone_zone_trylock` has none. The imbalance is the chapter in miniature —
the mutex mode is the one that gets used, and the only trylock with real volume
is the one hidden behind a channel-shaped name.

Two inline helpers round out the header, and they are worth quoting because of
what the counts say about them:

```c
/*!
 * \brief Increment reference count on an object and lock it
 * \since 13.9.0
 *
 * \param[in] obj A pointer to the ao2 object
 * \retval 0 The object is not an ao2 object or wasn't locked successfully
 * \retval 1 The object's reference count was incremented and was locked
 */
AST_INLINE_API(
int ao2_ref_and_lock(void *obj),
{
	ao2_ref(obj, +1);
	if (ao2_lock(obj)) {
		ao2_ref(obj, -1);
		return 0;
	}
	return 1;
}
)

/*!
 * \brief Unlock an object and decrement its reference count
 * \since 13.9.0
 *
 * \param[in] obj A pointer to the ao2 object
 * \retval 0 The object is not an ao2 object or wasn't unlocked successfully
 * \retval 1 The object was unlocked and it's reference count was decremented
 */
AST_INLINE_API(
int ao2_unlock_and_unref(void *obj),
{
	if (ao2_unlock(obj)) {
		return 0;
	}
	ao2_ref(obj, -1);

	return 1;
}
)
```

Both are correct. `ao2_ref_and_lock` bumps first and backs the bump out if the
lock fails, so it can never leave a reference behind; `ao2_unlock_and_unref`
unlocks before it unrefs, which is exactly the ordering `astobj2.h:449-450`
demands. Both have been in the tree since 13.9.0. Neither is called anywhere:
`ao2_ref_and_lock` appears once in the tree, at its own definition
(`include/asterisk/astobj2.h:771`), and so does `ao2_unlock_and_unref` (`:791`).
Two public functions, zero callers, ten years — 13.9.0 was tagged in May 2016.
If you are looking for the blessed way to hand a locked object to another
thread, this is it, and nobody uses it.

## 3.2 `__ao2_lock`, case by case

```c
int __ao2_lock(void *user_data, enum ao2_lock_req lock_how, const char *file, const char *func, int line, const char *var)
{
	struct astobj2 *obj = __INTERNAL_OBJ_CHECK(user_data, file, line, func);
	struct astobj2_lock *obj_mutex;
	struct astobj2_rwlock *obj_rwlock;
	struct astobj2_lockobj *obj_lockobj;
	int res = 0;

	if (obj == NULL) {
		return -1;
	}

	if (ref_log) {
		obj->priv_data.lockused  = 1;
	}

	switch (obj->priv_data.options & AO2_ALLOC_OPT_LOCK_MASK) {
	case AO2_ALLOC_OPT_LOCK_MUTEX:
		obj_mutex = INTERNAL_OBJ_MUTEX(user_data);
		res = __ast_pthread_mutex_lock(file, line, func, var, &obj_mutex->mutex.lock);
#ifdef AO2_DEBUG
		if (!res) {
			ast_atomic_fetchadd_int(&ao2.total_locked, 1);
		}
#endif
		break;
	case AO2_ALLOC_OPT_LOCK_RWLOCK:
		obj_rwlock = INTERNAL_OBJ_RWLOCK(user_data);
		switch (lock_how) {
		case AO2_LOCK_REQ_MUTEX:
		case AO2_LOCK_REQ_WRLOCK:
			res = __ast_rwlock_wrlock(file, line, func, &obj_rwlock->rwlock.lock, var);
			if (!res) {
				ast_atomic_fetchadd_int(&obj_rwlock->rwlock.num_lockers, -1);
#ifdef AO2_DEBUG
				ast_atomic_fetchadd_int(&ao2.total_locked, 1);
#endif
			}
			break;
		case AO2_LOCK_REQ_RDLOCK:
			res = __ast_rwlock_rdlock(file, line, func, &obj_rwlock->rwlock.lock, var);
			if (!res) {
				ast_atomic_fetchadd_int(&obj_rwlock->rwlock.num_lockers, +1);
#ifdef AO2_DEBUG
				ast_atomic_fetchadd_int(&ao2.total_locked, 1);
#endif
			}
			break;
		}
		break;
	case AO2_ALLOC_OPT_LOCK_NOLOCK:
		/* The ao2 object has no lock. */
		break;
	case AO2_ALLOC_OPT_LOCK_OBJ:
		obj_lockobj = INTERNAL_OBJ_LOCKOBJ(user_data);
		res = __ao2_lock(obj_lockobj->lockobj.lock, lock_how, file, func, line, var);
		break;
	default:
		ast_log(__LOG_ERROR, file, line, func, "Invalid lock option on ao2 object %p\n",
			user_data);
		return -1;
	}

	return res;
}
```

Read the cases in order of how much work they do.

**`AO2_ALLOC_OPT_LOCK_MUTEX`** (`:239-247`) ignores `lock_how` entirely. There
is no inner switch; `obj_mutex->mutex.lock` is taken with
`__ast_pthread_mutex_lock` and that is the whole case. `ao2_rdlock` on a mutex
object therefore takes the mutex *exclusively*, which is safe but gives up all
the concurrency the caller was asking for, silently. The `AO2_DEBUG` counter
bump at `:243-245` is guarded on `!res`, so the global `ao2.total_locked` tracks
successful acquisitions only.

**`AO2_ALLOC_OPT_LOCK_RWLOCK`** (`:248-271`) is the only case that reads
`lock_how`, and it reads it as two outcomes rather than three:
`AO2_LOCK_REQ_MUTEX` and `AO2_LOCK_REQ_WRLOCK` fall into the same arm at
`:251-252`. Each successful acquisition also adjusts `num_lockers` — `-1` for a
writer, `+1` for a reader — which is the bookkeeping `__ao2_unlock` and
`__adjust_lock` later depend on.

Note the inner `switch (lock_how)` has no `default:` and does not cover any
value outside the enum. If `lock_how` arrived as something else, `res` stays 0
and the function returns success having taken no lock at all. Every caller goes
through one of the six macros, so this is not reachable today. It is also not
unique: all three `switch (lock_how)` statements in the file lack a `default:`
— `__ao2_lock:250`, `__ao2_trylock:370` and `__adjust_lock:440` — so each of
them turns an out-of-range `lock_how` into silent success rather than the logged
error the *outer* switches give an out-of-range option bit pattern.

**`AO2_ALLOC_OPT_LOCK_NOLOCK`** (`:272-274`) is three lines, two of them a
comment:

```c
	case AO2_ALLOC_OPT_LOCK_NOLOCK:
		/* The ao2 object has no lock. */
		break;
```

`res` is never assigned, so the function returns 0. This is deliberate and it is
what makes the whole `OBJ_NOLOCK`/`__adjust_lock` machinery safe to run against a
lockless container — but from the caller's side it means a successful
`ao2_lock` is not evidence that anything is locked. Fact 2 above.

**`AO2_ALLOC_OPT_LOCK_OBJ`** (`:275-278`) recurses into the lender:
`__ao2_lock(obj_lockobj->lockobj.lock, lock_how, ...)`. The borrowed pointer is
itself an ao2 object, so it goes through `__INTERNAL_OBJ_CHECK` again on the way
in and its own option bits decide what actually happens. Recursion is bounded by
whatever chain the allocator built; nothing in `internal_ao2_alloc` prevents
lending a lock from an object that is itself a `LOCK_OBJ`, so the depth is a
property of the callers rather than of the code.

**`default:`** (`:279-282`) logs `"Invalid lock option on ao2 object %p"` and
returns -1. It is unreachable, for the same reason chapter 02's teardown
`default:` is unreachable: `AO2_ALLOC_OPT_LOCK_MASK` is `(3 << 0)` and
`AO2_ALLOC_OPT_LOCK_OBJ` is *defined as* the mask
(`include/asterisk/astobj2.h:369` and `:376`), so the masked value is one of 0, 1, 2, 3
and all four have case labels. That error string appears five times in the tree,
all in `main/astobj2.c` — `:280`, `:334`, `:400` and `:465`, once in each of the
four functions this chapter covers, plus `:656` in the teardown switch of
chapter 02's `__ao2_ref`. None of the five can print.

Finally, `lockused`. It is set at `:234-236`, before the switch, so it records
"someone called a lock function on this object", not "a lock was successfully
taken" — a lockless object that is `ao2_lock`ed gets `lockused = 1` and then
reports `lock-state:none` at destruction anyway, because chapter 02's teardown
switch hardcodes `"none"` for the `NOLOCK` arm (`:643-644`). The write is guarded
on `ref_log`:

```c
	if (ref_log) {
		obj->priv_data.lockused  = 1;
	}
```

There are exactly two writers of the field, `__ao2_lock` at `:235` and
`__ao2_trylock` at `:355`, both guarded the same way, and its only readers are
the three `lock_state` assignments in the teardown switch. So `lockused` is
observable only through the `**destructor**lock-state:` field of a `refs` line,
which is only written when `ref_log` is open — the guard is redundant with the
only consumer, and gating it anyway is what keeps the field from being written at
all on a production instance.

That matters more than it looks, because of the comment on `magic`:

```c
	/*!
	 * \brief Set to 1 when the lock is used if refdebug is enabled.
	 *
	 * \note This bit-field may be modified after object creation.  It
	 *       shares a uint32_t with \ref options and \ref magic.
	 */
	uint32_t lockused:1;
	/*!
	 * \brief Magic number.
	 *
	 * This is used to verify that a pointer is a valid astobj2 or ao2_weak
	 * reference.
	 *
	 * \note This field is constant after object creation.  It shares
	 *       a uint32_t with \p options and \p lockused.
	 *
	 * \warning Stealing bits for any additional writable fields would cause
	 *          reentrancy issues if using bitfields.  If any additional
	 *          writable bits are required in the future we will need to put
	 *          all bitfields into a single 'uint32_t flags' field and use
	 *          atomic operations from \file lock.h to perform writes.
	 */
	uint32_t magic:28;
```

`options`, `lockused` and `magic` share one `uint32_t`. Setting a bit-field is a
read-modify-write of the whole word, so `lockused` is the only writable bit in a
word that also holds the magic number — and the warning at `:87-91` says
explicitly that adding a second writable bit would "cause reentrancy issues".
One writable bit is safe because every writer writes the same value, so a lost
update changes nothing; two would not be.

The `magic` comment's *"constant after object creation"* is true only up to the
last moment of the object's life. `__ao2_ref` writes `obj->priv_data.magic = 0;`
at `main/astobj2.c:624` under the comment *"In case someone uses an object after
it's been freed"*, which is the same read-modify-write of the same word that
`lockused` performs. It is safe for the opposite reason: by then the refcount has
reached zero and no other thread is entitled to hold a pointer at all.

## 3.3 `__ao2_unlock` and the inferred lock kind

```c
int __ao2_unlock(void *user_data, const char *file, const char *func, int line, const char *var)
{
	struct astobj2 *obj = __INTERNAL_OBJ_CHECK(user_data, file, line, func);
	struct astobj2_lock *obj_mutex;
	struct astobj2_rwlock *obj_rwlock;
	struct astobj2_lockobj *obj_lockobj;
	int res = 0;
	int current_value;

	if (obj == NULL) {
		return -1;
	}

	switch (obj->priv_data.options & AO2_ALLOC_OPT_LOCK_MASK) {
	case AO2_ALLOC_OPT_LOCK_MUTEX:
		obj_mutex = INTERNAL_OBJ_MUTEX(user_data);
		res = __ast_pthread_mutex_unlock(file, line, func, var, &obj_mutex->mutex.lock);
#ifdef AO2_DEBUG
		if (!res) {
			ast_atomic_fetchadd_int(&ao2.total_locked, -1);
		}
#endif
		break;
	case AO2_ALLOC_OPT_LOCK_RWLOCK:
		obj_rwlock = INTERNAL_OBJ_RWLOCK(user_data);

		current_value = ast_atomic_fetchadd_int(&obj_rwlock->rwlock.num_lockers, -1) - 1;
		if (current_value < 0) {
			/* It was a WRLOCK that we are unlocking.  Fix the count. */
			ast_atomic_fetchadd_int(&obj_rwlock->rwlock.num_lockers, -current_value);
		}
		res = __ast_rwlock_unlock(file, line, func, &obj_rwlock->rwlock.lock, var);
#ifdef AO2_DEBUG
		if (!res) {
			ast_atomic_fetchadd_int(&ao2.total_locked, -1);
		}
#endif
		break;
	case AO2_ALLOC_OPT_LOCK_NOLOCK:
		/* The ao2 object has no lock. */
		break;
	case AO2_ALLOC_OPT_LOCK_OBJ:
		obj_lockobj = INTERNAL_OBJ_LOCKOBJ(user_data);
		res = __ao2_unlock(obj_lockobj->lockobj.lock, file, func, line, var);
		break;
	default:
		ast_log(__LOG_ERROR, file, line, func, "Invalid lock option on ao2 object %p\n",
			user_data);
		res = -1;
		break;
	}
	return res;
}
```

The mutex, nolock and lockobj cases mirror `__ao2_lock`. The `default:` case
does not, quite: `__ao2_lock:279-282` and `__ao2_trylock:399-402` `return -1`
straight out of the switch, while `__ao2_unlock:333-337` sets `res = -1` and
`break`s, reaching the `return res` at the bottom. The value a caller sees is
the same; only the control flow differs, and only in a case none of the three
can reach. The rwlock case does something you will not find in the lock path:

```c
		obj_rwlock = INTERNAL_OBJ_RWLOCK(user_data);

		current_value = ast_atomic_fetchadd_int(&obj_rwlock->rwlock.num_lockers, -1) - 1;
		if (current_value < 0) {
			/* It was a WRLOCK that we are unlocking.  Fix the count. */
			ast_atomic_fetchadd_int(&obj_rwlock->rwlock.num_lockers, -current_value);
		}
		res = __ast_rwlock_unlock(file, line, func, &obj_rwlock->rwlock.lock, var);
```

`ao2_unlock` was not told whether it is releasing a read lock or a write lock,
and `pthread_rwlock_unlock` does not need to be told. But `num_lockers` does,
and the only information available is the counter itself. So the code
speculatively decrements, and if the result went negative it concludes the
caller held the write lock and adds back `-current_value` to return the counter
to 0.

Work the two cases. A writer holds it, so `num_lockers` is `-1`: the fetch-add
returns `-1`, `current_value` is `-2`, the test fires, and `-current_value` is
`+2`, which restores 0. Three readers hold it, so `num_lockers` is 3: the
fetch-add returns 3, `current_value` is 2, the test does not fire, and the
counter is left at 2. Both correct.

What makes it correct is that a pthread rwlock cannot have a writer and a reader
at once, so the sign of the counter is unambiguous. What makes it *fragile* is
that the two operations are separate atomics. Between the decrement at `:314`
and the correction at `:317` the counter briefly reads `-2`, a value that means
nothing. No reader can observe it — `__adjust_lock` is the only reader and it
requires the lock to be held, which this thread is in the middle of releasing —
but if you are ever reading `num_lockers` in a debugger, `-2` is the window.

The counter is private: `num_lockers` appears eight times in the tree, all in
`main/astobj2.c` — the declaration at `:129`, six atomic updates, and the single
plain read at `:435`. It is not exposed through any CLI command or accessor.

## 3.4 `__adjust_lock` and promotion

`__adjust_lock` is declared in `main/astobj2_private.h:64` rather than the public
header, so its callers are only the four ao2 implementation files. There are 16
call sites: 13 in `main/astobj2_container.c`, one in `main/astobj2_hash.c:169`,
one in `main/astobj2_rbtree.c:853`, and one recursive call at
`main/astobj2.c:462`. It is tempting to read all 16 as the `OBJ_NOLOCK`
machinery, and that reading is wrong in a way that matters:

| Gate | Sites |
|---|---|
| `flags & OBJ_NOLOCK` | `main/astobj2_container.c` `:113`, `:155`, `:301`, `:303`, `:393`, `:452`, `:474` |
| `flags & AO2_ITERATOR_DONTLOCK` | `main/astobj2_container.c` `:514`, `:524`, `:580`, `:582`, `:622` |
| none — unconditional in a node destructor | `main/astobj2_hash.c:169`, `main/astobj2_rbtree.c:853` |
| none — inside the retry loop of `__ao2_weakproxy_find` | `main/astobj2_container.c:467` |
| the recursive `LOCK_OBJ` arm | `main/astobj2.c:462` |

Seven of 16, then. The five iterator sites are gated on a different flag of the
same shape, and the three marked `none` are the interesting ones, because they
promote a lock on a perfectly ordinary locked path.

`__ao2_weakproxy_find` is the clearest of the three, because both paths through
it end up promoting. A caller that passes `OBJ_NOLOCK` gets the read lock
adjusted at `:452`; a caller that does not gets `ao2_rdlock(c)` at `:455`. Then
the `while` body promotes at `:467` under the comment *"Upgrade to a write
lock"*, whichever way it arrived. `OBJ_NOLOCK` changes who took the lock, not
whether it gets dropped. The restore at `:474` is worth noting too — it passes
`keep_stronger` as 1 under the comment *"We'll keep any upgraded lock"*, so a
caller that handed in a read lock can be handed back a write lock, silently and
permanently. Chapter 05 returns to that.

The node destructors say the rest themselves, at `main/astobj2_hash.c:151-163`:

```c
		/*
		 * Promote to write lock if not already there.  Since
		 * adjust_lock() can potentially release and block waiting for a
		 * write lock, care must be taken to ensure that node references
		 * are released before releasing the container references.
		 *
		 * Node references held by an iterator can only be held while
		 * the iterator also holds a reference to the container.  These
		 * node references must be unreferenced before the container can
		 * be unreferenced to ensure that the node will not get a
		 * negative reference and the destructor called twice for the
		 * same node.
		 */
```

That is the hazard stated plainly by the code that creates it: dropping the last
reference to a linked node runs `hash_ao2_node_destructor`
(`main/astobj2_hash.c:143`), which calls
`__adjust_lock(my_container, AO2_LOCK_REQ_WRLOCK, 1)` at `:169` unconditionally,
inside `if (doomed->common.is_linked)`. "Potentially release and block" means an
`ao2_ref(node, -1)` can give up the container lock. `main/astobj2_rbtree.c`
carries the same warning at `:835-847` ahead of the same call at `:853`.

```c
/*!
 * \internal
 * \brief Adjust an object's lock to the requested level.
 *
 * \param user_data An ao2 object to adjust lock level.
 * \param lock_how What level to adjust lock.
 * \param keep_stronger TRUE if keep original lock level if it is stronger.
 *
 * \pre The ao2 object is already locked.
 *
 * \details
 * An ao2 object with a RWLOCK will have its lock level adjusted
 * to the specified level if it is not already there.  An ao2
 * object with a different type of lock is not affected.
 *
 * \return Original lock level.
 */
enum ao2_lock_req __adjust_lock(void *user_data, enum ao2_lock_req lock_how, int keep_stronger)
{
	struct astobj2 *obj = INTERNAL_OBJ(user_data);
	struct astobj2_rwlock *obj_rwlock;
	struct astobj2_lockobj *obj_lockobj;
	enum ao2_lock_req orig_lock;

	switch (obj->priv_data.options & AO2_ALLOC_OPT_LOCK_MASK) {
	case AO2_ALLOC_OPT_LOCK_RWLOCK:
		obj_rwlock = INTERNAL_OBJ_RWLOCK(user_data);
		if (obj_rwlock->rwlock.num_lockers < 0) {
			orig_lock = AO2_LOCK_REQ_WRLOCK;
		} else {
			orig_lock = AO2_LOCK_REQ_RDLOCK;
		}
		switch (lock_how) {
		case AO2_LOCK_REQ_MUTEX:
			lock_how = AO2_LOCK_REQ_WRLOCK;
			/* Fall through */
		case AO2_LOCK_REQ_WRLOCK:
			if (lock_how != orig_lock) {
				/* Switch from read lock to write lock. */
				ao2_unlock(user_data);
				ao2_wrlock(user_data);
			}
			break;
		case AO2_LOCK_REQ_RDLOCK:
			if (!keep_stronger && lock_how != orig_lock) {
				/* Switch from write lock to read lock. */
				ao2_unlock(user_data);
				ao2_rdlock(user_data);
			}
			break;
		}
		break;
	case AO2_ALLOC_OPT_LOCK_OBJ:
		obj_lockobj = INTERNAL_OBJ_LOCKOBJ(user_data);
		orig_lock = __adjust_lock(obj_lockobj->lockobj.lock, lock_how, keep_stronger);
		break;
	default:
		ast_log(LOG_ERROR, "Invalid lock option on ao2 object %p\n", user_data);
		/* Fall through */
	case AO2_ALLOC_OPT_LOCK_NOLOCK:
	case AO2_ALLOC_OPT_LOCK_MUTEX:
		orig_lock = AO2_LOCK_REQ_MUTEX;
		break;
	}

	return orig_lock;
}
```

The contract is in the comment: *"\pre The ao2 object is already locked."* The
function exists because the container engine wants a write lock for operations
that mutate and a read lock for operations that only search, but when the caller
passes `OBJ_NOLOCK` the engine is not allowed to take the lock — the caller
already holds it, at whatever strength the caller chose. So the engine adjusts.

Four things in this function are worth stopping on.

**It does not validate its argument.** Line `:427` is
`struct astobj2 *obj = INTERNAL_OBJ(user_data);` — the raw macro, not
`INTERNAL_OBJ_CHECK` or `__INTERNAL_OBJ_CHECK`. `INTERNAL_OBJ` is pointer
arithmetic with no NULL test (`main/astobj2.c:163-164`), and `obj->priv_data` is
dereferenced immediately at `:432`. `__adjust_lock(NULL, ...)` reads
`(char *) NULL - sizeof(struct astobj2)`. Every other function in this chapter
checks. This one is private and every caller has already validated, so the
omission is a deliberate saving, but it means a container with a corrupted
`my_container` back-pointer faults here rather than logging.

**The original level is inferred, not recorded.** At `:435-439` the function
decides what the caller held by testing `num_lockers < 0`. The read is not
atomic, and it does not need to be: the caller holds the lock, so if it is a
write lock the counter is pinned at `-1` and nobody else can touch it, and if it
is a read lock the counter is at least 1 and other readers can only move it
further positive. The sign cannot change underneath the test.

**Promotion releases the lock.** This is the one fact in the chapter with real
consequences:

```c
		case AO2_LOCK_REQ_WRLOCK:
			if (lock_how != orig_lock) {
				/* Switch from read lock to write lock. */
				ao2_unlock(user_data);
				ao2_wrlock(user_data);
			}
			break;
```

There is no atomic upgrade in POSIX and the code does not pretend otherwise. The
object is unlocked at `:447` and relocked at `:448`, and in between any other
thread may take the write lock and change anything. A caller that did
`ao2_rdlock(container)`, verified something, and then called
`ao2_find(container, ..., OBJ_UNLINK | OBJ_NOLOCK)` has had its verification
invalidated by the promotion inside the find. The demotion path at `:451-457`
has the same gap in the other direction, which is less dangerous but not free.

The tree comments on this in two places, and they warn about different halves of
it. The node-destructor comments quoted above are about an `ao2_ref` that
silently drops the container lock. `main/astobj2_container.c:774-776` is about
the opposite mistake — a clone container that must be locked *before*
`ao2_container_dup` is called with `OBJ_NOLOCK`, "otherwise we could have errors
in `__adjust_lock`", because `__adjust_lock` on an unlocked rwlock object would
unlock a lock it does not hold.

**The `default:` arm falls through into the cases it precedes.** At `:464-470`:

```c
	default:
		ast_log(LOG_ERROR, "Invalid lock option on ao2 object %p\n", user_data);
		/* Fall through */
	case AO2_ALLOC_OPT_LOCK_NOLOCK:
	case AO2_ALLOC_OPT_LOCK_MUTEX:
		orig_lock = AO2_LOCK_REQ_MUTEX;
		break;
```

Unlike the other three functions, whose `default:` returns -1, this one logs and
then adopts the mutex answer, because `orig_lock` is returned by value and
leaving it uninitialised would be worse than guessing. The ordering — `default:`
written above the two case labels it falls into — is unusual enough to read as a
mistake and is not one.

The consequence for the 562 mutex allocation sites and the 194 lockless ones is
that `__adjust_lock` is a no-op that returns `AO2_LOCK_REQ_MUTEX`, and the
restore call `__adjust_lock(self, orig_lock, 0)` is a second no-op. All of
§3.4's hazards exist only for the 26 sites that asked for an rwlock.

## 3.5 Who actually asks for an rwlock

The option counts are worth stating precisely, because the asymmetry is the
practical takeaway of the chapter. Parsing every call to the four object
allocation macros outside the ao2 implementation and its public header, with C
comments stripped first so that doc-comment mentions of `ao2_alloc()` do not
inflate the total — 524 sites — and taking the third argument where there is
one:

| Lock option | Object allocation sites |
|---|---|
| `AO2_ALLOC_OPT_LOCK_MUTEX` | 388 (387 of them implicit, via `ao2_alloc`/`ao2_t_alloc`) |
| `AO2_ALLOC_OPT_LOCK_NOLOCK` | 133 |
| `AO2_ALLOC_OPT_LOCK_RWLOCK` | 3 |

The same parse over the six container allocation macros — 259 sites, first
argument:

| Lock option | Container allocation sites |
|---|---|
| `AO2_ALLOC_OPT_LOCK_MUTEX` | 174 |
| `AO2_ALLOC_OPT_LOCK_NOLOCK` | 61 |
| `AO2_ALLOC_OPT_LOCK_RWLOCK` | 23 |
| a variable named `opts` | 1 |

783 allocation sites, 26 rwlocks. The three plain objects are
`res/res_pjsip/pjsip_distributor.c:828`, `res/res_pjsip_logger.c:152` and
`res/stasis/messaging.c:128`; the 23 containers are spread over 15 files, with
`main/` accounting for seven of them (`stasis_channels.c`, `codec.c`,
`bucket.c`, `format_cache.c`, `format.c`, `sorcery.c`, `stasis_cache.c`), and
`main/sorcery.c` alone accounting for six of the 23. The shape is what you would
expect: read-mostly caches and registries, where the read lock buys something.

Two caveats on the number 26, both in the direction of undercounting. Wrapper
allocators that take an `enum ao2_alloc_opts` and forward it are invisible to a
parse of the ao2 macros: `ast_str_container_alloc_options`
(`main/strings.c:200`) is one, and `res/res_pjsip_maintenance.c:716` passes it
`AO2_ALLOC_OPT_LOCK_RWLOCK`, making a 24th rwlock container. And named locks
alias the option into their own enum —
`AST_NAMED_LOCK_TYPE_RWLOCK = AO2_ALLOC_OPT_LOCK_RWLOCK`
(`include/asterisk/named_locks.h:61`) — so any `ast_named_lock_get` of that type
is another rwlock object arriving by a route the census cannot see. The 26 is a
floor, not a total; what it establishes is the order of magnitude.

`AO2_ALLOC_OPT_LOCK_OBJ` has exactly one user in the tree. It is not reachable
through `ao2_alloc_options` at all — the header says so at
`include/asterisk/astobj2.h:373-374`, *"This option is used internally by
ao2_alloc_with_lockobj and should never be passed directly to ao2_alloc"* — and
`ao2_alloc_with_lockobj` is called from one place:

```c
void *ast_sorcery_lockable_alloc(size_t size, ao2_destructor_fn destructor, void *lockobj)
{
	void *object = ao2_alloc_with_lockobj(size + sizeof(struct ast_sorcery_object),
		sorcery_object_destructor, lockobj, "");
	struct ast_sorcery_object_details *details = object;

	if (!object) {
		return NULL;
	}

	details->object = object + size;
	details->object->destructor = destructor;

	return object;
}
```

`ast_sorcery_lockable_alloc` is the entire reason the fourth lock mode exists.
Sorcery objects are configuration records; a caller that wants several of them to
serialise against one another passes the same `lockobj` to each. Chapter 02
showed the other end: the teardown switch drops a reference to the lender at
`main/astobj2.c:650` with the tag `"release lockobj"`, which is the one place in
ao2 where destroying one object unrefs another.

## 3.6 Scoped locks

`SCOPED_AO2LOCK` and its two siblings are not in `astobj2.h`; they are in
`include/asterisk/lock.h`, built on the generic `SCOPED_LOCK`, which is built on
the `RAII_VAR` of chapter 02:

```c
#define SCOPED_LOCK(varname, lock, lockfunc, unlockfunc) \
	RAII_VAR(typeof((lock)), varname, ({lockfunc((lock)); (lock); }), unlockfunc)

/*!
 * \brief scoped lock specialization for mutexes
 */
#define SCOPED_MUTEX(varname, lock) SCOPED_LOCK(varname, (lock), ast_mutex_lock, ast_mutex_unlock)

/*!
 * \brief scoped lock specialization for read locks
 */
#define SCOPED_RDLOCK(varname, lock) SCOPED_LOCK(varname, (lock), ast_rwlock_rdlock, ast_rwlock_unlock)

/*!
 * \brief scoped lock specialization for write locks
 */
#define SCOPED_WRLOCK(varname, lock) SCOPED_LOCK(varname, (lock), ast_rwlock_wrlock, ast_rwlock_unlock)

/*!
 * \brief scoped lock specialization for ao2 mutexes.
 */
#define SCOPED_AO2LOCK(varname, obj) SCOPED_LOCK(varname, (obj), ao2_lock, ao2_unlock)

/*!
 * \brief scoped lock specialization for ao2 read locks.
 */
#define SCOPED_AO2RDLOCK(varname, obj) SCOPED_LOCK(varname, (obj), ao2_rdlock, ao2_unlock)

/*!
 * \brief scoped lock specialization for ao2 write locks.
 */
#define SCOPED_AO2WRLOCK(varname, obj) SCOPED_LOCK(varname, (obj), ao2_wrlock, ao2_unlock)
```

`SCOPED_LOCK` is `RAII_VAR(typeof((lock)), varname, ({lockfunc((lock)); (lock); }), unlockfunc)`
— the initialiser is a statement expression that locks and then evaluates to the
object, so the variable holds the object and the cleanup handler is the unlock
function. The return value of `lockfunc` is discarded inside the statement
expression, which means a `SCOPED_AO2LOCK` on a lockless object and a
`SCOPED_AO2LOCK` that genuinely failed are indistinguishable at the call site.

The note at `:577-581` is the one to read twice:

```c
 * \note
 * Care should be taken when using SCOPED_LOCKS in conjunction with ao2 objects.
 * ao2 objects should be unlocked before they are unreffed. Since SCOPED_LOCK runs
 * once the variable goes out of scope, this can easily lead to situations where the
 * variable gets unlocked after it is unreffed.
```

This is chapter 02's rule from `astobj2.h:449-450` — never call `ao2_ref` with
the object's lock held — colliding with scope-based cleanup. Two `RAII_VAR`s in
the same scope are destroyed in reverse order of declaration, so
`RAII_VAR(..., ao2_cleanup)` declared *after* a `SCOPED_AO2LOCK` on the same
object unrefs first and unlocks second, which is the error. Declaring them the
other way round fixes it, and nothing enforces the ordering.

Usage outside the ao2 implementation: `SCOPED_AO2LOCK` 55 sites,
`SCOPED_AO2WRLOCK` 5, `SCOPED_AO2RDLOCK` 2.

## 3.7 Reaching the raw mutex

```c
/*!
 * \brief Return the mutex lock address of an object
 *
 * \param[in] obj A pointer to the object we want.
 * \return the address of the mutex lock, else NULL.
 *
 * This function comes in handy mainly for debugging locking
 * situations, where the locking trace code reports the
 * lock address, this allows you to correlate against
 * object address, to match objects to reported locks.
 *
 * \warning AO2 lock objects do not include tracking fields when
 * DEBUG_THREADS is not enabled.
 *
 * \since 1.6.1
 */
void *ao2_object_get_lockaddr(void *obj);
```

The documentation says this function *"comes in handy mainly for debugging
locking situations"*, and that is true of exactly two of its 22 call sites
outside the ao2 implementation — both in `include/asterisk/lock.h`, where
`AO2_DEADLOCK_AVOIDANCE` (`:343`) and `CHANNEL_DEADLOCK_AVOIDANCE` (`:365`) hand
the address to `ast_find_lock_info` so that a lock released for deadlock
avoidance can be re-reported at its original acquisition site. Those two are
also the reason the documented purpose is not wrong so much as outvoted.

The other 20 do something the doc comment does not mention: 19 of them want a
`pthread_mutex_t *` to hand to a condition variable, because `ast_cond_wait`
needs the actual mutex and `ao2_lock` will not give it to you. Sixteen of the 20
are production code, four are in `tests/`; `res/res_rtp_asterisk.c` alone has
five.

The function returns NULL for every lock mode except `MUTEX` — the switch at
`main/astobj2.c:487-495` has one case and a `default: break` — so an ao2 object
used with a condition variable must be allocated with a mutex, and if someone
later changes it to `AO2_ALLOC_OPT_LOCK_RWLOCK` the result is
`ast_cond_wait(cond, NULL)`. Exactly one call site guards against this:

```c
	int remaining;
	ast_mutex_t *lock;

	if (!shutdown_group) {
		return 0;
	}

	lock = ao2_object_get_lockaddr(shutdown_group);
	ast_assert(lock != NULL);
```

`main/bridge_basic.c:3021` is the twentieth, and the only one of the 20 that is
not about a condition variable at all:

```c
	SCOPED_MUTEX(lock, ao2_object_get_lockaddr(props));
```

It locks the raw mutex with the generic `SCOPED_MUTEX` rather than
`SCOPED_AO2LOCK`, so this one acquisition bypasses `__ao2_lock` and does not set
`lockused`. That is a quirk rather than a bug, and it does not hide the object
from the `refs` log, because the same `props` is locked through the ao2 API
elsewhere in the same file — `SCOPED_AO2LOCK(lock, props)` at `:1511` and
`ao2_lock(props)` at `:1560` and `:1571` — so the bit gets set anyway. The
lesson is narrower than "`lockused` can lie": it is that `lockused` counts
*calls into `__ao2_lock`*, not acquisitions of the lock, and the raw-mutex route
is a way to take an ao2 object's lock without going through either.

## 3.8 Deadlock avoidance, as the tree does it

The single `ao2_trywrlock` call site is also the tree's clearest statement of why
a trylock exists at all:

```c
	/*
	 * We need to do deadlock avoidance between a non-scheduler thread
	 * blocking when trying to delete the scheduled entry for this
	 * callback because the scheduler thread is running this callback
	 * and this callback waiting for the cache->objects container lock
	 * that the blocked non-scheduler thread already holds.
	 */
	while (ao2_trywrlock(cache->objects)) {
		if (cache->del_expire) {
			cache->expire_id = -1;
			ao2_ref(cache, -1);
			return 0;
		}
		sched_yield();
	}
```

A scheduler-thread callback cannot afford to block on the cache's container lock,
because the thread that holds that lock may be blocked deleting this very
scheduled entry. So the callback spins with `ao2_trywrlock` and `sched_yield()`,
re-checking `cache->del_expire` each time round so it can abandon the attempt
entirely.

`cache->del_expire` is a one-bit field (`:170`) read at `:529` *without* the
container lock held, and it is not a one-way latch: both of its writers set it
and clear it a line or two apart, bracketing exactly the call that could block.

```c
	cache->del_expire = 1;
	AST_SCHED_DEL_UNREF(sched, cache->expire_id, ao2_ref(cache, -1));
	cache->del_expire = 0;
```

That is `schedule_cache_expiration` at `:675-677`; `remove_all_from_cache`
repeats it verbatim at `:580-582`. So the flag does not mean "this cache is
being torn down", it means "a thread is inside `AST_SCHED_DEL_UNREF` right now",
which is precisely the window in which this callback must give up rather than
wait. The unlocked read is sound not because the transition is one-way but
because both answers are safe: a stale 0 costs one more `sched_yield()` round,
and a stale 1 abandons an expiration run that the thread clearing the flag is
about to reschedule anyway.

This is the general shape of lock-order recovery in Asterisk: there is no lock
ranking and no deadlock detector in a production build — `DETECT_DEADLOCKS` is a
separate compile-time option — so a cycle is broken by making one participant
willing to fail and retry. The channels guide's chapter on references and
locking has the same pattern with `ast_channel_trylock`.

## 3.9 Common bugs

| Symptom | Cause |
|---|---|
| A function that nests `ao2_lock` misbehaves only on some objects | the object is `AO2_ALLOC_OPT_LOCK_RWLOCK` and `ao2_lock` took the write lock, which POSIX does not allow to nest; the same code nests fine on a mutex object (`main/astobj2.c:251-252`) |
| Nested `ao2_lock` on an rwlock object neither blocks nor protects anything | the second `pthread_rwlock_wrlock` failed and the return value was discarded, as all but a handful of `ao2_lock` call sites discard it; the body then ran unprotected and the first `ao2_unlock` released the outer acquisition (`main/astobj2.c:248-259`, `main/lock.c:1036`) |
| Data races on an object that is locked everywhere | the object was allocated `AO2_ALLOC_OPT_LOCK_NOLOCK`; `ao2_lock` returned 0 and locked nothing (`main/astobj2.c:272-274`) |
| An invariant checked under `ao2_rdlock` is false by the time it is used, inside a container call passed `OBJ_NOLOCK` | `__adjust_lock` promoted the read lock by unlocking and relocking (`main/astobj2.c:447-448`) |
| `ast_cond_wait` crashes on a NULL mutex | `ao2_object_get_lockaddr` returns NULL for every mode but `MUTEX` (`main/astobj2.c:487-495`); the object's lock option was changed |
| `ao2_rdlock` gives no read concurrency | the object is a mutex; the `MUTEX` case never reads `lock_how` (`main/astobj2.c:239-247`) |
| Unlock-after-unref crash with `SCOPED_AO2LOCK` in scope | reverse-order `RAII_VAR` destruction; the cleanup that unrefs was declared after the scoped lock (`include/asterisk/lock.h:577-581`) |
| `__adjust_lock` unlocks a lock the thread does not hold | an `OBJ_NOLOCK` container call made without first locking the container (`main/astobj2_container.c:774-776`) |
| Segfault inside `__adjust_lock` with no log line | it uses the unchecked `INTERNAL_OBJ` (`main/astobj2.c:427`); a NULL or non-ao2 pointer faults rather than logging |
| A container's read lock is gone after an `ao2_ref(node, -1)` that looked unrelated | dropping the last reference to a *linked* node runs the node destructor, which promotes the container lock unconditionally — no `OBJ_NOLOCK` involved (`main/astobj2_hash.c:147-169`, `main/astobj2_rbtree.c:835-853`) |
| A `refs` line says `lock-state:unused` on an object you believe is locked | every acquisition went through `ao2_object_get_lockaddr` and the raw lock API rather than `ao2_lock`, so `__ao2_lock` never ran (`main/astobj2.c:234-236`) |
| Nested `ao2_rdlock` on the same rwlock object deadlocks on promotion | `num_lockers` is 2, so `__adjust_lock` releases one read lock and then blocks for the write lock it cannot get (`main/astobj2.c:435-448`) |

## 3.10 Rules

| Rule | Source |
|---|---|
| `ao2_lock` means "exclusive", not "mutex"; on an rwlock object it is a write lock | `main/astobj2.c:251-252` |
| Only the `RWLOCK` mode reads `lock_how`; `MUTEX` and `NOLOCK` ignore it | `main/astobj2.c:239-271` |
| A 0 return from `ao2_lock` does not mean a lock was taken | `main/astobj2.c:272-274` |
| Choose `AO2_ALLOC_OPT_LOCK_RWLOCK` only if nothing ever locks the object recursively | `include/asterisk/astobj2.h:363-365`; `main/lock.c:121` |
| Lock the container before any container call that takes `OBJ_NOLOCK` | `main/astobj2_container.c:774-776` |
| Treat `OBJ_NOLOCK` on an rwlock container as "my lock may be dropped and retaken" | `main/astobj2.c:447-448` |
| On an rwlock container, release node references before container references — the node destructor promotes the lock | `main/astobj2_hash.c:151-163` |
| An ao2 object used with a condition variable must be allocated `MUTEX` | `main/astobj2.c:487-495`; `main/serializer_shutdown_group.c:63-64` |
| Declare a `SCOPED_AO2LOCK` after, not before, the `RAII_VAR` that holds the reference | `include/asterisk/lock.h:577-581` |
| Break a lock cycle with `ao2_trylock` and a yield, not with a lock ordering ao2 does not have | `res/res_sorcery_memory_cache.c:528-535` |

## Next

[Chapter 04](04-containers.md) moves from the object to the container:
`struct ao2_container` and the vtable behind it, the two backends that implement
it, and why a list container is a hash container with one bucket.
