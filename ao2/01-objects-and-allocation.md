# 1. Objects and Allocation

Every `file:line` citation in this guide was verified against the `24` branch.
Newer releases and other branches may differ — line numbers drift — so if you
are reading this against a different tree, treat the citations as approximate
and re-grep for the symbol name.

An ao2 object is the only thing in Asterisk that you allocate without being able
to name its type. `ao2_alloc` takes a size and returns a `void *`, and the
bookkeeping it needs lives at negative offsets from the pointer it hands back.
This chapter is about that negative space: what is in it, how the four variants
differ, and what the consequences are for anyone holding the pointer.

```
         ao2_alloc(sizeof(struct foo), foo_dtor, options)
                               │
   ┌───────────────────────────┴──────────────────────────────────┐
   │                                                              │
   │  options & AO2_ALLOC_OPT_LOCK_MASK  selects one of four      │
   │  allocations.  All four end with __priv_data immediately     │
   │  before the returned pointer; they differ only in what       │
   │  sits before __priv_data.                                    │
   │                                                              │
   │  MUTEX (0)    [ ast_mutex_t        ][ priv ][ user data ]    │
   │  RWLOCK (1)   [ ast_rwlock_t + int ][ priv ][ user data ]    │
   │  NOLOCK (2)                         [ priv ][ user data ]    │
   │  LOCK_OBJ (3) [ void *lock         ][ priv ][ user data ]    │
   │                      ▲                         ▲             │
   │                      │                         │             │
   │           sizeof() differs per mode      always the          │
   │           → four INTERNAL_OBJ_* macros   returned pointer    │
   └──────────────────────────────────────────────────────────────┘

   One allocation, one free.  The lock is not a separate object (except in
   LOCK_OBJ mode, where it is a reference to one), the header is not a
   separate object, and there is no way to get the header's address
   without knowing which of the four modes the object was built in.
```

Three facts to carry forward:

1. **`__priv_data` is 3 or 4 machine words, and the last word is three
   bit-fields sharing a `uint32_t`.** `options` is 3 bits, `lockused` is 1,
   `magic` is 28. The header comment at `main/astobj2.c:87-91` explains why no
   more writable bits may be stolen from that word, and it is worth reading
   before you consider adding one.
2. **The magic check accepts weak proxies.** `IS_AO2_MAGIC_BAD` ORs the stored
   magic with 1 before comparing (`main/astobj2.c:98`), and the two magic values
   differ only in the low bit. A pointer that passes the generic check is
   *either* an object or a weak proxy, and the code that cares distinguishes
   them by testing `magic == AO2_MAGIC` explicitly.
3. **The `default:` arm of the allocation switch is unreachable.**
   `AO2_ALLOC_OPT_LOCK_MASK` is two bits and all four of its values are named
   cases, so the "Invalid lock option requested" path at `main/astobj2.c:739-742`
   cannot be taken. The same is true of the matching arm in the destruction
   switch, which is why the `refs` log can never print `lock-state:invalid`.

## 1.1 The private header

Everything ao2 knows about an object lives here:

```c
struct __priv_data {
	ao2_destructor_fn destructor_fn;
	/*! This field is used for astobj2 and ao2_weakproxy objects to reference each other */
	void *weakptr;
#if defined(AO2_DEBUG)
	/*! User data size for stats */
	size_t data_size;
#endif
	/*! Number of references held for this object */
	int32_t ref_counter;
	/*!
	 * \brief The ao2 object option flags.
	 *
	 * \note This field is constant after object creation.  It shares
	 *       a uint32_t with \p lockused and \p magic.
	 */
	uint32_t options:3;
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
};
```

Four fields, or five under `AO2_DEBUG`. `destructor_fn` may be NULL — an object
with no destructor is legal and common for plain data. `weakptr` is doing double
duty, as its comment says: on a real object it points at the weak proxy that
tracks it, and on a weak proxy it points at the object. Chapter 06 is about that
pair.

