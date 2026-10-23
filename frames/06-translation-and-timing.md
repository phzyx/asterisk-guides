# 06 — Translation, timing and packetization

Three related concerns that all live on the frame: converting between codecs
(`main/translate.c`), getting the right number of samples per frame
(`main/smoother.c`), and keeping `samples`/`datalen`/`ts`/`len` consistent.

## Translators

A translator converts frames from one format to another. `main/translate.c` owns
a registry and a cost matrix, and builds *chains* of translators when no direct
step exists.

### `struct ast_translator`

`include/asterisk/translate.h:137-193`. The interesting fields:

```c
struct ast_translator {
	char name[80];                         /*!< Name of translator */
	struct ast_codec src_codec;		       /*!< Source codec */
	struct ast_codec dst_codec;		   	   /*!< Destination codec */
	...
	int table_cost;                        /*!< Cost value associated with this translator based
	                                        *   on translation cost table. */
	int comp_cost;                         /*!< Cost value associated with this translator based
	                                        *   on computation time. ... */

	int (*newpvt)(struct ast_trans_pvt *);

	int (*framein)(struct ast_trans_pvt *pvt, struct ast_frame *in);
	struct ast_frame * (*frameout)(struct ast_trans_pvt *pvt);
	void (*feedback)(struct ast_trans_pvt *pvt, struct ast_frame *feedback);
	void (*destroy)(struct ast_trans_pvt *pvt);
	struct ast_frame * (*sample)(void);

	int buffer_samples;                    /*< size of outbuf, in samples */
	int buf_size;
	int desc_size;
	int native_plc;
	...
};
```

The contract a codec module signs up to (`include/asterisk/translate.h:122-135`):

> As a minimum, a translator should supply name, srcfmt and dstfmt, the required
> buf_size (in bytes) and buffer_samples (in samples), and a few callbacks
> (framein, frameout, feedback, sample). **The outbuf is automatically prepended
> by AST_FRIENDLY_OFFSET spare bytes** so generic routines can place data in
> there.
>
> Note, the translator **is not supposed to do any memory allocation or
> deallocation, nor any locking**, because all of this is done in the generic
> code.
>
> Translators using generic plc (packet loss concealment) should supply a non-zero
> plc_samples indicating the size (in samples) of artificially generated frames
> and incoming data. **Generic plc is only available for dstfmt = SLINEAR**.

So from a frame's point of view: a translator never frees your frame, and the
output frame's buffer always has an `AST_FRIENDLY_OFFSET` in front of it.

The two-cost design is explained at `include/asterisk/translate.h:45` and
`:145-149`: `table_cost` is the primary key, and `comp_cost` is a tie-breaker
measured at startup — which is how a hardware transcoder naturally outranks a
software one with the same formats.

### The `framein` / `frameout` split

`framein()` feeds samples in; `frameout()` produces a frame when enough have
accumulated. They are decoupled, which is why **a translator can output zero, one
or several frames per input frame**. The core handles all three cases, and that
multi-frame possibility is the source of a surprising amount of complexity in
`channel.c` (see [05-channel-path.md](05-channel-path.md)).

### Building and freeing a path

```c
struct ast_trans_pvt *ast_translator_build_path(struct ast_format *dst, struct ast_format *src);  /* main/translate.c:486 */
void ast_translator_free_path(struct ast_trans_pvt *p);                                           /* main/translate.c:476 */
```

`ast_translator_build_path()` walks the cost matrix one step at a time
(`main/translate.c:501-531`):

```c
	while (src_index != dst_index) {
		struct ast_trans_pvt *cur;
		struct ast_format *explicit_dst = NULL;
		struct ast_translator *t = matrix_get(src_index, dst_index)->step;
		if (!t) {
			ast_log(LOG_WARNING, "No translator path from %s to %s\n", ...);
			...
			return NULL;
		}
		...
		src_index = cur->t->dst_fmt_index;
	}
```

The result is a singly-linked chain of `struct ast_trans_pvt`. `matrix_get()`
returns the *next step* toward the destination, not the whole route — the matrix
is precomputed so each lookup is O(1) and the loop terminates.

`ast_translator_free_path()` walks and destroys the whole chain
(`main/translate.c:476-483`).

The channel keeps two of these: `ast_channel_readtrans(chan)` and
`ast_channel_writetrans(chan)`.

### `slin` as the pivot

Most codecs register translators only to and from signed linear. A G.711→G.729
conversion is therefore a two-step chain through `slin`. This is why:

