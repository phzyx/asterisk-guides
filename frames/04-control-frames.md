# 04 — Control frames

**Target tree:** Asterisk, the `24` branch. Line numbers and quoted code may
differ on newer releases and on other branches; re-grep for the symbol name if
you are reading against a different tree.

`AST_FRAME_CONTROL` is how Asterisk does in-band signalling between channel
drivers, the core, bridges and applications. `subclass.integer` holds an
`enum ast_control_frame_type`; `data.ptr`/`datalen` hold a control-specific
payload struct when one exists.

## The wire-compatibility warning

`include/asterisk/frame.h:265-276`, repeated verbatim again at `:316-328`:

> **IAX2 sends these values out over the wire.** To prevent future
> incompatibilities, pick the next value in the enum from whatever is on the
> current trunk. If you lose the merge race you need to fix the previous branches
> to match what is on trunk. In addition you need to change chan_iax2 to
> explicitly allow the control frame over the wire if it makes sense for the
> frame to be passed to another Asterisk instance.

Two practical consequences: control values are append-only, and a new control
frame is **not** forwarded over IAX2 unless `chan_iax2` is taught about it
explicitly. Default is deny.

## The catalog

Values 1-37 are the classic block (`include/asterisk/frame.h:278-314`). Values
1000+ are stream and recording manipulation, deliberately above the 8-bit range
(`include/asterisk/frame.h:330-332`):

> Control frames used to manipulate a stream on a channel. The values for these
> must be greater than the allowed value for a 8-bit char, so that they avoid
> conflicts with DTMF values.

| # | Name | Payload | Notes |
|---|---|---|---|
| 1 | `AST_CONTROL_HANGUP` | none | Other end hung up. Terminates the read loop. |
| 2 | `AST_CONTROL_RING` | none | Local ring. Audible. |
| 3 | `AST_CONTROL_RINGING` | none | Remote end is ringing. Audible. |
| 4 | `AST_CONTROL_ANSWER` | none; `subclass.topology` may be set | Handled specially in `__ast_read()`; owns a topology ref |
| 5 | `AST_CONTROL_BUSY` | none | Audible. Falls back to the "busy" tone. |
| 6 | `AST_CONTROL_TAKEOFFHOOK` | none | |
| 7 | `AST_CONTROL_OFFHOOK` | none | |
| 8 | `AST_CONTROL_CONGESTION` | none | Audible. Falls back to "congestion" tone. |
| 9 | `AST_CONTROL_FLASH` | none | Hook flash. Raises a channel event in `__ast_read()`. |
| 10 | `AST_CONTROL_WINK` | none | Same. |
| 11 | `AST_CONTROL_OPTION` | `struct ast_option_header` | Set a low-level tech option |
| 12 | `AST_CONTROL_RADIO_KEY` | none | |
| 13 | `AST_CONTROL_RADIO_UNKEY` | none | |
| 14 | `AST_CONTROL_PROGRESS` | none | Early media / call progress |
| 15 | `AST_CONTROL_PROCEEDING` | none | |
| 16 | `AST_CONTROL_HOLD` | optional music class string | Audible. Sets hold state. |
| 17 | `AST_CONTROL_UNHOLD` | none | The one control that *stops* being audible |
| 18 | `AST_CONTROL_VIDUPDATE` | none | Request a video keyframe |
| 19 | `_XXX_AST_CONTROL_T38` | — | **Deprecated, no longer supported.** Use `AST_CONTROL_T38_PARAMETERS` |
| 20 | `AST_CONTROL_SRCUPDATE` | none | Media source changed |
| 21 | `AST_CONTROL_TRANSFER` | `enum ast_control_transfer` | Transfer request status |
| 22 | `AST_CONTROL_CONNECTED_LINE` | packed buffer | Build with `ast_connected_line_build_data()` |
| 23 | `AST_CONTROL_REDIRECTING` | packed buffer | Build with `ast_redirecting_build_data()` |
| 24 | `AST_CONTROL_T38_PARAMETERS` | `struct ast_control_t38_parameters` | T.38 negotiation |
| 25 | `AST_CONTROL_CC` | CC-specific | Call completion is possible |
| 26 | `AST_CONTROL_SRCCHANGE` | none | Source changed; needs a **new RTP SSRC** |
| 27 | `AST_CONTROL_READ_ACTION` | `struct ast_control_read_action_payload` | Executed inside `__ast_read()`, then swallowed |
| 28 | `AST_CONTROL_AOC` | encoded AOC blob | Encode with `ast_aoc_encode()` |
| 29 | `AST_CONTROL_END_OF_Q` | none | Queue sentinel for softhangup |
| 30 | `AST_CONTROL_INCOMPLETE` | none | Audible (congestion tone) |
| 31 | `AST_CONTROL_MCID` | none | Malicious caller identification |
| 32 | `AST_CONTROL_UPDATE_RTP_PEER` | none | Interrupt the bridge to re-establish direct RTP |
| 33 | `AST_CONTROL_PVT_CAUSE_CODE` | `struct ast_control_pvt_cause_code` | Tech-specific cause for branching dials |
| 34 | `AST_CONTROL_MASQUERADE_NOTIFY` | notify data | **Never sent as a frame** — see below |
| 35 | `AST_CONTROL_STREAM_TOPOLOGY_REQUEST_CHANGE` | topology | Multistream renegotiation request |
| 36 | `AST_CONTROL_STREAM_TOPOLOGY_CHANGED` | topology | Renegotiation completed |
| 37 | `AST_CONTROL_STREAM_TOPOLOGY_SOURCE_CHANGED` | none | A source stream changed source |
| 1000 | `AST_CONTROL_STREAM_STOP` | none | Playback control |
| 1001 | `AST_CONTROL_STREAM_SUSPEND` | none | |
| 1002 | `AST_CONTROL_STREAM_RESTART` | none | |
| 1003 | `AST_CONTROL_STREAM_REVERSE` | none | |
| 1004 | `AST_CONTROL_STREAM_FORWARD` | none | |
| 1100 | `AST_CONTROL_RECORD_CANCEL` | none | Stop and discard the file |
| 1101 | `AST_CONTROL_RECORD_STOP` | none | Stop, keep the file |
| 1102 | `AST_CONTROL_RECORD_SUSPEND` | none | Toggle suspend |
| 1103 | `AST_CONTROL_RECORD_MUTE` | none | Toggle mute (write silence) |