`ref_counter` is a plain `int32_t`, not an `unsigned`. That is deliberate: the
refcount is allowed to go negative so that the error can be *detected* rather
than wrapping silently. Chapter 02 covers what happens when it does.

The last `uint32_t` is where the design gets tight:

```c
#define	AO2_MAGIC	0xa70b123
#define	AO2_WEAK	0xa70b122
#define IS_AO2_MAGIC_BAD(p) (AO2_MAGIC != (p->priv_data.magic | 1))
```

`AO2_MAGIC` is `0xa70b123` and `AO2_WEAK` is `0xa70b122` — consecutive values
differing only in bit 0. `IS_AO2_MAGIC_BAD` ORs in that bit before comparing, so
it answers "is this either kind of ao2 pointer", not "is this an object". Code
that needs the stricter answer tests the field directly; `__ao2_ref` does
exactly that at `main/astobj2.c:525` before following `weakptr`.

Note also what the magic check *cannot* do. It reads
`p->priv_data.magic` at a negative offset from a pointer the caller supplied. If
that pointer is not an ao2 object, the read is out of bounds before any
validation happens. `is_ao2_object` and `INTERNAL_OBJ_CHECK` detect a *corrupted
or already-destroyed* ao2 object reliably; they detect a `malloc`'d struct
passed by mistake only by luck.

## 1.2 Four layouts

The generic object is a header followed by a zero-length array, which is how the
user data gets an address without a type:

```c
/*!
 * What an astobj2 object looks like: fixed-size private data
 * followed by variable-size user data.
 */
struct astobj2 {
	struct __priv_data priv_data;
	void *user_data[0];
};
```

The three locking variants put their lock *before* the header, not after it, so
that the offset from the user pointer back to `priv_data` stays constant while
the offset back to the start of the allocation does not:

```c
struct ao2_lock_priv {
	ast_mutex_t lock;
};

/* AstObj2 with recursive lock. */
struct astobj2_lock {
	struct ao2_lock_priv mutex;
	struct __priv_data priv_data;
	void *user_data[0];
};

struct ao2_rwlock_priv {
	ast_rwlock_t lock;
	/*! Count of the number of threads holding a lock on this object. -1 if it is the write lock. */
	int num_lockers;
};

/* AstObj2 with RW lock. */
struct astobj2_rwlock {
	struct ao2_rwlock_priv rwlock;
	struct __priv_data priv_data;
	void *user_data[0];
};

struct ao2_lockobj_priv {
	void *lock;
};

/* AstObj2 with locking provided by a separate object. */
struct astobj2_lockobj {
	struct ao2_lockobj_priv lockobj;
	struct __priv_data priv_data;
	void *user_data[0];
};
```

`struct ao2_rwlock_priv` carries a `num_lockers` count alongside the lock, with
`-1` meaning a writer holds it. That field exists because a reader-writer lock
cannot be interrogated portably and `__adjust_lock` needs to know the current
state to promote and restore it; chapter 03 follows it.

`struct astobj2_lockobj` stores a `void *` rather than a lock. The object
borrows another ao2 object's lock, and it holds a reference on that object for
as long as it lives — taken at `main/astobj2.c:723` and released in the
destructor at `:650`.

Getting back to the header is therefore four different subtractions:

```c
#define INTERNAL_OBJ_MUTEX(user_data) \
	((struct astobj2_lock *) (((char *) (user_data)) - sizeof(struct astobj2_lock)))

#define INTERNAL_OBJ_RWLOCK(user_data) \
	((struct astobj2_rwlock *) (((char *) (user_data)) - sizeof(struct astobj2_rwlock)))

#define INTERNAL_OBJ_LOCKOBJ(user_data) \
	((struct astobj2_lockobj *) (((char *) (user_data)) - sizeof(struct astobj2_lockobj)))

#define INTERNAL_OBJ(user_data) \
	(struct astobj2 *) ((char *) user_data - sizeof(struct astobj2))
```

