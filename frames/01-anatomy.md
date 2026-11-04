# 01 — Anatomy of `struct ast_frame`

Every `file:line` citation in this guide was verified against the `24` branch.
Newer releases and other branches may differ — line numbers drift — so if you
are reading this against a different tree, treat the citations as approximate
and re-grep for the symbol name.

Source of truth: `include/asterisk/frame.h`.

## The struct, annotated

`include/asterisk/frame.h:165-198`:

```c
struct ast_frame {
	/*! Kind of frame */
	enum ast_frame_type frametype;
	/*! Subclass, frame dependent */
	struct ast_frame_subclass subclass;
	/*! Length of data */
	int datalen;
	/*! Number of samples in this frame */
	int samples;
	/*! Was the data malloc'd?  i.e. should we free it when we discard the frame? */
	int mallocd;
	/*! The number of bytes allocated for a malloc'd frame header */
	size_t mallocd_hdr_len;
	/*! How many bytes exist _before_ "data" that can be used if needed */
	int offset;
	/*! Optional source of frame for debugging */
	const char *src;
	/*! Pointer to actual data */
	union { void *ptr; uint32_t uint32; char pad[8]; } data;
	/*! Global delivery time */
	struct timeval delivery;
	/*! For placing in a linked list */
	AST_LIST_ENTRY(ast_frame) frame_list;
	/*! Misc. frame flags */
	unsigned int flags;
	/*! Timestamp in milliseconds */
	long ts;
	/*! Length in milliseconds */
	long len;
	/*! Sequence number */
	int seqno;
	/*! Stream number the frame originated from */
	int stream_num;
};
```

### `frametype`

The discriminator. Every other field's meaning depends on it. Values are
`enum ast_frame_type`, `include/asterisk/frame.h:97-138`. Full catalog in
[02-frame-types.md](02-frame-types.md).

**These values are wire-visible.** The enum carries this warning at
`include/asterisk/frame.h:93-95`:

> NOTE: It is important that the values of each frame type are never changed,
> because it will break backwards compatibility with older versions. This is
> because these constants are transmitted directly over IAX2.

So `AST_FRAME_DTMF_END = 1` forever, and new types only ever get appended.

### `subclass`

`include/asterisk/frame.h:150-161`:

```c
struct ast_frame_subclass {
	/*! A frame specific code */
	int integer;
	union {
		/*! The asterisk media format */
		struct ast_format *format;
		/*! The asterisk stream topology */
		struct ast_stream_topology *topology;
	};
	/*! For video formats, an indication that a frame ended */
	unsigned int frame_ending;
};
```

Note it is a **struct containing a union**, not a bare union: `integer` and
`format`/`topology` coexist. Which members are live depends on `frametype`:

| `frametype` | live subclass members |
|---|---|
| `AST_FRAME_VOICE` | `subclass.format` (ao2-refcounted) |
| `AST_FRAME_VIDEO` | `subclass.format` (ao2-refcounted), `frame_ending` |
| `AST_FRAME_IMAGE` | `subclass.format` (ao2-refcounted) |
| `AST_FRAME_CONTROL` | `subclass.integer` = an `AST_CONTROL_*`; **plus** `subclass.topology` when `integer == AST_CONTROL_ANSWER` or a topology control |
| `AST_FRAME_DTMF_BEGIN` / `_END` | `subclass.integer` = the ASCII digit character |
| `AST_FRAME_TEXT` | `subclass.integer` — `0` for plain, or a format-ish marker |
| `AST_FRAME_MODEM` | `subclass.integer` = `AST_MODEM_T38` (1) or `AST_MODEM_V150` (2), `include/asterisk/frame.h:239-243` |
| `AST_FRAME_HTML` | `subclass.integer` = `AST_HTML_*`, `include/asterisk/frame.h:247-263` |
| `AST_FRAME_NULL`, `AST_FRAME_CNG`, `AST_FRAME_RTCP` | unused |

The `integer`/`format` coexistence is why `AST_CONTROL_ANSWER` can carry both a
control code *and* a topology pointer — see `ast_queue_answer()` at
`main/channel.c:1304-1312`, which sets both:

```c
int ast_queue_answer(struct ast_channel *chan, const struct ast_stream_topology *topology)
{
	struct ast_frame f = {
		AST_FRAME_CONTROL,
		.subclass.integer = AST_CONTROL_ANSWER,
		.subclass.topology = (struct ast_stream_topology *)topology,
	};
	return ast_queue_frame(chan, &f);
}
```

**The ao2 reference in `subclass.format` / `subclass.topology` is owned by the
frame** once the frame is heap-allocated. `__ast_frdup()` bumps it
(`main/frame.c:354-359`) and `__frame_free()` releases it — twice over, once on
the cache path (`main/frame.c:137-143`) and once on the real-free path
(`main/frame.c:160-172`):

