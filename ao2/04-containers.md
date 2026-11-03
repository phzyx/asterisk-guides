# 4. Containers

**Target tree:** Asterisk, the `24` branch

Chapters 01 through 03 were about one object. This chapter is about the thing
that holds a lot of them, and the first surprise is that a container *is* one of
those objects: `__ao2_container_alloc_hash` calls `__ao2_alloc`
(`main/astobj2_hash.c:1079-1080`) and `__ao2_container_alloc_rbtree` does the
same (`main/astobj2_rbtree.c:2034-2035`). A container has the private header
from chapter 01, the refcount from chapter 02 and the lock mode from chapter 03,
and everything those chapters said about it still applies. What it adds on top
is a vtable.

The subsystem is 4,324 lines across three files — the generic engine
(`main/astobj2_container.c`, 1,196 lines), the hash backend
(`main/astobj2_hash.c`, 1,091) and the red-black tree backend
(`main/astobj2_rbtree.c`, 2,037) — plus a 318-line private header that nothing
outside those three may include. The split is the whole design: the engine knows
about links, unlinks, traversals and iterators but not about buckets or
rotations; the backends know about buckets and rotations but never *acquire*
the container's lock — the engine has already taken it before any backend method
runs.

```
   ao2_container_alloc_hash(ao2_opts, container_opts, n_buckets,
                            hash_fn, sort_fn, cmp_fn)
            │
            ▼
   ┌──────────────────┬───────────────────────┬──────────────────────────────┐
   │ struct           │ struct __priv_data    │ struct ao2_container_hash    │
   │ ao2_lock_priv    │                       │  ┌────────────────────────┐  │
   │  (ch. 01)        │  (ch. 01-02)          │  │ struct ao2_container   │  │
   │                  │                       │  │   v_table ────────────┼──┼──→ v_table_hash
   │                  │                       │  │   sort_fn             │  │    (astobj2_hash.c:1002)
   │                  │                       │  │   cmp_fn              │  │
   │                  │                       │  │   options             │  │
   │                  │                       │  │   elements            │  │
   │                  │                       │  │   destroying:1        │  │
   │                  │                       │  └────────────────────────┘  │
   │                  │                       │    hash_fn                   │
   │                  │                       │    n_buckets                 │
   │                  │                       │    buckets[0]  ← variable    │
   └──────────────────┴───────────────────────┴──────────────────────────────┘
                                              ▲
                                              └── what the caller holds, and
                                                  what every generic routine in
                                                  astobj2_container.c casts to
                                                  struct ao2_container *

   Eight vtable slots.  Neither backend fills all eight.

                        v_table_hash      v_table_rbtree
     destroy                ✓                  ✓
     alloc_empty_clone      ✓                  ✓
     new_node               ✓                  ✓
     insert                 ✓                  ✓
     traverse_first         ✓                  ✓
     traverse_next          ✓                  ✓
     traverse_cleanup       —                  —      ← nobody implements it
     iterator_next          ✓                  ✓
   ─────────────────────── AO2_DEBUG only ───────────────────────
     link_stat              ✓                  —
     unlink_stat            ✓                  —
     dump                   ✓                  ✓
     stats                  ✓                  ✓
     integrity              ✓                  ✓
                           12/13              10/13
```

Three facts carry the chapter:

1. **A list container is a hash container.** `__ao2_container_alloc_list` is a
   one-line forward to `__ao2_container_alloc_hash` with `n_buckets = 1` and
   `hash_fn = NULL` (`main/astobj2_hash.c:1085-1091`). There are two container
   implementations in the tree, not three, and the 233 hash-or-list allocation
   sites all land in the same 1,091 lines. Section 4.4.
2. **A NULL hash function silently collapses the container to one bucket.**
   `num_buckets = hash_fn ? n_buckets : 1` at `main/astobj2_hash.c:1076` — no
   warning, no log line, and `ao2_container_alloc_hash(..., 563, NULL, ...)`
   gives you a linked list. Section 4.5.
3. **The two backends disagree about whether a container may be unsorted.**
   `__ao2_container_alloc_rbtree` rejects a NULL `sort_fn` outright
   (`main/astobj2_rbtree.c:2028-2032`); the hash backend accepts one and
   degrades to unsorted bucket chains. Three of the four duplicate policies
   document "The container must be sorted" and nothing on the hash path enforces
   it. Section 4.7.

## 4.1 The generic container, and the "common first" rule

Every container begins with the same struct:

```c
struct ao2_container {
	/*! Container virtual method table. */
	const struct ao2_container_methods *v_table;
	/*! Container sort function if the container is sorted. */
	ao2_sort_fn *sort_fn;
	/*! Container traversal matching function for ao2_find. */
	ao2_callback_fn *cmp_fn;
	/*! The container option flags */
	uint32_t options;
	/*! Number of elements in the container. */
	int elements;
#if defined(AO2_DEBUG)
	/*! Number of nodes in the container. */
	int nodes;
	/*! Maximum number of empty nodes in the container. (nodes - elements) */
	int max_empty_nodes;
#endif	/* defined(AO2_DEBUG) */
	/*!
	 * \brief TRUE if the container is being destroyed.
	 *
	 * \note The destruction traversal should override any requested
	 * search order to do the most efficient order for destruction.
	 *
	 * \note There should not be any empty nodes in the container
	 * during destruction.  If there are then an error needs to be
	 * issued about container node reference leaks.
	 */
	unsigned int destroying:1;
};
```

Six fields in a production build, eight under `AO2_DEBUG`. `elements` is the
only one a caller can read, and only through a function:

```c
int ao2_container_count(struct ao2_container *c)
{
	return ast_atomic_fetchadd_int(&c->elements, 0);
}
```

The `ast_atomic_fetchadd_int(&c->elements, 0)` is an atomic read spelled as an
add of zero, which is why `ao2_container_count` needs no lock. It is also why
the count is a snapshot and nothing more: by the time it returns, another thread
may have linked or unlinked. `elements` is maintained by the engine, in
`__ao2_link` (`main/astobj2_container.c:132`) and in the unlink path, never by a
backend.

`destroying` is the field worth remembering. `container_destruct` sets it before
it starts unlinking (`main/astobj2_container.c:640`), and the comment on the
field explains what it licenses: the destruction traversal may ignore the
caller's requested search order, because order does not matter when you are
emptying the thing.

Nodes have their own common prefix:

```c
struct ao2_container_node {
	/*! Stored object in node. */
	void *obj;
	/*! Container holding the node.  (Does not hold a reference.) */
	struct ao2_container *my_container;
	/*! TRUE if the node is linked into the container. */
	unsigned int is_linked:1;
};
```