This is the single most important structural consequence in the subsystem.
`INTERNAL_OBJ` (the mode-independent one at `:163`) is correct for reaching
`priv_data`, because `priv_data` is always the last thing before the user data.
The three mode-specific macros are for reaching the *allocation*, and using the
wrong one yields a pointer into the middle of a block — which is why the
destruction switch in chapter 02 has to re-derive the right one from the stored
options rather than from anything the caller passed.

The wrapper that validates and converts in one step:

```c
/*!
 * \brief convert from a pointer _p to a user-defined object
 *
 * \return the pointer to the astobj2 structure
 */
#define __INTERNAL_OBJ_CHECK(user_data, file, line, func) \
	({ \
		struct astobj2 *p ## __LINE__; \
		if (!user_data \
			|| !(p ## __LINE__ = INTERNAL_OBJ(user_data)) \
			|| IS_AO2_MAGIC_BAD(p ## __LINE__)) { \
			log_bad_ao2(user_data, file, line, func); \
			p ## __LINE__ = NULL; \
		} \
		(p ## __LINE__); \
	})

#define INTERNAL_OBJ_CHECK(user_data) \
	__INTERNAL_OBJ_CHECK(user_data, __FILE__, __LINE__, __PRETTY_FUNCTION__)

/*!
 * \brief convert from a pointer _p to an astobj2 object
 *
 * \return the pointer to the user-defined portion.
 */
#define EXTERNAL_OBJ(_p)	((_p) == NULL ? NULL : (_p)->user_data)
```

`__INTERNAL_OBJ_CHECK` returns NULL and logs rather than crashing, and every
public entry point runs the pointer through it. The `p ## __LINE__` naming
(`main/astobj2.c:173`) looks like it is there to keep nested uses distinct, but
it is not: `##` suppresses macro expansion of its operands, so the local is
literally called `p__LINE__` in every file on every line. Nesting still works
only because each statement expression opens a scope and the inner declaration
shadows the outer one. Chapter 02 finds the same pattern in `ao2_bump`.

## 1.3 The option words

A caller controls three independent things with one `unsigned int`:

```c
enum ao2_alloc_opts {
	/*! The ao2 object has a recursive mutex lock associated with it. */
	AO2_ALLOC_OPT_LOCK_MUTEX = (0 << 0),
	/*! The ao2 object has a non-recursive read/write lock associated with it. */
	AO2_ALLOC_OPT_LOCK_RWLOCK = (1 << 0),
	/*! The ao2 object has no lock associated with it. */
	AO2_ALLOC_OPT_LOCK_NOLOCK = (2 << 0),
	/*! The ao2 object locking option field mask. */
	AO2_ALLOC_OPT_LOCK_MASK = (3 << 0),
	/*!
	 * \internal The ao2 object uses a separate object for locking.
	 *
	 * \note This option is used internally by ao2_alloc_with_lockobj and
	 * should never be passed directly to ao2_alloc.
	 */
	AO2_ALLOC_OPT_LOCK_OBJ = AO2_ALLOC_OPT_LOCK_MASK,
	/*! The ao2 object will not record any REF_DEBUG entries */
	AO2_ALLOC_OPT_NO_REF_DEBUG = (1 << 2),
};
```

Two bits for the lock mode, one for `NO_REF_DEBUG`, and that is the whole
vocabulary. The thing to notice is `AO2_ALLOC_OPT_LOCK_OBJ = AO2_ALLOC_OPT_LOCK_MASK`:
the fourth lock mode is the mask value itself. That is legal — the mask is
`(3 << 0)` and three is a valid two-bit value — but it means `options & MASK`
has exactly four possible results and all four are named, which is where the
dead `default:` arms come from. The header marks that value `\internal` and says
it "should never be passed directly to ao2_alloc" (`:370-375`); the supported
way in is `ao2_alloc_with_lockobj`, which supplies it for you.

