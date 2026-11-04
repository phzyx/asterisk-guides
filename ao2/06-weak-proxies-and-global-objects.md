# 06. Weak proxies and global objects

Two features share this chapter because they solve the same problem from
opposite ends. A weak proxy lets a container hold a *stand-in* for an object so
that the container's reference does not keep the object alive. A global object
holder lets exactly one reference live in a static variable, reachable from
anywhere, replaceable at runtime. Both exist because a plain ao2 reference is
unconditionally strong: chapter 02's rule is that the object dies when the last
reference goes, and neither a container nor a global variable has any way to
say "hold this, but don't vote".

The two are also wildly different in how much they are used. In the whole
`24` tree, with the eight astobj2 implementation files excluded and comments
stripped, `AO2_WEAKPROXY()` appears **7** times across **6** files — and one of
those files is the unit test. Production code that uses a weak proxy at all
amounts to five files: `main/named_locks.c`, `main/sorcery.c`, `main/stasis.c`,
`main/stasis_state.c` and `channels/chan_websocket.c`. `AO2_GLOBAL_OBJ_STATIC`,
by contrast, declares **43** holders in **32** files, read through **218**
`ao2_global_obj_ref` call sites.

So one of these is a niche mechanism with exactly one usage idiom, repeated
verbatim five times. The other is everywhere. Both are worth knowing, for
different reasons.

## 6.1 The field both objects share

Chapter 01 walked `struct __priv_data`, the hidden header in front of every ao2
object, and skipped one field. Here it is:

```c
 */
struct __priv_data {
	ao2_destructor_fn destructor_fn;
	/*! This field is used for astobj2 and ao2_weakproxy objects to reference each other */
	void *weakptr;
#if defined(AO2_DEBUG)
```

One `void *`, and the comment is the entire design: it is used *by both sides*
to point at the other. On a normal object, `weakptr` points at its proxy. On a
proxy, `weakptr` points at the real object. There is no separate weak-reference
type, no table, no registry — a weak proxy is an ordinary ao2 object with a
different magic number and the same single pointer, pointing the other way.

The two magic numbers are adjacent on purpose:

```c
#define	AO2_MAGIC	0xa70b123
#define	AO2_WEAK	0xa70b122
#define IS_AO2_MAGIC_BAD(p) (AO2_MAGIC != (p->priv_data.magic | 1))
```

`0xa70b123` and `0xa70b122` differ only in bit 0, and `IS_AO2_MAGIC_BAD` ORs
that bit back in before comparing. This is chapter 01's "one check accepts weak
proxies": every generic sanity check in the subsystem passes for both kinds,
while the eight places that care which kind they have compare
`priv_data.magic` against `AO2_MAGIC` or `AO2_WEAK` exactly.

The public surface of a proxy is one struct and one macro:

```c
/*! \brief ao2_weakproxy
 *
 * @{
 */
struct ao2_weakproxy_notification;
typedef void (*ao2_weakproxy_notification_cb)(void *weakproxy, void *data);

/*! \brief This struct should be opaque, but it's size is needed. */
struct ao2_weakproxy {
	AST_LIST_HEAD_NOLOCK(, ao2_weakproxy_notification) destroyed_cb;
};

/*! \brief Macro which must be used at the beginning of weakproxy capable objects.
 *
 * \note The primary purpose of user defined fields on weakproxy objects is to hold
 *       immutable container keys for the real object.
 */
#define AO2_WEAKPROXY() struct ao2_weakproxy __weakproxy##__LINE__

```

`struct ao2_weakproxy` is a single list head, and the comment says what the
list is for. `AO2_WEAKPROXY()` is a declaration macro, not a flag: it places
that struct first in the user's own structure, which is what makes a
`struct named_lock_proxy *` safe to pass to every `ao2_weakproxy_*` function
through a `void *`. The `##__LINE__` suffix does nothing: `##` suppresses macro
expansion of its operands, so the member is literally named `__weakproxy__LINE__`
in every file on every line, exactly as chapter 02 found for `__obj___LINE__`.
Using the macro twice in one struct would not compile.

Read the note on that macro again, because it is the whole usage pattern:
*"The primary purpose of user defined fields on weakproxy objects is to hold
immutable container keys for the real object."* The proxy is the thing the
container indexes. Section 6.7 shows what happens when authors take that
further than "keys".

## 6.2 "Weak" is implemented as a reference cycle plus a tripwire

The natural way to build a weak reference is an uncounted pointer. Asterisk
does the opposite. `__ao2_weakproxy_set_object` takes a reference on *both*
objects:

```c
int __ao2_weakproxy_set_object(void *weakproxy, void *obj, int flags,
	const char *tag, const char *file, int line, const char *func)
{
	struct astobj2 *weakproxy_internal = __INTERNAL_OBJ_CHECK(weakproxy, file, line, func);
	struct astobj2 *obj_internal = __INTERNAL_OBJ_CHECK(obj, file, line, func);
	int ret = -1;

	if (!weakproxy_internal
		|| weakproxy_internal->priv_data.magic != AO2_WEAK) {
		return -1;
	}

	if (!obj_internal
		|| obj_internal->priv_data.weakptr
		|| obj_internal->priv_data.magic != AO2_MAGIC) {
		return -1;
	}

	if (!(flags & OBJ_NOLOCK)) {
		ao2_lock(weakproxy);
	}

	if (!weakproxy_internal->priv_data.weakptr) {
		__ao2_ref(obj, +1, tag, file, line, func);
		__ao2_ref(weakproxy, +1, tag, file, line, func);

		weakproxy_internal->priv_data.weakptr = obj;
		obj_internal->priv_data.weakptr = weakproxy;

		ret = 0;
	}

	if (!(flags & OBJ_NOLOCK)) {
		ao2_unlock(weakproxy);
		/* It is possible for obj to be accessed now.  It's allowed
		 * for weakproxy to already be in a container.  Another thread
		 * could have been waiting for a lock on weakproxy to retrieve
		 * the object.
		 */
	}

```

The two `__ao2_ref(..., +1)` calls at `:841-842` create a deliberate cycle: the
proxy holds a reference to the object, the object holds a reference to the
proxy, and the two `weakptr` assignments at `:844-845` wire them together. Left
alone, that cycle would make both objects immortal.

