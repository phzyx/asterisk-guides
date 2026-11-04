# 5. Traversal and iterators

**Target tree:** Asterisk, the `24` branch

Chapter 04 built the container and never took anything out of it. This chapter
is the other half: the two ways a caller gets at the objects inside, and the
reference bookkeeping that makes each of them safe.

The two ways look unrelated in the API and are nearly the same underneath.
One is a single call that sweeps the container and hands back what it found —
`ao2_find` (477 call sites, plus 10 tagged), `ao2_callback` (291, plus 24
tagged) and `ao2_callback_data` (36, plus 9 tagged), all three of which are
thin wrappers over one 179-line static function. The other is a cursor the
caller advances itself — `ao2_iterator_init` (303 sites) and
`ao2_iterator_next` (304, plus 23 tagged). Both funnel into the same four
vtable slots that chapter 04 left undescribed, and both rest on a single
invariant: **the backend returns container *nodes*, and a node is an ao2 object
with its own refcount that is not the object you asked for.**

Almost every bug in this area is a confusion between those two refcounts.

```
   ao2_find / ao2_callback / ao2_callback_data
            │
            ▼
   internal_ao2_traverse  (astobj2_container.c:230)
            │  takes the container lock, allocs 100 bytes of
            │  traversal state on the stack, then:
            │
            ├──→ v_table->traverse_first(self, flags, arg, state)   → node (+1)
            ├──→ v_table->traverse_next(self, state, node)          → node (+1)
            └──→ v_table->traverse_cleanup(state)                   (never filled)

   ao2_iterator_init / ao2_iterator_next
            │
            ▼
   __ao2_iterator_next  (astobj2_container.c:556)
            │  takes the container lock, keeps the state in the
            │  caller's struct ao2_iterator instead, then:
            │
            └──→ v_table->iterator_next(self, last_node, flags)     → node (NOT reffed)

   The asymmetry on the right-hand side is real and is section 5.7.
```

## 5.1 `enum search_flags`, field by field

`enum search_flags` (`include/asterisk/astobj2.h:1034-1142`) is three
independent bits and two bit-fields packed into one `int`. The three bits come
first:

```c
enum search_flags {
	/*!
	 * Unlink the object for which the callback function returned
	 * CMP_MATCH.
	 */
	OBJ_UNLINK = (1 << 0),
	/*!
	 * On match, don't return the object hence do not increase its
	 * refcount.
	 */
	OBJ_NODATA = (1 << 1),
	/*!
	 * Don't stop at the first match in ao2_callback() unless the
	 * result of the callback function has the CMP_STOP bit set.
	 */
	OBJ_MULTIPLE = (1 << 2),
```

`OBJ_UNLINK` (163 uses) removes the matched object. `OBJ_NODATA` (285) says
"don't give it back to me". `OBJ_MULTIPLE` (153) says "don't stop at the first
match". Note what `OBJ_MULTIPLE`'s comment is careful to say: the default stop
is overridden *unless the callback set `CMP_STOP`*, so a callback can still cut
a multi-match sweep short.

Bit 3 is a hole. The next flag is at bit 4:

```c
	/*!
	 * \brief Assume that the ao2_container is already locked.
	 *
	 * \note For ao2_containers that have mutexes, no locking will
	 * be done.
	 *
	 * \note For ao2_containers that have RWLOCKs, the lock will be
	 * promoted to write mode as needed.  The lock will be returned
	 * to the original locked state.
	 *
	 * \note Only use this flag if the ao2_container is manually
	 * locked already.
	 */
	OBJ_NOLOCK = (1 << 4),
```

`OBJ_NOLOCK` (156 uses) is the subject of section 5.6, and its comment is the
most important four lines in the enum — "the lock will be promoted to write
mode as needed" is doing a great deal of work.

Then the search field, which is where the design shows its age:

```c
	/*!
	 * \brief Search option field mask.
	 *
	 * \todo Eventually OBJ_SEARCH_MASK will shrink to a two bit
	 * field when the codebase is made to use the search field
	 * values as a field instead of independent bits.
	 */
	OBJ_SEARCH_MASK = (0x07 << 5),
```

A three-bit mask holding four values encoded one-hot: `OBJ_SEARCH_NONE` is
`(0 << 5)`, `OBJ_SEARCH_OBJECT` is `(1 << 5)`, `OBJ_SEARCH_KEY` is `(2 << 5)`
and `OBJ_SEARCH_PARTIAL_KEY` is `(4 << 5)` (`:1074`, `:1087`, `:1101`,
`:1116`). Three bits for four states wastes four encodings, and the `\todo`
explains why nobody has fixed it: the values are *used as independent bits* at
some call sites, so shrinking the field to two bits would break them. Every
backend reads this field with `flags & OBJ_SEARCH_MASK` and `switch`es on the
result, which is the shape the `\todo` is asking the tree to adopt everywhere.

`OBJ_SEARCH_NONE` has zero uses outside astobj2 itself. It is the default, and
the default is spelled by omission.

Last is the order field:

```c
	/*! \brief Traverse order option field mask. */
	OBJ_ORDER_MASK = (0x03 << 8),
	/*! \brief Traverse in ascending order (First to last container object) */
	OBJ_ORDER_ASCENDING = (0 << 8),
	/*! \brief Traverse in descending order (Last to first container object) */
	OBJ_ORDER_DESCENDING = (1 << 8),
```

Two bits, four values, properly encoded as a field this time:
`OBJ_ORDER_ASCENDING`, `OBJ_ORDER_DESCENDING`, `OBJ_ORDER_PRE` (`:1132`) and
`OBJ_ORDER_POST` (`:1141`). The documentation for `PRE` and `POST` is honest
about them being tree concepts that a hash container has to reinterpret: for
hash and list containers pre-order means ascending and post-order means
descending.

**The entire order field has zero uses outside `main/astobj2_*.c` and
`tests/test_astobj2.c`.** Not one caller in the rest of the tree asks for a
traversal order. The one in-tree consumer is astobj2 forcing `OBJ_ORDER_POST`
on itself during container destruction, which is section 5.13.

And then there are the old names, which are not a small footnote:

```c
/*
 * Deprecated backward compatible flag names.
 *
 * Note: OBJ_POINTER, OBJ_KEY, and OBJ_PARTIAL_KEY are mutually
 * exclusive.
 */
#define OBJ_POINTER		OBJ_SEARCH_OBJECT		/*!< Deprecated name */
#define OBJ_KEY			OBJ_SEARCH_KEY			/*!< Deprecated name */
#define OBJ_PARTIAL_KEY	OBJ_SEARCH_PARTIAL_KEY	/*!< Deprecated name */
```

Counted across the tree with the eight astobj2 implementation files excluded
and comments stripped, the deprecated spellings are used **468** times
(`OBJ_POINTER` 115, `OBJ_KEY` 275, `OBJ_PARTIAL_KEY` 78) against **490** for
the modern ones (`OBJ_SEARCH_OBJECT` 101, `OBJ_SEARCH_KEY` 330,
`OBJ_SEARCH_PARTIAL_KEY` 59). Fifty-eight files still use a deprecated name;
`apps/confbridge/conf_config_parser.c` uses all three a combined 46 times
(`OBJ_KEY` 22, `OBJ_POINTER` 12, `OBJ_PARTIAL_KEY` 12). Reading this
subsystem means reading both vocabularies, and `OBJ_POINTER` in particular is
worth internalising as "the arg is an object" rather than anything to do with
pointers.

The callback's half of the contract is two bits:

```c
enum _cb_results {
	CMP_MATCH	= 0x1,	/*!< the object matches the request */
	CMP_STOP	= 0x2,	/*!< stop the search now */
};
```

## 5.2 The generic traversal engine

