# Frames in Asterisk — A Source-Level Guide

**Target tree:** Asterisk 22.2.0 (`.version` = `22.2.0`, git `a0db68c190`)

Every `file:line` citation in this guide points into an Asterisk 22.2.0 checkout
(written against `/usr/src/phzyx/asterisk-baseline`). Line numbers
drift between releases; if you are reading this against a different tree, treat the
citations as approximate and re-grep for the symbol name.

## Who this is for

A C developer working inside the Asterisk source tree who needs to know exactly
what a `struct ast_frame` is, who owns it at every point in its life, and what the
core does to it between the channel driver and the application.

## The one-page mental model

A frame is a small, flat, self-describing container for *one unit of media or
signalling*. It is not a packet, not an RTP frame, and not a codec frame — it is
Asterisk's internal currency. Channel drivers mint frames out of whatever the
network handed them; the core moves, copies, translates, inspects, and eventually
destroys them; applications read and write them.

```
                        ┌──────────────────────────────────────────┐
 network / hardware     │        channel driver (tech)             │
        │               │  chan_pjsip, chan_iax2, chan_dahdi, ...  │
        │               └──────────────────────────────────────────┘
        │                    │  tech->read / read_stream    ▲ tech->write / write_stream
        v                    v                              │
   ┌─────────────────────────────────────────────────────────────────────┐
   │                              main/channel.c                        │
   │                                                                    │
   │   __ast_read()                            ast_write_stream()       │
   │     ├─ timing fd / generator fd             ├─ framehook WRITE     │
   │     ├─ readq dequeue  ◄── __ast_queue_frame │                      │
   │     ├─ tech->read                           ├─ generator intercept │
   │     ├─ framehook READ                       ├─ audiohook (slin)    │
   │     ├─ per-frametype switch                 ├─ ast_translate(,,0)  │
   │     │    (DTMF emulation, ANSWER, ...)      ├─ audiohook (post)    │
   │     ├─ audiohook (slin)                     └─ tech_write()        │
   │     ├─ ast_translate(, , 1)                                        │
   │     └─ generator actions                                           │
   └─────────────────────────────────────────────────────────────────────┘
        │                                              ▲
        v  ast_read() returns a frame you own          │  ast_write() — you keep yours
   ┌─────────────────────────────────────────────────────────────────────┐
   │   applications, bridges, res_agi, app_dial, bridge_channel.c, ...   │
   └─────────────────────────────────────────────────────────────────────┘
```

Three facts carry most of the weight:

1. **A frame's header and its payload are owned independently.** The `mallocd`
   bitfield says which of the two (or three — `src` counts) the frame owns. See
   [03-memory-and-ownership.md](03-memory-and-ownership.md).
2. **Queuing duplicates.** `ast_queue_frame()` always `ast_frdup()`s, so the caller
   keeps ownership of what it passed in. That is why half the core queues frames
   that live on the stack. See [05-channel-path.md](05-channel-path.md).
3. **`&ast_null_frame` is a real, global, shared frame** and you must never free it.
   `main/frame.c:79`.

## The guide

| File | Covers |
|---|---|
| [01-anatomy.md](01-anatomy.md) | `struct ast_frame` field by field, the subclass union, `offset`, flags, `ast_null_frame` |
| [02-frame-types.md](02-frame-types.md) | Every `AST_FRAME_*` type: subclass meaning, payload, producers, consumers |
| [03-memory-and-ownership.md](03-memory-and-ownership.md) | `mallocd`, `frdup`/`frisolate`/`frfree`, the per-thread header cache, ownership rules table |
| [04-control-frames.md](04-control-frames.md) | The `AST_CONTROL_*` catalog, the indicate path, payload structs |
| [05-channel-path.md](05-channel-path.md) | `__ast_read`, `__ast_queue_frame`, `ast_write_stream`, waitfor, timing fds, generators, DTMF emulation |
| [06-translation-and-timing.md](06-translation-and-timing.md) | `translate.c`, smoothers, format refcounts, `samples`/`datalen`/`len`/`ts` |
| [07-interception.md](07-interception.md) | Framehooks, audiohooks, bridging and bridge action frames |
| [08-debugging-and-recipes.md](08-debugging-and-recipes.md) | `core set debug channel`, `ast_frame_dump`, known bugs, working recipes |

## Canonical source files

| Path | Role |
|---|---|
| `include/asterisk/frame.h` | The authoritative type, flag and enum definitions (670 lines) |
| `main/frame.c` | Lifecycle: alloc, dup, isolate, free, the thread cache, volume helpers (869 lines) |
| `main/channel.c` | `__ast_read`, `__ast_queue_frame`, `ast_write_stream`, `ast_indicate_data`, waitfor |
| `main/translate.c` | `ast_translate()` and the translator path machinery |
| `main/framehook.c`, `include/asterisk/framehook.h` | Per-channel frame interception |
| `main/audiohook.c`, `include/asterisk/audiohook.h` | Media tap / inject / manipulate |
| `main/bridge_channel.c` | The bridge's own frame queue and the bridge action frame types |
| `include/asterisk/smoother.h`, `main/smoother.c` | ptime repacketization (driver-side, *not* core-side) |

## Reading order

If you only read two files, read [01-anatomy.md](01-anatomy.md) and
[03-memory-and-ownership.md](03-memory-and-ownership.md) — ownership mistakes are
the single largest category of frame bug. If you are chasing a specific symptom,
start at [08-debugging-and-recipes.md](08-debugging-and-recipes.md).