It is broken by a tripwire in `__ao2_ref`. On the way in, every *decrement* of a
real object with a proxy attached grabs the proxy's lock first:

```c
	}

	if (delta < 0 && obj->priv_data.magic == AO2_MAGIC && (weakproxy = obj->priv_data.weakptr)) {
		ao2_lock(weakproxy);
	}

	/* we modify with an atomic operation the reference counter */
	ret = ast_atomic_fetch_add(&obj->priv_data.ref_counter, delta, __ATOMIC_RELAXED);
	current_value = ret + delta;
```

Three conditions, all necessary. `delta < 0` means increments skip this
entirely. `magic == AO2_MAGIC` means the check fires on the real object and
never on the proxy — unrefs of the proxy itself take the ordinary path. And the
assignment-in-condition means `weakproxy` is left NULL for the overwhelming
majority of objects, which have no proxy, so the cost to everything else is one
load and one branch.

Then, after the atomic:

```c
	if (weakproxy) {
		struct ao2_weakproxy cbs;

		if (current_value == 1) {
			/* The only remaining reference is the one owned by the weak object */
			struct astobj2 *internal_weakproxy;

			internal_weakproxy = INTERNAL_OBJ_CHECK(weakproxy);

			/* Unlink the obj from the weak proxy */
			internal_weakproxy->priv_data.weakptr = NULL;
			obj->priv_data.weakptr = NULL;

			/* transfer list to local copy so callbacks are run with weakproxy unlocked. */
			cbs.destroyed_cb = weakproxy->destroyed_cb;
			AST_LIST_HEAD_INIT_NOLOCK(&weakproxy->destroyed_cb);

			/* weak is already unlinked from obj so this won't recurse */
			ao2_ref(user_data, -1);
		}

		ao2_unlock(weakproxy);

		if (current_value == 1) {
			struct ao2_weakproxy_notification *destroyed_cb;

			/* Notify the subscribers that weakproxy now points to NULL. */
			while ((destroyed_cb = AST_LIST_REMOVE_HEAD(&cbs.destroyed_cb, list))) {
				destroyed_cb->cb(weakproxy, destroyed_cb->data);
				ast_free(destroyed_cb);
			}

			ao2_ref(weakproxy, -1);
		}
	}
```

`current_value == 1` is the tripwire. The object's refcount reaching exactly one
means every reference *except the proxy's* has been released — that is what the
comment at `:541` says — so the object is logically dead even though its count
is not zero. The code then, in order: NULLs both halves of the cycle
(`:546-548`), moves the subscriber list to a stack copy and reinitialises the
real one (`:551-552`), and drops the proxy's reference with
`ao2_ref(user_data, -1)` at `:555`. That last call takes the object to zero and
runs its destructor. The comment explains why it does not recurse: `weakptr` is
already NULL, so the re-entered `__ao2_ref` fails the `:525` test.

Only then does it unlock the proxy, run the callbacks with the proxy unlocked
(`:561-568`), and drop the object's reference to the proxy at `:569`. One
caveat on "unlocked": the teardown releases the lock *it* took at `:526`, and
ao2 mutexes are recursive, so an outer frame that already holds the proxy lock
still holds it across the callbacks. The only function that can do that is
`__ao2_weakproxy_ref_object`, which locks at `:875` before calling
`__ao2_ref` at `:879` — and that function has no production callers.

Two consequences fall straight out of that ordering, and both matter for
section 6.7:

* The object is **already destroyed** by the time the callbacks run.
  `destroyed_cb->cb(weakproxy, ...)` receives the proxy, never the object, and
  `ao2_weakproxy_get_object` on it will return NULL.
* The proxy is **still alive** while the callbacks run, because the object's
  reference to it is not released until `:569`, after the loop. A callback can
  safely read the proxy's own fields.

### A read-after-free on the ref-log path

There is a defect in this sequence. When the tripwire fires, `current_value` is
1, and nothing in the `if (weakproxy)` block changes it. So after the block,
execution reaches the "object still lives" branch:

```c
	if (0 < current_value) {
		/* The object still lives. */
#define EXCESSIVE_REF_COUNT		100000
```

That branch is correct for the refcount — but the object was freed at `:555`.
Everything in the branch except one line uses only locals, so with ref logging
off nothing happens. The exception is the ref-log write:

```c
		if (ref_log && !(obj->priv_data.options & AO2_ALLOC_OPT_NO_REF_DEBUG)) {
			fprintf(ref_log, "%p,%s%d,%d,%s,%d,%s,%d,%s\n", user_data,
				(delta < 0 ? "" : "+"), delta, ast_get_tid(),
				file, line, func, (int)ret, tag ?: "");
			fflush(ref_log);
		}
```

`obj->priv_data.options` at `:589` reads through a pointer into memory that
`ast_free` released inside the nested call. The condition is `ref_log && …`,
and as chapter 02 established, `ref_log` is opened at `main/astobj2.c:1173-1175`
under the *runtime* `refdebug = yes` in `asterisk.conf`, not under the
`REF_DEBUG` compile flag — so no special build is needed, only a restart. On a
run with ref logging on, it occurs every single time a proxied object is
destroyed, which is every stasis topic, every named lock and every sorcery
instance. Worth noting that the nested frame guards against exactly this: the
comment at `:625` says the options are saved locally "so the ref_log print at
the end doesn't access freed data". That fix protects the inner frame and not
the outer one. Chapter 07 covers the ref log in detail; the thing to know here is
that the weak-proxy teardown path is the one place where the ref log is read
from freed memory, so an unexplained corrupt `refs` line for a proxied object
is a known artefact rather than a new bug.

## 6.3 Reading through a proxy

Three functions read a proxy. The one that matters is `get_object`:

```c
void *__ao2_weakproxy_get_object(void *weakproxy, int flags,
	const char *tag, const char *file, int line, const char *func)
{
	struct astobj2 *internal = __INTERNAL_OBJ_CHECK(weakproxy, file, line, func);
	void *obj;

	if (!internal || internal->priv_data.magic != AO2_WEAK) {
		/* This method is meant to be run on weakproxy objects! */
		return NULL;
	}

	/* We have a weak object, grab reference to object within lock */
	if (!(flags & OBJ_NOLOCK)) {
		ao2_lock(weakproxy);
	}

	obj = internal->priv_data.weakptr;
	if (obj) {
		__ao2_ref(obj, +1, tag, file, line, func);
	}

	if (!(flags & OBJ_NOLOCK)) {
		ao2_unlock(weakproxy);
	}

	return obj;
```