`ao2_callback`, `ao2_callback_data` and `ao2_find` are all one function.
`__ao2_callback` and `__ao2_callback_data` differ only in the `data` and
`type` arguments they pass — `NULL, AO2_CALLBACK_DEFAULT` against
`data, AO2_CALLBACK_WITH_DATA` (`main/astobj2_container.c:414`, `:421`), and `__ao2_find` is
`__ao2_callback` with the container's own `cmp_fn` substituted for the caller's
(`:437`) — which is the whole difference between `find` and `callback`.

Everything happens in `internal_ao2_traverse`. It opens with the object and
vtable checks chapter 04 catalogued:

```c
static void *internal_ao2_traverse(struct ao2_container *self, enum search_flags flags,
	void *cb_fn, void *arg, void *data, enum ao2_callback_type type,
	const char *tag, const char *file, int line, const char *func)
{
	void *ret;
	ao2_callback_fn *cb_default = NULL;
	ao2_callback_data_fn *cb_withdata = NULL;
	struct ao2_container_node *node;
	void *traversal_state;

	enum ao2_lock_req orig_lock;
	struct ao2_container *multi_container = NULL;
	struct ao2_iterator *multi_iterator = NULL;

	if (!__is_ao2_object(self, file, line, func)) {
		return NULL;
	}

	if (!self->v_table
		|| !self->v_table->traverse_first || !self->v_table->traverse_next) {
		/* Sanity checks. */
		__ast_assert_failed(0, "invalid container v_table", file, line, func);
		return NULL;
	}
```

Note the return type. `void *` is not laziness: this function returns either a
user object or a `struct ao2_iterator *`, depending on the flags, and C has no
way to say that. Section 5.5 is about the consequences.

Next, the result container, if the flags call for one:

```c
	/*
	 * This logic is used so we can support OBJ_MULTIPLE with OBJ_NODATA
	 * turned off.  This if statement checks for the special condition
	 * where multiple items may need to be returned.
	 */
	if ((flags & (OBJ_MULTIPLE | OBJ_NODATA)) == OBJ_MULTIPLE) {
		/* we need to return an ao2_iterator with the results,
		 * as there could be more than one. the iterator will
		 * hold the only reference to a container that has all the
		 * matching objects linked into it, so when the iterator
		 * is destroyed, the container will be automatically
		 * destroyed as well.
		 */
		multi_container = ao2_t_container_alloc_list(AO2_ALLOC_OPT_LOCK_NOLOCK, 0, NULL,
			NULL, "OBJ_MULTIPLE return container creation");
		if (!multi_container) {
			return NULL;
		}
		if (!(multi_iterator = ast_calloc(1, sizeof(*multi_iterator)))) {
			ao2_t_ref(multi_container, -1, "OBJ_MULTIPLE iterator creation failed.");
			return NULL;
		}
	}
```

Then the callback is resolved — a NULL `cb_fn` becomes `cb_true` or
`cb_true_data`, which match everything (`:193-204`) — and the container is
locked:

```c
	/* avoid modifications to the content */
	if (flags & OBJ_NOLOCK) {
		if (flags & OBJ_UNLINK) {
			orig_lock = __adjust_lock(self, AO2_LOCK_REQ_WRLOCK, 1);
		} else {
			orig_lock = __adjust_lock(self, AO2_LOCK_REQ_RDLOCK, 1);
		}
	} else {
		orig_lock = AO2_LOCK_REQ_MUTEX;
		if (flags & OBJ_UNLINK) {
			ao2_wrlock(self);
		} else {
			ao2_rdlock(self);
		}
	}

	/* Create a buffer for the traversal state. */
	traversal_state = alloca(AO2_TRAVERSAL_STATE_SIZE);
```

Two things to carry forward. The lock taken depends on `OBJ_UNLINK`: a
traversal that might remove something takes a write lock, one that cannot takes
a read lock. And `traversal_state` is `alloca`'d, 100 bytes per traversal, on
the stack of whichever thread happened to call `ao2_find`. Chapter 04 explained
the compile-time assert that keeps the backends inside that budget; this is the
allocation it is guarding.

The loop itself is driven entirely by two vtable slots:

```c
	for (node = self->v_table->traverse_first(self, flags, arg, traversal_state);
		node;
		node = self->v_table->traverse_next(self, traversal_state, node)) {
		int match;

		/* Visit the current node. */
		match = (CMP_MATCH | CMP_STOP);
		if (type == AO2_CALLBACK_WITH_DATA) {
			match &= cb_withdata(node->obj, arg, data, flags);
		} else {
			match &= cb_default(node->obj, arg, flags);
		}
		if (match == 0) {
			/* no match, no stop, continue */
			continue;
		}
		if (match == CMP_STOP) {
			/* no match but stop, we are done */
			break;
		}
```

The `match = (CMP_MATCH | CMP_STOP); match &= cb(...)` idiom is a filter, not an
assignment: any bit a callback returns other than those two is discarded
silently. A callback that returns `-1` on error will be read as
`CMP_MATCH | CMP_STOP`, which is "found it, stop now" — the opposite of what
the author meant. Callbacks return only `0`, `CMP_MATCH`, `CMP_STOP`, or the
two ORed.

## 5.3 Who owns the returned reference

This is the body of the match case, and it is the most consequential twenty
lines in the subsystem:

```c
		if (node->obj) {
			/* The object is still in the container. */
			if (!(flags & OBJ_NODATA)) {
				/*
				 * We are returning the object, record the value.  It is
				 * important to handle this case before the unlink.
				 */
				if (multi_container) {
					/*
					 * Link the object into the container that will hold the
					 * results.
					 */
					__ao2_link(multi_container, node->obj, flags, tag, file, line, func);
				} else {
					ret = node->obj;
					/* Returning a single object. */
					if (!(flags & OBJ_UNLINK)) {
						/*
						 * Bump the ref count since we are not going to unlink and
						 * transfer the container's object ref to the returned object.
						 */
						__ao2_ref(ret, 1, tag ?: "Traversal found object", file, line, func);
					}
				}
			}

			if (flags & OBJ_UNLINK) {
				int ulflag = AO2_UNLINK_NODE_UNREF_NODE | AO2_UNLINK_NODE_DEC_COUNT;
				if (multi_container || (flags & OBJ_NODATA)) {
					ulflag |= AO2_UNLINK_NODE_UNLINK_OBJECT;
				}
				__container_unlink_node_debug(node, ulflag, tag, file, line, func);
			}
		}
```

Read the `if (!(flags & OBJ_UNLINK))` at `:360` carefully. On the no-unlink
path the engine takes a *new* reference for the caller at `:365`; the
container's own reference stays where it was. On the unlink path it takes
nothing — the comment at `:361-364` says why — and the container's reference
*becomes* the caller's. One object, two routes, the same obligation at the end:
the caller owes exactly one `ao2_ref(obj, -1)`.

The transfer is completed by the flags chosen at `:370-375`.
`AO2_UNLINK_NODE_UNLINK_OBJECT` is added **only** when the object is going into
a result container or is being discarded, never when it is being returned
singly. Here is what that bit controls:

```c
int __container_unlink_node_debug(struct ao2_container_node *node, uint32_t flags,
	const char *tag, const char *file, int line, const char *func)
{
	struct ao2_container *container = node->my_container;

	if (container == NULL && (flags & AO2_UNLINK_NODE_DEC_COUNT)) {
		return 0;
	}

	if ((flags & AO2_UNLINK_NODE_UNLINK_OBJECT)
		&& !(flags & AO2_UNLINK_NODE_NOUNREF_OBJECT)) {
		__ao2_ref(node->obj, -1, tag ?: "Remove obj from container", file, line, func);
	}

	node->obj = NULL;

	if (flags & AO2_UNLINK_NODE_DEC_COUNT) {
		ast_atomic_fetchadd_int(&container->elements, -1);
#if defined(AO2_DEBUG)
		{
			int empty = container->nodes - container->elements;

			if (container->max_empty_nodes < empty) {
				container->max_empty_nodes = empty;
			}
			if (container->v_table->unlink_stat) {
				container->v_table->unlink_stat(container, node);
			}
		}
#endif	/* defined(AO2_DEBUG) */
	}

	if (flags & AO2_UNLINK_NODE_UNREF_NODE) {
		/* Remove node from container */
		ao2_ref(node, -1);
	}

	return 1;
}
```