### `AST_CONTROL_MASQUERADE_NOTIFY` is not a frame

Its doc comment (`include/asterisk/frame.h:311`) says so outright:

> A masquerade is about to begin/end. (Never sent as a frame but directly with
> ast_indicate_data().)

A reminder that `enum ast_control_frame_type` is overloaded: it is simultaneously
the subclass space for control *frames* and the condition space for
`ast_indicate_data()`. Not every value is legal in both.

## Payload structs

### `struct ast_control_read_action_payload`

`include/asterisk/frame.h:352-369`:

```c
enum ast_frame_read_action {
	AST_FRAME_READ_ACTION_CONNECTED_LINE_MACRO,
	AST_FRAME_READ_ACTION_SEND_TEXT,
	AST_FRAME_READ_ACTION_SEND_TEXT_DATA,
};

struct ast_control_read_action_payload {
	/* An indicator to ast_read of what action to
	 * take with the frame;
	 */
	enum ast_frame_read_action action;
	/* The size of the frame's payload
	 */
	size_t payload_size;
	/* A payload for the frame.
	 */
	unsigned char payload[0];
};
```

A flexible-array struct, so `datalen` must be
`sizeof(struct ast_control_read_action_payload) + payload_size`.

This is the mechanism for "run this on the channel's own thread" — the three
actions are dispatched inside `__ast_read()`'s control case
(`main/channel.c:3835-3863`) and the frame is then **swallowed**, with
`&ast_null_frame` returned in its place. That is the point: the work has to happen
on the thread that owns the channel, not on whatever thread queued the request.

### `struct ast_control_t38_parameters`

`include/asterisk/frame.h:398-407`:

```c
struct ast_control_t38_parameters {
	enum ast_control_t38 request_response;			/*!< Request or response of the T38 control frame */
	unsigned int version;					/*!< Supported T.38 version */
	unsigned int max_ifp; 					/*!< Maximum IFP size supported */
	enum ast_control_t38_rate rate;				/*!< Maximum fax rate supported */
	enum ast_control_t38_rate_management rate_management;	/*!< Rate management setting */
	unsigned int fill_bit_removal:1;			/*!< Set if fill bit removal can be used */
	unsigned int transcoding_mmr:1;				/*!< Set if MMR transcoding can be used */
	unsigned int transcoding_jbig:1;			/*!< Set if JBIG transcoding can be used */
};
```