The `+1` at `:907` is the whole point, and it happens **inside the proxy's
lock**. Section 6.2 established that the object is destroyed while that same
lock is held (`:526` takes it, `:555` frees the object, `:558` releases it). So
the two cannot interleave: either `get_object` wins the lock and returns a
reference that is now strong enough to keep the object alive, or the teardown
wins and `weakptr` is NULL by the time `get_object` reads it. There is no window
in which a caller can be handed a pointer to a dying object. That mutual
exclusion is the reason the proxy is allocated `AO2_ALLOC_OPT_LOCK_MUTEX`
unconditionally in `__ao2_weakproxy_alloc` — the caller does not get to choose:

```c
void *__ao2_weakproxy_alloc(size_t data_size, ao2_destructor_fn destructor_fn,
	const char *tag, const char *file, int line, const char *func)
{
	struct ao2_weakproxy *weakproxy;

	if (data_size < sizeof(*weakproxy)) {
		ast_assert(0);
		ast_log(LOG_ERROR, "Requested data_size smaller than minimum.\n");
		return NULL;
	}

	weakproxy = __ao2_alloc(data_size, destructor_fn, AO2_ALLOC_OPT_LOCK_MUTEX,
		tag, file, line, func);

	if (weakproxy) {
		struct astobj2 *weakproxy_internal;

		/* Just created weakproxy, no need to check if it's valid. */
		weakproxy_internal = INTERNAL_OBJ(weakproxy);
		weakproxy_internal->priv_data.magic = AO2_WEAK;
	}

	return weakproxy;
}
```

Note also the `data_size` floor at `:798-802`: the proxy's user data must be at
least `sizeof(struct ao2_weakproxy)`, because the subsystem will write a list
head at offset zero. This is the runtime half of the convention that
`AO2_WEAKPROXY()` expresses — the macro only declares a member; nothing checks
that it is present or first — and it is why the header says the
struct "should be opaque, but its size is needed."

`ref_object` is the same shape with the `+1` replaced by an arbitrary delta:

```c
int __ao2_weakproxy_ref_object(void *weakproxy, int delta, int flags,
	const char *tag, const char *file, int line, const char *func)
{
	struct astobj2 *internal = __INTERNAL_OBJ_CHECK(weakproxy, file, line, func);
	int ret = -1;

	if (!internal || internal->priv_data.magic != AO2_WEAK) {
		/* This method is meant to be run on weakproxy objects! */
		return -2;
	}

	/* We have a weak object, grab lock. */
	if (!(flags & OBJ_NOLOCK)) {
		ao2_lock(weakproxy);
	}

	if (internal->priv_data.weakptr) {
		ret = __ao2_ref(internal->priv_data.weakptr, delta, tag, file, line, func);
	}

	if (!(flags & OBJ_NOLOCK)) {
		ao2_unlock(weakproxy);
	}

```

Its return value is three-way — `-2` for "not a proxy", `-1` for "no object",
and otherwise the refcount *before* the operation. A caller that treats any
negative result as failure cannot distinguish the two error cases, and a caller
that ignores the sign cannot distinguish `-1` from a successful operation on an
object whose count was... well, `-1` is never a valid prior count, so that one
is safe. All three call sites in the tree are in `tests/test_astobj2_weaken.c`
(`:224`, `:230`, `:242`); the API has no production callers at all.

The reverse direction gets you the proxy from the object:

```c
void *__ao2_get_weakproxy(void *obj, const char *tag, const char *file, int line, const char *func)
{
	struct astobj2 *obj_internal = __INTERNAL_OBJ_CHECK(obj, file, line, func);

	if (!obj_internal || obj_internal->priv_data.magic != AO2_MAGIC) {
		/* This method is meant to be run on normal ao2 objects! */
		return NULL;
	}

	if (!obj_internal->priv_data.weakptr) {
		return NULL;
	}

	__ao2_ref(obj_internal->priv_data.weakptr, +1, tag, file, line, func);
	return obj_internal->priv_data.weakptr;
}
```

This one takes no lock at all, and it does not need to: `obj_internal->weakptr`
can only be cleared by the teardown at `:547-548`, and the teardown only runs
when the object's count has already fallen to one. A caller holding a reference
to `obj` is holding one of those counted references, so the count cannot be one
while they are calling, so the field cannot be changing. The `+1` at `:930` is
on the proxy. Outside the unit test, nothing in the tree calls this.

## 6.4 Subscriptions

A subscription is a `cb`/`data` pair on a singly-linked list hanging off the
proxy:

```c
struct ao2_weakproxy_notification {
	ao2_weakproxy_notification_cb cb;
	void *data;
	AST_LIST_ENTRY(ao2_weakproxy_notification) list;
};
```

```c
int ao2_weakproxy_subscribe(void *weakproxy, ao2_weakproxy_notification_cb cb, void *data, int flags)
{
	struct astobj2 *weakproxy_internal = INTERNAL_OBJ_CHECK(weakproxy);
	int ret = -1;
	int hasobj;

	if (!weakproxy_internal || weakproxy_internal->priv_data.magic != AO2_WEAK) {
		return -1;
	}

	if (!(flags & OBJ_NOLOCK)) {
		ao2_lock(weakproxy);
	}

	hasobj = weakproxy_internal->priv_data.weakptr != NULL;
	if (hasobj) {
		struct ao2_weakproxy *weak = weakproxy;
		struct ao2_weakproxy_notification *sub = ast_calloc(1, sizeof(*sub));

		if (sub) {
			sub->cb = cb;
			sub->data = data;
			AST_LIST_INSERT_HEAD(&weak->destroyed_cb, sub, list);
			ret = 0;
		}
	}

	if (!(flags & OBJ_NOLOCK)) {
		ao2_unlock(weakproxy);
	}

	if (!hasobj) {
		cb(weakproxy, data);
		ret = 0;
	}

	return ret;
}
```