`node->obj = NULL` at `:53` is not gated by any flag; the object ref is
dropped at `:50` only when `UNLINK_OBJECT` was asked for and `NOUNREF_OBJECT`
was not.
So "unlink" means two separable things — detach the node from the object, and
release the object — and the single-return path does the first without the
second. That is the transfer, in one line of C.

That `node->obj = NULL` is also what absorbs a second unlink attempt: the
caller-side guards are `if (node->obj)` at `:344` in the engine and
`if (doomed->common.obj)` in the two node destructors
(`main/astobj2_hash.c:186`, `main/astobj2_rbtree.c:874`), and once `obj` is
NULL they all skip the whole block, so `elements` cannot be decremented twice.
The `container == NULL` guard at `:44` looks like it serves that purpose but
cannot: `node->my_container` is assigned once at node creation
(`main/astobj2_hash.c:221`, `main/astobj2_rbtree.c:906`) and is never cleared
anywhere in the subsystem, so for any node either backend can produce the
guard is unreachable.

Then the sweep ends:

```c
	if (self->v_table->traverse_cleanup) {
		self->v_table->traverse_cleanup(traversal_state);
	}
	if (node) {
		/* Unref the node from self->v_table->traverse_first/traverse_next() */
		ao2_ref(node, -1);
	}

	if (flags & OBJ_NOLOCK) {
		__adjust_lock(self, orig_lock, 0);
	} else {
		ao2_unlock(self);
	}

	/* if multi_container was created, we are returning multiple objects */
	if (multi_container) {
		*multi_iterator = ao2_iterator_init(multi_container,
			AO2_ITERATOR_UNLINK | AO2_ITERATOR_MALLOCD);
		ao2_t_ref(multi_container, -1,
			"OBJ_MULTIPLE for multiple objects traversal complete.");
		return multi_iterator;
	} else {
		return ret;
	}
```

`traverse_cleanup` is called only if the slot is filled, and chapter 04
established that *neither* backend fills it: `v_table_hash`
(`main/astobj2_hash.c:1002-1017`) and `v_table_rbtree`
(`main/astobj2_rbtree.c:1978-1991`) both omit the member, so the only
assignments to it in the tree are none and the call at `:384-385` is dead code.
It exists for a backend that would need to free heap state its traversal had
allocated; both in-tree backends keep everything in the `alloca`'d struct. The `ao2_ref(node, -1)` at `:389` is the
engine discharging the node reference the backend handed it, and it happens
only when the loop exited early via `break`; a loop that ran to completion
already has `node == NULL` because `traverse_next` dropped it (section 5.7).

## 5.4 `OBJ_NODATA`, and the four combinations that matter

`OBJ_MULTIPLE` and `OBJ_NODATA` interact, and the test that decides it is
exact-equality rather than a bit test (`:260`):

| `OBJ_MULTIPLE` | `OBJ_NODATA` | What happens | Return |
|---|---|---|---|
| clear | clear | stop at the first match, take a ref on it (or inherit the container's, under `OBJ_UNLINK`) | the object |
| clear | set | stop at the first match, take no ref; with `OBJ_UNLINK`, drop the container's | NULL |
| set | clear | visit everything, link each match into a throwaway container | `struct ao2_iterator *` |
| set | set | visit everything, take no refs; with `OBJ_UNLINK`, drop each container ref | NULL |

The fourth row is the one to recognise on sight: `OBJ_UNLINK | OBJ_NODATA |
OBJ_MULTIPLE` is "empty this container", and it appears in
`container_destruct` (`:641`), and with `OBJ_NOLOCK` added in
`ao2_container_dup`'s rollback (`:688`) and in
`ao2_container_dup_weakproxy_objs`'s (`:741`) — those two already hold the
destination's lock.

The published documentation for these two flags is not reliable:

```c
 *   - If OBJ_NODATA is set, ao2_callback will return NULL. No refcounts
 *     of any of the traversed objects will be incremented.
 *     On the converse, if it is NOT set (the default), the ref count
 *     of the first matching object will be incremented and returned.
 *   - If OBJ_MULTIPLE is set, the ref count of all matching objects will
 *     be incremented in an iterator for a temporary container and returned.
```

The first claim is right about the return value but wrong by omission about
refcounts — `OBJ_NODATA | OBJ_UNLINK` *decrements* the traversed objects, as
`:372-373` arranges. The second describes only the third row of the table;
with `OBJ_NODATA` also set there is no iterator and no container and no
increment. The flag summary further down has its own problems:

```c
 * The use of flags argument is the follow:
 *
 *      OBJ_UNLINK              unlinks the object found
 *      OBJ_NODATA              on match, do not return an object
 *                              Callbacks use OBJ_NODATA as a default
 *                              functions such as find() do
 *      OBJ_MULTIPLE            return multiple matches
 *                              Default is no.
 *      OBJ_SEARCH_OBJECT       the pointer is to an object
 *      OBJ_SEARCH_KEY          the pointer is to a search key
 *      OBJ_SEARCH_PARTIAL_KEY  the pointer is to a partial search key
```

"Callbacks use OBJ_NODATA as a default functions such as find() do" is garbled,
and whatever it was meant to say, there is no default: `internal_ao2_traverse`
reads the flags it was given and nothing substitutes `OBJ_NODATA` for a caller
who omitted it. `ao2_callback(c, 0, NULL, NULL)` returns a `+1` reference to
an arbitrary object, because `cb_true` matches everything and nothing told the
engine to withhold the result.

## 5.5 `OBJ_MULTIPLE` and the result container

When multiple objects must come back, the engine builds a container to hold
them and hands over an iterator instead of the container
(`:268-276`, `:399-404`). Four details make this work:

The result container is a **list** container with
`AO2_ALLOC_OPT_LOCK_NOLOCK`. No lock is needed because it is never visible to
a second thread: it is created inside the traversal, filled while the *source*
container's lock is held, and handed off inside an iterator that one caller
owns. `ao2_rdlock`/`ao2_wrlock` on a `LOCK_NOLOCK` object fall through to a
bare `break` (`main/astobj2.c:272-274`), so the iterator's per-`next` locking
costs nothing.

The iterator is `ast_calloc`'d and flagged `AO2_ITERATOR_MALLOCD` (`:273`,
`:401`), which is what licenses `ao2_iterator_destroy` to `ast_free` it
(`:544-546`). This is the only place in the tree that sets that flag — every
other iterator lives on the caller's stack.

The container reference is then dropped (`:402-403`), leaving the iterator's
reference from `ao2_iterator_init` as the only one. Destroying the iterator
destroys the container, which drops the last reference to every object still
in it. A caller that abandons the iterator leaks all of them.

`AO2_ITERATOR_UNLINK` is set on it (`:401`), so each `ao2_iterator_next`
removes the object as it yields it — meaning the reference the caller gets is
the one the result container was holding, not a fresh one. The arithmetic
chains correctly from the source container's node all the way out, with no
increment anywhere along the way except `__ao2_link`'s at `:356`.

Here is the pattern as a caller writes it:

