# ao2 Objects, Refcounting and Containers — A Source-Level Guide

**Target tree:** Asterisk, the `24` branch

Every `file:line` citation in this guide was verified against the `24` branch of
`/usr/src/phzyx/asterisk-baseline`. Newer releases and other branches may
differ — line numbers drift — so if you are reading this against a different
tree, treat the citations as approximate and re-grep for the symbol name.

## Who this is for

A C developer working inside the Asterisk source tree who has to reason about
who owns a pointer. ao2 is not a data-structure library you can take or leave:
`ao2_ref` is called from 2,961 sites outside the ao2 implementation itself and
`RAII_VAR` from 2,727. Almost every other subsystem is unreadable until you can
see the refcount discipline underneath it.

This guide is the companion to [the channels guide](../channels/README.md),
which spends a whole chapter on the difference between a reference and a lock
without ever explaining where either one lives. This one goes underneath it.

## The one-page mental model

An ao2 object is not a struct you declare. It is a pointer into the *middle* of
an allocation, with a private header behind it and, depending on how you asked
for it, a lock behind that:

```
   ao2_alloc(sizeof(struct foo), foo_dtor, AO2_ALLOC_OPT_LOCK_MUTEX)
            │
            ▼
   ┌──────────────────┬───────────────────────┬─────────────────────┐
   │ struct           │ struct __priv_data    │ your struct foo     │
   │ ao2_lock_priv    │                       │                     │
   │                  │  destructor_fn        │  (data_size bytes,  │
   │  ast_mutex_t     │  weakptr              │   calloc'd to zero) │
   │                  │  ref_counter          │                     │
   │                  │  options:3            │                     │
   │                  │  lockused:1           │                     │
   │                  │  magic:28             │                     │
   └──────────────────┴───────────────────────┴─────────────────────┘
                                              ▲
                                              └── what ao2_alloc returns,
                                                  and the ONLY address any
                                                  caller outside astobj2.c
                                                  ever holds

   Four layouts, chosen by the two option bits (main/astobj2.c:104-148):

     LOCK_MUTEX   struct astobj2_lock      mutex  + priv + user
     LOCK_RWLOCK  struct astobj2_rwlock    rwlock + priv + user
     LOCK_NOLOCK  struct astobj2           priv   + user
     LOCK_OBJ     struct astobj2_lockobj   void*  + priv + user
                                           (borrows another object's lock)
```

Everything else in the subsystem follows from that picture. Getting from the
user pointer back to the header is pointer arithmetic by a size that depends on
the option bits, which is why there are four `INTERNAL_OBJ_*` macros
(`main/astobj2.c:154-164`) and why handing a non-ao2 pointer to `ao2_ref` is a
wild read rather than a clean error.

Three facts carry most of the weight:

1. **The destructor runs with no lock held.** `__ao2_ref` calls
   `obj->priv_data.destructor_fn(user_data)` at `main/astobj2.c:614-616`
   without taking the object's own lock — it never had it. Anything your
   destructor touches that another thread could also be touching is a race, and
   the usual fix is not to lock inside the destructor but to arrange that the
   last reference is only droppable once nobody else can reach the object.
   Chapter 02.
2. **A reference is a count, not a handle.** The count is a single `int32_t`
   mutated with `ast_atomic_fetch_add(..., __ATOMIC_RELAXED)`
   (`main/astobj2.c:530`). Nothing records *who* holds a reference unless you
   compile with `REF_DEBUG` and pass tags, and even then the tag is a string
   you wrote. This is why leaked channels are hunted by reading a log rather
   than by asking the object. Chapter 07.
3. **A list container is a hash container.** `ao2_container_alloc_list` calls
   `__ao2_container_alloc_hash` with one bucket and no hash function
   (`main/astobj2_hash.c:1085-1091`), and `__ao2_container_alloc_hash` silently
   forces `num_buckets` to 1 whenever `hash_fn` is NULL (`:1076`). There are two
   container implementations in the tree, not three. Chapter 04.

## The guide

| File | Covers | Status |
|---|---|---|
| [01-objects-and-allocation.md](01-objects-and-allocation.md) | the hidden `__priv_data` header and the four layouts, the magic number and why one check accepts weak proxies, `enum ao2_alloc_opts`, `internal_ao2_alloc` branch by branch, the unreachable `default:`, `ao2_alloc_with_lockobj` | written |
| [02-refcounting-and-destruction.md](02-refcounting-and-destruction.md) | `__ao2_ref` end to end, the relaxed atomic, the destructor-runs-unlocked rule, the teardown switch and what each lock mode frees, `ao2_bump`/`ao2_cleanup`/`ao2_replace`, `RAII_VAR`, the excessive-refcount tripwire | written |
| [03-locking.md](03-locking.md) | the four lock modes as a user sees them, `__adjust_lock` and lock promotion, `ao2_lock` vs `ao2_rdlock` vs `ao2_wrlock` on an object that has the wrong kind of lock, `SCOPED_AO2LOCK`, `lockused`, lock-object borrowing and its ref | written |
| [04-containers.md](04-containers.md) | `struct ao2_container` and its 8-member vtable (13 under `AO2_DEBUG`), the hash and rbtree backends and which slots each fills, `enum ao2_container_opts`, duplicate handling, the bucket-collapse trap | written |
| [05-traversal-and-iterators.md](05-traversal-and-iterators.md) | `enum search_flags` field by field, the generic traversal engine, who owns the returned reference and the `OBJ_UNLINK` transfer, `OBJ_MULTIPLE` and the result container, `OBJ_NOLOCK` and lock restoration, the iterator | written |
| [06-weak-proxies-and-global-objects.md](06-weak-proxies-and-global-objects.md) | `ao2_weakproxy` and the `weakptr` field both objects share, subscription callbacks, the `AO2_GLOBAL_OBJ_STATIC` holder and why a global object is read through a function | written |
| `07-debugging.md` | `REF_DEBUG` and the `refs` log line formats, `refcounter.py` and `refstats.py`, `AO2_DEBUG` and the five CLI commands, container integrity checks, the compile-time traversal-state budget | planned |

## Canonical source files

| Path | Role |
|---|---|
| `include/asterisk/astobj2.h` | The entire public surface: 2,069 lines, almost all of it macros that capture `__FILE__`/`__LINE__` |
| `main/astobj2.c` | Objects: allocation, refcounting, destruction, locking, weak proxies (1,194 lines) |
| `main/astobj2_private.h` | The `is_ao2_object` check and `__adjust_lock`, shared by the four implementation files (66 lines) |
| `main/astobj2_container.c` | The generic container engine: link, unlink, traversal, iterators, CLI (1,196 lines) |
| `main/astobj2_container_private.h` | `struct ao2_container`, the node type and the vtable template (318 lines) |
| `main/astobj2_hash.c` | The hash backend, which is also the list backend (1,091 lines) |
| `main/astobj2_rbtree.c` | The red-black tree backend (2,037 lines) |
| `main/astobj2_global.c` | The global-object holder (97 lines) |

## Reading order

Read [01-objects-and-allocation.md](01-objects-and-allocation.md) first; the
header-behind-the-pointer layout explains every other file in the subsystem.
Then [02-refcounting-and-destruction.md](02-refcounting-and-destruction.md),
which is the one that prevents crashes. Containers can wait until you need
them.