The structurally interesting part is `hasobj`, read at `:948` under the lock and
acted on at `:965` *outside* it. If the proxy still has an object, a node is
allocated and inserted at the head (`:951-957`). If it does not — the object is
already gone — nothing is recorded and the callback is invoked immediately at
`:966`, after the unlock. The header documents this (`:650-651`), and it has two
practical implications: a callback may run on the subscriber's own thread before
`ao2_weakproxy_subscribe` returns, and a successful return does not mean a
subscription exists.

`AST_LIST_INSERT_HEAD` is also why the header promises callbacks run "in the
reverse order of subscriptions" (`:642`).

Failure is reported as `-1` for both "not a proxy" and "`ast_calloc` failed",
which all five production callers handle identically by tearing down the
half-built proxy. Note the asymmetry that creates: on the immediate-callback
path the function returns 0, so a caller cannot tell "subscribed" from "already
dead, and I just ran your callback."

Unsubscribing walks the same list:

```c
int ao2_weakproxy_unsubscribe(void *weakproxy, ao2_weakproxy_notification_cb destroyed_cb, void *data, int flags)
{
	struct astobj2 *internal_weakproxy = INTERNAL_OBJ_CHECK(weakproxy);
	struct ao2_weakproxy *weak;
	struct ao2_weakproxy_notification *sub;
	int ret = 0;

	if (!internal_weakproxy || internal_weakproxy->priv_data.magic != AO2_WEAK || !destroyed_cb) {
		return -1;
	}

	if (!(flags & OBJ_NOLOCK)) {
		ao2_lock(weakproxy);
	}

	weak = weakproxy;
	AST_LIST_TRAVERSE_SAFE_BEGIN(&weak->destroyed_cb, sub, list) {
		if (sub->cb == destroyed_cb && sub->data == data) {
			AST_LIST_REMOVE_CURRENT(list);
			ast_free(sub);
			ret++;
			if (!(flags & OBJ_MULTIPLE)) {
				break;
			}
		}
	}
	AST_LIST_TRAVERSE_SAFE_END;

	if (!(flags & OBJ_NOLOCK)) {
		ao2_unlock(weakproxy);
	}

```

Two details worth keeping. First, `OBJ_MULTIPLE` keeps only the core of its
chapter 05 meaning — "do not stop at the first match" — and none of the
machinery: there is no result iterator here, just "remove every matching pair".
Without it the loop `break`s after one removal (`:994-995`), which
matters because `subscribe` deliberately allows the same pair to be added
repeatedly. Second, the return value counts removals and is `-1` only for bad
parameters, so 0 is "found nothing" — and the header's note at `:674-677`
explains why that is not conclusive: the teardown moves the whole list to a
stack copy before running callbacks, so between `:551` and `:565` in
`__ao2_ref` the list is legitimately empty while the callbacks have not yet
run. Outside the unit test, nothing in the tree unsubscribes at all; every
production subscription lasts the proxy's lifetime.

## 6.5 `ao2_weakproxy_find`

A container full of proxies needs a find that returns real objects. That is a
separate entry point, in the container file rather than with the rest:

```c
void *__ao2_weakproxy_find(struct ao2_container *c, const void *arg, enum search_flags flags,
	const char *tag, const char *file, int line, const char *func)
{
	void *proxy;
	void *obj = NULL;
	enum ao2_lock_req orig_lock;

	ast_assert(!!c);
	ast_assert(flags & OBJ_SEARCH_MASK);
	ast_assert(!(flags & ~(OBJ_SEARCH_MASK | OBJ_NOLOCK)));

	if (flags & OBJ_NOLOCK) {
		orig_lock = __adjust_lock(c, AO2_LOCK_REQ_RDLOCK, 1);
	} else {
		orig_lock = AO2_LOCK_REQ_RDLOCK;
		ao2_rdlock(c);
	}

	while ((proxy = ao2_find(c, arg, flags | OBJ_NOLOCK))) {
		obj = __ao2_weakproxy_get_object(proxy, 0, tag ?: __PRETTY_FUNCTION__, file, line, func);

		if (obj) {
			ao2_ref(proxy, -1);
			break;
		}

		/* Upgrade to a write lock */
		__adjust_lock(c, AO2_LOCK_REQ_WRLOCK, 1);
		ao2_unlink_flags(c, proxy, OBJ_NOLOCK);
		ao2_ref(proxy, -1);
	}

	if (flags & OBJ_NOLOCK) {
		/* We'll keep any upgraded lock */
		__adjust_lock(c, orig_lock, 1);
	} else {
		ao2_unlock(c);
	}

	return obj;
}
```

The three asserts at `:447-449` are a tight contract: at least one search-mode
bit, and no flag other than `OBJ_NOLOCK`. Note the middle assert tests
`flags & OBJ_SEARCH_MASK`, so it catches zero search bits but not two, and
`ast_assert` is a no-op outside a dev-mode build. `OBJ_UNLINK`, `OBJ_MULTIPLE` and
`OBJ_NODATA` are all rejected — this function finds one object and returns one
reference, and chapter 05's whole flag vocabulary does not apply.

The `while` loop at `:458` is the interesting part. `ao2_find` matches against
the *proxy*, so `c->cmp_fn` and `c->hash_fn` must be written for the proxy's
fields — which is why the macro's note calls the proxy's user fields
"immutable container keys". If the matched proxy still has an object, the loop
takes the reference, drops the proxy and breaks (`:459-463`). If it does not,
the proxy is stale: the loop promotes the container to a write lock, unlinks the
proxy, drops it, and searches again (`:466-469`).

So `ao2_weakproxy_find` prunes as it searches, and it is the second mechanism
for removing stale proxies — the subscription callbacks of section 6.7 are the
first. It has to be, because a subscription callback is not guaranteed to have
run yet: the teardown drops the object's reference and releases the proxy lock
before invoking callbacks, so there is a real window in which the container
still holds a proxy whose object is gone.

The lock handling is the `keep_stronger` case chapter 05 deferred. Under
`OBJ_NOLOCK` the function promotes with `__adjust_lock(c, …, 1)` at `:452` and
restores with `keep_stronger = 1` at `:474` — the only site in the subsystem
that restores with the flag set. The comment at `:473` is explicit: *"We'll keep
any upgraded lock."* A caller who held a read lock and passed `OBJ_NOLOCK` can
therefore come back holding a write lock, with no indication that it happened.

