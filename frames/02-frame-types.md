# 02 — The frame type catalog

**Target tree:** Asterisk, the `24` branch. Line numbers and quoted code may
differ on newer releases and on other branches; re-grep for the symbol name if
you are reading against a different tree.

Every value of `enum ast_frame_type`, `include/asterisk/frame.h:97-138`. Values 1
through 16 are assigned; the enum is append-only because the numbers go on the
IAX2 wire (`include/asterisk/frame.h:92-96`).

## Quick reference

| # | Type | Subclass means | Payload |
|---|---|---|---|
| 1 | `AST_FRAME_DTMF_END` | ASCII digit | none; `len` = duration in ms |
| 2 | `AST_FRAME_VOICE` | `subclass.format` | encoded or linear audio, `samples` samples |
| 3 | `AST_FRAME_VIDEO` | `subclass.format`, `frame_ending` | one video packet/fragment |
| 4 | `AST_FRAME_CONTROL` | `AST_CONTROL_*` in `subclass.integer` | per-control struct, often none |
| 5 | `AST_FRAME_NULL` | unused | none |
| 6 | `AST_FRAME_IAX` | IAX2 private subcommand | IAX2 internal |
| 7 | `AST_FRAME_TEXT` | 0, or a text marker | UTF-8 bytes, **not NUL-terminated** |
| 8 | `AST_FRAME_IMAGE` | `subclass.format` | a still image |
| 9 | `AST_FRAME_HTML` | `AST_HTML_*` | URL / HTML fragment, or `data.uint32` |
| 10 | `AST_FRAME_CNG` | CNG level in -dBov | 0+ quantization coefficients |
| 11 | `AST_FRAME_MODEM` | `AST_MODEM_T38` / `AST_MODEM_V150` | UDPTL / modem payload |
| 12 | `AST_FRAME_DTMF_BEGIN` | ASCII digit | none |
| 13 | `AST_FRAME_BRIDGE_ACTION` | `enum bridge_channel_action_type` | action payload |
| 14 | `AST_FRAME_BRIDGE_ACTION_SYNC` | `enum bridge_channel_action_type` | `struct sync_payload` |
| 15 | `AST_FRAME_RTCP` | RTCP payload type | `struct ast_rtp_rtcp_report` or `_feedback` |
| 16 | `AST_FRAME_TEXT_DATA` | unused | `struct ast_msg_data` |

`AST_FRAME_DTMF` is an alias for `AST_FRAME_DTMF_END` (`include/asterisk/frame.h:139`) —
an artifact of DTMF originally being a single event. Prefer the explicit name.

---

## `AST_FRAME_VOICE` (2)

The workhorse. `subclass.format` is an ao2-refcounted `struct ast_format *`; the
frame owns a reference once heap-allocated.

- `datalen` = payload bytes, `samples` = sample count. For `slin` these differ by
  a factor of two; for compressed codecs there is no fixed ratio.
- `offset` should be `AST_FRIENDLY_OFFSET` for driver-produced frames so RTP can
  prepend its header in place.
- The core may translate it (`ast_translate()`), tap it (audiohooks), hook it
  (framehooks) and requeue it. Every one of those can replace the frame with a
  different one, or with a *list*.

`__ast_read()`'s voice handling is the largest single block in `channel.c`
(`main/channel.c:4073-4234`). See [05-channel-path.md](05-channel-path.md).

Voice frames are the only type the core will drop under queue pressure, alongside
video and null — `main/channel.c:1107-1120`.

## `AST_FRAME_VIDEO` (3)

Same ownership rules as voice. Two differences:

- `subclass.frame_ending` is meaningful: it marks the last fragment of a video
  frame, so a consumer can know when it has a complete decodable picture.
- `samples` is not meaningful for most video formats.

Video is never translated by `main/translate.c` — the translator registry is
audio-only. Video transcoding is out of scope for the core; it either passes
through or is handled by a specific module.

## `AST_FRAME_CONTROL` (4)

Signalling. `subclass.integer` is an `enum ast_control_frame_type`. The payload,
when present, is a control-specific struct pointed to by `data.ptr` with `datalen`
set to its size. Full catalog in [04-control-frames.md](04-control-frames.md).

Two special properties:

1. **It can carry a topology.** When `subclass.integer == AST_CONTROL_ANSWER`,
   `subclass.topology` holds an ao2 reference to a `struct ast_stream_topology`,
   which the frame owns (`main/frame.c:141-142`, `:165-166`).