```c
	struct ao2_iterator *monitor_iter;
	struct transport_monitor *monitored;
	char key[IP6ADDR_COLON_PORT_BUFLEN];
	int idx;

	AST_SIP_MAKE_REMOTE_IPADDR_PORT_STR(transport, key);

	monitor_iter = ao2_find(transports, key, OBJ_SEARCH_KEY | OBJ_MULTIPLE);
	while ((monitored = ao2_iterator_next(monitor_iter))) {
		if (monitored->transport == transport) {
			ao2_unlink(transports, monitored);
			for (idx = AST_VECTOR_SIZE(&monitored->monitors); idx--;) {
				struct transport_monitor_notifier *notifier;

				notifier = AST_VECTOR_GET_ADDR(&monitored->monitors, idx);
				ast_debug(3, "Transport %s(%s,%s) RefCnt: %ld : running callback %p(%p)\n",
						monitored->key, monitored->transport->obj_name,
						monitored->transport->type_name,
						pj_atomic_get(monitored->transport->ref_cnt), notifier->cb, notifier->data);
				notifier->cb(notifier->data);
			}
		} else {
			ast_debug(3, "Transport %s(%s,%s) RefCnt: %ld : ignored not matching %s\n",
					monitored->key, monitored->transport->obj_name,
					monitored->transport->type_name,
					pj_atomic_get(monitored->transport->ref_cnt), transport->obj_name);
		}
		ao2_ref(monitored, -1);
	}
	ao2_iterator_destroy(monitor_iter);
```

`ao2_find` returning something assignable to `struct ao2_iterator *` is the
type pun from section 5.2 in the wild, and the compiler cannot help: the
declared return type is `void *`. The per-object `ao2_ref(monitored, -1)` at
`:168` and the `ao2_iterator_destroy` at `:170` are both mandatory and are
both easy to forget.

One wart: `:149` dereferences `monitor_iter` without a NULL check, and
`internal_ao2_traverse` can return NULL on this path — if the source container
fails its `__is_ao2_object` check, or if either allocation at `:268-276`
fails. `__ao2_iterator_next` reads `iter->c` immediately (`:563`), so a NULL
iterator is a segfault rather than an empty loop. Under memory pressure this
is a crash. `res/res_pjsip_mwi.c:1453` guards the same pattern with
`if (mwi_subs)`; most sites do not.

## 5.6 `OBJ_NOLOCK` and lock restoration

Chapter 03 described `__adjust_lock` and deferred the question of who calls it
with `keep_stronger` set. Of the fifteen call sites outside `__adjust_lock`
itself, eleven pass `keep_stronger = 1`, and five of those eleven are in the
code this chapter covers: `main/astobj2_container.c:301` and `:303` in
`internal_ao2_traverse`, `:514` in `ao2_iterator_restart`, and `:580` and
`:582` in `__ao2_iterator_next`. Three more are in `__ao2_weakproxy_find`
(`:452`, `:467`, `:474`), which this section comes back to.

`OBJ_NOLOCK` promises "the container is already locked, don't lock it again".
On a mutex container that is exactly what happens, and nothing more:
`__adjust_lock` returns `AO2_LOCK_REQ_MUTEX` for both the mutex and the
no-lock cases and touches nothing (`main/astobj2.c:464-470`). On an **rwlock**
container it is not what happens at all:

```c
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
```

Promotion from a read lock to a write lock is `ao2_unlock` followed by
`ao2_wrlock` (`:447-448`). There is no atomic upgrade. **The container is
genuinely unlocked between those two calls**, and any other thread can link,
unlink or destroy during the gap. So `ao2_find(c, key, OBJ_SEARCH_KEY |
OBJ_NOLOCK | OBJ_UNLINK)` on an rwlock container that the caller is holding
read-locked will silently drop and retake that lock at `:301` before the sweep
begins — and the restore at `:393` drops and retakes it a second time on the
way out, because `keep_stronger` is 0 there and `lock_how != orig_lock` again.

Nothing in the flag's documentation says this; "the lock will be promoted to
write mode as needed" is the closest it comes. It is the hazard chapter 04
briefly attributed to the wrong mechanism, and it belongs here.

A caller holding the lock across an `OBJ_NOLOCK` traversal therefore cannot
assume anything it read before the call is still true after it, if the
container uses an rwlock and the traversal unlinks. The pattern that is safe is
the one that does not care:

```c
	ao2_lock(unsolicited_mwi);
	mwi_subs = ao2_find(unsolicited_mwi, contact->endpoint_name,
		OBJ_SEARCH_KEY | OBJ_MULTIPLE | OBJ_NOLOCK | OBJ_UNLINK);
	if (mwi_subs) {
		for (; (mwi_sub = ao2_iterator_next(mwi_subs)); ao2_cleanup(mwi_sub)) {
			unsubscribe(mwi_sub, NULL, 0);
		}
		ao2_iterator_destroy(mwi_subs);
	}
	ao2_unlock(unsolicited_mwi);
```

The explicit `ao2_lock` at `:1450` exists to make the whole find-and-unsubscribe
sequence atomic with respect to other writers, and `OBJ_NOLOCK` at `:1452`
stops the engine from taking the lock a second time. `unsolicited_mwi` is a
mutex container, so no promotion happens and no gap opens.

`__ao2_weakproxy_find` is the one place that deliberately keeps a promoted
lock:

```c
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
```

It promotes to a write lock at `:467` so it can unlink a proxy whose object has
already died, then restores with `keep_stronger = 1` at `:474` — "We'll keep
any upgraded lock". The caller, who was told to hold a read lock, gets a write
lock back. That is sound (a write lock satisfies every read-lock expectation)
but it means an `OBJ_NOLOCK` weakproxy find can strengthen a caller's lock
behind its back, and the caller cannot tell.

## 5.7 The backend contract: reffed nodes, and a `goto` into a loop

Chapter 04 listed `traverse_first`, `traverse_next`, `traverse_cleanup` and
`iterator_next` as four of the eight vtable slots. Their contracts differ in a
way the slot names do not reveal.

`traverse_first` and `traverse_next` return a node **with a reference taken**,
and `traverse_next` consumes the reference on the node it was passed. The
doc comments say so plainly — "\return node-ptr of found node (Reffed)" at
`main/astobj2_hash.c:330` and `main/astobj2_rbtree.c:1456`, and "The ref
ownership is passed back to this function" at `main/astobj2_hash.c:476`.
`iterator_next` returns a node with **no** reference taken
(`main/astobj2_hash.c:608`, `main/astobj2_rbtree.c:1620`), leaving all of the
bookkeeping to `__ao2_iterator_next`. Mixing the two contracts up in a new
backend would produce either a leak or a use-after-free, and nothing in the
vtable's type signature distinguishes them.

The state each backend keeps between calls is what chapter 04's 100-byte budget
is for. The hash backend needs to remember where in the bucket array it is:

```c
struct hash_traversal_state {
	/*! Active sort function in the traversal if not NULL. */
	ao2_sort_fn *sort_fn;
	/*! Saved comparison callback arg pointer. */
	void *arg;
	/*! Starting hash bucket */
	int bucket_start;
	/*! Stopping hash bucket */
	int bucket_last;
	/*! Saved search flags to control traversing the container. */
	enum search_flags flags;
	/*! TRUE if it is a descending search */
	unsigned int descending:1;
};
```

The rbtree backend needs nothing of the kind, because a tree node knows its own
parent:

```c
struct rbtree_traversal_state {
	/*! Active sort function in the traversal if not NULL. */
	ao2_sort_fn *sort_fn;
	/*! Saved comparison callback arg pointer. */
	void *arg;
	/*! Saved search flags to control traversing the container. */
	enum search_flags flags;
};
```

On x86-64 those are 32 and 24 bytes against a budget of 100, checked at compile
time by the division-by-zero trick chapter 04 described:

```c
struct hash_traversal_state_check {
	/*
	 * If we have a division by zero compile error here then there
	 * is not enough room for the state.  Increase AO2_TRAVERSAL_STATE_SIZE.
	 */
	char check[1 / (AO2_TRAVERSAL_STATE_SIZE / sizeof(struct hash_traversal_state))];
};
```

`1 / (100 / 32)` is `1 / 3`, which is `0`, so `check` is a zero-length array in
practice — legal as a GNU extension. The assert only bites when the state
exceeds 100 bytes, at which point `100 / sizeof(state)` is `0` and the division
is the error.

