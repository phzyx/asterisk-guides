# Channels in Asterisk — A Source-Level Guide

**Target tree:** Asterisk `master`, git `364f14214c` (2026-11-02)

Every `file:line` citation in this guide was verified against `266378a125`, two
commits back; neither intervening commit touches any file this guide cites, so the
citations hold verbatim at `364f14214c`. Note that this is a *different* tree from
the one [the frames guide](../frames/README.md) pins (the 22.2.0 release,
`a0db68c190`), which is not an ancestor of this commit — `master` carries 638
commits that the 22.2.0 branch does not, including about 700 changed lines in
`main/channel.c` alone. Line numbers drift between releases and between branches;
if you are reading this against a different tree, treat the citations as
approximate and re-grep for the symbol name.

## Who this is for

A C developer working inside the Asterisk source tree who needs to know what a
`struct ast_channel` actually is, how to get at its fields, who owns it, what
keeps it alive, and what can change underneath them while they hold it.

This guide is the companion to [the frames guide](../frames/README.md), which
cites `main/channel.c` 145 times without ever defining a channel. This one fills
that in.

## The one-page mental model

A channel is a *call leg*: one endpoint's participation in a call, plus the
dialplan state driving it and the media formats carrying it. It is an ao2 object,
it is registered in a global storage container, it has its own Stasis topic, and
it is almost always being touched by more than one thread.

The single most important structural fact is that you cannot see inside it:

```
      ┌──────────────────────────────────────────────────────────────────┐
      │  include/asterisk/channel.h:972                                  │
      │      struct ast_channel;        ← forward declaration only       │
      │                                                                  │
      │  ...plus ~330 accessor prototypes:                               │
      │      ast_channel_name(chan)           ast_channel_state(chan)    │
      │      ast_channel_tech_pvt_set(chan,p) ast_channel_context(chan)  │
      └──────────────────────────────────────────────────────────────────┘
                   ▲                                        ▲
    everything     │  can only call accessors               │
    in the tree ───┘  (main/channel.c included)             │
                                                            │
      ┌─────────────────────────────────────────────────────────────────┐
      │  main/channel_private.h:48    struct ast_channel { ... }        │
      │                                                                 │
      │  Included by exactly three translation units:                   │
      │    main/channel_internal_api.c:51      ← the accessors live here│
      │    main/channelstorage_ao2_legacy.c:24 ← the legacy container   │
      │    main/channelstorage.c:160           ← TEST_FRAMEWORK only    │
      └─────────────────────────────────────────────────────────────────┘
```

Note what is *not* in that list: `main/channel.c`. The 11,220-line heart of the
channel implementation reaches its own fields through the same public accessors
you do.

Three facts carry most of the weight:

1. **The struct is private to three files.** Adding or renaming a field means
   editing `main/channel_private.h` *and* hand-writing or macro-generating an
   accessor in `main/channel_internal_api.c`. See
   [01-anatomy.md](01-anatomy.md).
2. **A reference is not a lock.** An ao2 reference guarantees the channel still
   exists; it says nothing about whether another thread is mutating it. These are
   two independent things to get right, and conflating them is the largest
   category of channel bug. Chapter 03.
3. **Setters have side effects.** Several field setters invalidate a snapshot
   segment, and three of them publish a Stasis message synchronously, inside the setter
   (`main/channel_internal_api.c:84-106`). Assigning to a channel field is not a
   cheap operation and is not free of ordering concerns.

## The guide

| File | Covers | Status |
|---|---|---|
| [01-anatomy.md](01-anatomy.md) | `struct ast_channel` field group by field group, the accessor layer and its generating macros, snapshot invalidation, `ast_channel_id` | written |
| [02-lifecycle.md](02-lifecycle.md) | `__ast_channel_alloc` → `finalized` → `ast_hangup` → the ao2 destructor; channel storage registration; soft vs hard hangup; who holds references | written |
| [03-references-and-locking.md](03-references-and-locking.md) | refs vs locks, `ast_channel_lock` as a recursive mutex, the string field hazard, lock ordering, `ast_channel_lock_both`, `CHANNEL_DEADLOCK_AVOIDANCE`, `core show locks` | written |
| [04-state-and-flags.md](04-state-and-flags.md) | `enum ast_channel_state` and `ast_setstate`'s side effects, the `softhangup` bits and `ast_check_hangup`, the `AST_FLAG_*` catalog including `ZOMBIE`, `DEAD` and `SNAPSHOT_STAGE` | written |
| 05-io-and-waiting.md | the fd vector, `ast_waitfor` family, timing fds, and how this meets the frame read path | planned |
| 06-masquerade.md | masquerade and `ast_channel_move`: what swaps, what does not, fixup callbacks, local channels | planned |
| 07-tech-and-drivers.md | `struct ast_channel_tech`, `ast_request`/`ast_call`/`ast_answer`, what a driver must implement | planned |
| 08-snapshots-and-debugging.md | snapshots and Stasis coupling, `core show channel`, `REF_DEBUG`, recipes | planned |

## Canonical source files

| Path | Role |
|---|---|
| `main/channel_private.h` | The real `struct ast_channel` definition (210 lines) |
| `include/asterisk/channel.h` | The opaque type plus the entire public accessor and API surface (5,114 lines) |
| `main/channel_internal_api.c` | The accessor layer — 233 function definitions (1,638 lines) |
| `main/channel.c` | Everything else: alloc, hangup, read/write, waitfor, masquerade, indications (11,220 lines) |
| `include/asterisk/channel_internal.h` | The internal API that `channel.c` and friends share but modules do not |
| `main/channelstorage.c`, `main/channelstorage_ao2_legacy.c` | The global channel container and its pluggable backends |
| `main/stasis_channels.c` | Channel snapshots and the channel message types |

## Reading order

Read [01-anatomy.md](01-anatomy.md) first — the accessor indirection explains the
shape of every other file in the subsystem. Then [02-lifecycle.md](02-lifecycle.md)
and [03-references-and-locking.md](03-references-and-locking.md): those two are
the ones that prevent crashes.