- `transcode_via_slin` exists as an option (and is **on by default** —
  `main/channel.c:4146-4149` warns about it).
- `generic_plc` only works for `slin` (`include/asterisk/translate.h:135`).
- The read path forces `core_format` to slin whenever the core already wanted slin
  (`main/channel.c:4159-4163`), because core consumers like softmix bridges,
  recording and AMD need linear audio.

## `ast_translate()`

`main/translate.c:566-680`:

```c
struct ast_frame *ast_translate(struct ast_trans_pvt *path, struct ast_frame *f, int consume);
```

### The `consume` argument

The whole ownership story in one parameter (`main/translate.c:676-679`):

```c
	if (consume) {
		ast_frfree(f);
	}
	return out;
```

| Caller | `consume` | Why |
|---|---|---|
| `__ast_read()` voice path, `main/channel.c:4188` | **1** | The frame came from the driver/queue and the core owns it |
| `ast_write_stream()` voice path, `main/channel.c:5366` | **0** | The frame belongs to the caller of `ast_write()` |
| `__ast_read()` RTCP path, `main/channel.c:4239` | 0 | The frame continues up to the reader |

Getting this backwards leaks one way and double-frees the other. It is the single
most important thing to check when adding a translation call site.

### The RTCP short-circuit

`main/translate.c:576-585`:

```c
	if (f->frametype == AST_FRAME_RTCP) {
		/* Just pass the feedback to the right callback, if it exists.
		 * This "translation" does nothing so return a null frame. */
		struct ast_trans_pvt *tp;
		for (tp = p; tp; tp = tp->next) {
			if (tp->t->feedback)
				tp->t->feedback(tp, f);
		}
		return &ast_null_frame;
	}
```

Note this returns **before** the `consume` check at the bottom. So
`ast_translate(path, rtcp_frame, 1)` does **not** free the frame. Both in-tree
RTCP call sites pass 0, so this is latent rather than live, but do not rely on
`consume` for RTCP.

This is how a codec with a rate controller (Opus, VP8) learns about packet loss:
the receiver report arrives as a frame and is handed to every `feedback()`
callback in the path.

### Timing prediction

`main/translate.c:587-613` and `:632-675`. The path carries `nextin` and `nextout`
predictions, maintained only when `f->delivery` is non-zero.

Incoming (`:592-613`): if the actual delivery time differs from the prediction,
the output prediction is shifted by the same delta rather than reset — so a
one-off jitter does not permanently skew output timing:

```c
			if (!ast_tveq(path->nextin, f->delivery)) {
				/* The time has changed between what we expected and this
				   most recent time on the new packet.  If we have a
				   valid prediction adjust our output time appropriately */
				if (!ast_tvzero(path->nextout)) {
					path->nextout = ast_tvadd(path->nextout,
								  ast_tvsub(f->delivery, path->nextin));
				}
				path->nextin = f->delivery;
			}
```

Outgoing (`:634-661`): each output frame's `delivery` is set from `nextout`, and
`nextout` advances by the frame's own sample count. A CNG frame **invalidates** the
prediction, because silence has no reliable duration
(`main/translate.c:646-648`, repeated at `:670-673`):

```c
				/* Invalidate prediction if we're entering a silence period */
				if (current->frametype == AST_FRAME_CNG) {
					path->nextout = ast_tv(0, 0);
```

And if the sample count changed, `AST_FRFLAG_HAS_TIMING_INFO` is **cleared**,
because `ts`/`len`/`seqno` no longer describe the output
(`main/translate.c:656-659`):

```c
				if (f->samples != current->samples && ast_test_flag(current, AST_FRFLAG_HAS_TIMING_INFO)) {
					ast_debug(4, "Sample size different %d vs %d\n", f->samples, current->samples);
					ast_clear_flag(current, AST_FRFLAG_HAS_TIMING_INFO);
				}
```

That is a correct and easily-missed behaviour: **translation can strip timing
information.** Anything downstream that relies on `ts` must re-check the flag.

When `delivery` is zero the else-branch (`:662-674`) copies the saved
`has_timing_info`/`ts`/`len`/`seqno` straight across instead.

### The translation loop

`main/translate.c:615-626`:

```c
	for (out = f; out && p ; p = p->next) {
		struct ast_frame *current = out;

		do {
			framein(p, current);
			current = AST_LIST_NEXT(current, frame_list);
		} while (current);
		if (out != f) {
			ast_frfree(out);
		}
		out = p->t->frameout(p);
	}
```