`hash_ao2_find_first` sets the state up and then walks:

```c
	/*
	 * If lookup by pointer or search key, run the hash and optional
	 * sort functions.  Otherwise, traverse the whole container.
	 */
	switch (flags & OBJ_SEARCH_MASK) {
	case OBJ_SEARCH_OBJECT:
	case OBJ_SEARCH_KEY:
		/* we know hash can handle this case */
		bucket_cur = abs(self->hash_fn(arg, flags & OBJ_SEARCH_MASK)
				% self->n_buckets);
		state->sort_fn = self->common.sort_fn;
		break;
	case OBJ_SEARCH_PARTIAL_KEY:
		/* scan all buckets for partial key matches */
		bucket_cur = -1;
		state->sort_fn = self->common.sort_fn;
		break;
	default:
		/* don't know, let's scan all buckets */
		bucket_cur = -1;
		state->sort_fn = NULL;
		break;
	}
```

Note what the `default:` arm at `:372-376` does: for `OBJ_SEARCH_NONE` it scans
every bucket *and* clears `state->sort_fn`, because an `arg` that means nothing
to astobj2 cannot be fed to a sort function. `OBJ_SEARCH_PARTIAL_KEY` at
`:367-371` keeps the sort function but also scans all buckets, since a partial
key does not hash.

And then `hash_ao2_find_next` has to resume the nested loop it left:

```c
	} else {
		goto hash_ascending_resume;

		/* For each bucket */
		for (; bucket_cur < state->bucket_last; ++bucket_cur) {
			/* For each node in the bucket. */
			for (node = AST_DLLIST_FIRST(&self->buckets[bucket_cur].list);
				node;
				node = AST_DLLIST_NEXT(node, links)) {
				if (!node->common.obj) {
					/* Node is empty */
					continue;
				}

				if (state->sort_fn) {
					/* Filter node through the sort_fn */
					cmp = state->sort_fn(node->common.obj, arg, flags & OBJ_SEARCH_MASK);
					if (cmp < 0) {
						continue;
					}
					if (cmp > 0) {
						/* No more nodes in this bucket are possible to match. */
						break;
					}
				}

				/* We have the next traversal node */
				ao2_ref(node, +1);

				/*
				 * Dereferencing the prev node may result in our next node
				 * object being removed by another thread.  This could happen if
				 * the container uses RW locks and the container was read
				 * locked.
				 */
				ao2_ref(prev, -1);
				if (node->common.obj) {
					return node;
				}
				prev = node;

hash_ascending_resume:;
			}
		}
	}
```

The `goto hash_ascending_resume;` at `:546` jumps *into* the body of a doubly
nested `for`, past both loop initialisers, landing on a label at `:586` placed
immediately before the inner loop's increment. `bucket_cur` and `node` were
primed from `prev` at `:491-492`, so the loops pick up exactly where
`hash_ao2_find_first` stopped. The comment at `:494-500` tells you the two
functions are structured identically on purpose; they must be edited in
lockstep, and there are four copies of the inner body across the two functions
(ascending and descending in each).

The reference dance at `:572-583` repays attention. The new node is reffed
*before* the previous one is unreffed, and then the new node's `obj` is checked
*again*. The comment at `:574-578` explains: dropping the previous node can run
a destructor, which can let another thread remove the object the new node
points at. If that happened, the loop keeps the node as the new `prev` and
carries on rather than returning an empty node.

## 5.8 Empty nodes, and why every loop checks `obj`

`!node->common.obj` appears as a skip condition a dozen times across the two
backends — ten `if` tests and, in the rbtree's ordered walks, two `while`
loops (`main/astobj2_rbtree.c:1571`, `:1594`). Chapter 04's `node->obj = NULL` is the reason.

A node is unlinked from its object before it is unlinked from the container's
structure. The structural removal happens in the node destructor —
`AST_DLLIST_REMOVE` at `main/astobj2_hash.c:178` and `rb_delete_node` at
`main/astobj2_rbtree.c:861`, each inside an `if (doomed->common.is_linked)`
block (`main/astobj2_hash.c:147`, `main/astobj2_rbtree.c:832`) — which runs
only when
the node's last reference goes. Any traversal or iterator still holding a node
reference therefore keeps a node in the bucket list or the tree *after* its
object has been removed, with `obj == NULL`. Those are empty nodes, and every
walk has to step over them:

```c
		/* For each bucket */
		for (; bucket_cur < state->bucket_last; ++bucket_cur) {
			/* For each node in the bucket. */
			for (node = AST_DLLIST_FIRST(&self->buckets[bucket_cur].list);
				node;
				node = AST_DLLIST_NEXT(node, links)) {
				if (!node->common.obj) {
					/* Node is empty */
					continue;
				}

				if (state->sort_fn) {
					/* Filter node through the sort_fn */
					cmp = state->sort_fn(node->common.obj, arg, flags & OBJ_SEARCH_MASK);
					if (cmp < 0) {
						continue;
					}
					if (cmp > 0) {
						/* No more nodes in this bucket are possible to match. */
						break;
					}
				}

				/* We have the first traversal node */
				ao2_ref(node, +1);
				return node;
			}
		}
```

Under `AO2_DEBUG` the engine tracks how many accumulate —
`container->nodes - container->elements`, kept as a high-water mark at
`main/astobj2_container.c:59-63`. A container whose `max_empty_nodes` grows
without bound has a node reference leaking somewhere, which is chapter 04's
`Node ref leak.  Hash container still has nodes!` assert waiting to happen at
shutdown.

This is also why the rbtree backend cannot simply delete a node on unlink: the
tree structure has to survive until the node's refcount reaches zero, so an
unlinked-but-referenced node stays in the tree as an empty placeholder and the
rebalance is deferred to the destructor.

## 5.9 The iterator

The iterator is four fields, and the header apologises for showing them to you:

```c
struct ao2_iterator {
	/*! The container (Has a reference) */
	struct ao2_container *c;
	/*! Last container node (Has a reference) */
	void *last_node;
	/*! Nonzero if the iteration has completed. */
	int complete;
	/*! operation flags (enum ao2_iterator_flags) */
	int flags;
};
```

`last_node` is `void *` rather than `struct ao2_container_node *` because the
node type lives in `main/astobj2_container_private.h`, which nothing outside
the four implementation files may include. Both `c` and `last_node` hold
references, as the comments say; `complete` latches once the end is reached.

`ao2_iterator_init` is almost nothing — a struct literal and one reference:

```c
struct ao2_iterator ao2_iterator_init(struct ao2_container *c, int flags)
{
	struct ao2_iterator a = {
		.c = c,
		.flags = flags
	};

	ao2_t_ref(c, +1, "Init iterator with container.");

	return a;
}
```

It is declared `attribute_warn_unused_result` (`include/asterisk/astobj2.h:1893`),
which matters because it returns the iterator *by value*. Discarding the return
value leaks the container reference it just took, and the attribute is what
catches that.

The flags are four:

```c
enum ao2_iterator_flags {
	/*!
	 * \brief Assume that the ao2_container is already locked.
	 *
	 * \note For ao2_containers that have mutexes, no locking will
	 * be done.
	 *
	 * \note For ao2_containers that have RWLOCKs, the lock will be
	 * promoted to write mode as needed.  The lock will be returned
	 * to the original locked state.
	 *
	 * \note Only use this flag if the ao2_container is manually
	 * locked already.  You should hold the lock until after
	 * ao2_iterator_destroy().  If you must release the lock then
	 * you must at least hold the lock whenever you call an
	 * ao2_iterator_xxx function with this iterator.
	 */
	AO2_ITERATOR_DONTLOCK = (1 << 0),
	/*!
	 * Indicates that the iterator was dynamically allocated by
	 * astobj2 API and should be freed by ao2_iterator_destroy().
	 */
	AO2_ITERATOR_MALLOCD = (1 << 1),
	/*!
	 * Indicates that before the iterator returns an object from
	 * the container being iterated, the object should be unlinked
	 * from the container.
	 */
	AO2_ITERATOR_UNLINK = (1 << 2),
```