Three fields, one of which is a back-pointer that explicitly does *not* hold a
reference. That is deliberate and it is the reverse of the obvious ownership:
the container holds a reference to each node, each node holds a reference to the
stored object (taken in `new_node`, `main/astobj2_hash.c:219`), and the node's
pointer back to the container is raw. A node therefore cannot outlive its
container by holding it alive, which is the property that makes
`container_destruct` finite.

Both backends then embed these as their first member, and both say so in a
`\note`:

```c
struct rbtree_node {
	/*!
	 * \brief Items common to all container nodes.
	 * \note Must be first in the specific node struct.
	 */
	struct ao2_container_node common;
	/*! Parent node of this node. NULL if this is the root node. */
	struct rbtree_node *parent;
	/*! Left child node of this node.  NULL if does not have this child. */
	struct rbtree_node *left;
	/*! Right child node of this node.  NULL if does not have this child. */
	struct rbtree_node *right;
	/*! TRUE if the node is red. */
	unsigned int is_red:1;
};
```

```c
struct ao2_container_hash {
	/*!
	 * \brief Items common to all containers.
	 * \note Must be first in the specific container struct.
	 */
	struct ao2_container common;
	ao2_hash_fn *hash_fn;
	/*! Number of hash buckets in this container. */
	int n_buckets;
	/*! Hash bucket array of n_buckets.  Variable size. */
	struct hash_bucket buckets[0];
};
```

The `\note Must be first in the specific container struct.` is not a style
preference. The entire generic engine receives `struct ao2_container *` and the
backends receive their own type; the two are converted by a plain cast, both on
the way in (`return (struct ao2_container *) self;`,
`main/astobj2_hash.c:1064`) and on the way out, by every
`(ao2_container_insert_fn)` cast in the vtable initialisers. Reorder the members
and nothing fails to compile.

`struct ao2_container_hash` ends in `struct hash_bucket buckets[0];` — a
zero-length array, which is why the allocation size is computed rather than
`sizeof`:

```c
	container_size = sizeof(struct ao2_container_hash) + num_buckets * sizeof(struct hash_bucket);
```

So a hash container is a single allocation containing its own bucket array, and
`n_buckets` is immutable for the container's lifetime. There is no rehashing in
ao2. Pick the number at allocation time or live with it.

## 4.2 The vtable: eight slots, seven filled

```c
struct ao2_container_methods {
	/*! Destroy this container. */
	ao2_container_destroy_fn destroy;
	/*! \brief Create an empty copy of this container. */
	ao2_container_alloc_empty_clone_fn alloc_empty_clone;
	/*! Create a new container node. */
	ao2_container_new_node_fn new_node;
	/*! Insert a node into this container. */
	ao2_container_insert_fn insert;
	/*! Traverse the container, find the first node. */
	ao2_container_find_first_fn traverse_first;
	/*! Traverse the container, find the next node. */
	ao2_container_find_next_fn traverse_next;
	/*! Traverse the container, cleanup state. */
	ao2_container_find_cleanup_fn traverse_cleanup;
	/*! Find the next iteration element in the container. */
	ao2_iterator_next_fn iterator_next;
#if defined(AO2_DEBUG)
	/*! Increment the container linked object statistic. */
	ao2_link_node_stat_fn link_stat;
	/*! Decrement the container linked object statistic. */
	ao2_unlink_node_stat_fn unlink_stat;
	/*! Display container contents. (Method for debug purposes) */
	ao2_container_display dump;
	/*! Display container debug statistics. (Method for debug purposes) */
	ao2_container_statistics stats;
	/*! Perform an integrity check on the container. (Method for debug purposes) */
	ao2_container_integrity integrity;
#endif	/* defined(AO2_DEBUG) */
};
```

Eight function pointers, then five more under `AO2_DEBUG`. The interesting one
is the seventh:

```c
	/*! Traverse the container, cleanup state. */
	ao2_container_find_cleanup_fn traverse_cleanup;
```

Neither backend sets it. Here are both initialisers in full:

```c
static const struct ao2_container_methods v_table_hash = {
	.alloc_empty_clone = (ao2_container_alloc_empty_clone_fn) hash_ao2_alloc_empty_clone,
	.new_node = (ao2_container_new_node_fn) hash_ao2_new_node,
	.insert = (ao2_container_insert_fn) hash_ao2_insert_node,
	.traverse_first = (ao2_container_find_first_fn) hash_ao2_find_first,
	.traverse_next = (ao2_container_find_next_fn) hash_ao2_find_next,
	.iterator_next = (ao2_iterator_next_fn) hash_ao2_iterator_next,
	.destroy = (ao2_container_destroy_fn) hash_ao2_destroy,
#if defined(AO2_DEBUG)
	.link_stat = hash_ao2_link_node_stat,
	.unlink_stat = hash_ao2_unlink_node_stat,
	.dump = (ao2_container_display) hash_ao2_dump,
	.stats = (ao2_container_statistics) hash_ao2_stats,
	.integrity = (ao2_container_integrity) hash_ao2_integrity,
#endif	/* defined(AO2_DEBUG) */
};
```

```c
static const struct ao2_container_methods v_table_rbtree = {
	.alloc_empty_clone = (ao2_container_alloc_empty_clone_fn) rb_ao2_alloc_empty_clone,
	.new_node = (ao2_container_new_node_fn) rb_ao2_new_node,
	.insert = (ao2_container_insert_fn) rb_ao2_insert_node,
	.traverse_first = (ao2_container_find_first_fn) rb_ao2_find_first,
	.traverse_next = (ao2_container_find_next_fn) rb_ao2_find_next,
	.iterator_next = (ao2_iterator_next_fn) rb_ao2_iterator_next,
	.destroy = (ao2_container_destroy_fn) rb_ao2_destroy,
#if defined(AO2_DEBUG)
	.dump = (ao2_container_display) rb_ao2_dump,
	.stats = (ao2_container_statistics) rb_ao2_stats,
	.integrity = (ao2_container_integrity) rb_ao2_integrity,
#endif	/* defined(AO2_DEBUG) */
};
```

`traverse_cleanup` appears in neither, and the engine's sole call site is
guarded:

```c
	}
	if (self->v_table->traverse_cleanup) {
		self->v_table->traverse_cleanup(traversal_state);
	}
```

So the slot is dead code in the `24` branch — a designated-initialiser
`static const` leaves it NULL, the guard sees NULL, nothing happens. It is worth
knowing for two reasons. First, if you ever add a third backend whose traversal
state owns a resource, the hook is already wired. Second, it means the
`v_table`-validity checks the engine performs are per-slot and not
"is this table complete". There are four that assert: `__ao2_link` checks exactly
`new_node` and `insert` (`main/astobj2_container.c:106`),
`internal_ao2_traverse` checks exactly `traverse_first` and `traverse_next`
(`:248-249`), `__ao2_container_clone` checks exactly `alloc_empty_clone`
(`:763`), and `__ao2_iterator_next` checks exactly `iterator_next` (`:567-569`).
The first three assert with the same string, `"invalid container v_table"`
(`:108`, `:251`, `:765`), so a crash report naming it tells you which operation
was attempted but not which slot was missing; the iterator check is the one that
names itself, `"invalid iterator container v_table"` (`:569`). Three further
per-slot checks do not assert at all — `dump` (`:807`), `stats` (`:847`) and
`integrity` (`:866-868`) silently do nothing when their slot is empty.