## 6.6 Bulk conversion

One more helper converts a container of proxies into a container of objects:

```c
static int dup_weakproxy_cb(void *proxy, void *arg, int flags)
{
	void *obj = ao2_weakproxy_get_object(proxy, 0);
	struct ao2_container *dest = arg;
	int ret;

	if (!obj) {
		return 0;
	}

	ret = ao2_link_flags(dest, obj, OBJ_NOLOCK) ? 0 : (CMP_MATCH | CMP_STOP);
	ao2_ref(obj, -1);

	return ret;
}
```

```c
int ao2_container_dup_weakproxy_objs(struct ao2_container *dest, struct ao2_container *src, enum search_flags flags)
{
	void *obj;
	int res = 0;

	if (!(flags & OBJ_NOLOCK)) {
		ao2_rdlock(src);
		ao2_wrlock(dest);
	}
	obj = ao2_callback(src, OBJ_NOLOCK, dup_weakproxy_cb, dest);
	if (obj) {
		/* Failed to put this obj into the dest container. */
		ao2_t_ref(obj, -1, "Failed to put this object into the dest container.");

		/* Remove all items from the dest container. */
		ao2_callback(dest, OBJ_NOLOCK | OBJ_UNLINK | OBJ_NODATA | OBJ_MULTIPLE, NULL,
			NULL);
		res = -1;
	}
	if (!(flags & OBJ_NOLOCK)) {
		ao2_unlock(dest);
		ao2_unlock(src);
	}

	return res;
```

`dup_weakproxy_cb` returns `0` for a stale proxy at `:716-718`, which in
chapter 05's vocabulary means "no match, keep going" — stale entries are
skipped silently, exactly as the header documents. It signals *failure* by
returning `CMP_MATCH | CMP_STOP` (`:720`), so a non-NULL return from the
`ao2_callback` at `:735` is an error indication rather than a result, and the
rollback at `:740-742` empties the destination.

The header's note on this function is at best ambiguous:

```c
 *
 * \note Every object inside the container is locked by \ref ao2_weakproxy_get_object.
 *       Any weakproxy in \p src with no associated object is ignored.
```

Nothing locks the objects. `ao2_weakproxy_get_object(proxy, 0)` locks the
*proxy* — the object is only ref-bumped. The distinction matters because a
reader who believes the objects are locked might assume this function
serialises against concurrent mutation of them, and it does not.

Nothing in the tree outside `tests/test_astobj2_weaken.c` calls
`ao2_container_dup_weakproxy_objs`.

## 6.7 What the five production users actually do

All five do the same five things in the same order, which makes the idiom worth
learning once. `main/named_locks.c` is the smallest and clearest, and it is a
useful read in its own right because the "real" object is empty:

```c
struct named_lock_proxy {
	AO2_WEAKPROXY();
	char key[0];
};

struct ast_named_lock {
};

AO2_STRING_FIELD_HASH_FN(named_lock_proxy, key)
AO2_STRING_FIELD_CMP_FN(named_lock_proxy, key)
```

`struct ast_named_lock` has no members at all. The object *is* its ao2 lock;
the proxy carries the key, and the hash and compare functions are generated for
the proxy. That is the division of labour in its purest form: the container
indexes the proxy by key, and the object it stands for carries no key of its
own.

```c
	/* We have exclusive access to proxy and lock, no need for locking here. */
	if (ao2_weakproxy_set_object(proxy, lock, OBJ_NOLOCK)) {
		goto failure_cleanup;
	}

	if (ao2_weakproxy_subscribe(proxy, named_lock_proxy_cb, NULL, OBJ_NOLOCK)) {
		goto failure_cleanup;
	}

	strcpy(proxy->key, concat_key); /* Safe */
	ao2_link_flags(named_locks, proxy, OBJ_NOLOCK);
	ao2_unlock(named_locks);
	ao2_t_ref(proxy, -1, "Release allocation reference");

```

The steps, in order: allocate the proxy (`:90`), allocate the real object
(`:95`), `ao2_weakproxy_set_object` with `OBJ_NOLOCK` because nothing else can
see either object yet (`:101`), `ao2_weakproxy_subscribe` (`:105`), write the
key and link the *proxy* — not the object — into the container (`:109-110`),
then drop the allocation reference on the proxy (`:112`). After that the proxy
has exactly two references, one from the container and one from the object, and
the object has exactly two — the caller's allocation reference, which it
returns at `:114`, and the proxy's, taken at `main/astobj2.c:841`. That second
one is the whole mechanism: when the caller releases the lock, the count falls
to 1 and the tripwire fires.

The subscription callback is the other half:

```c
static void named_lock_proxy_cb(void *weakproxy, void *data)
{
	ao2_unlink(named_locks, weakproxy);
}
```

That is the mechanism in one line. When the last caller releases the named
lock, the tripwire in `__ao2_ref` destroys it, then runs this callback, which
unlinks the proxy. The container's reference goes, the object's reference went
at `:569`, and the proxy is freed too. The entry cleans itself up with no
reaping pass, no timer, and no explicit "release the named lock" API.

`main/sorcery.c` (proxy struct at `:225`, callback at `:660-663`, construction
at `:681-700`) and `channels/chan_websocket.c` (`:1438`, `:1443-1448`,
`:1467-1542`) are the same steps with a different key. The two stasis files
reorder the first two: `main/stasis_state.c` allocates the real object at
`:183` and the proxy at `:216`, and in `main/stasis.c` the topic is created by
the caller and arrives as a parameter, so `topic_proxy_create` never allocates
it at all. Everything from `set_object` onward is identical in all five. Three
files add something.

**`main/stasis.c` passes a reference as callback data.** Its proxy is a topic
key plus some metadata:

```c
struct topic_proxy {
	AO2_WEAKPROXY();

	char *name;
	char *detail;

	struct timeval creationtime;

	char buf[0];
};

```

and its callback needs the container, which is not a global the callback can
reach, so the subscription carries one:

```c
static void proxy_dtor(void *weakproxy, void *container)
{
	ao2_unlink(container, weakproxy);
	ao2_cleanup(container);
}
```