With the state enum (`include/asterisk/frame.h:371-378`):

```c
enum ast_control_t38 {
	AST_T38_REQUEST_NEGOTIATE = 1,	/*!< Request T38 on a channel (voice to fax) */
	AST_T38_REQUEST_TERMINATE,	/*!< Terminate T38 on a channel (fax to voice) */
	AST_T38_NEGOTIATED,		/*!< T38 negotiated (fax mode) */
	AST_T38_TERMINATED,		/*!< T38 terminated (back to voice) */
	AST_T38_REFUSED,		/*!< T38 refused for some reason (usually rejected by remote end) */
	AST_T38_REQUEST_PARMS,		/*!< request far end T.38 parameters for a channel in 'negotiating' state */
};
```

`AST_T38_RATE_14400` is deliberately `0` so an unspecified rate defaults to the
highest (`include/asterisk/frame.h:386-390`), per the ITU T.38 Implementors'
Guide.

`indicate_data_internal()` maintains the channel's T.38 active flag from these
states (`main/channel.c:4518-4529`):

```c
	case AST_CONTROL_T38_PARAMETERS:
		t38_parameters = data;
		switch (t38_parameters->request_response) {
		case AST_T38_REQUEST_NEGOTIATE:
		case AST_T38_NEGOTIATED:
			ast_channel_set_is_t38_active_nolock(chan, 1);
			break;
		case AST_T38_REQUEST_TERMINATE:
		case AST_T38_TERMINATED:
		case AST_T38_REFUSED:
			ast_channel_set_is_t38_active_nolock(chan, 0);
			break;
```

### `struct ast_control_pvt_cause_code`

`include/asterisk/frame.h:417-422`:

```c
struct ast_control_pvt_cause_code {
	char chan_name[AST_CHANNEL_NAME];	/*!< Name of the channel that originated the cause information */
	unsigned int emulate_sip_cause:1;	/*!< Indicates whether this should be used to emulate SIP_CAUSE support */
	int ast_cause;				/*!< Asterisk cause code associated with this message */
	char code[1];				/*!< Tech-specific cause code information, beginning with the name of the tech */
};
```

Another flexible-array-by-`[1]` struct. Consumed by
`ast_channel_hangupcause_hash_set()` (`main/channel.c:4368-4380`), which stores it
and optionally writes the `HASH(SIP_CAUSE,<chan>)` dialplan variable
(`main/channel.c:4376-4379`):

```c
	if (cause_code->emulate_sip_cause) {
		snprintf(causevar, sizeof(causevar), "HASH(SIP_CAUSE,%s)", cause_code->chan_name);
		ast_func_write(chan, causevar, cause_code->code);
	}
```

### `AST_CONTROL_TRANSFER`

The payload is just the enum (`include/asterisk/frame.h:409-415`):

```c
enum ast_control_transfer {
	AST_TRANSFER_SUCCESS = 0, /*!< Transfer request on the channel worked */
	AST_TRANSFER_FAILED,      /*!< Transfer request on the channel failed */
	AST_TRANSFER_PROGRESS,    /*!< Transfer request on the channel is in progress */
	AST_TRANSFER_UNAVAILABLE, /*!< Transfer request on the channel is unavailable */
	AST_TRANSFER_INVALID,     /*!< Invalid state for none of the above. */
};
```

It is worth knowing that this used to be a two-valued success/failure enum. The
three later members carry real meaning, so a consumer that only tests for
`AST_TRANSFER_SUCCESS` and treats everything else as failure is now wrong:

- `AST_TRANSFER_PROGRESS` is *not* terminal. `res_pjsip_refer` maps it to a SIP
  `100 Trying` NOTIFY and leaves the subscription `PJSIP_EVSUB_STATE_ACTIVE`,
  whereas `SUCCESS`, `FAILED` and `UNAVAILABLE` all terminate it
  (`res/res_pjsip_refer.c:408-423`).
- `AST_TRANSFER_UNAVAILABLE` is distinct from `FAILED` on the wire — `503`
  rather than `603` — even though `ast_channel_transfer_state2str()` collapses
  both to `"channel_declined"` for ARI (`main/stasis_channels.c:1652-1665`).