The `AO2_DEBUG` half is more lopsided. `link_stat` and `unlink_stat` are
hash-only, and they too are NULL-guarded at both call sites
(`main/astobj2_container.c:135-136` and `:64-65`). They maintain the
per-bucket `elements`/`max_elements` counters that `struct hash_bucket` carries
only under `AO2_DEBUG`; a red-black tree has no buckets, so it has nothing to
count and leaves the slots empty. Both backends do fill `stats`, so
`astobj2 container stats` works on either — it just prints entirely different
things: a bucket histogram from `hash_ao2_stats`
(`main/astobj2_hash.c:817-829`) or a rebalancing histogram from `rb_ao2_stats`
(`main/astobj2_rbtree.c:1729-1749`).

| Slot | Filled by hash | Filled by rbtree | Called from |
|---|---|---|---|
| `destroy` | `hash_ao2_destroy` | `rb_ao2_destroy` | `container_destruct` (`main/astobj2_container.c:645-647`) |
| `alloc_empty_clone` | `hash_ao2_alloc_empty_clone` | `rb_ao2_alloc_empty_clone` | `__ao2_container_clone` (`:769`) |
| `new_node` | `hash_ao2_new_node` | `rb_ao2_new_node` | `__ao2_link` (`:120`) |
| `insert` | `hash_ao2_insert_node` | `rb_ao2_insert_node` | `__ao2_link` (`:129`) |
| `traverse_first` | `hash_ao2_find_first` | `rb_ao2_find_first` | `internal_ao2_traverse` (`:318`) |
| `traverse_next` | `hash_ao2_find_next` | `rb_ao2_find_next` | `internal_ao2_traverse` (`:320`) |
| `traverse_cleanup` | *(NULL)* | *(NULL)* | `internal_ao2_traverse` (`:384-385`), guarded |
| `iterator_next` | `hash_ao2_iterator_next` | `rb_ao2_iterator_next` | `__ao2_iterator_next` (`:593`) |
| `link_stat` | `hash_ao2_link_node_stat` | *(NULL)* | `__ao2_link` (`:135-136`), guarded |
| `unlink_stat` | `hash_ao2_unlink_node_stat` | *(NULL)* | `__container_unlink_node_debug` (`:64-65`), guarded |
| `dump` | `hash_ao2_dump` | `rb_ao2_dump` | `ao2_container_dump` (`:807-809`) |
| `stats` | `hash_ao2_stats` | `rb_ao2_stats` | `ao2_container_stats` (`:847-849`) |
| `integrity` | `hash_ao2_integrity` | `rb_ao2_integrity` | `ao2_container_check` (`:866-870`) |

## 4.3 Two backends, one of them wearing two hats

The hash backend's state is the bucket array and a hash function; the tree's is
a root pointer and, under `AO2_DEBUG`, a histogram of its own rebalancing cases:

```c
struct ao2_container_rbtree {
	/*!
	 * \brief Items common to all containers.
	 * \note Must be first in the specific container struct.
	 */
	struct ao2_container common;
	/*! Root node of the tree.  NULL if the tree is empty. */
	struct rbtree_node *root;
#if defined(AO2_DEBUG)
	struct {
		/*! Fixup insert left cases 1-3 */
		int fixup_insert_left[3];
		/*! Fixup insert right cases 1-3 */
		int fixup_insert_right[3];
		/*! Fixup delete left cases 1-4 */
		int fixup_delete_left[4];
		/*! Fixup delete right cases 1-4 */
		int fixup_delete_right[4];
		/*! Deletion of node with number of children (0-2). */
		int delete_children[3];
	} stats;
#endif	/* defined(AO2_DEBUG) */
};
```

Those `fixup_insert_left[3]`, `fixup_delete_right[4]` and `delete_children[3]`
counters are the only per-container statistics in the subsystem that describe an
algorithm rather than a population, and they are what `astobj2 container stats`
on a tree prints in place of the bucket histogram.

Choosing between the two is the one container decision that cannot be changed
later, and it comes down to three questions.

| | hash / list | rbtree |
|---|---|---|
| Lookup by key | O(1) amortised with a good `hash_fn`; O(n) with one bucket | O(log n) |
| Ordered traversal | only within a bucket, and only if `sort_fn` was given | always, globally |
| `sort_fn` required | no | **yes** — NULL is rejected at allocation (`main/astobj2_rbtree.c:2028-2032`) |
| Partial-key search | yes, but as a scan of every bucket — `hash_fn` is not consulted (`main/astobj2_hash.c:367-371`) | yes, as a directed search via the bias machinery (`main/astobj2_rbtree.c:96-103`, `:1511-1524`) |
| Allocation sites outside ao2 | 233 | 26 |

The ratio in that last row is the honest summary: nine out of ten containers in
Asterisk are hash containers, and most of those want a key lookup and nothing
else.

## 4.4 Allocation: three public entry points, two implementations

The public surface is six macros — three plus a `_t_` tagged variant of each —
and all six expand to one of three `__`-prefixed functions, capturing
`__FILE__`, `__LINE__` and `__PRETTY_FUNCTION__` the way everything else in
`astobj2.h` does:

```c
#define ao2_container_alloc_hash(ao2_options, container_options, n_buckets, hash_fn, sort_fn, cmp_fn) \
	__ao2_container_alloc_hash((ao2_options), (container_options), (n_buckets), (hash_fn), (sort_fn), (cmp_fn), NULL,  __FILE__, __LINE__, __PRETTY_FUNCTION__)
```

Those three functions reduce to two. First the real one:

```c
struct ao2_container *__ao2_container_alloc_hash(unsigned int ao2_options,
	unsigned int container_options, unsigned int n_buckets, ao2_hash_fn *hash_fn,
	ao2_sort_fn *sort_fn, ao2_callback_fn *cmp_fn,
	const char *tag, const char *file, int line, const char *func)
{
	unsigned int num_buckets;
	size_t container_size;
	struct ao2_container_hash *self;

	num_buckets = hash_fn ? n_buckets : 1;
	container_size = sizeof(struct ao2_container_hash) + num_buckets * sizeof(struct hash_bucket);

	self = __ao2_alloc(container_size, container_destruct, ao2_options,
		tag ?: __PRETTY_FUNCTION__, file, line, func);
	return hash_ao2_container_init(self, container_options, num_buckets, hash_fn,
		sort_fn, cmp_fn);
}
```