`AO2_ALLOC_OPT_NO_REF_DEBUG` is the odd one out: it does not change the
allocation at all, only whether this object appears in the `refs` log. It has
two users in the tree, both of them container internals that would otherwise
drown the log — `main/astobj2_hash.c:212` for a hash bucket node and
`main/astobj2_rbtree.c:899` for an rbtree node. Chapter 07 covers the asymmetry
it creates: the two `**invalid**` paths log regardless of this flag, while
constructor, destructor and ordinary ref lines honour it.

## 1.4 The allocation path

One function serves every public entry point:

```c
static void *internal_ao2_alloc(size_t data_size, ao2_destructor_fn destructor_fn, unsigned int options,
	void *lockobj, const char *tag, const char *file, int line, const char *func)
{
	/* allocation */
	struct astobj2 *obj;
	struct astobj2_lock *obj_mutex;
	struct astobj2_rwlock *obj_rwlock;
	struct astobj2_lockobj *obj_lockobj;
	size_t overhead;

	switch (options & AO2_ALLOC_OPT_LOCK_MASK) {
	case AO2_ALLOC_OPT_LOCK_MUTEX:
		overhead = sizeof(*obj_mutex);
		obj_mutex = __ast_calloc(1, overhead + data_size, file, line, func);
		if (obj_mutex == NULL) {
			return NULL;
		}

		ast_mutex_init(&obj_mutex->mutex.lock);
		obj = (struct astobj2 *) &obj_mutex->priv_data;
		break;
	case AO2_ALLOC_OPT_LOCK_RWLOCK:
		overhead = sizeof(*obj_rwlock);
		obj_rwlock = __ast_calloc(1, overhead + data_size, file, line, func);
		if (obj_rwlock == NULL) {
			return NULL;
		}

		ast_rwlock_init(&obj_rwlock->rwlock.lock);
		obj = (struct astobj2 *) &obj_rwlock->priv_data;
		break;
	case AO2_ALLOC_OPT_LOCK_NOLOCK:
		overhead = sizeof(*obj);
		obj = __ast_calloc(1, overhead + data_size, file, line, func);
		if (obj == NULL) {
			return NULL;
		}
		break;
	case AO2_ALLOC_OPT_LOCK_OBJ:
		lockobj = ao2_t_bump(lockobj, "set lockobj");
		if (!lockobj) {
			ast_log(__LOG_ERROR, file, line, func, "AO2_ALLOC_OPT_LOCK_OBJ requires a non-NULL lockobj.\n");
			return NULL;
		}

		overhead = sizeof(*obj_lockobj);
		obj_lockobj = __ast_calloc(1, overhead + data_size, file, line, func);
		if (obj_lockobj == NULL) {
			ao2_t_ref(lockobj, -1, "release lockobj for failed alloc");
			return NULL;
		}

		obj_lockobj->lockobj.lock = lockobj;
		obj = (struct astobj2 *) &obj_lockobj->priv_data;
		break;
	default:
		/* Invalid option value. */
		ast_log(__LOG_DEBUG, file, line, func, "Invalid lock option requested\n");
		return NULL;
	}
```

Four branches, one per mode, each doing the same three things in the same order:
compute the overhead, `__ast_calloc` the overhead plus the user size as a single
block, and set `obj` to the address of the `priv_data` member inside whichever
wrapper struct it just allocated. That last step is the trick that makes the
rest of the file mode-agnostic — after the switch, `obj` is a `struct astobj2 *`
regardless of what was really allocated, and `EXTERNAL_OBJ(obj)` is the user
pointer.

Three details are worth stopping on.

The allocation is `__ast_calloc`, so **user data is zero-filled**. Code
throughout the tree relies on this: an ao2 object's pointer fields are NULL and
its counters are zero without any explicit initialization. That is a contract,
not an accident, and it is why `ao2_alloc` has no "uninitialized" variant.