```c

	/* We have exclusive access to proxy, no need for locking here. */
	if (ao2_t_weakproxy_set_object(proxy, topic, OBJ_NOLOCK, "weakproxy link")) {
		ao2_cleanup(proxy);
		ao2_unlock(topic_all);

		return -1;
	}

	if (ao2_weakproxy_subscribe(proxy, proxy_dtor, ao2_bump(topic_all), OBJ_NOLOCK)) {
		ao2_cleanup(proxy);
		ao2_unlock(topic_all);
		ao2_cleanup(topic_all);

		return -1;
	}

	/* setting the topic point to the proxy */
	topic->name = proxy->name;
	topic->detail = proxy->detail;
	topic->creationtime = &(proxy->creationtime);

	ao2_link_flags(topic_all, proxy, OBJ_NOLOCK);
	ao2_ref(proxy, -1);

	ao2_unlock(topic_all);
```

`ao2_bump(topic_all)` at `:614` takes a reference *for the subscription*, and
`proxy_dtor` releases it at `:482` after unlinking. The header's note that
"it is the caller's responsibility to ensure that `*data` is valid until after
`cb()` is run" (`:647-648`) is being satisfied by refcounting rather than by
lifetime reasoning, which is the right way to satisfy it. The error path at
`:615-619` has to undo that bump by hand, which is why there are two
`ao2_cleanup` calls there and one on the path above it.

**Three of the five let the object borrow the proxy's storage.** At `:623-625`
above, the topic's `name` and `detail` are pointed into the proxy's trailing
`buf[0]` and its `creationtime` at the proxy's own `struct timeval` member
(`:470`). `main/sorcery.c` does the same for one field — `sorcery->module_name
= proxy->module_name` at `:692`, with the struct comment at `:238` saying so
outright. And `stasis_state.c` does it and says so in a comment:

```c

	state->id = proxy->id;
	proxy->manager = ao2_bump(manager);
	state->manager = proxy->manager; /* state->manager is owned by the proxy */

	state->forward = stasis_forward_all(state->topic, manager->all_topic);
```

```c
struct stasis_state_proxy {
	AO2_WEAKPROXY();
	/*! The manager that owns and handles this state */
	struct stasis_state_manager *manager;
	/*! A unique id for this state object. */
	char id[0];
};
```

`state->id` points at `proxy->id`, and `state->manager` is a reference the
*proxy* owns, released by the proxy's own destructor:

```c
static void state_proxy_dtor(void *obj) {
	struct stasis_state_proxy *proxy = obj;

	ao2_cleanup(proxy->manager);
}

static void state_proxy_sub_cb(void *obj, void *data)
{
	struct stasis_state_proxy *proxy = obj;

	ao2_unlink(proxy->manager->states, proxy);
}
```

This is safe, and the reason is the ordering from section 6.2: the object holds
a reference to its proxy, and that reference is not released until `:569` in
`__ao2_ref` — after the object's destructor has run. So the proxy, and
everything stored in it, outlives the object it stands for. `stasis_state.c` is
also the only one of the five to give its proxy a destructor at all; the other
four pass NULL.

It is worth being clear about what this costs. `stasis_topic_name(topic)`
returns a pointer into the proxy's buffer. That is fine for a caller holding a
topic reference and wrong for anyone who caches the pointer past it. The design
is deliberate — it avoids a second allocation and keeps the key in exactly one
place, so the container's key and the object's name can never disagree — but it
means the proxy is not merely an index entry. It is where some of the object's
immutable state lives.

## 6.8 Global object holders

Now the other end of the problem. A holder is a two-field struct:

```c
/*! Global ao2 object holder structure. */
struct ao2_global_obj {
	/*! Access lock to the held ao2 object. */
	ast_rwlock_t lock;
	/*! Global ao2 object. */
	void *obj;
};
```

An rwlock and a `void *`. Declaring one is a macro with two variants:

```c
 */
#ifndef HAVE_PTHREAD_RWLOCK_INITIALIZER
#define AO2_GLOBAL_OBJ_STATIC(name)										\
	struct ao2_global_obj name;											\
	static void  __attribute__((constructor)) __init_##name(void)		\
	{																	\
		ast_rwlock_init(&name.lock);									\
		name.obj = NULL;												\
	}																	\
	static void  __attribute__((destructor)) __fini_##name(void)		\
	{																	\
		if (name.obj) {													\
			ao2_ref(name.obj, -1);										\
			name.obj = NULL;											\
		}																\
		ast_rwlock_destroy(&name.lock);									\
	}																	\
	struct __dummy_##name
#else
#define AO2_GLOBAL_OBJ_STATIC(name)										\
	struct ao2_global_obj name = {										\
		.lock = AST_RWLOCK_INIT_VALUE,									\
	}
#endif
```

On a platform without `PTHREAD_RWLOCK_INITIALIZER` the macro emits a
constructor to initialise the lock and a destructor to release the object and
destroy the lock, then a `struct __dummy_##name` declaration whose only job is
to consume the semicolon the caller writes. On a platform that has the
initialiser — which includes every Linux build, since `configure` defines
`HAVE_PTHREAD_RWLOCK_INITIALIZER` on glibc; the tracked template carries it as
`#undef` at `include/asterisk/autoconfig.h.in:812` and the generated header
defines it to 1 — the macro collapses to the `#else` body at
`include/asterisk/astobj2.h:847-850`: a statically initialised struct with
**no destructor**. So on the platform
Asterisk is actually built for, releasing the held object at module unload is
entirely manual.

The tree is disciplined about it: all 43 holders in all 32 files are matched by
an `ao2_global_obj_release` or an `ao2_global_obj_replace` in the same file.
But nothing enforces that, and the enforcement that the macro *appears* to
provide only exists on the branch most builds do not compile.