Then the list "backend", in its entirety:

```c
struct ao2_container *__ao2_container_alloc_list(unsigned int ao2_options,
	unsigned int container_options, ao2_sort_fn *sort_fn, ao2_callback_fn *cmp_fn,
	const char *tag, const char *file, int line, const char *func)
{
	return __ao2_container_alloc_hash(ao2_options, container_options, 1, NULL,
		sort_fn, cmp_fn, tag, file, line, func);
}
```

That is the whole of it. `ao2_container_alloc_list` has no struct, no vtable, no
file and no code beyond those seven lines; a list container reports
`v_table_hash`, allocates a `struct ao2_container_hash`, and is destroyed by
`hash_ao2_destroy`. If you are reading a stack trace and wondering why a list
container is inside `hash_ao2_insert_node`, this is why.

The shared initialiser is where the vtable and the hash function are bound:

```c
static struct ao2_container *hash_ao2_container_init(
	struct ao2_container_hash *self, unsigned int options, unsigned int n_buckets,
	ao2_hash_fn *hash_fn, ao2_sort_fn *sort_fn, ao2_callback_fn *cmp_fn)
{
	if (!self) {
		return NULL;
	}

	self->common.v_table = &v_table_hash;
	self->common.sort_fn = sort_fn;
	self->common.cmp_fn = cmp_fn;
	self->common.options = options;
	self->hash_fn = hash_fn ? hash_fn : hash_zero;
	self->n_buckets = n_buckets;

#ifdef AO2_DEBUG
	ast_atomic_fetchadd_int(&ao2.total_containers, 1);
#endif	/* defined(AO2_DEBUG) */

	return (struct ao2_container *) self;
}
```

Note `self->hash_fn = hash_fn ? hash_fn : hash_zero;` at `:1057`. A list
container's hash function is not NULL — it is `hash_zero`
(`main/astobj2_hash.c:1028`), a function that returns 0 for every object. That
matters because `hash_ao2_new_node` calls it unconditionally:

```c
static struct hash_bucket_node *hash_ao2_new_node(struct ao2_container_hash *self, void *obj_new, const char *tag, const char *file, int line, const char *func)
{
	struct hash_bucket_node *node;
	int i;

	node = ao2_alloc_options(sizeof(*node), hash_ao2_node_destructor,
		AO2_ALLOC_OPT_LOCK_NOLOCK | AO2_ALLOC_OPT_NO_REF_DEBUG);
	if (!node) {
		return NULL;
	}

	i = abs(self->hash_fn(obj_new, OBJ_SEARCH_OBJECT) % self->n_buckets);

	__ao2_ref(obj_new, +1, tag ?: "Container node creation", file, line, func);
	node->common.obj = obj_new;
	node->common.my_container = (struct ao2_container *) self;
	node->my_bucket = i;

	return node;
}
```

`i = abs(self->hash_fn(obj_new, OBJ_SEARCH_OBJECT) % self->n_buckets)` at
`:217`. With `hash_zero` and `n_buckets == 1` that is always bucket 0, and the
`abs` is there because a user `hash_fn` is free to return a negative `int` —
`%` in C keeps the sign of the dividend, so without `abs` a negative hash would
index before the array.

Note also the `!self` check at the top of `hash_ao2_container_init` (`:1049`).
Both `*_container_init` functions begin with it, and both `*_alloc_*` functions
pass the result of `__ao2_alloc` straight in without checking. So an allocation
failure propagates as a NULL return from the macro, with no log line from the
container layer.

The tree's allocator is the one that refuses work:

```c
struct ao2_container *__ao2_container_alloc_rbtree(unsigned int ao2_options, unsigned int container_options,
	ao2_sort_fn *sort_fn, ao2_callback_fn *cmp_fn,
	const char *tag, const char *file, int line, const char *func)
{
	struct ao2_container_rbtree *self;

	if (!sort_fn) {
		/* Sanity checks. */
		ast_log(__LOG_ERROR, file, line, func, "Missing sort_fn()!\n");
		return NULL;
	}

	self = __ao2_alloc(sizeof(*self), container_destruct, ao2_options,
		tag ?: __PRETTY_FUNCTION__, file, line, func);
	return rb_ao2_container_init(self, container_options, sort_fn, cmp_fn);
}
```

The `ast_log(__LOG_ERROR, file, line, func, "Missing sort_fn()!\n")` uses the
caller's file and line rather than its own, so the error points at the
allocation site in the calling module — one of the few places in ao2 where a
misuse produces a message you can act on directly.

## 4.5 The bucket-collapse trap

Put `:1076` and `:1089-1090` next to each other:

```c
	num_buckets = hash_fn ? n_buckets : 1;
```

```c
	return __ao2_container_alloc_hash(ao2_options, container_options, 1, NULL,
		sort_fn, cmp_fn, tag, file, line, func);
```

The second line is how the list container is *meant* to use the collapse. The
first line is what happens to anyone who gets there by accident. The following
is the one code block in this chapter that is **not** quoted from the tree — it
is the mistake, written out:

```c
	/* Looks like a 563-bucket hash container.  Is a linked list. */
	c = ao2_container_alloc_hash(AO2_ALLOC_OPT_LOCK_MUTEX, 0, 563, NULL,
		NULL, my_cmp_fn);
```

No warning is logged, `ao2_container_count` works, `ao2_find` works, every test
passes, and lookups are O(n). The behaviour is not undocumented — the macro's
doxygen says so, at `include/asterisk/astobj2.h:1295`: *"\param hash_fn Pointer
to a function computing a hash value. (NULL if everyting goes in first
bucket.)"*, typo and all. But nothing enforces or logs it, and the bug is
invisible at the call site because the bucket count you asked for is simply
discarded. Three things make it easy to reach by mistake: `hash_fn` is the fourth of six arguments, so it is easy to put
in the wrong slot; `NULL` is a legal value for the neighbouring `sort_fn` and
`cmp_fn`, so the compiler cannot help; and the symptom is latent — a container
with twenty objects in it behaves acceptably either way.

Two ways to catch it. In an `AO2_DEBUG` build, `astobj2 container stats` on a
registered container opens with `Number of buckets:`
(`main/astobj2_hash.c:817`), and because the collapse happens *before* the
allocation — `num_buckets` is forced to 1 at `:1076` and that collapsed value is
what `:1058` stores — a collapsed container reports `1`. Nothing anywhere
remembers the 563 you asked for, so the discrepancy is between the CLI output
and the source line, not inside the container. Which is the second way: grep the
allocation site. If the argument immediately before `sort_fn` is `NULL`,
`n_buckets` is a lie, and the honest spelling of what you built is
`ao2_container_alloc_list`.