`AO2_ALLOC_OPT_LOCK_OBJ` is the only branch that can fail for a reason other
than memory. It bumps the supplied lock object *first* (`:723`), so a NULL
`lockobj` is caught before anything is allocated, and if the subsequent calloc
fails it must undo that bump (`:732`) before returning. It is the only branch
with two failure exits.

The `default:` arm logs at `__LOG_DEBUG`, not `__LOG_ERROR`. Given that it is
unreachable, the log level is moot — but the asymmetry with the destruction
switch, which logs the same condition at `__LOG_ERROR` (`main/astobj2.c:655`),
is a small sign that neither arm has been exercised.

Then the common tail:

```c
	/* Initialize common ao2 values. */
	obj->priv_data.destructor_fn = destructor_fn;	/* can be NULL */
	obj->priv_data.ref_counter = 1;
	obj->priv_data.options = options;
	obj->priv_data.magic = AO2_MAGIC;

#ifdef AO2_DEBUG
	obj->priv_data.data_size = data_size;
	ast_atomic_fetchadd_int(&ao2.total_objects, 1);
	ast_atomic_fetchadd_int(&ao2.total_mem, data_size);
	ast_atomic_fetchadd_int(&ao2.total_refs, 1);
#endif

	if (ref_log && !(obj->priv_data.options & AO2_ALLOC_OPT_NO_REF_DEBUG)) {
		fprintf(ref_log, "%p,+1,%d,%s,%d,%s,**constructor**%zu**%zu**,%s\n",
			EXTERNAL_OBJ(obj), ast_get_tid(), file, line, func, overhead, data_size, tag ?: "");
		fflush(ref_log);
	}

	/* return a pointer to the user data */
	return EXTERNAL_OBJ(obj);
}
```

`ref_counter` starts at 1. There is no "allocate with zero references" form, so
the caller of `ao2_alloc` owns a reference the moment it returns, and the
canonical error path for a half-built object is `ao2_cleanup` rather than
`ast_free`.

The constructor log line records both `overhead` and `data_size` as separate
fields. Chapter 07 reads them back: they are how `refstats.py` reports
allocation by type without knowing any types.

## 1.5 The two public allocators

```c
void *__ao2_alloc(size_t data_size, ao2_destructor_fn destructor_fn, unsigned int options,
	const char *tag, const char *file, int line, const char *func)
{
	return internal_ao2_alloc(data_size, destructor_fn, options, NULL, tag, file, line, func);
}

void *__ao2_alloc_with_lockobj(size_t data_size, ao2_destructor_fn destructor_fn, void *lockobj,
	const char *tag, const char *file, int line, const char *func)
{
	return internal_ao2_alloc(data_size, destructor_fn, AO2_ALLOC_OPT_LOCK_OBJ, lockobj,
		tag, file, line, func);
}
```

`__ao2_alloc` passes the caller's options through; `__ao2_alloc_with_lockobj`
hard-codes `AO2_ALLOC_OPT_LOCK_OBJ` and threads the lock object through the
parameter that `__ao2_alloc` leaves NULL. Everything else in the public header
is a macro over one of these two, and the macros exist to capture
`__FILE__`, `__LINE__` and `__PRETTY_FUNCTION__` at the call site:

```c
#define ao2_alloc_options(data_size, destructor_fn, options) \
	__ao2_alloc((data_size), (destructor_fn), (options), NULL,  __FILE__, __LINE__, __PRETTY_FUNCTION__)

#define ao2_t_alloc(data_size, destructor_fn, debug_msg) \
	__ao2_alloc((data_size), (destructor_fn), AO2_ALLOC_OPT_LOCK_MUTEX, (debug_msg),  __FILE__, __LINE__, __PRETTY_FUNCTION__)
#define ao2_alloc(data_size, destructor_fn) \
	__ao2_alloc((data_size), (destructor_fn), AO2_ALLOC_OPT_LOCK_MUTEX, NULL,  __FILE__, __LINE__, __PRETTY_FUNCTION__)

```

