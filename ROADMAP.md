# Roadmap

Planned guides, in the order they're worth writing. Line counts are approximate
and are there to set expectations about scope.

All work is verified against the `24` branch. Newer releases and other branches
may differ.

The ordering is driven by dependency rather than interest: each tier assumes the
tiers above it. The first two entries are set by what the frames guide leans on
without explaining: it cites `main/channel.c` 145 times without ever defining
what a channel is, and it relies on ao2 refcounting (format and topology refs,
§3) without a guide to ao2.

## Tier 1 — unblocks everything else

### 1. Channels — *written* ([channels/](channels/README.md))

| | |
|---|---|
| Core source | `main/channel.c` (11,220), `include/asterisk/channel.h` (5,114), `main/channel_internal_api.c` (1,638), `main/channelstorage*.c` (870) |
| Size | Comparable to the frames guide, probably larger |

`struct ast_channel` is opaque — `channel.h:972` is a bare forward declaration and
every field sits behind an accessor in `channel_internal_api.c`. Covers the ao2
lifecycle and why holding a reference is not holding a lock, `ast_channel_alloc`
through `ast_hangup`, the snapshot and Stasis coupling, masquerade, autoservice,
the `ast_waitfor` family, and the channel storage backends.

Shipped as nine files: an overview plus chapters on anatomy and the accessor
layer, lifecycle, references versus locking, state and flags, I/O and waiting,
masquerade, the tech vtable and drivers, and snapshots and debugging. All 245
quoted blocks verify whitespace-exact against the `24` tree, the one exception
being a deliberate anti-pattern.

### 2. ao2 objects, refcounting and containers — *written* ([ao2/](ao2/README.md))

| | |
|---|---|
| Core source | `main/astobj2.c` + `astobj2_hash.c` / `astobj2_rbtree.c` / `astobj2_container.c` (5,400), `include/asterisk/astobj2.h` (2,069) |
| Size | Smaller than channels; a fast write |

Looks like plumbing, is where correctness actually goes wrong. Counting call
sites outside the ao2 implementation itself: `ao2_ref` 2,961, `RAII_VAR` 2,727,
`ao2_cleanup` 1,467, the five `ao2_*lock` forms 1,640 between them. No other
subsystem reads cleanly without this. Covers destructors and the "destructor runs
with no lock held" trap, `ao2_bump`/`cleanup`/`replace`, container types and
traversal callbacks, `OBJ_NOLOCK` vs `OBJ_MULTIPLE`, weak proxies, `REF_DEBUG`.

Shipped as eight files: an overview plus chapters on objects and allocation,
refcounting and destruction, locking, containers, traversal and iterators, weak
proxies and global objects, and debugging. All 217 quoted blocks verify
whitespace-exact against the `24` tree, the two exceptions being deliberate
anti-patterns.

### 3. Threading, locking and deadlock avoidance

| | |
|---|---|
| Core source | `main/lock.c` (1,335), `include/asterisk/lock.h` (782), `main/taskprocessor.c` (1,646), `main/threadpool.c` (1,350), `main/sched.c` (868), `main/autoservice.c` (391) |
| Size | Medium |

Placed before bridging on purpose: bridging is where the thread model bites. 185
files use the lock wrappers, 136 sites call `ast_pthread_create*`, and
`DEADLOCK_AVOIDANCE` appears at 41 sites — the macro has two definitions
(`lock.h:383`, and `:486` under `DEBUG_THREADS`) and deserves its own section.
Covers the thread inventory (PBX thread, bridge thread, driver threads,
serializers), lock ordering between channel, bridge and ao2 locks,
`DEBUG_THREADS` tracking and `core show locks`, taskprocessors vs serializers vs
the threadpool, and taskprocessor overload.

## Tier 2 — the large subsystems

### 4. Bridging

| | |
|---|---|
| Core source | `main/bridge.c` (5,782), `main/bridge_basic.c` (3,710), `main/bridge_channel.c` (3,138), `bridges/*.c` (5,313), `main/bridge_after.c`, `main/bridge_roles.c` |
| Size | ~18k lines of source; the second-largest topic here |
| Needs | Guides 1 and 3 |

Picks up where frames §7.3 stopped. Covers `smart_bridge_operation()`
(`bridge.c:1061`) and capability-based technology switching, softmix's mixing
thread against native RTP's passthrough, imparting and departing channels,
features and DTMF hooks, and after-bridge callbacks.

### 5. The Stasis message bus

| | |
|---|---|
| Core source | `main/stasis.c` (3,419), `main/stasis_channels.c` (2,380), `main/stasis_bridges.c` (1,478), `main/stasis_cache.c` (1,021), `main/stasis_message_router.c` (426) |
| Size | Medium-large |
| Needs | Guides 1 and 2 |

130 publish sites, 78 subscribe sites, and its own configurable threadpool
(`stasis.c:2416`). This is how anything outside the core learns what happened:
snapshots, topic forwarding, the cache, subscription filtering. It is the shared
substrate under AMI, ARI, CDR and CEL, so writing it once pays for four later
guides.

## Tier 3 — media and execution

### 6. Formats, codecs and negotiation

`main/format*.c` and `main/codec*.c` (2,645). Extends frames §6 into capability
sets, the format cache and `format_compatibility.c`. Modest size, high everyday
value.

### 7. RTP and the media path

`main/rtp_engine.c` (4,366) as the abstraction against `res/res_rtp_asterisk.c` as
the implementation: ICE, DTLS/SRTP, RTCP, the jitterbuffer, and how native
bridging hands off the stream.

### 8. Dialplan and the PBX core

`main/pbx.c` (7,049) and `main/extension_state.c` (1,751): extension matching and
pattern precedence, the PBX thread, hints and device state.

### 9. PJSIP architecture

`res/res_pjsip*` totals 70,663 lines, so this must be scoped to a slice — session
lifecycle in `res_pjsip_session.c` (6,300), serializers, SDP handling in
`res_pjsip_sdp_rtp.c` (2,735) — or it becomes its own guide set rather than a
guide.

## Tier 4 — supporting, write when needed

- **Modules, loader, config and sorcery** — `loader.c`, `config.c`, `sorcery.c` (~9,600)
- **AMI and ARI as event consumers** — `manager.c` alone is 10,300
- **CDR and CEL** — `cdr.c` and `cel.c` (6,700)
- **Memory debugging** — `astmm.c`, `MALLOC_DEBUG`, `REF_DEBUG`; small, and a
  natural companion to the frames debugging chapter