One other wrinkle in the macro: it is named `AO2_GLOBAL_OBJ_STATIC`, and it does
not emit `static`. The caller has to. Thirty-eight of the 43 uses write
`static AO2_GLOBAL_OBJ_STATIC(name);`. Five do not —
`main/features_config.c:712`, `res/res_pjsip_authenticator_digest.c:45`,
`res/res_pjsip_mwi.c:46`, `:47` and `res/res_pjsip_notify.c:268` — which gives
those holders external linkage, and two of the five are both named `globals`.
In practice nothing collides, for two independent reasons: modules are dlopened
`RTLD_NOW | RTLD_LOCAL` unless they declare `AST_MODFLAG_GLOBAL_SYMBOLS`
(`main/loader.c:1169-1178`), and under GNU ld the build passes
`--version-script` for both the core binary and every module
(`main/Makefile:119`, `Makefile.rules:120`, each inside an
`ifeq ($(GNU_LD),1)`), which localises anything not explicitly exported. The
symbols really are local in the linked artefacts. It is a latent problem rather
than a live one, but `static` is still the correct thing to write.

## 6.9 Why a global object is read through a function

The whole implementation is 97 lines, about 65 of them code:

```c
void *__ao2_global_obj_ref(struct ao2_global_obj *holder, const char *tag, const char *file, int line, const char *func, const char *name)
{
	void *obj;

	if (!holder) {
		/* For sanity */
		ast_log(LOG_ERROR, "Must be called with a global object!\n");
		ast_assert(0);
		return NULL;
	}

	if (__ast_rwlock_rdlock(file, line, func, &holder->lock, name)) {
		/* Could not get the read lock. */
		ast_assert(0);
		return NULL;
	}

	obj = holder->obj;
	if (obj) {
		__ao2_ref(obj, +1, tag, file, line, func);
	}

	__ast_rwlock_unlock(file, line, func, &holder->lock, name);

	return obj;
}
```

The read bumps the refcount *inside the read lock*. That is the entire reason
this is a function call and not `holder.obj`. Reading the pointer directly and
bumping it afterwards is a use-after-free: between the load and the increment
another thread can call `ao2_global_obj_replace_unref`, which drops the
holder's reference, which may be the last one. Taking the read lock first makes
the pointer stable for long enough to make it strong. The reference the caller
gets back cannot be revoked by a later replace — it is theirs until they drop
it, and a replace simply means the *next* reader gets a different object.

That is what makes the dominant read idiom correct:

```c
struct ast_ha *ast_named_acl_find(const char *name, int *is_realtime, int *is_undefined)
{
	struct ast_ha *ha = NULL;

	RAII_VAR(struct named_acl_config *, cfg, ao2_global_obj_ref(globals), ao2_cleanup);
	RAII_VAR(struct named_acl *, named_acl, NULL, ao2_cleanup);
```

96 of the 218 `ao2_global_obj_ref` call sites are inside a `RAII_VAR` like this
one. The pattern is: take a reference on entry, use it for the whole function
without caring whether a reload happened, release it on exit. A function that
calls `ao2_global_obj_ref` twice can legitimately get two *different* objects,
so the convention is to call it once and hold the result.

Two caveats on the read. `__ao2_global_obj_ref` returns NULL both for an empty
holder and for a failed `rdlock` (`:83-87`), so a NULL return is not
conclusively "nothing installed" — though the failure path asserts first. And
the `holder == NULL` check at `:76-81` cannot fire from the public macros, which
all pass `&holder` of a real struct; it is there for direct callers of the
underscore function, of which there are none.

Writing is two functions:

```c
void *__ao2_global_obj_replace(struct ao2_global_obj *holder, void *obj, const char *tag, const char *file, int line, const char *func, const char *name)
{
	void *obj_old;

	if (!holder) {
		/* For sanity */
		ast_log(LOG_ERROR, "Must be called with a global object!\n");
		ast_assert(0);
		return NULL;
	}
	if (__ast_rwlock_wrlock(file, line, func, &holder->lock, name)) {
		/* Could not get the write lock. */
		ast_assert(0);
		return NULL;
	}

	if (obj) {
		__ao2_ref(obj, +1, tag, file, line, func);
	}
	obj_old = holder->obj;
	holder->obj = obj;

	__ast_rwlock_unlock(file, line, func, &holder->lock, name);

	return obj_old;
}
```

```c
int __ao2_global_obj_replace_unref(struct ao2_global_obj *holder, void *obj, const char *tag, const char *file, int line, const char *func, const char *name)
{
	void *obj_old;

	obj_old = __ao2_global_obj_replace(holder, obj, tag, file, line, func, name);
	if (obj_old) {
		__ao2_ref(obj_old, -1, tag, file, line, func);
		return 1;
	}
	return 0;
}
```

`__ao2_global_obj_replace` takes the write lock, bumps the incoming object,
swaps, and **returns the old object with the holder's reference still on it**.
The caller inherits that reference and owes an unref.
`__ao2_global_obj_replace_unref` is a one-line wrapper that drops it for you and
returns 0 or 1 depending on whether the holder had been empty.

The usage split is lopsided and instructive. `replace_unref` has **36** uses
across **25** files, counting the three `ao2_t_` sites
(`main/manager.c:9852`, `res/res_pjsip/config_global.c:241`,
`tests/test_astobj2.c:899`); the ref-returning `replace` has **4**, of which two are in the
unit test. Both production uses do the same thing — atomically take the object
*out* of the holder so it can be torn down outside the lock:

```c
		unsolicited_mwi = ao2_global_obj_replace(mwi_unsolicited, NULL);
		if (unsolicited_mwi) {
			ao2_callback(unsolicited_mwi, OBJ_UNLINK | OBJ_NODATA | OBJ_MULTIPLE, unsubscribe, NULL);
			ao2_ref(unsolicited_mwi, -1);
		}

```

```c
	/* Destroy agent holding bridge. */
	holding = ao2_global_obj_replace(agent_holding, NULL);
	if (holding) {
		ast_bridge_destroy(holding, 0);
	}

```

So in practice `ao2_global_obj_replace` is not "swap in a new object" at all —
it is "steal the object", and both callers pass NULL. If you are installing a
new configuration on reload, `ao2_global_obj_replace_unref` is the one you
want; `ao2_global_obj_release(holder)` is the same call with NULL, via a macro.

One loose end: `replace_unref` returns whether the holder had been empty, and
**no production caller checks it.** The only site in the tree that tests the
return value is `tests/test_astobj2.c:899`. The return value is, for all
practical purposes, documentation of an invariant nobody asserts.

## 6.10 Symptom → cause