```c
	if (fr->mallocd & AST_MALLOCD_HDR) {
		if (fr->frametype == AST_FRAME_VOICE
			|| fr->frametype == AST_FRAME_VIDEO
			|| fr->frametype == AST_FRAME_IMAGE) {
			ao2_cleanup(fr->subclass.format);
		} else if (fr->frametype == AST_FRAME_CONTROL && fr->subclass.integer == AST_CONTROL_ANSWER) {
			ao2_cleanup(fr->subclass.topology);
		}

		ast_free(fr);
	} else {
		fr->mallocd = 0;
	}
```

### `datalen` and `data`

`datalen` is the payload length in **bytes**. `data` is an 8-byte union, so a
frame can carry either a pointer or a small inline integer:

- `data.ptr` — the normal case; points at the payload.
- `data.uint32` — used when the payload *is* a number and `datalen` is 0. For
  example `AST_FRAME_HTML` link/URL subclasses, and the `__ast_frisolate()`
  shortcut at `main/frame.c:273-277`:

```c
		if (!fr->datalen && fr->frametype != AST_FRAME_TEXT) {
			/* Actually it's just an int so we can simply copy it. */
			out->data.uint32 = fr->data.uint32;
			return out;
		}
```

- `pad[8]` exists purely to force the union to 8 bytes on 32-bit builds so the
  struct layout is ABI-stable.

Note the asymmetry in that shortcut: `AST_FRAME_TEXT` is explicitly excluded,
because a text frame with `datalen == 0` still has a meaningful `data.ptr`
(an empty string).

### `samples`

Number of **samples** (not bytes) in a voice frame. For 8 kHz signed linear,
`datalen == samples * 2`. For a compressed codec there is no fixed relationship —
20 ms of G.729 is 20 bytes and 160 samples. Translators and the smoother both
rely on `samples` being correct; a driver that gets it wrong produces audio that
drifts. See [06-translation-and-timing.md](06-translation-and-timing.md).

### `mallocd` and `mallocd_hdr_len`

The ownership bitfield. `include/asterisk/frame.h:232-237`:

```c
/*! Need the header be free'd? */
#define AST_MALLOCD_HDR		(1 << 0)
/*! Need the data be free'd? */
#define AST_MALLOCD_DATA	(1 << 1)
/*! Need the source be free'd? (haha!) */
#define AST_MALLOCD_SRC		(1 << 2)
```

`mallocd == 0` means the frame is entirely borrowed — typically a stack frame or a
driver-owned static. `ast_frfree()` on such a frame is a no-op by design
(`main/frame.c:124-125`):

```c
	if (!fr->mallocd)
		return;
```

`mallocd_hdr_len` records how many bytes were actually allocated for the header
*plus any trailing payload space*, which is how the per-thread header cache can
decide whether a recycled header is big enough. Full treatment in
[03-memory-and-ownership.md](03-memory-and-ownership.md).

### `offset`

`offset` is the number of writable bytes that exist **before** `data.ptr`. It
exists so that a lower layer can prepend a header (RTP, IAX2, zaptel) without
reallocating or memmoving the payload.

`include/asterisk/frame.h:219-230`:

```c
/*! \brief Offset into a frame's data buffer.
 *
 * By providing some "empty" space prior to the actual data of an ast_frame,
 * this gives any consumer of the frame ample space to prepend other necessary
 * information without having to create a new buffer.
 *
 * As an example, RTP can use the data from an ast_frame and simply prepend the
 * RTP header information into the space provided by AST_FRIENDLY_OFFSET instead
 * of having to create a new buffer with the necessary space allocated.
 */
#define AST_FRIENDLY_OFFSET 	64
#define AST_MIN_OFFSET 		32	/*! Make sure we keep at least this much handy */
```

The contract: if you allocate your own buffer, reserve `AST_FRIENDLY_OFFSET` bytes
at the front, point `data.ptr` past them, and set `offset = AST_FRIENDLY_OFFSET`.
`res_rtp_asterisk.c` is the canonical example — `res/res_rtp_asterisk.c:396`
declares the buffer with the offset baked in:

```c
	unsigned char rawdata[8192 + AST_FRIENDLY_OFFSET];
```

and `res/res_rtp_asterisk.c:6302-6304` wires it up:

```c
		rtp->f.data.ptr = rtp->rawdata + AST_FRIENDLY_OFFSET;
		rtp->f.datalen = len - 1;
		rtp->f.offset = AST_FRIENDLY_OFFSET;
```

**This is load-bearing for `free()`.** When a frame owns its data,
`__frame_free()` frees `data.ptr - offset`, not `data.ptr`
(`main/frame.c:152-156`):

```c
	if (fr->mallocd & AST_MALLOCD_DATA) {
		if (fr->data.ptr) {
			ast_free(fr->data.ptr - fr->offset);
		}
	}
```

So if you set `AST_MALLOCD_DATA` with a stale or wrong `offset`, you hand `free()`
a pointer that is not the start of the allocation. Use the macro
(`include/asterisk/frame.h:208-213`) instead of doing it by hand:

```c
#define	AST_FRAME_SET_BUFFER(fr, _base, _ofs, _datalen)	\
	{					\
	(fr)->data.ptr = (char *)_base + (_ofs);	\
	(fr)->offset = (_ofs);			\
	(fr)->datalen = (_datalen);		\
	}
```

