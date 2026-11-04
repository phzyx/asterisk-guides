# 01 — Anatomy of `struct ast_channel`

Every `file:line` citation in this guide was verified against the `24` branch.
Newer releases and other branches may differ — line numbers drift — so if you
are reading this against a different tree, treat the citations as approximate
and re-grep for the symbol name.

## 1.1 Where the struct lives

`struct ast_channel` is opaque to almost the entire tree. `include/asterisk/channel.h:972`
contains nothing but:

```c
struct ast_channel;
```

The real definition is in `main/channel_private.h:48-204`, a private header that
is `#include`d by exactly three translation units — and in a default build, only
two of those three includes are live:

| File | Why it needs the layout |
|---|---|
| `main/channel_internal_api.c:51` | It *is* the accessor layer |
| `main/channelstorage_ao2_legacy.c:24` | Sets `linked_in_container` directly (`:47`, `:56`) |
| `main/channelstorage.c:160` | Test-only: the include sits inside `#ifdef TEST_FRAMEWORK` (`:158`), for `mock_channel_destructor` |

`main/channel.c` is not on that list. The 11,220-line core of the channel
implementation reaches its own object through the same public accessors that
`app_dial` uses. If you are adding to `channel.c` and reach for `chan->field`,
it will not compile.

The practical consequence: **adding a field is a two-file change.** The field goes
in `main/channel_private.h`, and a getter (and usually a setter) goes in
`main/channel_internal_api.c` — which currently holds 233 `ast_channel_*`
definitions. `include/asterisk/channel.h` carries roughly 330 `ast_channel_*`
prototypes.

## 1.2 The field-ordering convention

Before adding a field, read the note attached to the struct
(`main/channel_private.h:37-48`):

```c
/*!
 * \brief Main Channel structure associated with a channel.
 *
 * \note When adding fields to this structure, it is important to add the field
 *       'in position' with like-aligned fields, so as to keep the compiler from
 *       having to add padding to align fields. The structure's fields are sorted
 *       in this order: pointers, structures, long, int/enum, short, char. This
 *       is especially important on 64-bit architectures, where mixing 4-byte
 *       and 8-byte fields causes 4 bytes of padding to be added before many
 *       8-byte fields.
 */
struct ast_channel {
```

This is why the field order looks arbitrary when read top to bottom: it is sorted
by alignment, not by meaning. The groupings below are therefore a reading aid, not
the physical layout.

## 1.3 The field groups

### Technology and driver private state

```c
	const struct ast_channel_tech *tech;		/*!< Technology (point to channel driver) */
	void *tech_pvt;					/*!< Private data used by the technology driver */
```

`main/channel_private.h:49-50`. `tech` is the driver's vtable — covered in
chapter 07. `tech_pvt` is the driver's own per-channel object and the core never
looks inside it.

### The masquerade pair

```c
	struct ast_channel *masq;			/*!< Channel that will masquerade as us */
	struct ast_channel *masqr;			/*!< Who we are masquerading as */
```

`main/channel_private.h:54-55`. Two pointers that let one channel take over
another's identity. This is the mechanism behind transfers and local channel
optimisation, and it is why a channel pointer you hold can change what it
represents. Chapter 06.

### Media state

```c
	struct ast_format_cap *nativeformats;         /*!< Kinds of data this channel can natively handle */
	struct ast_format *readformat;            /*!< Requested read format (after translation) */
	struct ast_format *writeformat;           /*!< Requested write format (before translation) */
	struct ast_format *rawreadformat;         /*!< Raw read format (before translation) */
	struct ast_format *rawwriteformat;        /*!< Raw write format (after translation) */
```

`main/channel_private.h:168-172`. Note the parenthetical comments: `readformat` is
what the core hands *you* after translating, `rawreadformat` is what the driver
produces. The translation paths between them are `readtrans` and `writetrans`
(`:65-66`). Every one of these five is an ao2-refcounted object. The frames guide
covers how they are used; [06-translation-and-timing.md](../frames/06-translation-and-timing.md)
is the relevant chapter.