The converse mistake is cheaper but still worth naming: passing a hash function
and `n_buckets = 1`. Nothing collapses, because `hash_fn` is non-NULL, so
`num_buckets` is the 1 you asked for — you just pay for a hash computation on
every link and every find and get a linked list anyway.

## 4.6 `enum ao2_container_opts` and where the object lands

Container options are a separate word from the ao2 allocation options of chapter
01, and they are passed as the *second* argument, right after them. Six values
in one bit field:

```c
enum ao2_container_opts {
	/*!
	 * \brief Insert objects at the beginning of the container.
	 * (Otherwise it is the opposite; insert at the end.)
	 *
	 * \note If an ao2_sort_fn is provided, the object is inserted
	 * before any objects with duplicate keys.
	 *
	 * \note Hash containers insert the object in the computed hash
	 * bucket in the indicated manner.
	 */
	AO2_CONTAINER_ALLOC_OPT_INSERT_BEGIN = (1 << 0),

	/*!
	 * \brief The ao2 container objects with duplicate keys option field mask.
	 */
	AO2_CONTAINER_ALLOC_OPT_DUPS_MASK = (3 << 1),
	/*!
	 * \brief Allow objects with duplicate keys in container.
	 */
	AO2_CONTAINER_ALLOC_OPT_DUPS_ALLOW = (0 << 1),
	/*!
	 * \brief Reject objects with duplicate keys in container.
	 *
	 * \note The container must be sorted.  i.e. have an
	 * ao2_sort_fn.
	 */
	AO2_CONTAINER_ALLOC_OPT_DUPS_REJECT = (1 << 1),
	/*!
	 * \brief Reject duplicate objects in container.
	 *
	 * \details Don't link the same object into the container twice.
	 * However, you can link a different object with the same key.
	 *
	 * \note The container must be sorted.  i.e. have an
	 * ao2_sort_fn.
	 *
	 * \note It is assumed that the objects are located where the
	 * search key says they should be located.
	 */
	AO2_CONTAINER_ALLOC_OPT_DUPS_OBJ_REJECT = (2 << 1),
	/*!
	 * \brief Replace objects with duplicate keys in container.
	 *
	 * \details The existing duplicate object is removed and the new
	 * object takes the old object's place.
	 *
	 * \note The container must be sorted.  i.e. have an
	 * ao2_sort_fn.
	 */
	AO2_CONTAINER_ALLOC_OPT_DUPS_REPLACE = (3 << 1),
};
```

One position bit and a two-bit duplicate policy, which is the entire vocabulary.
Across the tree outside the ao2 implementation there are 46 mentions of these
constants against 259 container allocation sites, so the overwhelming majority
of containers are allocated with a container-options word of plain `0` —
insert-at-end, duplicates allowed.

`AO2_CONTAINER_ALLOC_OPT_INSERT_BEGIN` is the one with a per-backend reading,
and the enum's own `\note` says so: *"Hash containers insert the object in the
computed hash bucket in the indicated manner."* So on a hash container the flag
positions the object within its bucket and nowhere else, which — with a real
hash function — means it positions it relative to objects it mostly does not
share a bucket with. On a tree the flag is global, and it is consumed by being
translated into a search bias rather than applied at insertion:

```c
	switch (options & AO2_CONTAINER_ALLOC_OPT_DUPS_MASK) {
	default:
	case AO2_CONTAINER_ALLOC_OPT_DUPS_ALLOW:
		if (options & AO2_CONTAINER_ALLOC_OPT_INSERT_BEGIN) {
			bias = BIAS_FIRST;
		} else {
			bias = BIAS_LAST;
		}
		break;
	case AO2_CONTAINER_ALLOC_OPT_DUPS_REJECT:
	case AO2_CONTAINER_ALLOC_OPT_DUPS_OBJ_REJECT:
	case AO2_CONTAINER_ALLOC_OPT_DUPS_REPLACE:
		bias = BIAS_EQUAL;
		break;
	}
```

`BIAS_FIRST` or `BIAS_LAST` when duplicates are allowed, `BIAS_EQUAL` when they
are not: the flag decides which end of a run of equal keys the search for an
insertion point aims at. With any of the three rejecting or replacing policies
the bias is `BIAS_EQUAL`, and `INSERT_BEGIN` has no effect on where a *new* key
goes, because a sorted tree has exactly one place to put it. The tree backend
reads the flag in exactly two places — here at `:1030`, and again at `:1149`
inside the `DUPS_OBJ_REJECT` arm, where it decides whether a permitted duplicate
becomes a left or a right child of the existing run.

## 4.7 Duplicate handling: four policies, two implementations

| Policy | Value | Meaning | Documented requirement |
|---|---|---|---|
| `DUPS_ALLOW` | `0 << 1` | link it regardless | none |
| `DUPS_REJECT` | `1 << 1` | refuse any object with a key already present | must be sorted |
| `DUPS_OBJ_REJECT` | `2 << 1` | refuse the *same object* twice; a different object with the same key is fine | must be sorted |
| `DUPS_REPLACE` | `3 << 1` | the new object takes the existing one's place | must be sorted |

The "must be sorted" in the last column is the enum's own `\note The container
must be sorted.  i.e. have an ao2_sort_fn.`, repeated verbatim on all three
non-default policies. On a red-black tree it is enforced, because allocation
fails without a `sort_fn`. On a hash container nothing enforces it, and the
consequence is visible in the insert routine:

```c
				switch (options & AO2_CONTAINER_ALLOC_OPT_DUPS_MASK) {
				default:
				case AO2_CONTAINER_ALLOC_OPT_DUPS_ALLOW:
					break;
				case AO2_CONTAINER_ALLOC_OPT_DUPS_REJECT:
					/* Reject all objects with the same key. */
					return AO2_CONTAINER_INSERT_NODE_REJECTED;
				case AO2_CONTAINER_ALLOC_OPT_DUPS_OBJ_REJECT:
					if (cur->common.obj == node->common.obj) {
						/* Reject inserting the same object */
						return AO2_CONTAINER_INSERT_NODE_REJECTED;
					}
					break;
				case AO2_CONTAINER_ALLOC_OPT_DUPS_REPLACE:
					SWAP(cur->common.obj, node->common.obj);
					ao2_ref(node, -1);
					return AO2_CONTAINER_INSERT_NODE_OBJ_REPLACED;
				}
