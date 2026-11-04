# Roadmap

Planned guides, in the order they're worth writing. Line counts are approximate
and are there to set expectations about scope.

All work is verified against the `24` branch in
`/usr/src/phzyx/asterisk-baseline`, which is the definitive tree. Newer versions
and other branches may differ.

The ordering is driven by dependency rather than interest: each tier assumes the
tiers above it. Two debts from the frames guide set the first two entries — it
cites `main/channel.c` 145 times without ever defining what a channel is, and it
relies on ao2 refcounting (format and topology refs, §3) without a guide to ao2.

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

## Maintenance debt

**The frames guide has been re-verified against `24`** — *done*. It was
originally written against 22.2.0 and later relabelled, which rewrote the header
banner but not the line numbers; 44 of its 171 quoted C blocks did not verify.
All eight chapters now read 162 verbatim, 0 not found, with 14 blocks explicitly
marked `<!-- illustrative -->` (see below). Every `file:line` citation in the
prose was re-grepped as well, not just the machine-checkable blocks, and all ten
items in chapter 08's bug list were re-confirmed live on `24`.

Three of the corrections were substantive rather than positional:

- Chapter 05 picked up three branch-`24` behaviour changes — commit `5963e624e2`
  demoted the missing-`fdno` warning from `LOG_ERROR` to `ast_debug(3, ...)`,
  `e200f51ea4` removed the generator-fd branch and relabelled
  `AST_GENERATOR_FD` unused (the 13-step walkthrough is now 12), and `7b8339fb52`
  added a NULL-timer guard at the four 50 fps DTMF timer arm sites.
- Chapter 06 picked up commit `4b9ef3f5e2` (configurable `comp_cost` sample
  types, the `translate sampletype` CLI, the pre-encode pool) and `b49e36c1e4`,
  whose `ast_frame_adjust_volume` dB-versus-linear-gain documentation fix also
  exposed a wrong "power of two shift" claim in the guide itself.
- Chapter 08's bug 3 described the wrong mechanism. The "returns success after
  giving up" behaviour in `ast_write_stream()` is not `CHECK_BLOCKING`; it is the
  `ast_channel_trylock()` retry loop that bails out after eleven attempts, and
  because it returns before the `done:` label the frame is not freed either. Bug
  6's latency claim was also over-broad and is now scoped to the call sites that
  can actually carry an RTCP frame.

**New: the `<!-- illustrative -->` marker.** `checkall.py` compares every fenced
block against the tree, which means a block the guide wrote itself — an
anti-pattern shown deliberately, a recipe skeleton with invented names — could
never verify and showed up as a permanent failure. Fourteen such blocks in
frames, one in channels and two in ao2 now carry an HTML comment on the line
above the fence; both `checkall.py` and `pin.py` honour it and report those
blocks in their own category instead of as failures. It is invisible in the
rendered page and explicit in the markdown, so a clean run means something again.

**Channels and ao2 citation pinning has not been done.** `checkall.py` only asks
whether a quoted block exists *somewhere* in the tree, so a correct quote with a
wrong line number passes clean — which is exactly how the frames drift went
unnoticed. The new `pin.py` closes that gap by binding each fenced block to its
adjacent citation and comparing the exact cited range. Run across channels and
ao2 it reports 202 OK against 51 `OTHER`, 30 `SHIFT`, 23 `NOFILE` and 48
`UNPAIRED`, with 269 blocks it could not pair with a citation at all. Those
figures are an upper bound on real drift, not a measurement of it: both guides
place the citation on the line *after* the closing fence but as a sentence-initial
code span, and `pin.py`'s `pick_cite()` only accepts a post-fence citation when
the line opens with a dash or a parenthesis. It therefore falls back to a 14-line
lookback and picks up the *previous* block's citation, producing a visible
lag-by-one cascade. A further 23 citations are written as bare basenames
(`astobj2.h:373`) that `pin.py` cannot resolve against its full-path corpus.
Teaching `pick_cite()` the sentence-initial convention and resolving basenames is
the prerequisite for measuring these two guides honestly; some of the same-file
`SHIFT` entries may well be real and will need checking individually.