### Dialplan position

```c
	char context[AST_MAX_CONTEXT];			/*!< Dialplan: Current extension context */
	char exten[AST_MAX_EXTENSION];			/*!< Dialplan: Current extension number */
	char lastcontext[AST_MAX_CONTEXT];		/*!< Dialplan: Previous extension context */
	char lastexten[AST_MAX_EXTENSION];		/*!< Dialplan: Previous extension number */
```

`main/channel_private.h:183-186`, with `priority` at `:154` and the `pbx` pointer
at `:64`. These four are fixed-size char arrays rather than string fields, so the
setters copy.

### Identity

```c
	struct ast_channel_id uniqueid;		/*!< Unique Channel Identifier - can be specified on creation */
	struct ast_channel_id linkedid;		/*!< Linked Channel Identifier - oldest propagated when bridged */
```

`main/channel_private.h:89-90`, where `ast_channel_id` is itself declared just
above at `:30-35`:

```c
struct ast_channel_id {
	time_t creation_time;				/*!< Creation time */
	int creation_unique;				/*!< sub-second unique value */
	char unique_id[AST_MAX_UNIQUEID];	/*!< Unique Identifier */
	char tenant_id[AST_MAX_TENANT_ID];	/*!< Multi-tenant identifier */
};
```

`AST_MAX_TENANT_ID` is 64 (`include/asterisk/channel.h:149`). The `creation_time`
and `creation_unique` fields are not decoration: the doc comment at `:27-28` notes
that creation time is what determines linkedid propagation, so the older channel's
linkedid wins when two channels are bridged.

The getters for both assert that the id is non-empty rather than returning an
empty string (`main/channel_internal_api.c:161-165`, and `:167-171` for
`linkedid`):

```c
const char *ast_channel_uniqueid(const struct ast_channel *chan)
{
	ast_assert(chan->uniqueid.unique_id[0] != '\0');
	return chan->uniqueid.unique_id;
}
```

### Party identification

Five separate structures, at `main/channel_private.h:100-122`: `dialed` (`:100`,
who was originally asked for), `caller` (`:107`, this channel's own caller id when
originating), `connected` (`:114`, who is on the other end right now),
`connected_indicated` (`:119`, what was last actually signalled outward), and
`redirecting` (`:122`). The split between `connected`
and `connected_indicated` exists so the core can tell "what is true" from "what the
far end has been told", which is the distinction that makes connected line updates
idempotent.

### I/O and blocking

```c
	int blocker_tid;					/*!< If anyone is blocking, this is their thread id */
	AST_VECTOR(, int) fds;				/*!< File descriptors for channel -- Drivers will poll on
							 *   these file descriptors, so at least one must be non -1.
							 *   See \arg \ref AstFileDesc */
```

`main/channel_private.h:141-144`. The fd set is a vector, not a fixed array, with
`fdno` at `:147` recording which one fired, `alertpipe[2]` at `:167`, `timingfd` at
`:151` and the `timer` object that provided it at `:181`. The read queue (`readq`,
`:132`) and jitterbuffer state (`jb`, `:133`) sit alongside. Who blocks is tracked
by `blocker` (`:93`, a `pthread_t`), `blocker_tid` (`:141`) and `blockproc`
(`:56`, the name of the function that blocked). `blockproc` is set by the
`CHECK_BLOCKING` macro (`include/asterisk/channel.h:2932-2945`) to
`__PRETTY_FUNCTION__`, and it exists to name the blocking site in diagnostics:
`CHECK_BLOCKING`'s own debug log when a second thread tries to block on the same
channel (`channel.h:2938`), the warning on a hard hangup of a blocked channel
(`main/channel.c:2605`), and `DumpChan` (`apps/app_dumpchan.c:169`). Note that
`core show channel` does *not* print it — `handle_showchan` in `main/cli.c` never
reads `blockproc`.