```

That switch is inside `if (sort_fn) {`. The whole bucket walk is. A hash
container allocated with `AO2_CONTAINER_ALLOC_OPT_DUPS_REJECT` and a NULL
`sort_fn` therefore never evaluates the policy at all: it falls through to
`AST_DLLIST_INSERT_HEAD` (`main/astobj2_hash.c:282`) or
`AST_DLLIST_INSERT_TAIL` (`:315`) and links the duplicate, silently, as though
`DUPS_ALLOW` had been requested. `ao2_link` returns 1. There is no error path
because the code that would reject is unreachable.

The switch appears twice in `hash_ao2_insert_node`, once per insertion
direction — `:261-278` under `INSERT_BEGIN`, `:294-311` otherwise — and the two
copies are identical in effect. The bucket is chosen before either runs, from
the node's own `my_bucket`, which `new_node` already computed:

```c
	bucket = &self->buckets[node->my_bucket];
```

That is the second thing to know about duplicate handling on a hash container:
*it is per-bucket*. The walk is a walk of one bucket's doubly-linked list, so
`DUPS_REJECT` rejects a key already present **in the same bucket**. With a
correct `hash_fn` — one that maps equal keys to equal buckets — that is the same
thing as rejecting a key present in the container. With an inconsistent
`hash_fn`, one whose `OBJ_SEARCH_OBJECT` and `OBJ_SEARCH_KEY` paths can disagree
for the same logical key, it is not, and you get duplicates past a policy that
forbids them. The tree has no equivalent failure mode: there is one ordering and
`rb_ao2_insert_node` walks all of it.

The tree's answer to hash-function inconsistency is a set of macros that
generate the function for you, from a struct type and a field name, so that the
`OBJ_SEARCH_OBJECT` and `OBJ_SEARCH_KEY` arms cannot disagree:
`AO2_FIELD_HASH_FN` (`include/asterisk/astobj2.h:1957`) and the string
specialisation `AO2_STRING_FIELD_HASH_FN` (`:2032`), with matching
`AO2_FIELD_CMP_FN` (`:1999`), `AO2_STRING_FIELD_CMP_FN` (`:2048`) and
`AO2_STRING_FIELD_SORT_FN` (`:2064`). Use them in preference to a hand-written
`hash_fn` whenever the key is a struct field. Note that the generated hash
function `ast_assert(0)`s on `OBJ_SEARCH_PARTIAL_KEY`
(`include/asterisk/astobj2.h:1969-1971`), which is harmless only because
`hash_ao2_find_first` never calls `hash_fn` on the partial-key path
(`main/astobj2_hash.c:367-371`).

The tree's duplicate arm is also where the one unreachable-by-construction
assert in the backend lives:

```c
	/* Node is a duplicate */
	switch (options & AO2_CONTAINER_ALLOC_OPT_DUPS_MASK) {
	default:
	case AO2_CONTAINER_ALLOC_OPT_DUPS_ALLOW:
		ast_assert(0);/* Case already handled by BIAS_FIRST/BIAS_LAST. */
		return AO2_CONTAINER_INSERT_NODE_REJECTED;
	case AO2_CONTAINER_ALLOC_OPT_DUPS_REJECT:
		/* Reject all objects with the same key. */
		return AO2_CONTAINER_INSERT_NODE_REJECTED;
```

`ast_assert(0);/* Case already handled by BIAS_FIRST/BIAS_LAST. */` — under
`DUPS_ALLOW` the bias was `BIAS_FIRST` or `BIAS_LAST`, the search for an
insertion point ran off the end of any run of equal keys, and control cannot
reach the "Node is a duplicate" switch. Reaching it means the bias machinery is
inconsistent with the options word, which is a backend bug rather than a caller
one. `DUPS_OBJ_REJECT` on a tree is the expensive policy: it walks the entire run of
equal keys looking for pointer identity, in both directions —
`rb_node_next_full` forward (`:1151-1181`) and `rb_node_prev_full` backward
(`:1192-1222`) — because "the same object" cannot be answered by the sort
function.

`DUPS_REPLACE` is the policy with a refcount consequence, and both backends
spell it the same way:

```c
				case AO2_CONTAINER_ALLOC_OPT_DUPS_REPLACE:
					SWAP(cur->common.obj, node->common.obj);
					ao2_ref(node, -1);
					return AO2_CONTAINER_INSERT_NODE_OBJ_REPLACED;
```

The *node* stays in the container and the objects trade places, so the new
object inherits the existing node and the fresh node — now holding the old
object — is unreferenced. Dropping that node's last reference runs the node
destructor, which is what releases the old object
(`main/astobj2_hash.c:186-188`).

It is worth being precise about what that destructor does *not* do here, because
chapter 03 §3.9 warned that the node destructor promotes the container lock. It
does, but only for a node that is linked: the promotion lives inside
`if (doomed->common.is_linked) {` at `main/astobj2_hash.c:147`, and
`is_linked` is set by the engine at `main/astobj2_container.c:131` — a line the
`OBJ_REPLACED` return never reaches. The displaced node was never linked, so its
destructor skips the whole block and goes straight to the object unref. A
`DUPS_REPLACE` link does not drop the container's lock. The chapter-03 hazard is
real but it belongs to node references obtained from a traversal or an iterator,
which is chapter 05's subject.

## 4.8 Linking: `enum ao2_container_insert`

The backends do not report success and failure. They report one of three
outcomes, from a private enum:

```c
enum ao2_container_insert {
	/*! The node was inserted into the container. */
	AO2_CONTAINER_INSERT_NODE_INSERTED,
	/*! The node object replaced an existing node object. */
	AO2_CONTAINER_INSERT_NODE_OBJ_REPLACED,
	/*! The node was rejected (duplicate). */
	AO2_CONTAINER_INSERT_NODE_REJECTED,
};
```

And `__ao2_link` is the only thing that reads it:

```c
		/* Insert the new node. */
		switch (self->v_table->insert(self, node)) {
		case AO2_CONTAINER_INSERT_NODE_INSERTED:
			node->is_linked = 1;
			ast_atomic_fetchadd_int(&self->elements, 1);
#if defined(AO2_DEBUG)
			AO2_DEVMODE_STAT(++self->nodes);
			if (self->v_table->link_stat) {
				self->v_table->link_stat(self, node);
			}
#endif	/* defined(AO2_DEBUG) */
			/* Fall through */
		case AO2_CONTAINER_INSERT_NODE_OBJ_REPLACED:
#if defined(AO2_DEBUG)
			if (ao2_container_check(self, OBJ_NOLOCK)) {
				ast_log(LOG_ERROR, "Container integrity failed after insert or replace.\n");
			}
#endif	/* defined(AO2_DEBUG) */
			res = 1;
			break;
		case AO2_CONTAINER_INSERT_NODE_REJECTED:
			ao2_ref(node, -1);
			break;
		}
```

Three things in that switch matter.

The `/* Fall through */` from `NODE_INSERTED` into `NODE_OBJ_REPLACED` is the
reason the two cases share the `res = 1`. Both are successes from the caller's
point of view and `ao2_link` cannot distinguish them — a replace returns 1
exactly as an insert does. If you need to know whether you displaced something,
the container cannot tell you; you have to `ao2_find` first.

`node->is_linked = 1` and `ast_atomic_fetchadd_int(&self->elements, 1)` happen
only on `NODE_INSERTED`. A replace leaves both alone, correctly: the node was
already linked and the population did not change.

`NODE_REJECTED` does `ao2_ref(node, -1)` and leaves `res` at 0. That single
unref is what releases the reference the backend's `new_node` took on the object
at `main/astobj2_hash.c:219` — the node destructor drops it. So a rejected
`ao2_link` leaves the caller's reference count exactly as it was, and
`ao2_link`'s return value of 0 is the only signal that anything was refused. It
is the same 0 you get from a NULL `new_node` (an allocation failure) and the
same 0 you get from a non-ao2 argument, which is why "`ao2_link` returned 0" is
rarely enough information on its own.

`__ao2_link` takes `ao2_wrlock(self)` for the whole operation
(`main/astobj2_container.c:115`), or promotes an already-held lock when passed
`OBJ_NOLOCK` (`:113`). So the backend `insert` always runs under whatever
exclusion the container's lock mode provides, already established by the engine
— which is why no backend acquires a lock of its own. On an
`AO2_ALLOC_OPT_LOCK_NOLOCK` container that exclusion is nothing at all, and the
engine is unbothered: the `OBJ_MULTIPLE` result container is allocated exactly
that way (`main/astobj2_container.c:268`) precisely because no other thread can
see it.

## 4.9 Destruction, and the node-leak assert

A container is destroyed like any other ao2 object — the last `ao2_ref(-1)` runs
the destructor passed to `__ao2_alloc`, which for every container is
`container_destruct`:

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

The order is the point. `destroying = 1` first, then a single
`OBJ_UNLINK | OBJ_NODATA | OBJ_MULTIPLE` callback with a NULL callback function,
which unlinks and unreferences every object, and only then the backend's
`destroy`. By the time a backend's `destroy` runs the container is supposed to
be empty, and both backends check exactly that:

```c
static void hash_ao2_destroy(struct ao2_container_hash *self)
{
	int idx;

	/* Check that the container no longer has any nodes */
	for (idx = self->n_buckets; idx--;) {
		if (!AST_DLLIST_EMPTY(&self->buckets[idx].list)) {
			ast_log(LOG_ERROR, "Node ref leak.  Hash container still has nodes!\n");
			ast_assert(0);
			break;
		}
	}
}
```

```c
static void rb_ao2_destroy(struct ao2_container_rbtree *self)
{
	/* Check that the container no longer has any nodes */
	if (self->root) {
		ast_log(LOG_ERROR, "Node ref leak.  Red-Black tree container still has nodes!\n");
		ast_assert(0);
	}
}
```

`"Node ref leak.  Hash container still has nodes!"` followed by `ast_assert(0)`.
This fires when something outside the container still holds a reference to a
*node* — not to a stored object, which is normal and fine, but to the node
itself. The only way to acquire a node reference is through
`traverse_first`/`traverse_next` or the iterator, so in practice this message
means a traversal or an `ao2_iterator` leaked, and the container's last object
reference went away before that leaked node reference did.
[Chapter 05](05-traversal-and-iterators.md) is about who owns those
references.

Note what the destructor does *not* do: it never takes the container's lock.
That is chapter 02's destructor-runs-unlocked rule, and `container_destruct`
depends on it, because the `ao2_t_callback` at `:641` takes the lock itself.
Setting `destroying` before that call is safe for the same reason the rule
exists — if another thread could still reach the container, the refcount would
not have hit zero.

## 4.10 The compile-time traversal-state budget

A traversal needs per-call state, and the engine allocates it without knowing
which backend will use it:

```c
	/* Create a buffer for the traversal state. */
	traversal_state = alloca(AO2_TRAVERSAL_STATE_SIZE);
```

`AO2_TRAVERSAL_STATE_SIZE` is 100 (`main/astobj2_container_private.h:60`), the
buffer is `void *`, and each backend casts it to its own state struct. Nothing
at that call site knows how big the state actually needs to be — so each backend
asserts its own fit, at compile time, with a struct that exists only to be
declared:

```c
struct hash_traversal_state_check {
	/*
	 * If we have a division by zero compile error here then there
	 * is not enough room for the state.  Increase AO2_TRAVERSAL_STATE_SIZE.
	 */
	char check[1 / (AO2_TRAVERSAL_STATE_SIZE / sizeof(struct hash_traversal_state))];
};
```

```c
struct rbtree_traversal_state_check {
	/*
	 * If we have a division by zero compile error here then there
	 * is not enough room for the state.  Increase AO2_TRAVERSAL_STATE_SIZE.
	 */
	char check[1 / (AO2_TRAVERSAL_STATE_SIZE / sizeof(struct rbtree_traversal_state))];
};
```

`char check[1 / (AO2_TRAVERSAL_STATE_SIZE / sizeof(struct hash_traversal_state))]`
is a division-by-zero trap dressed as an array bound. If the state fits, the
inner division is at least 1 and the array bound is a legal 0 or 1. If the state
exceeds 100 bytes the inner division truncates to 0 and the outer division is
`1 / 0` — a constant expression the compiler must reject. The comment says
exactly what to do about it: *"Increase AO2_TRAVERSAL_STATE_SIZE."* This predates
`_Static_assert` being usable here and does the same job with no dependencies.
It is worth noting what the passing case actually declares: with the real sizes
the bound is `1 / (100 / 32)` and `1 / (100 / 24)`, both of which are 0, so both
backends declare a zero-length array — a GCC extension rather than ISO C, and
harmless because the struct is never instantiated.

It is a floor, not an exact fit. `struct hash_traversal_state` is 32 bytes and
`struct rbtree_traversal_state` is 24 on a 64-bit build, so the budget is three
to four times what either backend needs, and the check would also pass at 51
bytes or 99. If you add a backend, the number to grow is the one in the private
header, and growing it costs stack on every traversal in the process.

## 4.11 Clone, and locking a container nobody else can see

`ao2_container_clone` is the one routine that needs the `alloc_empty_clone`
slot, and the reason it exists as a vtable entry rather than a generic function
is that "the same kind of container, configured the same way" is knowledge only
the backend has:

```c
static struct ao2_container *hash_ao2_alloc_empty_clone(struct ao2_container_hash *self,
	const char *tag, const char *file, int line, const char *func)
{
	if (!__is_ao2_object(self, file, line, func)) {
		return NULL;
	}

	return __ao2_container_alloc_hash(ao2_options_get(self), self->common.options,
		self->n_buckets, self->hash_fn, self->common.sort_fn, self->common.cmp_fn,
		tag, file, line, func);
}
```

```c
static struct ao2_container *rb_ao2_alloc_empty_clone(struct ao2_container_rbtree *self,
	const char *tag, const char *file, int line, const char *func)
{
	if (!__is_ao2_object(self, file, line, func)) {
		return NULL;
	}

	return __ao2_container_alloc_rbtree(ao2_options_get(self), self->common.options,
		self->common.sort_fn, self->common.cmp_fn, tag, file, line, func);
}
```

Both reconstruct themselves by calling their own allocator with their own
settings — including `ao2_options_get(self)`, so the clone inherits the
original's *lock mode* as well as its container options. A clone of an rwlock
container is an rwlock container.

The `OBJ_NOLOCK` handling in `__ao2_container_clone` is worth reading alongside
chapter 03, because `ao2_wrlock(clone)` at `main/astobj2_container.c:778` is the
only `ao2_*lock` call in the subsystem on a container no other thread can reach:

```c
	/* This test is correct.  clone must be locked before calling
	 * ao2_container_dup when the OBJ_NOLOCK flag is set, otherwise
	 * we could have errors in __adjust_lock. */
	if (flags & OBJ_NOLOCK) {
		ao2_wrlock(clone);
	}
```

`ao2_container_dup` links into `clone` with `OBJ_NOLOCK`
(`main/astobj2_container.c:670`, via `dup_obj_cb`), and `OBJ_NOLOCK` means
"`__adjust_lock` will adjust a lock I already hold". The clone is brand new and
unlocked, so somebody has to take that lock first, and it is this function. The
`/* This test is correct. */` comment is there because the code reads like a
mistake: the flag asks for *no* locking and the function responds by locking.
Compare the `OBJ_MULTIPLE` result container, which has the same "nobody can see
it" property and solves it the other way, by being allocated
`AO2_ALLOC_OPT_LOCK_NOLOCK` (`:268`) so that the `OBJ_NOLOCK` link at `:356`
has no lock to adjust.

`ao2_container_dup` (`:673`) and `ao2_container_dup_weakproxy_objs` (`:726`) are
the other two public container-to-container operations. The second exists
because copying a container of weak proxies usually means wanting the objects
they point at rather than the proxies themselves; it is chapter 06's subject.

## 4.12 Symptom → cause

| Symptom | Cause |
|---|---|
| `astobj2 container stats` reports `Number of buckets: 1` for a container whose allocation site asked for hundreds | `hash_fn` was NULL, so `num_buckets` collapsed to 1 before the allocation and `n_buckets` stores the collapsed value (`main/astobj2_hash.c:1076`, `:1058`, `:817`) |
| `ao2_container_alloc_rbtree` returns NULL and logs `Missing sort_fn()!` at your line number | the tree backend rejects an unsorted container (`main/astobj2_rbtree.c:2028-2032`) |
| `AO2_CONTAINER_ALLOC_OPT_DUPS_REJECT` lets duplicates in | no `sort_fn`, so the whole policy switch is inside unreachable code (`main/astobj2_hash.c:251`, `:284`) |
| `DUPS_REJECT` lets duplicates in on a container that *has* a `sort_fn` | the walk is per-bucket; an inconsistent `hash_fn` put equal keys in different buckets (`main/astobj2_hash.c:246`) |
| A stack trace shows a *list* container inside `hash_ao2_insert_node` | there is no list backend; `ao2_container_alloc_list` is a one-bucket hash container (`main/astobj2_hash.c:1085-1091`) |
| `Node ref leak.  Hash container still has nodes!` plus an assert at shutdown | a traversal or `ao2_iterator` leaked a node reference; the container outlived its own objects (`main/astobj2_hash.c:737-740`) |
| `ao2_link` returned 1 but the object you passed is not the one in the container | `DUPS_REPLACE` swapped the objects and kept the node (`main/astobj2_hash.c:274-277`) |
| `__ast_assert_failed` with `"invalid container v_table"` | one of three per-operation checks fired on a missing slot, and all three share the string, so read the backtrace (`main/astobj2_container.c:106`, `:248-249`, `:763`); the fourth check has its own string, `"invalid iterator container v_table"` (`:567-569`) |
| `ao2_container_count` disagrees with what a traversal visits | the count is an atomic read of a snapshot, taken without the lock (`main/astobj2_container.c:36`) |
| `astobj2 container stats` prints no bucket table for one container | it is an rbtree; that backend fills the `stats` slot too, but with tree fixup and delete counts instead of buckets (`main/astobj2_hash.c:817-829` vs `main/astobj2_rbtree.c:1729-1749`) |
| A compile error about division by zero in `astobj2_hash.c` or `astobj2_rbtree.c` | a traversal state struct grew past `AO2_TRAVERSAL_STATE_SIZE` (`main/astobj2_container_private.h:60`) |
| Growing a container's bucket count at runtime has no effect | there is no rehash; `buckets[0]` is sized once in the single allocation (`main/astobj2_hash.c:1077`) |

## 4.13 Rules

| Rule | Source |
|---|---|
| Never pass a NULL `hash_fn` to `ao2_container_alloc_hash` — say `ao2_container_alloc_list` instead | `main/astobj2_hash.c:1076`, `:1085-1091` |
| A duplicate policy other than `DUPS_ALLOW` requires a `sort_fn`, and only the rbtree backend enforces it | `include/asterisk/astobj2.h:1188`, `:1201`, `:1211`; `main/astobj2_rbtree.c:2028-2032` |
| A `hash_fn` must send an object and a key that matches it to the same bucket — insertion hashes with `OBJ_SEARCH_OBJECT`, lookup with whatever the caller passed | `main/astobj2_hash.c:217`, `:363-364` |
| Pick `n_buckets` at allocation time; there is no rehash | `main/astobj2_hash.c:1077` |
| Treat `ao2_link` returning 1 as "inserted or replaced", not "inserted" | `main/astobj2_container.c:139-146` |
| Release every node reference a traversal or iterator gave you before the container's last object reference goes | `main/astobj2_hash.c:737-740`, `main/astobj2_rbtree.c:1669-1672` |
| Do not treat `ao2_container_count` as a value you can act on without the container locked | `main/astobj2_container.c:36` |
| A new backend must keep its common struct first and its traversal state under 100 bytes | `main/astobj2_container_private.h:60`, `main/astobj2_hash.c:67-70` |
| The backend `v_table` is validated per operation, not as a whole; a new backend can compile and link and still assert | `main/astobj2_container.c:106`, `:248-249`, `:763`, `:567-569` |

## Next

[Chapter 05](05-traversal-and-iterators.md) is about getting objects back
out: `enum search_flags` field by field, the generic traversal engine the two `traverse_*` slots feed, who owns
the reference `ao2_find` hands back and what `OBJ_UNLINK` transfers,
`OBJ_MULTIPLE` and the container it returns, `OBJ_NOLOCK` and the lock
restoration chapter 03 deferred, and `ao2_iterator`.