| Symptom | Likely cause |
|---|---|
| A container keeps growing and entries are never reaped | The proxies' subscription callback is not unlinking, or `ao2_weakproxy_subscribe` returned without subscribing because the object was already gone (`main/astobj2.c:948`, `:965-967`) |
| `ao2_weakproxy_find` returns NULL for a key you just linked | You linked the object instead of the proxy, or the container's `hash_fn`/`cmp_fn` are written for the object's fields rather than the proxy's (`main/astobj2_container.c:458`) |
| `ao2_weakproxy_set_object` returns -1 on a fresh object | The object already has a `weakptr`, or its magic is not `AO2_MAGIC` — most often the "object" is itself a proxy (`main/astobj2.c:830-833`) |
| `ao2_weakproxy_set_object` returns -1 on a reused proxy | The proxy still has an object set; the only way to clear it is destruction (`main/astobj2.c:840`, header `:573-577`) |
| `ao2_weakproxy_alloc` logs "Requested data_size smaller than minimum" | The user struct does not begin with `AO2_WEAKPROXY()` and is smaller than `struct ao2_weakproxy` (`main/astobj2.c:798-802`) |
| A destroy callback sees `ao2_weakproxy_get_object` return NULL | Correct and unavoidable: the object is freed at `main/astobj2.c:555`, before callbacks run at `:565` |
| A destroy callback runs on the subscribing thread, before `subscribe` returns | The proxy had no object at subscribe time, so the callback was invoked immediately (`main/astobj2.c:965-967`) |
| A caller that passed `OBJ_NOLOCK` with a read lock finds it holding a write lock | `ao2_weakproxy_find` restores with `keep_stronger = 1` and keeps any promotion (`main/astobj2_container.c:474`) |
| Corrupt or impossible `refs` log line when a proxied object dies | The ref-log write at `main/astobj2.c:589` reads `obj->priv_data.options` after the object was freed at `:555`; gated on runtime `refdebug = yes`, not on a build flag |
| A string borrowed from a topic or state goes bad | It points into the proxy's storage (`main/stasis.c:623-625`, `main/stasis_state.c:223`, `main/sorcery.c:692`); it is valid only while you hold a reference to the object |
| A held global object leaks at module unload | `AO2_GLOBAL_OBJ_STATIC` emits no destructor on platforms with `PTHREAD_RWLOCK_INITIALIZER`, which is all Linux builds; the release is manual (`include/asterisk/astobj2.h:847-850`, `include/asterisk/autoconfig.h.in:812`) |
| Two reads of the same holder in one function disagree | A replace landed in between; each `ao2_global_obj_ref` is an independent snapshot (`main/astobj2_global.c:89-92`) |
| A replaced object is freed while another thread is still using it | That thread read `holder.obj` directly instead of calling `ao2_global_obj_ref`, so it never took a reference (`main/astobj2_global.c:83-92`) |
| `ao2_global_obj_replace` leaks the old object | It returns the holder's reference; the caller owes an unref. Use `ao2_global_obj_replace_unref` unless you want the object (`main/astobj2_global.c:52-57`) |

## 6.11 Rules

| Rule | Evidence |
|---|---|
| Put `AO2_WEAKPROXY()` first in the proxy struct, and keep container keys there | `include/asterisk/astobj2.h:533-538`, `main/astobj2.c:798-802` |
| Link the proxy into the container, never the real object | `main/named_locks.c:110`, `main/stasis.c:627`, `main/astobj2_container.c:458` |
| Write the container's `hash_fn` and `cmp_fn` against the proxy's fields | `main/named_locks.c:44-45`, `main/stasis.c:475-477` |
| Call `ao2_weakproxy_set_object` before anything else can reach either object, and `OBJ_NOLOCK` is then safe | header `:570-571`, `main/named_locks.c:100-101` |
| Subscribe a callback that unlinks the proxy, or the container will accumulate stale entries | `main/named_locks.c:65-68`, `main/sorcery.c:660-663` |
| Treat a 0 return from `ao2_weakproxy_subscribe` as "either subscribed or already called" | `main/astobj2.c:957`, `:965-967` |
| Assume a destroy callback can run before `subscribe` returns, on your thread, and with the proxy unlocked | `main/astobj2.c:962-967` |
| Never touch the real object from a destroy callback; it is already freed | `main/astobj2.c:555` then `:565` |
| Keep a reference on anything you pass as callback `data` | header `:647-648`; `main/stasis.c:614` bumps, `:482` releases |
| Read through a proxy only with `ao2_weakproxy_get_object` or `ao2_weakproxy_find`; both return a reference you must drop | `main/astobj2.c:907`, `main/astobj2_container.c:458-461` |
| Pass only one `OBJ_SEARCH_*` bit and at most `OBJ_NOLOCK` to `ao2_weakproxy_find` | `main/astobj2_container.c:447-449` |
| Expect `ao2_weakproxy_find` with `OBJ_NOLOCK` to leave your read lock promoted to a write lock | `main/astobj2_container.c:467`, `:473-474` |
| If the object borrows storage from its proxy, document it; it is only safe because the object's proxy reference outlives its destructor | `main/stasis_state.c:225`, `main/astobj2.c:555` vs `:569` |
| Write `static AO2_GLOBAL_OBJ_STATIC(name);` — the macro does not supply the `static` its name implies | `include/asterisk/astobj2.h:830-831`, and the five sites that omit it |
| Release every holder explicitly at unload; no destructor runs on Linux builds | `include/asterisk/astobj2.h:847-850`, `include/asterisk/autoconfig.h.in:812` |
| Read a holder once per function and hold the reference, ideally with `RAII_VAR` | `main/named_acl.c:306`, `main/astobj2_global.c:89-92` |
| Never read `holder.obj` directly | `main/astobj2_global.c:83-92` |
| Use `ao2_global_obj_replace_unref` to install; use `ao2_global_obj_replace` only to steal the object out | `main/astobj2_global.c:60-70`; `res/res_pjsip_mwi.c:1545`, `apps/app_agent_pool.c:2668` |

## Next

Chapter 07 is the last one: `REF_DEBUG` and the two `refs`
log line formats, `refcounter.py` and `refstats.py`, what `AO2_DEBUG` turns on
and the CLI commands it adds, the container integrity checks, and the
compile-time traversal-state budget chapter 05 measured.