- `AST_TRANSFER_INVALID` is a sentinel for "no state yet", not an outcome.
  `res_pjsip_refer` initialises `last_response` to it
  (`res/res_pjsip_refer.c:1599`) and the ARI `ChannelTransfer` event omits the
  `state` field entirely when it still holds that value
  (`main/stasis_channels.c:1755-1757`).

Note also that `res_pjsip_refer` guards on
`f->datalen >= sizeof(enum ast_control_transfer)` before dereferencing
(`res/res_pjsip_refer.c:405-406`) — the usual discipline for a control payload.

### Connected line, redirecting and AOC payloads

These three carry a *serialized* representation rather than a struct, because they
cross process and protocol boundaries. Pack and unpack with the dedicated helpers
rather than by hand:

| Control | Pack | Unpack |
|---|---|---|
| `AST_CONTROL_CONNECTED_LINE` | `ast_connected_line_build_data()` — `include/asterisk/channel.h:3832` | `ast_connected_line_parse_data()` — `:3851` |
| `AST_CONTROL_REDIRECTING` | `ast_redirecting_build_data()` — `include/asterisk/channel.h:3897` | `ast_redirecting_parse_data()` — `:3915` |
| `AST_CONTROL_AOC` | `ast_aoc_encode()` — `include/asterisk/aoc.h:257` | `ast_aoc_decode()` — `:244` |

The pack signatures. `include/asterisk/channel.h:3832`:

```c
int ast_connected_line_build_data(unsigned char *data, size_t datalen, const struct ast_party_connected_line *connected, const struct ast_set_party_connected_line *update);
```

`include/asterisk/channel.h:3897`:

```c
int ast_redirecting_build_data(unsigned char *data, size_t datalen, const struct ast_party_redirecting *redirecting, const struct ast_set_party_redirecting *update);
```

`include/asterisk/aoc.h:257`:

```c
struct ast_aoc_encoded *ast_aoc_encode(struct ast_aoc_decoded *decoded, size_t *out_size, struct ast_channel *chan);
```

The `build_data` functions return the number of bytes written (or -1), which is
what you put in the frame's `datalen`. The AOC encoded object must be released
with `ast_aoc_destroy_encoded()` (`include/asterisk/aoc.h:231`).

`indicate_data_internal()` preprocesses the first two before handing them to the
driver (`main/channel.c:4502-4507`), and will quietly succeed without calling the
driver if the preprocessing says the update is redundant:

```c
	case AST_CONTROL_CONNECTED_LINE:
		if (indicate_connected_line(chan, data, datalen)) {
			res = 0;
			return res;
		}
		break;
```

## Queuing control frames

Three convenience functions, all building **stack** frames —
`main/channel.c:1288-1312`:

```c
/*! \brief Queue a control frame */
int ast_queue_control(struct ast_channel *chan, enum ast_control_frame_type control)
{
	struct ast_frame f = { AST_FRAME_CONTROL, .subclass.integer = control };
	return ast_queue_frame(chan, &f);
}

/*! \brief Queue a control frame with payload */
int ast_queue_control_data(struct ast_channel *chan, enum ast_control_frame_type control,
			   const void *data, size_t datalen)
{
	struct ast_frame f = { AST_FRAME_CONTROL, .subclass.integer = control, .data.ptr = (void *) data, .datalen = datalen };
	return ast_queue_frame(chan, &f);
}

/*! \brief Queue an ANSWER control frame with topology */
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

`ast_queue_control_data()` takes a `const void *` and the queue `frdup`s it, so
**the payload only needs to outlive the call**, not the frame. That is why
callers can pass a struct that lives on their stack.

`ast_queue_answer()`'s cast away from `const` is safe because `frdup` bumps the
topology reference rather than mutating it.

## Indicating control frames: the `ast_indicate` path

The *other* direction — telling a channel driver to signal something — does not go
through the write path. It goes through `ast_indicate_data()`.

```
ast_indicate(chan, condition)                        main/channel.c:4296
    └─> ast_indicate_data(chan, condition, NULL, 0)  main/channel.c:4672
            └─> indicate_data_internal(...)          main/channel.c:4489