Note that the two shorthand forms both hard-code `AO2_ALLOC_OPT_LOCK_MUTEX`. A
mutex is the default in the sense that it is what you get for not choosing, and
`ao2_alloc_options` is the form that lets you choose. This is the single most
common avoidable inefficiency in the subsystem: an object that is immutable
after construction, or that is only ever touched under some other lock, still
carries and initializes a full `ast_mutex_t` unless somebody passed
`AO2_ALLOC_OPT_LOCK_NOLOCK` deliberately.

Reading the options back out is a one-liner, and it is the supported way to ask
an object what it is:

```c
unsigned int ao2_options_get(void *obj)
{
	struct astobj2 *orig_obj;

	orig_obj = INTERNAL_OBJ_CHECK(obj);
	if (!orig_obj) {
		return 0;
	}
	return orig_obj->priv_data.options;
}
```

## 1.6 Common bugs

| Symptom | Cause |
|---|---|
| `ao2_ref` logs "bad magic number" on a pointer that looks fine | The pointer is not the one `ao2_alloc` returned. Interior pointers, pointers to an embedded struct, and pointers offset by a container node all fail here. The header is at a fixed negative offset from the *returned* address only. |
| Crash inside `ao2_ref` with no log line at all | A non-ao2 pointer was passed. The magic read at a negative offset happens before any validation can reject it (`main/astobj2.c:98`, reached from `:176`). Nothing in ao2 can defend against this; the caller has to not do it. |
| An object's fields are garbage immediately after `ao2_alloc` | They are not — `__ast_calloc` zero-fills (`main/astobj2.c:697`, `:707`, `:717`, `:730`). Garbage means the object is being read after destruction, or the pointer is wrong. |
| `ao2_alloc_with_lockobj` returns NULL and the log says nothing about memory | The `lockobj` argument was NULL; the error is "AO2_ALLOC_OPT_LOCK_OBJ requires a non-NULL lockobj" at `main/astobj2.c:725`, logged before any allocation is attempted. |
| A lock-object-mode object keeps another object alive forever | By design: the bump at `main/astobj2.c:723` is held until the destructor releases it at `:650`. If the lock donor is large, the borrower pins it. |
| A weak proxy passes `is_ao2_object` and then behaves strangely | It would. `IS_AO2_MAGIC_BAD` accepts both magic values (`main/astobj2.c:98`). Test `magic == AO2_MAGIC` if you need to exclude proxies, as `__ao2_ref` does at `:525`. |
| Memory use is higher than the sum of the structs | Every object pays `sizeof(struct astobj2_lock)` unless it was allocated `NOLOCK`, and `ao2_alloc` without `_options` always asks for a mutex (`include/asterisk/astobj2.h:408-410`). |

## 1.7 Rules

| Rule | Why |
|---|---|
| Pass around only the pointer `ao2_alloc` returned | It is the only address from which the header offset is defined (`main/astobj2.c:163-164`). |
| Choose the lock mode deliberately; `ao2_alloc` means mutex | The two-argument macros hard-code `AO2_ALLOC_OPT_LOCK_MUTEX` (`include/asterisk/astobj2.h:408-410`); `NOLOCK` is free and correct for immutable or externally-locked data. |
| Rely on zero-initialization rather than writing it out | `__ast_calloc` guarantees it for the user portion, and the rest of the tree assumes it. |
| Release a failed half-built object with `ao2_cleanup`, never `ast_free` | The allocation began before `priv_data`, so `ast_free` on the user pointer frees the wrong address, and the destructor and lock teardown are skipped. |
| Do not add a writable bit-field to `__priv_data` | The three fields share one `uint32_t`; a fourth writable one needs atomics, as the warning at `main/astobj2.c:87-91` spells out. |
| Treat `ao2_options_get` as the only way to ask about an object's mode | The options are inside the private header and are constant after construction (`main/astobj2.c:781-790`). |

## Next

[Chapter 02](02-refcounting-and-destruction.md) follows a reference from
`ao2_bump` to the destructor, and explains why the destructor runs with no lock
held.