Each stage of the chain is fed **the whole list** produced by the previous stage,
and the previous stage's output is freed once consumed — except the original `f`,
which is left for the `consume` check. This is why a two-step chain can turn one
input frame into several outputs.

### Interpolated frames and generic PLC

`main/translate.c:628-630`:

```c
	if (!out) {
		out = generate_interpolated_slin(path, f);
	}
```

`generate_interpolated_slin()` (`main/translate.c:537-563`) manufactures an
**empty** slin frame — `datalen = 0`, `data.ptr = NULL`, but `samples` copied from
the input — when the translator produced nothing for a zero-length (lost) input
frame:

```c
	if (!ast_opt_generic_plc || f->datalen != 0 ||
		ast_format_cmp(p->explicit_dst, ast_format_slin) == AST_FORMAT_CMP_NOT_EQUAL) {
		return NULL;
	}

	res.subclass.format = ast_format_cache_get_slin_by_rate(8000); /* ref bumped on dup */
	res.samples = f->samples;
	res.datalen = 0;
	res.data.ptr = NULL;
	res.offset = AST_FRIENDLY_OFFSET;

	return ast_frdup(&res);
```

with the explanation at `:541-550`:

> If we've gotten here then we should have an interpolated frame that was not
> handled by the translation codec. So create an interpolated frame in the
> appropriate format that was going to be written. This frame might be handled
> later by other resources. For instance, generic plc. ... Generic plc is filled
> in later on frame write.

**So a voice frame with `datalen == 0` and `samples > 0` is a legitimate thing to
receive.** It means "this much audio is missing". `apply_plc()` in the write path
(`main/channel.c:5125`) fills it in. Any code that reads voice frames must handle
`datalen == 0` without dividing by it or dereferencing `data.ptr`.

Note the "ref bumped on dup" comment: the stack frame holds a borrowed format
reference, and `ast_frdup()` is what takes the real one.

## Smoothers (repacketization)

A smoother turns variable-size frames into fixed-size ones. From the page doc at
`include/asterisk/smoother.h:42-54`:

> The ast_smoother interface was designed specifically to take frames of variant
> sizes and produce frames of a single expected size, precisely what you want to
> do.
>
> The basic interface is:
> - Initialize with ast_smoother_new()
> - Queue input frames with ast_smoother_feed()
> - Get output frames with ast_smoother_read()
> - when you're done, free the structure with ast_smoother_free()

### The API

`include/asterisk/smoother.h:34-82`:

```c
#define AST_SMOOTHER_FLAG_G729		(1 << 0)
#define AST_SMOOTHER_FLAG_BE		(1 << 1)
#define AST_SMOOTHER_FLAG_FORCED	(1 << 2)

struct ast_smoother *ast_smoother_new(int bytes);
void ast_smoother_set_flags(struct ast_smoother *smoother, int flags);
int ast_smoother_get_flags(struct ast_smoother *smoother);
int ast_smoother_test_flag(struct ast_smoother *s, int flag);
void ast_smoother_free(struct ast_smoother *s);
void ast_smoother_reset(struct ast_smoother *s, int bytes);
void ast_smoother_reconfigure(struct ast_smoother *s, int bytes);

int __ast_smoother_feed(struct ast_smoother *s, struct ast_frame *f, int swap);
struct ast_frame *ast_smoother_read(struct ast_smoother *s);
#define ast_smoother_feed(s,f) __ast_smoother_feed(s, f, 0)
```

The endian variants are compile-time aliases (`include/asterisk/smoother.h:76-82`):

```c
#if __BYTE_ORDER == __LITTLE_ENDIAN
#define ast_smoother_feed_be(s,f) __ast_smoother_feed(s, f, 1)
#define ast_smoother_feed_le(s,f) __ast_smoother_feed(s, f, 0)
#else
#define ast_smoother_feed_be(s,f) __ast_smoother_feed(s, f, 0)
#define ast_smoother_feed_le(s,f) __ast_smoother_feed(s, f, 1)
#endif
```

So `ast_smoother_feed_be()` is a no-op swap on a big-endian build. Use the
direction-named macros rather than `__ast_smoother_feed()` directly, and you get
correct behaviour on both.

Note `ast_smoother_new(int bytes)` — the smoother is configured in **bytes**, not
samples or milliseconds. The caller converts from the desired ptime.

### The smoother is *not* in the core write path

This is worth stating plainly because it is a common misconception. There is no
smoother in `channel.c`:

```
$ grep -c smoother main/channel.c
0
```

`ast_smoother_new` appears in exactly six files in this tree:

| File | Role |
|---|---|
| `main/smoother.c` | the implementation |
| `include/asterisk/smoother.h` | the header |
| `res/res_rtp_asterisk.c` | RTP ptime enforcement |
| `res/res_rtp_multicast.c` | multicast RTP |
| `res/res_fax.c` | fax |
| `addons/chan_mobile.c` | Bluetooth HFP |

**Packetization is a driver concern, not a core concern.** The core passes frames
of whatever size it has; the driver decides how to put them on the wire. If you
are writing a channel driver with a fixed packet size, you need your own smoother.
Do not expect `ast_write()` to hand you uniform frames.

## `samples`, `datalen`, `ts` and `len`

Four fields that describe duration, each with a different authority.

| Field | Unit | Set by | Valid when |
|---|---|---|---|
| `samples` | samples | the producer | always, for voice |
| `datalen` | bytes | the producer | always |
| `len` | milliseconds | jitterbuffer / IAX2 / DTMF | `AST_FRFLAG_HAS_TIMING_INFO`, or for DTMF_END always |
| `ts` | milliseconds | jitterbuffer / IAX2 | `AST_FRFLAG_HAS_TIMING_INFO` |

Rules that actually matter:

1. **`samples` is authoritative for voice duration.** `ast_samp2tv(samples, rate)`
   converts; that is what `ast_translate()` uses (`main/translate.c:611-612`,
   `:652-653`).
2. **`datalen` and `samples` are independent.** For `slin`,
   `datalen == samples * 2`. For a compressed codec there is no ratio you can
   assume. For an interpolated frame, `datalen == 0` with `samples > 0`.
3. **`len` on a DTMF_END frame is the digit duration** and is meaningful
   regardless of `AST_FRFLAG_HAS_TIMING_INFO`. This is an exception to the flag
   rule, and the DTMF code reads and writes it freely
   (`main/channel.c:3903-4016`).
4. **Translation can clear `AST_FRFLAG_HAS_TIMING_INFO`** when the sample count
   changes (`main/translate.c:656-659`). Re-check the flag after translating.
5. **`delivery` and `ts` are different things.** `delivery` is an absolute
   wall-clock `struct timeval`; `ts` is a relative millisecond timestamp from the
   protocol. `ast_translate()` maintains `delivery` and merely copies `ts`.

### Getting the sample rate

`ast_format_get_sample_rate(f->subclass.format)`. Never hardcode 8000 — Asterisk
has been wideband-capable for a long time, and the `XXX` comment at
`main/channel.c:4151-4157` describes real corner cases that arise when sample
rates differ between the read and write paths on bridged channels.

## Frame math helpers

`main/frame.c` provides a few in-place operations on signed-linear frames.

### `ast_frame_adjust_volume()`

`main/frame.c:787-810`:

```c
int ast_frame_adjust_volume(struct ast_frame *f, int adjustment);
```

`adjustment` is a power of two shift: positive multiplies, negative divides. Only
works on slin frames; returns -1 otherwise. There is a float variant,
`ast_frame_adjust_volume_float()` (`main/frame.c:812-835`), taking a
multiplicative `float`.

### `ast_frame_slinear_sum()`

`main/frame.c:837-857`. Mixes `f2` into `f1` in place. Note the format check uses
a deliberately inverted-looking comparison:

```c
	if (ast_format_cmp(f1->subclass.format, ast_format_slin) != AST_FORMAT_CMP_NOT_EQUAL)
```

`ast_format_cmp()` returns one of three values (`EQUAL`, `NOT_EQUAL`,
`SUBSET`), so "not NOT_EQUAL" accepts both `EQUAL` and `SUBSET`. It is correct;
it just reads like a typo. Do not "fix" it to `== AST_FORMAT_CMP_EQUAL`.

### `ast_frame_clear()`

`main/frame.c:859-869`. Zeroes the payload of a frame **and every frame linked
after it** — another list-walking function. Useful for muting without
reallocating.

### `ast_swapcopy_samples()` and the byteswap helpers

`main/frame.c:396-...` plus `ast_frame_byteswap_le()` / `ast_frame_byteswap_be()`
declared in `include/asterisk/frame.h`. On a matching-endian build the byteswap
macros compile to nothing.

---

Next: [07-interception.md](07-interception.md) — framehooks, audiohooks and
bridging.