```

`ast_indicate()` is a one-liner (`main/channel.c:4298-4301`):

```c
int ast_indicate(struct ast_channel *chan, int condition)
{
	return ast_indicate_data(chan, condition, NULL, 0);
}
```

### `indicate_data_internal()` in order

`main/channel.c:4492-4674`. The structure matters because it determines what a
driver can and cannot rely on:

1. **Pre-processing switch** (`main/channel.c:4501-4536`) — connected line, redirecting, hold
   state, T.38 active flag. Note `AST_CONTROL_HOLD`/`UNHOLD` set the channel's
   hold state *before* the driver sees the indication (`main/channel.c:4514-4517`):
   ```c
   	case AST_CONTROL_HOLD:
   	case AST_CONTROL_UNHOLD:
   		ast_channel_hold_state_set(chan, _condition);
   		break;
   ```

2. **Visible-indication bookkeeping** (`main/channel.c:4538-4544`):
   ```c
   	if (is_visible_indication(condition)) {
   		/* A new visible indication is requested. */
   		ast_channel_visible_indication_set(chan, _condition);
   	} else if (condition == AST_CONTROL_UNHOLD || _condition < 0) {
   		/* Visible indication is cleared/stopped. */
   		ast_channel_visible_indication_set(chan, 0);
   	}
   ```

3. **Ask the driver** (`main/channel.c:4546-4557`). If `tech->indicate` returns 0, we are done.
   If there is no `indicate` callback at all, `res = -1` and we fall through to
   emulation.

4. **Negative condition means stop** (`main/channel.c:4565-4570`):
   ```c
   	if (_condition < 0) {
   		/* Stop any tones that are playing */
   		ast_playtones_stop(chan);
   		res = 0;
   		return res;
   	}
   ```
   With a comment at `main/channel.c:4562-4564` explaining why the comparison is on the `int`
   and not the enum: a compiler may optimise away a `< 0` test on an enum with no
   negative constants.

5. **Tone emulation switch** (`main/channel.c:4573-4654`). RINGING → "ring", BUSY → "busy",
   INCOMPLETE/CONGESTION → "congestion". Everything else is a no-op that sets
   `res = 0`.

6. **Play the tone if we found one** (`main/channel.c:4656-4666`), logging at debug level 1 that
   the driver did not support the indication and we are faking it.

7. **Warn if nothing handled it** (`main/channel.c:4668-4671`).

### `is_visible_indication()`

`main/channel.c:4303-4366`. It answers "can the far end *hear* this?", which
determines whether the core needs to remember and re-apply it across a
masquerade or bridge change.

Returns 1 for exactly six conditions (`main/channel.c:4351-4358`):

```c
	case AST_CONTROL_INCOMPLETE:
	case AST_CONTROL_CONGESTION:
	case AST_CONTROL_BUSY:
	case AST_CONTROL_RINGING:
	case AST_CONTROL_RING:
	case AST_CONTROL_HOLD:
		/* You can hear these */
		return 1;
```

and `AST_CONTROL_UNHOLD` is called out as the inverse (`:4357-4359`):

```c
	case AST_CONTROL_UNHOLD:
		/* This is a special case.  You stop hearing this. */
		break;
```

Note the deliberate absence of a `default:` label (`main/channel.c:4303-4304`):

> Don't include a default case here so that we get compiler warnings when a new
> type is added.

Both the big switches in this function follow that pattern. **If you add a control
frame type, the compiler will tell you to come here** — which is the intended
design, and a good reason not to add a `default:` when you do.

## `AST_CONTROL_END_OF_Q`: the softhangup sentinel

Worth its own section because it is unintuitive.

When a channel is soft-hung-up, there may still be useful frames sitting in the
readq. Dropping them would lose, say, the DTMF digit that triggered the hangup. So
the core queues an `AST_CONTROL_END_OF_Q` frame to mark the boundary
(`main/channel.c:3558-3571`), and `__ast_read()` keeps serving the readq until it
reaches that marker.

`__ast_read()` then treats `END_OF_Q` like `HANGUP` — frees the frame and returns
NULL (`main/channel.c:3706-3718`, the free itself at `main/channel.c:3713-3714`):

```c
				ast_frfree(f);
				f = NULL;
```

And `__ast_queue_frame()` collapses the two: if a HANGUP arrives while an
`END_OF_Q` is already the last frame in the queue, the `END_OF_Q` is replaced
rather than both being kept (`main/channel.c:1057-1086`).

---

Next: [05-channel-path.md](05-channel-path.md) — the read and write paths in
full.