```c
	/*!
	 * Iterate in descending order (Last to first container object)
	 * (Otherwise ascending order)
	 *
	 * \note Other traversal orders such as pre-order and post-order
	 * do not make sense because they require the container
	 * structure to be static during the traversal.  Iterators just
	 * about guarantee that is not going to happen because the
	 * container is allowed to change by other threads during the
	 * iteration.
	 */
	AO2_ITERATOR_DESCENDING = (1 << 3),
```

The note at `:1868-1873` is the clearest statement in the subsystem of what an
iterator is *not*: pre-order and post-order are refused because they need the
container to hold still, and an iterator exists precisely for the case where it
will not. The tree-order flags belong to `ao2_callback`, which holds the lock
for the whole sweep. And as section 5.1 recorded, nothing outside the tests
asks for descending iteration either.

`__ao2_iterator_next` locks the same way `internal_ao2_traverse` does, keyed on
`AO2_ITERATOR_UNLINK` instead of `OBJ_UNLINK`:

```c
void *__ao2_iterator_next(struct ao2_iterator *iter,
	const char *tag, const char *file, int line, const char *func)
{
	enum ao2_lock_req orig_lock;
	struct ao2_container_node *node;
	void *ret;

	if (!__is_ao2_object(iter->c, file, line, func)) {
		return NULL;
	}

	if (!iter->c->v_table || !iter->c->v_table->iterator_next) {
		/* Sanity checks. */
		__ast_assert_failed(0, "invalid iterator container v_table", file, line, func);
		return NULL;
	}

	if (iter->complete) {
		/* Don't return any more objects. */
		return NULL;
	}

	if (iter->flags & AO2_ITERATOR_DONTLOCK) {
		if (iter->flags & AO2_ITERATOR_UNLINK) {
			orig_lock = __adjust_lock(iter->c, AO2_LOCK_REQ_WRLOCK, 1);
		} else {
			orig_lock = __adjust_lock(iter->c, AO2_LOCK_REQ_RDLOCK, 1);
		}
	} else {
		orig_lock = AO2_LOCK_REQ_MUTEX;
		if (iter->flags & AO2_ITERATOR_UNLINK) {
			ao2_wrlock(iter->c);
		} else {
			ao2_rdlock(iter->c);
		}
	}
```

and then does the reference arithmetic:

```c
	node = iter->c->v_table->iterator_next(iter->c, iter->last_node, iter->flags);
	if (node) {
		ret = node->obj;

		if (iter->flags & AO2_ITERATOR_UNLINK) {
			/* Transfer the object ref from the container to the returned object. */
			__container_unlink_node_debug(node, AO2_UNLINK_NODE_DEC_COUNT, tag, file, line, func);

			/* Transfer the container's node ref to the iterator. */
		} else {
			/* Bump ref of returned object */
			__ao2_ref(ret, +1, tag ?: "Next iterator object.", file, line, func);

			/* Bump the container's node ref for the iterator. */
			ao2_ref(node, +1);
		}
	} else {
		/* The iteration has completed. */
		iter->complete = 1;
		ret = NULL;
	}

	/* Replace the iterator's node */
	if (iter->last_node) {
		ao2_ref(iter->last_node, -1);
	}
	iter->last_node = node;

	if (iter->flags & AO2_ITERATOR_DONTLOCK) {
		__adjust_lock(iter->c, orig_lock, 0);
	} else {
		ao2_unlock(iter->c);
	}

	return ret;
}
```

Both branches end with the iterator holding exactly one node reference and the
caller owing exactly one object unref — but they get there oppositely. Without
`AO2_ITERATOR_UNLINK`, both the object and the node are bumped (`:604`,
`:607`). With it, neither is: `__container_unlink_node_debug` is called with
only `AO2_UNLINK_NODE_DEC_COUNT`, so the object reference transfers to the
caller (no `UNLINK_OBJECT`) and the node reference transfers to the iterator
(no `UNREF_NODE`). The two comments at `:598` and `:601` are the specification.

The previous node is released at `:616-618`, which is why an iterator holds at
most one node reference at a time, and why the container can be mutated between
two `next` calls without the iterator pinning history.

`ao2_iterator_count` is a trap:

```c
int ao2_iterator_count(struct ao2_iterator *iter)
{
	return ao2_container_count(iter->c);
}
```

It reports the number of objects in the **container**, not the number remaining
in the iteration. There is no way to ask an iterator how much is left. It has
zero callers outside astobj2.

## 5.10 `AO2_ITERATOR_UNLINK` and the drain idiom

Sixteen sites outside astobj2 pass `AO2_ITERATOR_UNLINK`, and ten of them do
the same thing: build a throwaway sorted container, copy the real container
into it, then drain the copy.

```c
	/*
	 * No need to lock the container since it's temporary and static.
	 * We also unlink the entries as we use them so the container is
	 * empty when the iterator finishes.  We can then just unref the container.
	 */
	queue_iter = ao2_iterator_init(sorted_queues, AO2_ITERATOR_DONTLOCK | AO2_ITERATOR_UNLINK);
	while ((q = ao2_t_iterator_next(&queue_iter, "Iterate through queues"))) {
```

The container being drained was allocated fourteen lines earlier as
`ao2_container_alloc_rbtree(AO2_ALLOC_OPT_LOCK_NOLOCK, 0, call_queue_sort_fn,
NULL)` (`apps/app_queue.c:10495`) and filled with `ao2_container_dup` from the
live `queues` container (`:10499`). The rbtree is there purely to sort; the
`UNLINK` iterator is there so that the scratch container is empty when the loop
ends, which is what makes the single `ao2_ref(sorted_queues, -1)` at `:10535`
sufficient. Drain-as-you-go means there is no second cleanup pass and no
double-free window.

The same shape appears in `main/taskprocessor.c:369-375` and `:652-662`,
`main/stasis.c:2582-2592`, `res/res_pjsip/pjsip_scheduler.c:534-558`,
`res/res_pjsip/pjsip_transport_events.c:617-645`,
`res/res_geolocation/geoloc_config.c:282-311`, `:373-402` and `:453-479`, and
with a list rather than a tree in `res/res_mwi_external_ami.c:196-224`.

The other six sites drain something real. `res/stasis/control.c:1560`, `:1577`
and `:1623` drain a live `AO2_ALLOC_OPT_LOCK_MUTEX` command queue — the
control structure's own queue for the first two (allocated at
`res/stasis/control.c:157-158`) and the per-channel prestart queue for the
third (`res/stasis/command.c:142`, same lock mode) — legitimate, since `__ao2_iterator_next` takes the
mutex for each `next`, so a producer can keep appending between iterations.
`res/res_mwi_external.c:271` and `:887` drain a container handed back by a
sorcery retrieval. And `res/res_pjsip/pjsip_options.c:3251` drains the
long-lived, `AO2_ALLOC_OPT_LOCK_RWLOCK` `sip_options_aors` container at module
cleanup, which is the only site where `UNLINK` iteration and an rwlock meet.

`ao2_iterator_restart`'s documentation (`include/asterisk/astobj2.h:1922-1924`)
warns that restarting an `UNLINK` iterator does nothing useful, since whatever
it already yielded is gone. It has no callers at all outside astobj2 — the
only mention in the tree is in the example in the header's own comment at
`:1794`.

## 5.11 `AO2_ITERATOR_DONTLOCK`, and the restart that promotes a lock

`AO2_ITERATOR_DONTLOCK` is `OBJ_NOLOCK` for iterators, with the same rwlock
caveat and one extra requirement: the documentation says to hold the lock until
after `ao2_iterator_destroy`, not merely until the last `next`
(`include/asterisk/astobj2.h:1846-1850`). Four sites outside astobj2 use it.
`main/codec.c` shows why the destroy matters:

```c
	ao2_rdlock(codecs);
	i = ao2_iterator_init(codecs, AO2_ITERATOR_DONTLOCK);

	for (; (codec = ao2_iterator_next(&i)); ao2_ref(codec, -1)) {
```

```c
	ao2_iterator_destroy(&i);
	ao2_unlock(codecs);
```

The `ao2_iterator_destroy` at `:183` precedes the `ao2_unlock` at `:184`, and
that order is not cosmetic. Destroying the iterator releases its node
reference, and releasing a node reference can need the container lock:

```c
void ao2_iterator_restart(struct ao2_iterator *iter)
{
	if (!is_ao2_object(iter->c)) {
		/* Sanity check. */
		return;
	}

	/* Release the last container node reference if we have one. */
	if (iter->last_node) {
		enum ao2_lock_req orig_lock;

		/*
		 * Do a read lock in case the container node unref does not
		 * destroy the node.  If the container node is destroyed then
		 * the lock will be upgraded to a write lock.
		 */
		if (iter->flags & AO2_ITERATOR_DONTLOCK) {
			orig_lock = __adjust_lock(iter->c, AO2_LOCK_REQ_RDLOCK, 1);
		} else {
			orig_lock = AO2_LOCK_REQ_MUTEX;
			ao2_rdlock(iter->c);
		}

		ao2_ref(iter->last_node, -1);
		iter->last_node = NULL;

		if (iter->flags & AO2_ITERATOR_DONTLOCK) {
			__adjust_lock(iter->c, orig_lock, 0);
		} else {
			ao2_unlock(iter->c);
		}
	}

	/* The iteration is no longer complete. */
	iter->complete = 0;
}
```

`ao2_iterator_restart` — which `ao2_iterator_destroy` calls first thing
(`:537`) — takes a **read** lock around the node unref at `:520`, and the
comment at `:508-512` explains the choice: if that unref destroys the node, the
node destructor will promote the lock to a write lock itself. This is the
`__adjust_lock` promotion chapter 04 looked for in the wrong place. It is real,
it is here, and under `AO2_ITERATOR_DONTLOCK` on an rwlock container it means
`ao2_iterator_destroy` can drop and retake a lock the caller believes it holds
continuously.

Get the order wrong — unlock before destroy — and the destructor's structural
removal from the bucket list or tree runs unsynchronised.

The other `DONTLOCK` sites are `main/manager.c:9342`,
`apps/app_queue.c:10152` and the `apps/app_queue.c:10509` seen in section
5.10, and that last one uses the flag for a different reason than the
documentation gives. Its container is `AO2_ALLOC_OPT_LOCK_NOLOCK`, so there is
no lock to hold; the comment at `:10504-10508` says "No need to lock the
container since it's temporary and static". `DONTLOCK` there means "there is
no lock", not "I am already holding it". Both readings produce correct code,
but only one of them is written down.

## 5.12 `ao2_unlink`, and three callbacks worth knowing

`ao2_unlink` is a traversal, and a surprisingly opinionated one:

```c
int ao2_match_by_addr(void *user_data, void *arg, int flags)
{
	return (user_data == arg) ? (CMP_MATCH | CMP_STOP) : 0;
}

/*
 * Unlink an object from the container
 * and destroy the associated * bucket_entry structure.
 */
void *__ao2_unlink(struct ao2_container *c, void *user_data, int flags,
	const char *tag, const char *file, int line, const char *func)
{
	if (!__is_ao2_object(user_data, file, line, func)) {
		/* Sanity checks. */
		return NULL;
	}

	flags &= ~OBJ_SEARCH_MASK;
	flags |= (OBJ_UNLINK | OBJ_SEARCH_OBJECT | OBJ_NODATA);
	__ao2_callback(c, flags, ao2_match_by_addr, user_data, tag, file, line, func);

	return NULL;
}
```

Three things here catch people. It **clears** `OBJ_SEARCH_MASK` at `:183`
before forcing `OBJ_SEARCH_OBJECT`, so any search mode the caller passed is
discarded. It forces `OBJ_NODATA`, so there is no reference to release. And
despite being declared `void *`, it returns NULL unconditionally at `:187` —
the return value carries no information whatsoever, not even success.

The forced `OBJ_SEARCH_OBJECT` has a sharp consequence on a hash container. It
makes the backend call `hash_fn(obj, OBJ_SEARCH_OBJECT)` to pick a bucket
(`main/astobj2_hash.c:363-364`), and `ao2_match_by_addr` is then only consulted
within that one bucket. If the object's key fields have been mutated since it
was linked, it now hashes to a different bucket than the one it is sitting in,
and `ao2_unlink` will scan the wrong bucket, find nothing, and return — with no
error, no log line and no indication that the object is still in the container.
That is the strongest argument in the subsystem for treating an object's key
fields as immutable for as long as it is linked.

The callback it forces is as simple as it looks, and `cb_true` is simpler
still:

```c
static int cb_true(void *user_data, void *arg, int flags)
{
	return CMP_MATCH;
}

/*!
 * \brief similar to cb_true, but is an ao2_callback_data_fn instead
 */
static int cb_true_data(void *user_data, void *arg, void *data, int flags)
{
	return CMP_MATCH;
}
```

`cb_true` returns `CMP_MATCH` without `CMP_STOP`, which is what lets
`ao2_callback(c, OBJ_UNLINK | OBJ_NODATA | OBJ_MULTIPLE, NULL, NULL)` sweep an
entire container: every node matches, and only `OBJ_MULTIPLE` keeps the loop
going (`:379`).

`ao2_match_by_addr` is public (`include/asterisk/astobj2.h:1229`) and is used
outside astobj2 three times, in all three cases as a function *pointer* rather
than a direct call: as a container's `cmp_fn` in `res/res_odbc.c:1197`, and
passed to `ao2_callback` in `res/res_musiconhold.c:237` and
`tests/test_astobj2.c:239`.

`ao2_container_dup` shows the last idiom — using the single-object return as an
error channel:

```c
static int dup_obj_cb(void *obj, void *arg, int flags)
{
	struct ao2_container *dest = arg;

	return ao2_link_flags(dest, obj, OBJ_NOLOCK) ? 0 : (CMP_MATCH | CMP_STOP);
}

int ao2_container_dup(struct ao2_container *dest, struct ao2_container *src, enum search_flags flags)
{
	void *obj;
	int res = 0;

	if (!(flags & OBJ_NOLOCK)) {
		ao2_rdlock(src);
		ao2_wrlock(dest);
	}
	obj = ao2_callback(src, OBJ_NOLOCK, dup_obj_cb, dest);
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
}
```

`dup_obj_cb` returns `0` on success, so the traversal visits every node and
returns NULL. It returns `CMP_MATCH | CMP_STOP` only when the link fails, which
stops the sweep and hands the offending object back with a `+1` that `:685`
releases. Then `:688-689` rolls the destination back with the empty-the-
container flag triple. A callback that signals failure by matching is unusual
enough to be worth recognising; `dup_weakproxy_cb` (`:710-724`) does the same.

`__ao2_container_clone` wraps `ao2_container_dup` with one piece of
counter-intuitive locking (`:774-783`): when the caller passes `OBJ_NOLOCK` the
clone is explicitly **write-locked**, because `OBJ_NOLOCK` tells
`ao2_container_dup` not to lock the destination itself. The comment at
`:774-776` opens with "This test is correct", which is a fair warning that it
does not read that way.

## 5.13 Destruction is a traversal

The last piece ties the chapter to chapter 04. A container's destructor does
not walk buckets or rotate trees; it issues one `ao2_callback`:

```c
void container_destruct(void *_c)
{
	struct ao2_container *c = _c;

	/* Unlink any stored objects in the container. */
	c->destroying = 1;
	ao2_t_callback(c, OBJ_UNLINK | OBJ_NODATA | OBJ_MULTIPLE, NULL, NULL,
		"container_destruct called");

	/* Perform any extra container cleanup. */
	if (c->v_table && c->v_table->destroy) {
		c->v_table->destroy(c);
	}

#if defined(AO2_DEBUG)
	ast_atomic_fetchadd_int(&ao2.total_containers, -1);
#endif
}
```

`OBJ_UNLINK | OBJ_NODATA | OBJ_MULTIPLE` — the fourth row of section 5.4's
table — visits every node, drops the container's reference to every object, and
returns nothing. Only then does the backend's `destroy` slot run.

The `c->destroying = 1` at `:640` is what makes this affordable for the rbtree,
and it reaches into the traversal through a flag override:

```c
static struct rbtree_node *rb_ao2_find_first(struct ao2_container_rbtree *self, enum search_flags flags, void *arg, struct rbtree_traversal_state *state)
{
	struct rbtree_node *node;
	enum equal_node_bias bias;

	if (self->common.destroying) {
		/* Force traversal to be post order for tree destruction. */
		flags = OBJ_UNLINK | OBJ_NODATA | OBJ_MULTIPLE | OBJ_ORDER_POST;
	}
```

A destroying rbtree discards the caller's flags entirely and substitutes
post-order. Post-order visits children before parents, so the tree collapses
from the leaves upward and never needs to rebalance — which the delete path
confirms by skipping fixups outright when the flag is set:
`need_fixup = (!doomed->is_red && !self->common.destroying)`
(`main/astobj2_rbtree.c:778`). That is the only place the flag suppresses real
work on the delete path. The node destructors also test it
(`main/astobj2_hash.c:172`, `main/astobj2_rbtree.c:856` and `:863`), but those
tests sit inside `#if defined(AO2_DEBUG)` and gate an `ao2_container_check()`
integrity assertion, not the removal itself — a container mid-destruction is
legitimately inconsistent, so the check would cry wolf. The structural removal
below each of them is unconditional.

This is also the one in-tree consumer of `OBJ_ORDER_POST`, and it never passes
through a caller's hands.

## 5.14 Symptom → cause

| Symptom | Cause |
|---|---|
| An object found with `ao2_find` is freed too early, or leaks | `OBJ_UNLINK` changes where the returned reference came from but not that you owe one unref (`main/astobj2_container.c:360-366`) |
| `ao2_find` returns a pointer that crashes when dereferenced as your object | `OBJ_MULTIPLE` was set, so the return is a `struct ao2_iterator *` (`main/astobj2_container.c:399-404`) |
| Segfault inside `__ao2_iterator_next` right after an `OBJ_MULTIPLE` find | the find returned NULL on an allocation failure and was not checked; `:563` dereferences `iter->c` (`main/astobj2_container.c:268-276`) |
| Every object in an `OBJ_MULTIPLE` result is freed while you are still using one | the iterator held the only container reference; destroying it destroyed them all (`main/astobj2_container.c:402-403`) |
| A callback returning `-1` on error behaves as "found it, stop" | the return is masked to `CMP_MATCH \| CMP_STOP`, and `-1` has both bits (`main/astobj2_container.c:324-329`) |
| `ao2_callback` with no callback returns a reference to a random object | `cb_true` matches everything and nothing implied `OBJ_NODATA` (`main/astobj2_container.c:193-196`, `:279-285`) |
| A container mutated under a lock the caller never released, during an `OBJ_NOLOCK \| OBJ_UNLINK` traversal | rwlock promotion is `ao2_unlock` then `ao2_wrlock`, with a real gap (`main/astobj2.c:445-449`, `main/astobj2_container.c:301`) |
| A read lock became a write lock across `ao2_weakproxy_find` | it restores with `keep_stronger = 1` on purpose (`main/astobj2_container.c:467`, `:474`) |
| `ao2_unlink` silently fails to remove an object that is definitely in the container | it forces `OBJ_SEARCH_OBJECT`, so a mutated key field sends it to the wrong bucket (`main/astobj2_container.c:183-184`, `main/astobj2_hash.c:363-364`) |
| `ao2_unlink`'s return value is always NULL | it is, unconditionally, despite the `void *` type (`main/astobj2_container.c:187`) |
| `ao2_iterator_count` disagrees with how many objects the loop still has to go | it counts the container, not the remainder (`main/astobj2_container.c:630-633`) |
| A container's `max_empty_nodes` grows without bound, then asserts at shutdown | a node reference from a traversal or iterator is leaking (`main/astobj2_container.c:59-63`) |
| A crash in a node destructor's list or tree removal, under `AO2_ITERATOR_DONTLOCK` | the container was unlocked before `ao2_iterator_destroy`, which needs the lock to drop the last node ref (`main/astobj2_container.c:505-528`) |
| `OBJ_ORDER_DESCENDING` on a multi-bucket hash container gives an order that makes no sense | the walk is bucket-by-bucket reversed, not a global reversal (`main/astobj2_hash.c:394-398`) |
| A compiler warning about discarding the result of `ao2_iterator_init` | it returns the iterator by value and is `attribute_warn_unused_result`; discarding it leaks the container ref (`include/asterisk/astobj2.h:1893`) |

## 5.15 Rules

| Rule | Source |
|---|---|
| Release exactly one object reference per non-`OBJ_NODATA` match, whether or not `OBJ_UNLINK` was set | `main/astobj2_container.c:360-366` |
| Never assign an `OBJ_MULTIPLE` result to your object type; it is a `struct ao2_iterator *` | `main/astobj2_container.c:399-404` |
| NULL-check an `OBJ_MULTIPLE` result before the first `ao2_iterator_next` | `main/astobj2_container.c:268-276`, `:563` |
| Destroy an `OBJ_MULTIPLE` iterator even on an early exit, or leak every object it still holds | `main/astobj2_container.c:402-403` |
| Return only `0`, `CMP_MATCH`, `CMP_STOP` or the two ORed from a callback | `main/astobj2_container.c:324-329` |
| Do not hold `OBJ_NOLOCK` state across an unlinking traversal of an rwlock container | `main/astobj2.c:445-449`, `main/astobj2_container.c:301`, `:393` |
| Call `ao2_iterator_destroy` before releasing a lock you took for `AO2_ITERATOR_DONTLOCK` | `main/astobj2_container.c:505-528`, `main/codec.c:183-184` |
| Treat an object's hash and sort key fields as immutable while it is linked | `main/astobj2_container.c:183-184`, `main/astobj2_hash.c:217`, `:363-364` |
| Do not read `ao2_unlink`'s return value | `main/astobj2_container.c:187` |
| Do not use `ao2_iterator_count` to drive a loop | `main/astobj2_container.c:630-633` |
| In a new backend, ref the node in `traverse_first`/`traverse_next` and do not in `iterator_next` | `main/astobj2_hash.c:330`, `:476`, `:608` |
| Keep a new backend's traversal state at or under 100 bytes | `main/astobj2_container_private.h:60`, `main/astobj2_hash.c:94-100` |
| Skip nodes whose `obj` is NULL in every backend walk | `main/astobj2_hash.c:441-444`, `main/astobj2_rbtree.c:1571-1576` |
| Prefer `OBJ_SEARCH_*` to `OBJ_POINTER`/`OBJ_KEY`/`OBJ_PARTIAL_KEY` in new code | `include/asterisk/astobj2.h:1144-1152` |

## Next

[Chapter 06](06-weak-proxies-and-global-objects.md) is about the two ways an
object can be reached without a strong reference to it: `ao2_weakproxy` and the
`weakptr` field that a proxy and its object share, the subscription callbacks
that fire when the object dies, and the `AO2_GLOBAL_OBJ_STATIC` holder — why a
global ao2 object is read through a function rather than named directly.