### `src`

A short string naming the producer, used only for debugging and logging ("RTP",
"DAHDI", "ast_prod"). Owned by the frame only when `AST_MALLOCD_SRC` is set,
which in practice only happens after `__ast_frisolate()` or `__ast_frdup()` copy
it. Drivers set it to a string literal and leave the bit clear —
`res/res_rtp_asterisk.c:6051`:

```c
	rtp->f.src = "RTP";
```

### `delivery`

A `struct timeval` giving the wall-clock time at which the frame's media should be
played. A zero `delivery` means "as soon as possible". Used mainly by jitter
buffers and by file playback; most of the core passes it through untouched.
`__ast_frdup()` copies it verbatim.

### `frame_list`

The intrusive list hook. **Frames form lists**, and several APIs operate on a
whole list rather than a single frame:

- `ast_frame_free()` walks `frame_list` and frees every frame in the chain
  (`main/frame.c:176-185`).
- Translators can emit multiple frames from one input frame; `__ast_read()` and
  `ast_write_stream()` both have explicit multi-frame handling.
- `__ast_queue_frame()` accepts a list and queues all of it atomically.

A single frame has `AST_LIST_NEXT(f, frame_list) == NULL`. A common bug is to
`ast_frfree()` a frame that is still linked into a list you intend to keep, which
takes the rest of the chain with it.

### `flags`

`include/asterisk/frame.h:141-148`:

```c
enum {
	/*! This frame contains valid timing information */
	AST_FRFLAG_HAS_TIMING_INFO = (1 << 0),
	/*! This frame has been requeued */
	AST_FRFLAG_REQUEUED = (1 << 1),
	/*! This frame contains a valid sequence number */
	AST_FRFLAG_HAS_SEQUENCE_NUMBER = (1 << 2),
};
```

- `AST_FRFLAG_HAS_TIMING_INFO` — `ts` and `len` are meaningful. Without it, do not
  trust them.
- `AST_FRFLAG_HAS_SEQUENCE_NUMBER` — `seqno` is meaningful.
- `AST_FRFLAG_REQUEUED` — set by the core when it pushes an already-processed
  frame back onto the readq so the second pass through `__ast_read()` does not
  re-translate or re-hook it. Set in the multi-frame translator output path at
  `main/channel.c:4217-4220`.

### `ts`, `len`, `seqno`

Timestamp (ms), duration (ms) and sequence number, all only valid when the
corresponding flag is set. `ts` and `len` originate in the jitterbuffer and IAX2
paths. `len` is also how DTMF duration travels: a `AST_FRAME_DTMF_END` frame's
`len` is the digit's duration in milliseconds, which the core clamps against
`option_dtmfminduration` — see the DTMF section of
[05-channel-path.md](05-channel-path.md).

### `stream_num`

Added for multistream (SFU / multi-video) support. Identifies which
`struct ast_stream` within the channel's topology the frame belongs to. `-1` means
"the default stream for this frame's media type". `__ast_read()` resolves it at
`main/channel.c:3720-3758` and will **drop the frame entirely** if it cannot map
it to a stream and the caller asked for default-stream-only behaviour.

The two-wrapper split in `ast_read` vs `ast_read_stream`
(`main/channel.c:4278-4286`) is exactly about this field.

## `ast_null_frame`

`main/frame.c:79`:

```c
struct ast_frame ast_null_frame = { AST_FRAME_NULL, };
```

Declared `extern` at `include/asterisk/frame.h:217`. One global instance, shared
by every thread, with `mallocd == 0`.

It means **"something happened, but there is no media"** — a timing tick fired, a
generator ran, a control frame was consumed internally, a translator swallowed its
input. Returning it (rather than NULL) keeps callers in their read loop; returning
NULL means hangup.

Rules:

- **Never `ast_frfree(&ast_null_frame)`.** It happens to be harmless today because
  `mallocd == 0` short-circuits `__frame_free()`, but it is still wrong and will
  mask real bugs.
- **Never modify it.** It is shared across all threads with no locking.
- Compare by pointer: `if (f == &ast_null_frame)`.
- Do not assume `ast_read()` returning a null frame means "no work" — it usually
  means the core did work you cannot see.

`__ast_read()` returns it from many places; the timing-fd path at
`main/channel.c:3645` is a good illustration of why it is a `return` and not a
`goto done`:

```c
			return &ast_null_frame;
```

with the preceding comment noting the channel is already unlocked at that point,
so the shared cleanup at `done:` must be skipped.

## Size and alignment notes

`sizeof(struct ast_frame)` is not part of any public contract, but
`mallocd_hdr_len` is compared against it (`main/frame.c:103`:
`f->mallocd_hdr_len = sizeof(*f);`), so never allocate a frame header with
anything other than `ast_frame_header_new()` or `ast_frdup()`. A stack frame
declared as `struct ast_frame f = { ... };` is fine precisely because it is never
freed.

---

Next: [02-frame-types.md](02-frame-types.md) — what each `frametype` actually
carries.