### DTMF emulation state

`emulate_dtmf_duration` (`:173`), `dtmf_digit_to_emulate` (`:190`),
`sending_dtmf_digit` (`:191`), `sending_dtmf_tv` (`:192`), `dtmf_tv` (`:134`) and
the scratch frame `dtmff` (`:124`). The state machine that drives all of these
lives in `__ast_read` and is documented in
[the frames guide's channel path chapter](../frames/05-channel-path.md).

### Bridge attachment

```c
	struct ast_bridge *bridge;                      /*!< Bridge this channel is participating in */
	struct ast_bridge_channel *bridge_channel;/*!< The bridge_channel this channel is linked with. */
```

`main/channel_private.h:179-180`, with `unbridged` at `:187-188` — a flag whose
comment explains that it means "the bridge core needs to re-evaluate the current
bridging technology". Setting it is how the rest of the core asks for a technology
switch without knowing anything about technologies.

### Interception and attached state

`audiohooks` (`:67`), `framehooks` (`:68`), `datastores` (`:136`), `autochans`
(`:137`) and `hangup_handlers` (`:135`). The first two are the subject of
[the frames guide's interception chapter](../frames/07-interception.md). Datastores
are the general-purpose "attach my module's data to this channel" mechanism and are
the correct answer whenever you are tempted to add a field to the struct.

### Stasis and snapshots

```c
	struct stasis_topic *topic;		/*!< Topic for this channel */
	struct stasis_forward *channel_forward; /*!< Subscription for event forwarding to all channel topic */
	struct stasis_forward *endpoint_forward;	/*!< Subscription for event forwarding to endpoint's topic */
```

`main/channel_private.h:193-195`, plus `snapshot` (`:199`) and
`snapshot_segment_flags` (`:200`). Every channel owns a Stasis topic and forwards
to the all-channels topic and, if it has one, its endpoint's topic. The cached
`snapshot` and the invalidation flags are what make §1.5 below work.

### Streams

`stream_topology` (`:196`), `stream_topology_change_source` (`:197`) and
`default_streams[AST_MEDIA_TYPE_END]` (`:198`) — the multi-stream support added
in Asterisk 15 and the reason frames carry a stream number.

### Storage bookkeeping

```c
	int linked_in_container; /*!< Whether this channel is linked in a storage container */
```

`main/channel_private.h:201`. Small, and load-bearing — see §1.6.

## 1.4 The accessor layer

Most string-field accessors are macro-generated. The plain setter macro
(`main/channel_internal_api.c:64-82`) is:

```c
#define DEFINE_STRINGFIELD_SETTERS_FOR(field, assert_on_null) \
void ast_channel_##field##_set(struct ast_channel *chan, const char *value) \
{ \
	if ((assert_on_null)) ast_assert(!ast_strlen_zero(value)); \
	if (!strcmp(value, chan->field)) return; \
	ast_string_field_set(chan, field, value); \
} \
```

and the getter macro is as thin as it looks (`:119-122`):

```c
#define DEFINE_STRINGFIELD_GETTER_FOR(field) const char *ast_channel_##field(const struct ast_channel *chan) \
{ \
	return chan->field; \
}
```

Two things to notice in the setter. First, it short-circuits on an unchanged value
— `if (!strcmp(value, chan->field)) return;` — so setting a field to what it
already holds is free, including of the side effects below. Second,
`assert_on_null` is a per-field policy in principle, but **no invocation in the
tree actually enables it**: every macro call at `:108-117` passes `0` for that
argument, so no macro-generated string setter rejects an empty value. Mind the
argument order when reading those lines — the four-argument variant in §1.5 is
`(field, publish, assert_on_null, invalidate)`, so the `1` in
`DEFINE_STRINGFIELD_SETTERS_AND_INVALIDATE_FOR(language, 1, 0, ...)` is `publish`,
not an assertion. The only live null assertion on a string field is the
hand-written `ast_channel_name_set` in §1.6.

## 1.5 Setters with side effects

The reason to care which macro generated a setter is that the second variant does
more than assign. From `main/channel_internal_api.c:84-92`:

```c
#define DEFINE_STRINGFIELD_SETTERS_AND_INVALIDATE_FOR(field, publish, assert_on_null, invalidate) \
void ast_channel_##field##_set(struct ast_channel *chan, const char *value) \
{ \
	if ((assert_on_null)) ast_assert(!ast_strlen_zero(value)); \
	if (!strcmp(value, chan->field)) return; \
	ast_string_field_set(chan, field, value); \
	ast_channel_snapshot_invalidate_segment(chan, invalidate); \
	if (publish && ast_channel_internal_is_finalized(chan)) ast_channel_publish_snapshot(chan); \
} \
```

So assigning to one of these fields can publish a Stasis message synchronously,
from your thread, while you hold the channel lock. The fields and their segments
(`:108-117`):

| Field | Invalidates | Publishes |
|---|---|---|
| `language` | `..._BASE` | yes |
| `accountcode` | `..._BASE` | yes |
| `peeraccount` | `..._PEER` | yes |
| `userfield` | `..._BASE` | no |
| `hangupsource` | `..._HANGUP` | no |
| `musicclass`, `latest_musicclass`, `call_forward`, `parkinglot`, `dialcontext` | — | no |

Only three of the ten string setters publish: `language`, `accountcode` and
`peeraccount`. `userfield` and `hangupsource` invalidate their segment and stop
there, leaving the publish to whoever next asks for a snapshot.

The segment flags are a bitfield —
`enum ast_channel_snapshot_segment_invalidation`,
`include/asterisk/stasis_channels.h:36-51`, with the seven enumerators
(`BRIDGE`, `DIALPLAN`, `CONNECTED`, `CALLER`, `HANGUP`, `PEER`, `BASE`) at
`:38-50` and starting at `(1 << 1)` rather than `(1 << 0)` — so a snapshot is
rebuilt piecewise rather than wholesale.

The `publish &&` guard matters: `ast_channel_internal_is_finalized()`
(`main/channel_internal_api.c:1454`) is false for a channel still being
constructed, which is what stops allocation from emitting a storm of snapshot
messages for a channel nobody has seen yet. Chapter 02 covers when `finalized`
gets set.

## 1.6 The name is immutable once the channel is linked

`ast_channel_name_set` is hand-written rather than macro-generated, and the reason
is its second assertion (`main/channel_internal_api.c:136-143`):

```c
void ast_channel_name_set(struct ast_channel *chan, const char *value)
{
	ast_assert(!ast_strlen_zero(value));
	ast_assert(!chan->linked_in_container);
	if (!strcmp(value, chan->name)) return;
	ast_string_field_set(chan, name, value);
	ast_channel_snapshot_invalidate_segment(chan, AST_CHANNEL_SNAPSHOT_INVALIDATE_BASE);
}
```

The channel storage container indexes by name, so renaming a linked channel would
corrupt the index. All three name-setting entry points carry the assertion
(`:139`, `:147`, `:155`). Under a production build with asserts compiled out this
becomes silent corruption rather than a crash, which is worth knowing when you
read a bug report about channels that cannot be found by name.

## 1.7 Rules

| Rule | Why |
|---|---|
| Never add `chan->field` access outside the three private-header files | It will not compile, and the fix is an accessor, not an `#include` |
| Prefer a datastore over a new struct field | Datastores are the designed extension point; fields cost an accessor pair and a layout review |
| Treat a setter as potentially publishing | Some setters emit Stasis messages synchronously (§1.5) |
| Do not rename a linked channel | Asserted, and silently corrupting with asserts off (§1.6) |
| Respect the alignment-sorted field order when adding | `main/channel_private.h:40-46` |
| Assume every pointer-to-ao2-object field needs a ref bump to outlive the lock | Formats, topology, snapshot, bridge — covered in chapter 03 |