2. **The core consumes many of them silently.** `__ast_read()`'s control case
   handles `AST_CONTROL_ANSWER`, `AST_CONTROL_READ_ACTION`, the topology controls
   and the FLASH/WINK events itself and returns a null frame instead
   (`main/channel.c:3835-3902`). Do not assume a queued control frame will reach
   your read loop.

## `AST_FRAME_NULL` (5)

"Nothing happened that you can see." Always `&ast_null_frame` in practice.
`ast_write()` silently ignores it (`main/channel.c:5463-5464`). The queue-pressure
flush treats it as droppable. See the `ast_null_frame` section of
[01-anatomy.md](01-anatomy.md).

## `AST_FRAME_IAX` (6)

Private to `chan_iax2`. Carries IAX2 protocol subcommands between the channel
driver and its peer. The core must never generate or interpret one.
`ast_write()` silently ignores it too (`main/channel.c:5465-5466`), grouped with
`AST_FRAME_NULL`:

```c
	case AST_FRAME_NULL:
	case AST_FRAME_IAX:
		/* Ignore these */
		res = 0;
		break;
```

## `AST_FRAME_TEXT` (7)

The header comment is the important part (`include/asterisk/frame.h:110-113`):

> Text messages. The character data may not be zero-terminated, so care should be
> taken when passing it to functions that expect a zero-terminated string. The
> frame's datalen member should be used as it indicates the actual number of bytes
> available.

So: **never `strlen()` a text frame's payload.** Use `datalen`. This is a real
source of out-of-bounds reads.

`ast_write_stream()` routes text two ways depending on format
(`main/channel.c:5281-5291`): T.140 real-time text goes to `tech->write_text`,
everything else to `tech->send_text`.

`__ast_frisolate()` carves out an explicit exception for text frames with
`datalen == 0` (`main/frame.c:273`) — they still have a real `data.ptr`, so the
`data.uint32` shortcut must not apply.

## `AST_FRAME_IMAGE` (8)

A still image, `subclass.format` identifying the encoding. Used by
`ast_send_image()` and the (rarely exercised) image-capable channel drivers. Owns
its format reference like voice and video (`main/frame.c:137-140`, `:161-164`).

## `AST_FRAME_HTML` (9)

Legacy web-integration frames, subclasses at `include/asterisk/frame.h:247-263`:
`AST_HTML_URL`, `AST_HTML_DATA`, `AST_HTML_BEGIN`, `AST_HTML_END`,
`AST_HTML_LDCOMPLETE`, `AST_HTML_NOSUPPORT`, `AST_HTML_LINKURL`,
`AST_HTML_UNLINK`, `AST_HTML_LINKREJECT`.

Interesting mainly as the clearest user of `data.uint32`: the link-status
subclasses carry a number, not a buffer, so `datalen` is 0 and `data.uint32`
holds the value. This is what the `__ast_frisolate()` shortcut at
`main/frame.c:273-279` exists for.

## `AST_FRAME_CNG` (10)

Comfort noise. `include/asterisk/frame.h:119-120`:

> Comfort Noise frame (subclass is level of CNG in -dBov), body may include zero
> or more 8-bit quantization coefficients

So `subclass.integer` is a *level*, not a format, and the payload is optional.
Produced by RTP when it receives payload type 13 —
`res/res_rtp_asterisk.c:6311`:

```c
	rtp->f.frametype = AST_FRAME_CNG;
```

**Gotcha:** `ast_frame_type2str()` has no case for `AST_FRAME_CNG`
(`main/frame.c:671-728`), so a CNG frame prints as the default "Unknown
Frametype". See [08-debugging-and-recipes.md](08-debugging-and-recipes.md).

## `AST_FRAME_MODEM` (11)

Modem-over-IP. `include/asterisk/frame.h:239-243`:

```c
/* MODEM subclasses */
/*! T.38 Fax-over-IP */
#define AST_MODEM_T38		1
/*! V.150 Modem-over-IP */
#define AST_MODEM_V150		2
```

T.38 fax carries UDPTL payloads in these frames. The *negotiation* happens through
`AST_CONTROL_T38_PARAMETERS` control frames and
`struct ast_control_t38_parameters` — media and signalling are cleanly separated.
See [04-control-frames.md](04-control-frames.md).

## `AST_FRAME_DTMF_BEGIN` (12) and `AST_FRAME_DTMF_END` (1)

`subclass.integer` is the ASCII character of the digit (`'0'`..`'9'`, `'*'`,
`'#'`, `'A'`..`'D'`). No payload. On an END frame, `len` is the digit duration in
milliseconds.

Not every path produces both. Some signalling only reports digit completion, which
is why the core has a **DTMF emulation state machine** that synthesizes the
missing half and enforces minimum durations and inter-digit gaps. The relevant
constants:

| Constant | Value | Location |
|---|---|---|
| `AST_DEFAULT_EMULATE_DTMF_DURATION` | 100 | `main/channel.c:102` |
| `DEFAULT_EMULATE_MF_DURATION` | 55 | `main/channel.c:106` |
| `AST_MIN_DTMF_GAP` | 45 | `main/channel.c:112` |
| `option_dtmfminduration` | `AST_MIN_DTMF_DURATION` | `main/options.c:83` |

Full state machine in [05-channel-path.md](05-channel-path.md).

In the write direction, DTMF frames do not go to `tech->write`; they go to
`ast_senddigit_begin()` / `ast_senddigit_end()` with the channel unlocked
(`main/channel.c:5255-5280`).

## `AST_FRAME_BRIDGE_ACTION` (13) and `AST_FRAME_BRIDGE_ACTION_SYNC` (14)

Internal to the bridging framework. `subclass.integer` is an
`enum bridge_channel_action_type` (`include/asterisk/bridge_channel_internal.h:41-71`).

The header carries an explicit warning about the sync variant
(`include/asterisk/frame.h:128-133`):

> Internal synchronous bridge module action. Synchronous bridge actions may be
> queued onto bridge channels, but they absolutely must not ever be written
> directly into bridges.

enforced by an assertion at `main/bridge_channel.c:653`:

```c
	ast_assert(frame->frametype != AST_FRAME_BRIDGE_ACTION_SYNC);
```

A sync action's payload is a `struct sync_payload` whose freeing *signals the
waiting thread* — so freeing it with plain `ast_frfree()` would hang the sender.
That is why bridging has its own `bridge_frame_free()`
(`main/bridge_channel.c:984-1003`). Details in
[07-interception.md](07-interception.md).

## `AST_FRAME_RTCP` (15)

RTCP feedback surfaced as a frame so that modules (notably `bridge_softmix` and
anything doing congestion control) can see it. `subclass.integer` is the RTCP
payload type; `data.ptr` is a `struct ast_rtp_rtcp_report` or
`struct ast_rtp_rtcp_feedback`.

Constructed in `res/res_rtp_asterisk.c:7041` and `:7109`, with the payload built
into the per-instance `rtcp->frame_buf` and `mallocd = 0`
(`res/res_rtp_asterisk.c:7043-7060`).

Three oddities worth knowing:

1. **`ast_translate()` short-circuits RTCP.** It calls each translator's
   `feedback()` callback and then returns `&ast_null_frame`
   (`main/translate.c:566-600`) — an RTCP frame is never translated into
   anything.
2. **`__ast_read()` feeds read-direction RTCP to the *write* translator**
   (`main/channel.c:4235-4241`):
   ```c
   		ast_translate(ast_channel_writetrans(chan), f, 0);
   ```
   Which is correct: RTCP arriving on the read side is a receiver report *about
   what we sent*, so it is feedback for the encoder.
3. **`ast_frame_subclass2str()` is buggy for RTCP** — the case at
   `main/frame.c:661-662` is missing a `break` and falls through to `default`.

## `AST_FRAME_TEXT_DATA` (16)

Modern structured text messaging. `data.ptr` is a `struct ast_msg_data`, which
carries sender, recipient, body and content type rather than raw bytes. Preferred
over `AST_FRAME_TEXT` for anything that needs metadata. Handled in bridging by
`sendtext_safe()` alongside `AST_FRAME_TEXT`.

---

## Which types the core will silently drop or swallow

Worth internalizing, because it explains a lot of "my frame never arrived":

| Situation | Types affected | Location |
|---|---|---|
| Readq over 128 total / 96 voice | VOICE, VIDEO, NULL (up to 64 dropped) | `main/channel.c:1101-1120` |
| `ast_write()` ignores | NULL, IAX | `main/channel.c:5463-5467` |
| `__ast_read()` consumes internally | many CONTROL subclasses | `main/channel.c:3835-3902` |
| Generator active on write | everything except DTMF_END and CONTROL/UNHOLD | `main/channel.c:5222-5246` |
| `ast_translate()` swallows | RTCP (returns null frame) | `main/translate.c:566-600` |
| `ast_frame_dump()` prints nothing | VOICE, VIDEO, RTCP | `main/frame.c:731-785` |
| `__ast_read()` drops unmappable | any frame with an unresolvable `stream_num` | `main/channel.c:3720-3758` |

---

Next: [03-memory-and-ownership.md](03-memory-and-ownership.md) — who frees what.
