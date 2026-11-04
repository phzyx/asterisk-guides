# Snapshots, Stasis and Debugging

Every `file:line` citation in this guide was verified against the `24` branch.
Newer releases and other branches may differ — line numbers drift — so if you
are reading this against a different tree, treat the citations as approximate
and re-grep for the symbol name.

Everything that observes a channel from outside the thread that owns it — AMI,
ARI, CDR, CEL, the CLI's channel list — does not look at the channel. It looks
at a *snapshot*: a frozen, immutable, refcounted copy of the parts of the
channel that outside observers care about. There are therefore two
representations of every call leg in a running Asterisk, and they are not
updated at the same moment by the same code.

Almost every channel observability bug is a disagreement between the two.

## Two views of one channel

```
   the owning thread                        every observer
   ─────────────────                        ──────────────

   struct ast_channel                   struct ast_channel_snapshot
   (private, mutable,                   (public, immutable,
    lock-protected)                      refcounted, 7 segments)
        │                                        ▲
        │  ast_channel_exten_set()               │
        │  ast_channel_caller_set()              │
        │  ast_setstate()                        │
        ▼                                        │
   snapshot_segment_flags                        │
   ┌───────────────────────┐                     │
   │ BASE DIALPLAN CALLER  │   ast_channel_snapshot_create()
   │ CONNECTED BRIDGE      │─────────────────────┤  dirty segment → rebuild
   │ PEER HANGUP           │                     │  clean segment → ao2_bump
   └───────────────────────┘                     │
        │                                        │
        │  ast_channel_publish_snapshot()        │
        ▼                                        │
   ┌──────────────────────────┐          ┌───────┴────────────────┐
   │ channel_cache            │          │ chan->snapshot         │
   │ channel_cache_by_name    │◄─────────┤ (the latest, on the    │
   │ (1567 buckets each)      │  linked  │  channel itself)       │
   └──────────────────────────┘          └────────────────────────┘
        │        ▲                               │
        │        └── unlinked by                  │ stasis_publish()
        │            ast_channel_publish_         │
        │            final_snapshot()             ▼
        │                               ast_channel_topic(chan)
        │                                        │
        │                                        │ stasis_forward_all(),
        │                                        │ set up at alloc
        │                                        ▼
        │                                  channel:all
        │                                        │
        ├─ ast_channel_cache_by_name()           ├─ main/manager_channels.c (AMI)
        │     → core show channels               ├─ res/stasis/app.c        (ARI)
        ├─ ast_channel_cache_all()               ├─ main/cdr.c
        │     → ast_complete_channels            ├─ main/cel.c
        └─ ast_channel_snapshot_get_latest()     └─ res/res_chan_stats.c
              → channel blobs
```

Three facts carry most of the weight:

1. **The snapshot is segmented, and segments are shared.** A snapshot is seven
   independently refcounted objects behind one container object. Publishing a new
   snapshot rebuilds only the segments whose dirty bit is set and `ao2_bump`s the
   rest straight out of the previous snapshot. Two consecutive snapshots of a
   channel usually share five or six of their seven segments.
2. **Nothing enforces "don't publish if nothing changed."** There is a `memcmp`
   in `ast_channel_publish_snapshot` that exists to suppress a no-op publish, and
   it can never fire. Section 8.5 shows why, and section 8.4 shows the one place
   in the tree that hand-rolls a working substitute.
3. **The CLI and the event stream do not read the same thing.** `core show
   channel` reads the live channel under its lock; `core show channels` lists the
   snapshot cache but prints a count that comes from a third source entirely.
   Section 8.8 catalogues the three counts.

## 8.1 The snapshot object

`struct ast_channel_snapshot` is seven segment pointers plus six flat fields:

```c
struct ast_channel_snapshot {
	struct ast_channel_snapshot_base *base;           /*!< Base information about the channel */
	struct ast_channel_snapshot_peer *peer;           /*!< Peer information */
	struct ast_channel_snapshot_caller *caller;       /*!< Information about the caller */
	struct ast_channel_snapshot_connected *connected; /*!< Information about who this channel is connected to */
	struct ast_channel_snapshot_bridge *bridge;       /*!< Information about the bridge */
	struct ast_channel_snapshot_dialplan *dialplan;   /*!< Information about the dialplan */
	struct ast_channel_snapshot_hangup *hangup;       /*!< Hangup information */
	enum ast_channel_state state;                     /*!< State of line */
	int amaflags;                                     /*!< AMA flags for billing */
	struct ast_flags flags;                           /*!< channel flags of AST_FLAG_ type */
	struct ast_flags softhangup_flags;                /*!< softhangup channel flags */
	struct varshead *manager_vars;                    /*!< Variables to be appended to manager events */
	struct varshead *ari_vars;                        /*!< Variables to be appended to ARI events */
};
```

The doxygen above it states the contract the code relies on but does not check:

> While not enforced programmatically, this object is shared across multiple
> threads, and should be treated as an immutable object.
>
> It is guaranteed that the segments of this snapshot will always exist when
> accessing the snapshot.

The second sentence is the useful one. A consumer handed a snapshot may
dereference `snapshot->base->name` or `snapshot->dialplan->exten` without a NULL
check, because `ast_channel_snapshot_create` fails the whole allocation if any
segment fails. The exception is `old_snapshot` in a subscriber callback, which
*is* NULL for the first publish of a channel's life — section 8.6.

Each segment is its own ao2 object. The base segment is the largest and is the
only one using string fields:

```c
struct ast_channel_snapshot_base {
	AST_DECLARE_STRING_FIELDS(
		AST_STRING_FIELD(name);        /*!< ASCII unique channel name */
		AST_STRING_FIELD(uniqueid);    /*!< Unique Channel Identifier */
		AST_STRING_FIELD(accountcode); /*!< Account code for billing */
		AST_STRING_FIELD(userfield);   /*!< Userfield for CEL billing */
		AST_STRING_FIELD(language);    /*!< The default spoken language for the channel */
		AST_STRING_FIELD(type);        /*!< Type of channel technology */
		AST_STRING_FIELD(tenantid);    /*!< Channel tenant identifier */
	);
	struct timeval creationtime; /*!< The time of channel creation */
	int tech_properties;         /*!< Properties of the channel's technology */
	AST_STRING_FIELD_EXTENDED(protocol_id); /*!< Channel driver protocol id (i.e. Call-ID for chan_pjsip) */
};
```

`tech_properties` is a straight copy of `ast_channel_tech(chan)->properties`,
and it is the field that the AMI, CDR and CEL filters in section 8.6 test for
`AST_CHAN_TP_INTERNAL`. `protocol_id` is filled from the driver's
`get_pvt_uniqueid` callback — for `chan_pjsip` that is the SIP Call-ID. Both are
set by `channel_snapshot_base_create`:

```c
static struct ast_channel_snapshot_base *channel_snapshot_base_create(struct ast_channel *chan)
{
	struct ast_channel_snapshot_base *snapshot;

	snapshot = ao2_alloc_options(sizeof(*snapshot), channel_snapshot_base_dtor,
		AO2_ALLOC_OPT_LOCK_NOLOCK);
	if (!snapshot) {
		return NULL;
	}

	if (ast_string_field_init(snapshot, 256) || ast_string_field_init_extended(snapshot, protocol_id)) {
		ao2_ref(snapshot, -1);
		return NULL;
	}

	ast_string_field_set(snapshot, name, ast_channel_name(chan));
	ast_string_field_set(snapshot, type, ast_channel_tech(chan)->type);
	ast_string_field_set(snapshot, accountcode, ast_channel_accountcode(chan));
	ast_string_field_set(snapshot, userfield, ast_channel_userfield(chan));
	ast_string_field_set(snapshot, uniqueid, ast_channel_uniqueid(chan));
	ast_string_field_set(snapshot, language, ast_channel_language(chan));
	ast_string_field_set(snapshot, tenantid, ast_channel_tenantid(chan));

	snapshot->creationtime = ast_channel_creationtime(chan);
	snapshot->tech_properties = ast_channel_tech(chan)->properties;

	if (ast_channel_tech(chan)->get_pvt_uniqueid) {
		ast_string_field_set(snapshot, protocol_id, ast_channel_tech(chan)->get_pvt_uniqueid(chan));
	}

	return snapshot;
}
```

Note that the base segment dereferences `ast_channel_tech(chan)` four times
without a NULL check. That is safe only because `ast_channel_snapshot_create`
refuses to run at all on a tech-less channel — see section 8.2.

The remaining segments are small. Several use the `char x[0]` trailing-buffer
trick rather than string fields, over-allocating the segment and pointing into
the tail — `channel_snapshot_peer_create` (`main/stasis_channels.c:320-338`)
asks `ao2_alloc_options` for `sizeof(*snapshot) + linkedid_len +
peeraccount_len` and lays both strings out by hand:

```c
struct ast_channel_snapshot_peer {
	char *linkedid;   /*!< Linked Channel Identifier -- gets propagated by linkage */
	char account[0]; /*!< Peer account code for billing */
};
```

```c
struct ast_channel_snapshot_hangup {
	int cause;      /*!< Why is the channel hanged up. See causes.h */
	char *source;   /*!< Who is responsible for hanging up this channel */
	int tech_cause; /*!< Technology-specific hangup cause */
	char buffer[0]; /*!< \private Buffer to store source in */
};
```

The full set, with the header line each is defined at:

| Segment | Header | Contents | Invalidation flag |
|---|---|---|---|
| `base` | `stasis_channels.h:105` | name, uniqueid, accountcode, userfield, language, type, tenantid, creationtime, tech_properties, protocol_id | `BASE` |
| `peer` | `stasis_channels.h:124` | linkedid, peer account code | `PEER` |
| `caller` | `stasis_channels.h:79` | caller/dialed/redirecting number, name, subaddress, ANI, RDNIS, `pres` | `CALLER` |
| `connected` | `stasis_channels.h:96` | connected line number and name | `CONNECTED` |
| `bridge` | `stasis_channels.h:57` | bridge unique id, or empty | `BRIDGE` |
| `dialplan` | `stasis_channels.h:65` | appl, data, context, exten, priority | `DIALPLAN` |
| `hangup` | `stasis_channels.h:133` | cause, tech_cause, hangup source | `HANGUP` |

The flags are a bitfield on the channel, not on the snapshot:

```c
enum ast_channel_snapshot_segment_invalidation {
	/*! Invalidate the bridge segment */
	AST_CHANNEL_SNAPSHOT_INVALIDATE_BRIDGE = (1 << 1),
	/*! Invalidate the dialplan segment */
	AST_CHANNEL_SNAPSHOT_INVALIDATE_DIALPLAN = (1 << 2),
	/*! Invalidate the connected segment */
	AST_CHANNEL_SNAPSHOT_INVALIDATE_CONNECTED = (1 << 3),
	/*! Invalidate the caller segment */
	AST_CHANNEL_SNAPSHOT_INVALIDATE_CALLER = (1 << 4),
	/*! Invalidate the hangup segment */
	AST_CHANNEL_SNAPSHOT_INVALIDATE_HANGUP = (1 << 5),
	/*! Invalidate the peer segment */
	AST_CHANNEL_SNAPSHOT_INVALIDATE_PEER = (1 << 6),
	/*! Invalidate the base segment */
	AST_CHANNEL_SNAPSHOT_INVALIDATE_BASE = (1 << 7),
};
```

Bit 0 is unused — the enum starts at `(1 << 1)`, a detail
[chapter 01](01-anatomy.md) also notes. There is no "all" value; the code that
wants one uses `AST_FLAGS_ALL` against the `struct ast_flags` directly.

## 8.2 Creating a snapshot

`ast_channel_snapshot_create` is the only producer. It starts by refusing
tech-less channels outright, which is how dummy channels — the ones
`ast_dummy_channel_alloc` hands out for `pbx_substitute_variables_helper` — stay
out of the event stream entirely:

```c
struct ast_channel_snapshot *ast_channel_snapshot_create(struct ast_channel *chan)
{
	struct ast_channel_snapshot *old_snapshot;
	struct ast_channel_snapshot *snapshot;

	/* no snapshots for dummy channels */
	if (!ast_channel_tech(chan)) {
		return NULL;
	}

	snapshot = ao2_alloc_options(sizeof(*snapshot), channel_snapshot_dtor,
		AO2_ALLOC_OPT_LOCK_NOLOCK);
	if (!snapshot) {
		return NULL;
	}

	old_snapshot = ast_channel_snapshot(chan);
```

Then, once per segment, the same shape: dirty bit set means build a new one,
dirty bit clear means take a reference on the one already in the previous
snapshot.

```c
	 */
	if (ast_test_flag(ast_channel_snapshot_segment_flags(chan), AST_CHANNEL_SNAPSHOT_INVALIDATE_BASE)) {
		/* The base information has changed so update our snapshot */
		snapshot->base = channel_snapshot_base_create(chan);
		if (!snapshot->base) {
			ao2_ref(snapshot, -1);
			return NULL;
		}
	} else {
		snapshot->base = ao2_bump(old_snapshot->base);
```

```c
	if (ast_test_flag(ast_channel_snapshot_segment_flags(chan), AST_CHANNEL_SNAPSHOT_INVALIDATE_PEER)) {
		/* The peer information has changed so update our snapshot */
		snapshot->peer = channel_snapshot_peer_create(chan);
		if (!snapshot->peer) {
			ao2_ref(snapshot, -1);
			return NULL;
		}
	} else {
		snapshot->peer = ao2_bump(old_snapshot->peer);
	}
```

Two segments do not get that treatment. `caller` and `connected` are rebuilt
unconditionally on every single snapshot, and the source says why:

```c
	/* Unfortunately both caller and connected information do not have an enforced contract with
	 * the channel API. This has allowed consumers to directly get the caller or connected structure
	 * and manipulate it. Until such time as there is an enforced contract (which is being tracked under
	 * ASTERISK-28164) they are each regenerated every time a channel snapshot is created.
	 */
	snapshot->caller = channel_snapshot_caller_create(chan);
	if (!snapshot->caller) {
		ao2_ref(snapshot, -1);
		return NULL;
	}

	snapshot->connected = channel_snapshot_connected_create(chan);
	if (!snapshot->connected) {
		ao2_ref(snapshot, -1);
		return NULL;
	}
```

`ast_channel_caller(chan)` returns a pointer to the live `struct
ast_party_caller` inside the channel, so a caller can mutate caller ID without
going through `ast_channel_caller_set` and without ever setting the `CALLER`
dirty bit. Rebuilding every time is the workaround. It has two consequences
worth carrying forward: it is what makes the `memcmp` in section 8.5 dead, and
it means every snapshot publish allocates at least two segments no matter how
little changed.

The tail copies the flat fields:

```c
	snapshot->state = ast_channel_state(chan);
	snapshot->amaflags = ast_channel_amaflags(chan);
	ast_copy_flags(&snapshot->flags, ast_channel_flags(chan), 0xFFFFFFFF);
	ast_set_flag(&snapshot->softhangup_flags, ast_channel_softhangup_internal_flag(chan));

	/* These have to be recreated as they may have changed, unfortunately */
	snapshot->manager_vars = ast_channel_get_manager_vars(chan);
	snapshot->ari_vars = ast_channel_get_ari_vars(chan);

	return snapshot;
```

`ast_copy_flags(..., 0xFFFFFFFF)` copies the whole `AST_FLAG_*` word, which is
how `AST_FLAG_DEAD` and `AST_FLAG_ZOMBIE` reach consumers — AMI's hangup
detection in section 8.6 depends on it. `softhangup_flags` carries the
`AST_SOFTHANGUP_*` bits from [chapter 04](04-state-and-flags.md).

The pairing object a subscriber actually receives is a two-pointer struct:

```c
struct ast_channel_snapshot_update {
	struct ast_channel_snapshot *old_snapshot; /*!< The old channel snapshot */
	struct ast_channel_snapshot *new_snapshot; /*!< The new channel snapshot */
};
```

built by holding a reference on whatever the channel's current snapshot is and
creating a new one:

```c
static struct ast_channel_snapshot_update *channel_snapshot_update_create(struct ast_channel *chan)
{
	struct ast_channel_snapshot_update *update;

	update = ao2_alloc_options(sizeof(*update), channel_snapshot_update_dtor,
		AO2_ALLOC_OPT_LOCK_NOLOCK);
	if (!update) {
		return NULL;
	}

	update->old_snapshot = ao2_bump(ast_channel_snapshot(chan));
	update->new_snapshot = ast_channel_snapshot_create(chan);
	if (!update->new_snapshot) {
		ao2_ref(update, -1);
		return NULL;
	}

	return update;
}
```

The `ao2_bump` at `main/stasis_channels.c:604` is the reason the old snapshot —
and every segment it owns — survives into the subscriber callback. It is also
the reason the `memcmp` cannot work.

## 8.3 Invalidation: who marks what dirty

The dirty bits live on the channel and are set by the accessors in
`main/channel_internal_api.c`. Every segment is marked dirty at birth, inside
`__ast_channel_internal_alloc_with_initializers`
(`main/channel_internal_api.c:1242`):

```c
	AST_VECTOR_INIT(&tmp->ari_report_vars, 8);

	/* Force all channel snapshot segments to be created on first use, so we don't have to check if
	 * an old snapshot exists.
	 */
	ast_set_flag(&tmp->snapshot_segment_flags, AST_FLAGS_ALL);
```

so the first snapshot builds all seven, and the `old_snapshot->base` dereference
on the clean path can never run against a NULL. (`__ast_channel_internal_alloc`
at `main/channel_internal_api.c:1320` is a thin wrapper that delegates to it.)

Two macros generate the string-field setters, and
[chapter 01](01-anatomy.md) covers both, along with the table of which string
fields invalidate what and which of them publish. The one that matters here is
`DEFINE_STRINGFIELD_SETTERS_AND_INVALIDATE_FOR`
(`main/channel_internal_api.c:84`):

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
  \
void ast_channel_##field##_build_va(struct ast_channel *chan, const char *fmt, va_list ap) \
{ \
	ast_string_field_build_va(chan, field, fmt, ap); \
	ast_channel_snapshot_invalidate_segment(chan, invalidate); \
	if (publish && ast_channel_internal_is_finalized(chan)) ast_channel_publish_snapshot(chan); \
} \
```

The guard to carry forward is `ast_channel_internal_is_finalized(chan)`: before
`ast_channel_internal_finalize` runs near the end of `__ast_channel_alloc_ap`, a
setter invalidates but does not publish, so a half-built channel never reaches
the event stream through a field assignment. The macro is invoked five times
(`main/channel_internal_api.c:108-117`) — for `language` and `accountcode` (both
`BASE`, both publishing), `peeraccount` (`PEER`, publishing), `userfield`
(`BASE`, silent) and `hangupsource` (`HANGUP`, silent).

Everything else invalidates by hand. There are twenty calls to
`ast_channel_snapshot_invalidate_segment` in `main/channel_internal_api.c`: the
two inside the macro above, and eighteen written out longhand. Five more exist
outside the accessor layer — `main/channel.c:7404`, `:7417`, `:8493`, `:9304`
and `pbx/pbx_realtime.c:352` — for twenty-five tree-wide. Those five mutate a
field through a path no setter covers, and they are why the function is public
in `include/asterisk/stasis_channels.h:455` rather than static to the accessor
layer.

| Site | Setter | Invalidates |
|---|---|---|
| `:142` | `ast_channel_name_set` | `BASE` |
| `:149` | `ast_channel_name_build_va` | `BASE` |
| `:185` | `ast_channel_tenantid_set` | `BASE` |
| `:195` | `ast_channel_appl_set` | `DIALPLAN` |
| `:212` | `ast_channel_data_set` | `DIALPLAN` |
| `:230` | `ast_channel_context_set` | `DIALPLAN` |
| `:247` | `ast_channel_exten_set` | `DIALPLAN` |
| `:305` | `ast_channel_hangupcause_set` | `HANGUP` |
| `:314` | `ast_channel_tech_hangupcause_set` | `HANGUP` |
| `:323` | `ast_channel_priority_set` | `DIALPLAN` |
| `:445` | `ast_channel_tech_pvt_set` | `BASE`, **only when non-NULL** |
| `:814` | `ast_channel_caller_set` | `CALLER` |
| `:819` | `ast_channel_connected_set` | `CONNECTED` |
| `:824` | `ast_channel_dialed_set` | `CALLER` |
| `:829` | `ast_channel_redirecting_set` | `CALLER` |
| `:850` | `ast_channel_creationtime_set` | `BASE` |
| `:1152` | `ast_channel_internal_bridge_set` | `BRIDGE` |
| `:1351` | `ast_channel_internal_copy_linkedid` | `PEER` |

Note that `ast_channel_name_set` and `ast_channel_tenantid_set` are hand-written
rather than macro-generated, despite setting string fields — chapter 01 explains
why. The practical consequence is that neither publishes.

Publication is a separate axis from invalidation, and the two do not line up.
Five sites in the file call `ast_channel_publish_snapshot` synchronously, inside
the setter: the two in the macro, plus `ast_channel_amaflags_set` (`:288`),
`ast_channel_internal_bridge_set` (`:1153`) and
`ast_channel_internal_copy_linkedid` (`:1352`). `ast_channel_amaflags_set` is
the odd one out in the other direction — it publishes while invalidating
nothing at all, because `amaflags` is one of the flat fields copied in the
snapshot tail rather than a segment.

`ast_channel_tech_pvt_set` is the one asymmetric entry. It invalidates on the
way in and stays silent on the way out:

```c
void ast_channel_tech_pvt_set(struct ast_channel *chan, void *value)
{
	chan->tech_pvt = value;
	if (value != NULL) {
		ast_channel_snapshot_invalidate_segment(chan, AST_CHANNEL_SNAPSHOT_INVALIDATE_BASE);
	}
}
```

The source records no reason for the guard, so treat the rationale as unknown.
The observable effect is the part that matters: clearing `tech_pvt` does not
mark `BASE` dirty, so a channel whose driver has detached keeps whatever
`protocol_id` the last `BASE` rebuild captured. An observer reading
`snapshot->base->protocol_id` is reading the value from when the driver was
still attached, not a cleared one.

`caller`, `dialed` and `redirecting` all invalidate `CALLER` — one segment
covers all three parties:

```c
void ast_channel_caller_set(struct ast_channel *chan, struct ast_party_caller *value)
{
	chan->caller = *value;
	ast_channel_snapshot_invalidate_segment(chan, AST_CHANNEL_SNAPSHOT_INVALIDATE_CALLER);
}
void ast_channel_connected_set(struct ast_channel *chan, struct ast_party_connected_line *value)
{
	chan->connected = *value;
	ast_channel_snapshot_invalidate_segment(chan, AST_CHANNEL_SNAPSHOT_INVALIDATE_CONNECTED);
}
void ast_channel_dialed_set(struct ast_channel *chan, struct ast_party_dialed *value)
{
	chan->dialed = *value;
	ast_channel_snapshot_invalidate_segment(chan, AST_CHANNEL_SNAPSHOT_INVALIDATE_CALLER);
}
void ast_channel_redirecting_set(struct ast_channel *chan, struct ast_party_redirecting *value)
{
	chan->redirecting = *value;
	ast_channel_snapshot_invalidate_segment(chan, AST_CHANNEL_SNAPSHOT_INVALIDATE_CALLER);
}
```

And none of those four setters publishes. Setting caller ID updates nothing
visible until somebody else calls `ast_channel_publish_snapshot` or ends a
staging region.

The segment flags are reachable through one accessor, alongside the snapshot
pointer itself:

```c
struct ast_channel_snapshot *ast_channel_snapshot(const struct ast_channel *chan)
{
	return chan->snapshot;
}

void ast_channel_snapshot_set(struct ast_channel *chan, struct ast_channel_snapshot *snapshot)
{
	ao2_cleanup(chan->snapshot);
	chan->snapshot = ao2_bump(snapshot);
}

struct ast_flags *ast_channel_snapshot_segment_flags(struct ast_channel *chan)
{
	return &chan->snapshot_segment_flags;
}
```

`ast_channel_snapshot_set` is an unref-then-bump, so the channel owns exactly one
reference to its current snapshot and dropping in a new one releases the old.

## 8.4 Staging

Setting a channel's `accountcode` and then its `peeraccount` would otherwise
publish twice — both of those setters come from the publishing macro in section
8.3. Staging coalesces a burst of such changes into one publish:

```c
void ast_channel_stage_snapshot(struct ast_channel *chan)
{
	ast_set_flag(ast_channel_flags(chan), AST_FLAG_SNAPSHOT_STAGE);
}

void ast_channel_stage_snapshot_done(struct ast_channel *chan)
{
	ast_clear_flag(ast_channel_flags(chan), AST_FLAG_SNAPSHOT_STAGE);
	ast_channel_publish_snapshot(chan);
}

void ast_channel_snapshot_invalidate_segment(struct ast_channel *chan,
	enum ast_channel_snapshot_segment_invalidation segment)
{
	ast_set_flag(ast_channel_snapshot_segment_flags(chan), segment);
}
```

The header documents a precondition on all three — and on
`ast_channel_publish_snapshot` itself:

```c
/*!
 * \since 12
 * \brief Publish a \ref ast_channel_snapshot for a channel.
 *
 * \pre chan is locked
 *
 * \param chan Channel to publish.
 */
void ast_channel_publish_snapshot(struct ast_channel *chan);
```

`\pre chan is locked` is a comment. Nothing in any of the four functions asserts
it, and `ast_channel_publish_snapshot` reads a dozen channel fields through
accessors that do no locking of their own. Publishing without the channel lock is
a data race that will not be diagnosed.

Construction itself is one large staging region. `__ast_channel_alloc_ap`
(`main/channel.c:746`) opens it immediately after the allocation succeeds, at
`main/channel.c:775`:

```c
	ast_channel_stage_snapshot(tmp);
```

and closes it 185 lines later, after finalization and after the channel is in the
storage container — so the very first snapshot a channel publishes is already
complete and already findable:

```c

	/* Finalize and link into the channels container. */
	ast_channel_internal_finalize(tmp);
	ast_atomic_fetchadd_int(&chancount, +1);
	CHANNELSTORAGE_API(current_channel_storage_instance, insert, tmp, OBJ_NOLOCK, 0);
	CHANNELSTORAGE_API(current_channel_storage_instance, unlock);


	if (endpoint) {
		ast_channel_endpoint_set(tmp, endpoint);
	}

	/*
	 * And now, since the channel structure is built, and has its name, let
	 * the world know of its existence
	 */
	ast_channel_stage_snapshot_done(tmp);
```

Outside `stasis_channels.c` there are 29 `ast_channel_stage_snapshot(` call sites
across 19 files. The paired examples worth knowing are `main/channel.c:5962/5965`
in `ast_call_forward` (`:5921`) and `main/channel.c:6041/6044` in
`__ast_request_and_dial` (`:5997`) — both wrap an `accountcode` and a
`peeraccount` assignment, which is the two-publish case the mechanism exists for
— plus `main/core_unreal.c:875/928` in `ast_unreal_call_setup` (`:870`), which
stages across call setup rather than construction, and `main/pbx.c:6063/6066`.

The flag itself is a single bit:

```c
	AST_FLAG_SNAPSHOT_STAGE = (1 << 25),
```

Not a counter. Nested staging regions therefore do not compose: an inner
`ast_channel_stage_snapshot_done` clears the bit and publishes, and the outer
region silently continues unstaged, publishing on every subsequent setter. There
are only seven references to the flag in the whole tree, which is what keeps that
from being a routine problem, but it is a real constraint on any new code that
wants to stage around a call into something that might stage too.

Two of those seven references are a hand-rolled workaround. `main/rtp_engine.c`
stages a channel, sets a long run of `RTPAUDIOQOS*` variables, and then — if it
turns out none of them actually changed — clears the staging flag *without*
calling `_done`, specifically to suppress the publish:

```c
	if (chanchanges) {
		ast_channel_stage_snapshot_done(chan);
	} else {
		ast_clear_flag(ast_channel_flags(chan), AST_FLAG_SNAPSHOT_STAGE);
	}
	ast_channel_unlock(chan);
	if (bridge) {
		if (bridgechanges) {
			ast_channel_stage_snapshot_done(bridge);
		} else {
			ast_clear_flag(ast_channel_flags(bridge), AST_FLAG_SNAPSHOT_STAGE);
		}
		ast_channel_unlock(bridge);
		ast_channel_unref(bridge);
	}
```

That is the only code in the tree that does this, and it exists because the
suppression built into `ast_channel_publish_snapshot` does not work — section
8.5 shows why.

## 8.5 Publication, the cache swap, and the dead `memcmp`

`ast_channel_publish_snapshot` opens with the staging check, which is the whole
mechanism behind section 8.4:

```c
void ast_channel_publish_snapshot(struct ast_channel *chan)
{
	struct ast_channel_snapshot_update *update;
	struct stasis_message *message;

	if (!ast_channel_snapshot_type()) {
		return;
	}

	if (ast_test_flag(ast_channel_flags(chan), AST_FLAG_SNAPSHOT_STAGE)) {
		return;
	}

	update = channel_snapshot_update_create(chan);
```

Then the suppression attempt:

```c
	/* If an old snapshot exists and is the same as this newly created one don't bother
	 * raising a message as it hasn't changed.
	 */
	if (update->old_snapshot && !memcmp(update->old_snapshot, update->new_snapshot, sizeof(struct ast_channel_snapshot))) {
		ao2_ref(update, -1);
		return;
	}
```

**This condition can never be true.** `memcmp` over `sizeof(struct
ast_channel_snapshot)` compares, among other things, the seven segment pointers.
`caller` and `connected` were just freshly allocated by
`ast_channel_snapshot_create` (section 8.2), and the old snapshot's `caller` and
`connected` are still alive — `channel_snapshot_update_create` took a reference
on the old snapshot, which owns them. Two live ao2 objects cannot share an
address, so the pointers differ, so the `memcmp` is non-zero, every time. The
`caller`/`connected` pair alone is sufficient; `manager_vars` and `ari_vars` are
also rebuilt unconditionally, but they are usually NULL on both snapshots —
`channel_get_external_vars` (`main/channel.c:7985`) returns NULL when no channel
variables are configured for AMI or ARI — so they are not what carries the
argument.

The consequence is that *every* call to `ast_channel_publish_snapshot` publishes.
The optimisation the comment describes does not exist, and it has not existed
since the segmented snapshot landed in Asterisk 17. If you are measuring Stasis
message volume, this is where it comes from.

The cache update is the one piece of locking the function does by hand:

```c
	/* We lock these ourselves so that the update is atomic and there isn't time where a
	 * snapshot is not in the cache.
	 */
	ao2_wrlock(channel_cache);
	ao2_find(channel_cache, ast_channel_uniqueid(chan), OBJ_SEARCH_KEY | OBJ_UNLINK | OBJ_NODATA | OBJ_NOLOCK);

	ao2_link_flags(channel_cache, update->new_snapshot, OBJ_NOLOCK);
	ao2_unlock(channel_cache);

	/* The same applies here. */
	ao2_wrlock(channel_cache_by_name);
	ao2_find(channel_cache_by_name, ast_channel_name(chan), OBJ_SEARCH_KEY | OBJ_UNLINK | OBJ_NODATA | OBJ_NOLOCK);

	ao2_link_flags(channel_cache_by_name, update->new_snapshot, OBJ_NOLOCK);
```

Taking the container write lock explicitly and passing `OBJ_NOLOCK` to both the
`ao2_find`-with-unlink and the `ao2_link_flags` makes unlink-then-link atomic
from a reader's point of view. Without it there would be a window in which
`ast_channel_cache_by_name()` returns a container with no entry for a channel
that very much exists — and `core show channels` would intermittently omit live
channels. The same dance runs twice, once per cache.

The tail stores the snapshot on the channel, clears every dirty bit, and
publishes:

```c

	ast_channel_snapshot_set(chan, update->new_snapshot);

	/* As this is now the new snapshot any existing invalidated segments have been
	 * created fresh and are up to date.
	 */
	ast_clear_flag(ast_channel_snapshot_segment_flags(chan), AST_FLAGS_ALL);

	ast_assert(ast_channel_topic(chan) != NULL);
	stasis_publish(ast_channel_topic(chan), message);
```

The ordering matters: the caches and `chan->snapshot` are updated *before*
`stasis_publish`, so a subscriber that reacts by calling
`ast_channel_snapshot_get_latest()` sees the snapshot it was just handed, not the
previous one.

The caches and the all-channels topic are created at startup:

```c
	channel_topic_all = stasis_topic_create("channel:all");
	if (!channel_topic_all) {
		return -1;
	}

	channel_cache = ao2_container_alloc_hash(AO2_ALLOC_OPT_LOCK_RWLOCK,
		0, AST_NUM_CHANNEL_BUCKETS, channel_snapshot_uniqueid_hash_cb,
		NULL, channel_snapshot_uniqueid_cmp_cb);
	if (!channel_cache) {
		return -1;
	}

	channel_cache_by_name = ao2_container_alloc_hash(AO2_ALLOC_OPT_LOCK_RWLOCK,
		0, AST_NUM_CHANNEL_BUCKETS, channel_snapshot_hash_cb,
		NULL, channel_snapshot_cmp_cb);
	if (!channel_cache_by_name) {
		return -1;
	}
```

```c
#ifdef LOW_MEMORY
#define AST_NUM_CHANNEL_BUCKETS 61
#else
#define AST_NUM_CHANNEL_BUCKETS 1567
#endif
```

1567 buckets per cache, two caches, is a deliberate memory/lookup trade — the
same bucket count the legacy channel storage container uses. `ast_stasis_channels_init`
then runs 26 `STASIS_MESSAGE_TYPE_INIT` calls, one per channel message type
declared in the file. `ast_channel_snapshot_type` is among them, and it is
declared with no `to_ami` and no `to_json` formatter: AMI and ARI do not render
snapshot updates generically, they each subscribe and decide what event to emit.
Section 8.6 is about that decision.

### Channel blobs, and the second way a snapshot is attached

`ast_channel_snapshot_type` is one of 26 message types declared in
`stasis_channels.c`. Twenty-three of the other 25 — `DTMFBegin`, `Hold`,
`VarSet`, `MusicOnHoldStart`, `HangupRequest` and the rest — are *blobs*: a JSON
payload plus a channel snapshot, wrapped in `struct ast_channel_blob`. The two
that are not are `ast_channel_dial_type`, which carries a
`struct ast_multi_channel_blob` so that it can name both legs of a dial, and
`ast_channel_transfer_request_type` (`:1883`), which carries its own
`struct transfer_message` and is published from `main/refer.c:587`. Two
functions build an ordinary blob, and the difference between them is the whole
theme of this chapter.

```c
struct stasis_message *ast_channel_blob_create_from_cache(const char *channel_id,
					       struct stasis_message_type *type,
					       struct ast_json *blob)
{
	struct ast_channel_snapshot *snapshot;
	struct stasis_message *msg;

	if (!type) {
		return NULL;
	}

	snapshot = ast_channel_snapshot_get_latest(channel_id);
	msg = create_channel_blob_message(snapshot, type, blob);
	ao2_cleanup(snapshot);
	return msg;
}
```

```c
struct stasis_message *ast_channel_blob_create(struct ast_channel *chan,
	struct stasis_message_type *type, struct ast_json *blob)
{
	struct ast_channel_snapshot *snapshot;
	struct stasis_message *msg;

	if (!type) {
		return NULL;
	}

	snapshot = chan ? ao2_bump(ast_channel_snapshot(chan)) : NULL;
	msg = create_channel_blob_message(snapshot, type, blob);
	ao2_cleanup(snapshot);
	return msg;
}
```

`ast_channel_blob_create_from_cache` looks the snapshot up in `channel_cache` by
uniqueid. `ast_channel_blob_create` bumps the snapshot hanging off the channel.
Those are the same object on the ordinary path — `ast_channel_publish_snapshot`
links the new snapshot into both caches *and* stores it on the channel before
publishing — but they diverge in two windows, one wide and one narrow.

The wide one is teardown: after `ast_channel_publish_final_snapshot` has
unlinked the channel from both caches and set `chan->snapshot` to NULL
(section 8.7), the cached lookup returns NULL and so does the channel one, and
`create_channel_blob_message` tolerates a NULL snapshot by simply not attaching
one. A blob published during teardown therefore arrives with no channel
information at all.

The narrow one is inside `ast_channel_publish_snapshot` itself. It links the new
snapshot into `channel_cache` at `main/stasis_channels.c:1124` and into
`channel_cache_by_name` at `:1131`, but only calls
`ast_channel_snapshot_set(chan, update->new_snapshot)` at `:1134`. Between those
points the caches hold snapshot N+1 while the channel still holds N. A thread
that takes the channel lock cannot observe it, because the publisher is required
to hold that lock for the duration (`\pre chan is locked`) — but
`ast_channel_blob_create` imposes no such requirement on *its* caller, so an
unlocked caller can attach the older snapshot while a `_from_cache` sibling
attaches the newer. The window is three statements wide and the consequence is a
stale field in one event, not a crash.

The two also differ in what they accept. `ast_channel_blob_create` takes
`chan == NULL` and produces a snapshotless message; `ast_channel_blob_create_from_cache`
would trip the `ast_assert(!ast_strlen_zero(uniqueid))` inside
`ast_channel_snapshot_get_latest` if handed nothing to look up.

The two public wrappers mirror the split: `ast_channel_publish_cached_blob`
(`main/stasis_channels.c:1146`) uses the cache, `ast_channel_publish_blob`
(`:1161`) uses the channel. Both publish to `ast_channel_topic(chan)`.

One publisher in the file accepts a NULL channel, and it routes through a helper
that redirects to the all-channels topic:

```c
static void publish_message_for_channel_topics(struct stasis_message *message, struct ast_channel *chan)
{
	if (chan) {
		stasis_publish(ast_channel_topic(chan), message);
	} else {
		stasis_publish(ast_channel_topic_all(), message);
	}
}
```

Its one caller is the dial publisher at `main/stasis_channels.c:707`, which
passes `caller ?: peer` — a dial with no originating channel still has to be
announced, and the all-channels topic is where it goes. No other publisher in
the file uses this helper.

The two cache lookups are public API, and they are the supported way for code
outside `stasis_channels.c` to read a snapshot without holding a channel:

```c
struct ast_channel_snapshot *ast_channel_snapshot_get_latest(const char *uniqueid)
{
	ast_assert(!ast_strlen_zero(uniqueid));

	return ao2_find(channel_cache, uniqueid, OBJ_SEARCH_KEY);
}

struct ast_channel_snapshot *ast_channel_snapshot_get_latest_by_name(const char *name)
{
	ast_assert(!ast_strlen_zero(name));

	return ao2_find(channel_cache_by_name, name, OBJ_SEARCH_KEY);
}
```

Both assert on an empty key and both return a reference the caller must drop.
Neither takes a channel lock, which is the point — this is how a subsystem
handed only a uniqueid, such as an ARI request thread, gets at channel state
without joining the locking order in [chapter 03](03-references-and-locking.md).

## 8.6 The consumers

The all-channels topic has more subscribers than this chapter covers: besides
the core four — AMI (`main/manager_channels.c:1444`), CDR (`main/cdr.c:4481`),
CEL (`main/cel.c:1543`) and the statistics module (`res/res_chan_stats.c:146`) —
there are routers in `apps/app_queue.c` (`:7012` and `:12189`),
`apps/app_meetme.c:1044` and `apps/confbridge/confbridge_manager.c:852`. ARI's
subscription at `res/stasis/app.c:152` is a special case: it is per-app and
conditional, `chan ? ast_channel_topic(chan) : ast_channel_topic_all()`, so an
app watching one channel does not see the firehose.

The four this chapter follows — AMI, ARI, CDR and CEL — are the ones that turn
snapshot updates into externally visible events, and they all face the same
problem: they are handed a before-and-after pair and have to work out what
actually changed. None of them can trust the dirty bits — those were cleared
before publication.

### AMI

`main/manager_channels.c` declares a callback type and an array of five of them:

```c
/*! \brief Typedef for callbacks that get called on channel snapshot updates */
typedef struct ast_manager_event_blob *(*channel_snapshot_monitor)(
	struct ast_channel_snapshot *old_snapshot,
	struct ast_channel_snapshot *new_snapshot);
```

```c
channel_snapshot_monitor channel_monitors[] = {
	channel_state_change,
	channel_newexten,
	channel_new_callerid,
	channel_new_accountcode,
	channel_new_connected_line,
};
```

`channel_snapshot_update` (`main/manager_channels.c:780`) runs all five in order
and emits an event for each one that returns a blob, building the shared
channel-state string lazily on the first hit.

The first monitor is the interesting one, because it is three AMI events in one
function and it emits at most one of them:

```c
static struct ast_manager_event_blob *channel_state_change(
	struct ast_channel_snapshot *old_snapshot,
	struct ast_channel_snapshot *new_snapshot)
{
	int is_hungup, was_hungup;

	/* The Newchannel, Newstate and Hangup events are closely related, in
	 * in that they are mutually exclusive, basically different flavors
	 * of a new channel state event.
	 */

	if (!old_snapshot) {
		return ast_manager_event_blob_create(
			EVENT_FLAG_CALL, "Newchannel", NO_EXTRA_FIELDS);
	}

	was_hungup = ast_test_flag(&old_snapshot->flags, AST_FLAG_DEAD) ? 1 : 0;
	is_hungup = ast_test_flag(&new_snapshot->flags, AST_FLAG_DEAD) ? 1 : 0;

	if (!was_hungup && is_hungup) {
		if (new_snapshot->hangup->tech_cause) {
			return ast_manager_event_blob_create(
				EVENT_FLAG_CALL, "Hangup",
				"Cause: %d\r\n"
				"Cause-txt: %s\r\n"
				"TechCause: %d\r\n",
				new_snapshot->hangup->cause,
				ast_cause2str(new_snapshot->hangup->cause),
				new_snapshot->hangup->tech_cause);
		} else {
			return ast_manager_event_blob_create(
				EVENT_FLAG_CALL, "Hangup",
				"Cause: %d\r\n"
				"Cause-txt: %s\r\n",
				new_snapshot->hangup->cause,
				ast_cause2str(new_snapshot->hangup->cause));
		}
	}

	if (old_snapshot->state != new_snapshot->state) {
		return ast_manager_event_blob_create(
			EVENT_FLAG_CALL, "Newstate", NO_EXTRA_FIELDS);
	}

	/* No event */
	return NULL;
```

The `!old_snapshot` test at `main/manager_channels.c:640` is the only Newchannel
trigger in the tree:
AMI's Newchannel is literally "this is the first snapshot I have seen for this
channel." Hangup is detected not from a message type but from the
`AST_FLAG_DEAD` bit copied into the snapshot flags back in section 8.2 — which is
exactly why `ast_channel_destructor` sets that flag *before* publishing the final
snapshot (section 8.7). And `Newstate` fires only if neither of the first two
did, which is why a channel that goes from `Ring` straight to hung up produces a
`Hangup` and no `Newstate`.

The other four monitors are straightforward diffs: `channel_newexten`
(`main/manager_channels.c:677`), `channel_new_callerid` (`:707`),
`channel_new_connected_line` (`:738`) and `channel_new_accountcode` (`:755`).
Three of them lean on the comparison helpers in `stasis_channels.c`, which exist
precisely because the segment pointers are useless for this:

```c
int ast_channel_snapshot_caller_id_equal(
	const struct ast_channel_snapshot *old_snapshot,
	const struct ast_channel_snapshot *new_snapshot)
{
	ast_assert(old_snapshot != NULL);
	ast_assert(new_snapshot != NULL);
	return strcmp(old_snapshot->caller->number, new_snapshot->caller->number) == 0 &&
		strcmp(old_snapshot->caller->name, new_snapshot->caller->name) == 0;
}

```

`ast_channel_snapshot_cep_equal` (`main/stasis_channels.c:1366`),
`..._caller_id_equal` (`:1387`) and `..._connected_line_equal` (`:1397`) have six
consumers between them — `main/manager_channels.c:692`, `:719`, `:747` and
`res/stasis/app.c:479`, `:508`, `:538`. The CEP variant carries a deliberate
asymmetry documented in its own comment: an empty application becoming non-empty
counts as a change even though context, extension and priority are unchanged.

### The `AST_CHAN_TP_INTERNAL` filter

All four event consumers drop updates for channels whose tech is marked
internal — `Announcer`, `Recorder`, the bridge-internal media channels — but
each does it in a different place and by a different mechanism. AMI refuses to
build the channel-state string at all, which makes every monitor's event
unbuildable:

```c
struct ast_str *ast_manager_build_channel_state_string_prefix(
		const struct ast_channel_snapshot *snapshot,
		const char *prefix)
{
	struct ast_str *out;
	char *caller_name;
	char *connected_name;
	int res;

	if (!snapshot || (snapshot->base->tech_properties & AST_CHAN_TP_INTERNAL)) {
		return NULL;
	}
```

CDR filters both halves of the pair:

```c
static int filter_channel_snapshot(struct ast_channel_snapshot *snapshot)
{
	return snapshot->base->tech_properties & AST_CHAN_TP_INTERNAL;
}

/*!
 * \internal
 * \brief Filter a channel snapshot update
 */
static int filter_channel_snapshot_message(struct ast_channel_snapshot *old_snapshot,
		struct ast_channel_snapshot *new_snapshot)
{
	int ret = 0;

	/* Drop cache updates from certain channel technologies */
	if (old_snapshot) {
		ret |= filter_channel_snapshot(old_snapshot);
	}
	if (new_snapshot) {
		ret |= filter_channel_snapshot(new_snapshot);
	}

	return ret;
}
```

and CEL does the same thing with a NULL-tolerant variant:

```c
static int cel_filter_channel_snapshot(struct ast_channel_snapshot *snapshot)
{
	if (!snapshot) {
		return 0;
	}
	return snapshot->base->tech_properties & AST_CHAN_TP_INTERNAL;
}

static void cel_snapshot_update_cb(void *data, struct stasis_subscription *sub,
	struct stasis_message *message)
{
	struct ast_channel_snapshot_update *update = stasis_message_data(message);
	size_t i;

	if (cel_filter_channel_snapshot(update->old_snapshot) || cel_filter_channel_snapshot(update->new_snapshot)) {
		return;
	}
```

ARI filters too, but not with a dedicated filter function — it goes through the
Stasis *sanitizer*, a vtable of three callbacks that every ARI JSON rendering
path consults:

```c
/*! \brief Sanitization callback for channel snapshots */
static int channel_snapshot_sanitizer(const struct ast_channel_snapshot *snapshot)
{
	if (!snapshot || !(snapshot->base->tech_properties & AST_CHAN_TP_INTERNAL)) {
		return 0;
	}
	return 1;
}

/*! \brief Sanitization callback for channels */
static int channel_sanitizer(const struct ast_channel *chan)
{
	if (!chan || !(ast_channel_tech(chan)->properties & AST_CHAN_TP_INTERNAL)) {
		return 0;
	}
	return 1;
}

/*! \brief Sanitization callback for channel unique IDs */
static int channel_id_sanitizer(const char *id)
{
	struct ast_channel_snapshot *snapshot;
	int ret;

	snapshot = ast_channel_snapshot_get_latest(id);
	ret = channel_snapshot_sanitizer(snapshot);
	ao2_cleanup(snapshot);

	return ret;
}

/*! \brief Sanitization callbacks for communication to Stasis applications */
struct stasis_message_sanitizer app_sanitizer = {
	.channel_id = channel_id_sanitizer,
	.channel_snapshot = channel_snapshot_sanitizer,
	.channel = channel_sanitizer,
};
```

`channel_snapshot_sanitizer` returns 1 — meaning "hide this" — for exactly the
same `tech_properties` test the other three use. Every ARI monitor renders its
channel through `ast_channel_snapshot_to_json(new_snapshot,
stasis_app_get_sanitizer())`, which returns NULL when the sanitizer says hide,
and each monitor then bails out. Three of the four call it directly, at
`res/stasis/app.c:483`, `:512` and `:542`, bailing at `:484`, `:513` and `:543`.
The fourth, `channel_state`, renders through the shared
`simple_channel_event` helper, which performs the same call and bails at `:383`
and `:407`. So the event is never sent.

The practical effect: a channel whose `ast_channel_tech` sets
`AST_CHAN_TP_INTERNAL` still allocates snapshots, still publishes them, still
occupies both caches and still shows up in `core show channels` — it is
invisible to all four event consumers but not to the caches or the CLI. If a
channel is in the CLI list but produces no AMI or ARI events, this is the first
thing to check.

### ARI

`res/stasis/app.c` keeps its own parallel `channel_monitors[]`
(`res/stasis/app.c:553`), driven by `sub_channel_update_handler` immediately
below it. It is a different shape from AMI's in two ways. It has four entries
rather than five — `channel_state`, `channel_dialplan`, `channel_callerid`,
`channel_connected_line`; there is no accountcode equivalent — and its callback
type takes a third parameter:

```c
	struct ast_channel_snapshot *old_snapshot,
	struct ast_channel_snapshot *new_snapshot,
	const struct timeval *tv);

static struct ast_json *simple_channel_event(
```

The `const struct timeval *tv` is `stasis_message_timestamp(message)`, which ARI
emits as the event's `timestamp` field. AMI gets its timestamps from the manager
event layer instead, which is why its typedef needs only the two snapshots.

## 8.7 The final snapshot

A channel going away is not signalled by a NULL `new_snapshot`. The header is
explicit:

> This structure will not have a transition of an old snapshot with no new
> snapshot to indicate that a channel has gone away. A new snapshot will always
> exist and a channel going away can be determined by checking for the
> AST_FLAG_DEAD flag on the new snapshot.

So teardown publishes one more, normal-looking snapshot — with `AST_FLAG_DEAD`
set — and removes the channel from both caches in the same breath:

```c
void ast_channel_publish_final_snapshot(struct ast_channel *chan)
{
	struct ast_channel_snapshot_update *update;
	struct stasis_message *message;

	if (!ast_channel_snapshot_type()) {
		return;
	}

	update = channel_snapshot_update_create(chan);
	if (!update) {
		return;
	}

	message = stasis_message_create(ast_channel_snapshot_type(), update);
	/* In the success path message holds a reference to update so it will be valid
	 * for the lifetime of this function until the end.
	 */
	ao2_ref(update, -1);
	if (!message) {
		return;
	}

	ao2_find(channel_cache, ast_channel_uniqueid(chan), OBJ_SEARCH_KEY | OBJ_UNLINK | OBJ_NODATA);
	ao2_find(channel_cache_by_name, ast_channel_name(chan), OBJ_SEARCH_KEY | OBJ_UNLINK | OBJ_NODATA);

	ast_channel_snapshot_set(chan, NULL);

	stasis_publish(ast_channel_topic(chan), message);
	ao2_ref(message, -1);
}
```

Three differences from `ast_channel_publish_snapshot` are worth naming. There is
no staging check — a final snapshot publishes even mid-stage. There is no
`memcmp`. And the two `ao2_find`-with-unlink calls take the container locks
normally rather than wrapping them in an explicit write lock: the channel is
being destroyed, nothing will link it back, and a reader that sees it gone from
one cache and still present in the other is seeing a channel it must not use
anyway.

`ast_channel_snapshot_set(chan, NULL)` at `main/stasis_channels.c:971` drops the
channel's own reference. The snapshot itself lives on inside the message until
the last subscriber is done with it. This is also the moment that makes
`ast_channel_blob_create` and `ast_channel_blob_create_from_cache` disagree, as
section 8.5 described: both now come back empty.

It has exactly one caller, inside the ao2 destructor —
[chapter 02](02-lifecycle.md) walks the whole destructor, this is the snapshot
part of it:

```c
	/* Things that may possibly raise Stasis messages shouldn't occur after this point */
	ast_set_flag(ast_channel_flags(chan), AST_FLAG_DEAD);

	if (ast_channel_internal_is_finalized(chan)) {
		/* A channel snapshot should not be in the process of being staged now. */
		ast_assert(!ast_test_flag(ast_channel_flags(chan), AST_FLAG_SNAPSHOT_STAGE));

		ast_channel_lock(chan);
		ast_channel_publish_final_snapshot(chan);
		ast_channel_unlock(chan);
	}
```

Read that sequence carefully, because it is the contract the whole of section 8.6
depends on:

- `AST_FLAG_DEAD` is set at `main/channel.c:2201`, *before* the publish, so it
  is copied into the final snapshot's flags and AMI's hangup detection fires.
- The `ast_channel_internal_is_finalized` guard at `main/channel.c:2203` means a
  channel that died during construction — one of the seven
  `return ast_channel_unref(tmp)` abort paths in `__ast_channel_alloc_ap`
  (`:793`, `:807`, `:822`, `:830`, `:844`, `:848`, `:942`) — publishes nothing
  at all. It never published a first snapshot, so there is nothing to retract.
- The `ast_assert` at `main/channel.c:2205` checks what section 8.4 cannot: that
  nobody left a staging region open. If they had,
  `ast_channel_publish_final_snapshot` would still publish, since it does not
  test the flag.
- The channel is locked across the publish, satisfying the undocumented
  precondition.

## 8.8 The CLI's two views, and three channel counts

`core show channels` is the clearest illustration of the split. It is
`handle_chanlist` (`main/cli.c:1107-1231`), and it takes three optional
modifiers — `concise`, `verbose` and `count` — which only change formatting:
`verbose` prints twelve columns instead of four, `concise` prints
exclamation-separated fields for machine parsing *and* suppresses the trailing
summary, and `count` suppresses the per-row output but still walks the whole
container. All three read the same source. Its rows come from the snapshot
cache:

```c
	channels = ast_channel_cache_by_name();
```

and the loop over that container — iterator initialised at `main/cli.c:1158`,
`for` at `:1159` — reads nothing but snapshot fields —
`cs->base->name`, `cs->dialplan->context`, `cs->caller->number`,
`cs->bridge->id`. No channel is referenced and no channel lock is taken, which is
why the command cannot deadlock against a wedged channel and why it can show a
channel whose live state has already moved on.

Its trailing count comes from somewhere else entirely:

```c
		numchans = ast_active_channels();
```

So the number of rows printed and the number reported on the next line are read
from two different data structures at two different moments. They normally agree.
When they do not — and a channel that has published its final snapshot but has
not finished running its destructor is exactly such a window — the count is the
one to trust for "does the core still think this channel exists."

There are in fact three counts, and they mean three different things:

```c
/*! \brief returns number of active/allocated channels */
int ast_active_channels(void)
{
	return current_channel_storage_instance ? CHANNELSTORAGE_API(current_channel_storage_instance, active_channels, 1) : 0;
}

int ast_undestroyed_channels(void)
{
	return ast_atomic_fetchadd_int(&chancount, 0);
}
```

| Count | Source | Meaning |
|---|---|---|
| `ast_active_channels()` | `ao2_container_count` of the channel storage container (`channelstorage_ao2_legacy.c:61`) | channels currently findable by `ast_channel_get_by_name` |
| `ast_undestroyed_channels()` | the `chancount` atomic (`channel.c:97`, `+1` at `:947`, `-1` at `:2320`) | channels allocated and not yet through the destructor |
| cache count | `ao2_container_count(ast_channel_cache_by_name())` | channels with a live snapshot |

`ast_undestroyed_channels` has exactly one consumer: the graceful-shutdown wait
at `main/asterisk.c:1954`. The gap between it and `ast_active_channels` is
channels that have been unlinked from storage but whose destructors have not
finished — which during a `core stop gracefully` is precisely the set you are
waiting on.

`core show channel <name>`, by contrast, reads the **live** channel. It takes a
reference, allocates an 8KB accumulation buffer, locks the channel, builds the
entire output into the buffer, unlocks, and only then writes it out with a single
`ast_cli`:

```c
			ast_variables_destroy(metadata);
		}
	}

	ast_channel_unlock(chan);

	ast_cli(a->fd, "%s", ast_str_buffer(output));
	ast_free(output);

	ao2_cleanup(bridge);
```

Building into a buffer under the lock rather than emitting line by line is what
keeps the channel lock hold time bounded by memcpy rather than by socket writes
to a possibly-slow CLI client. The command is the right tool when snapshot and
reality seem to disagree, because it is the only one that shows reality. The one
place it honours the internal-tech rule is skipping the CDR-variables block
(`main/cli.c:1883`) — everything else about an internal channel is printed:

```c

	if (!(ast_channel_tech(chan)->properties & AST_CHAN_TP_INTERNAL)
		&& ast_cdr_serialize_variables(ast_channel_name(chan), &obuf, '=', '\n')) {
		ast_str_append(&output, 0, "  CDR Variables:\n%s\n", ast_str_buffer(obuf));
```

`ast_complete_channels` (`main/cli.c:1950`) is the third reader, and it uses a
third container: `ast_channel_cache_all()`, the by-uniqueid cache rather than the
by-name one. It has 12 call sites across the tree — most commands that take a
channel name, though not all of them; `group show channels`, for instance, has
no completion at all — so tab completion is largely driven by the snapshot
cache, and can therefore offer a name that `ast_channel_get_by_name` will then
fail to find.

The channel commands registered by `main/cli.c` are six, and they split cleanly
along the line this section is about:

```c
static struct ast_cli_entry cli_channels_cli[] = {
	AST_CLI_DEFINE(handle_chanlist, "Display information on channels"),
	AST_CLI_DEFINE(handle_showcalls, "Display information on calls"),
	AST_CLI_DEFINE(handle_showchan, "Display information on a specific channel"),
	AST_CLI_DEFINE(handle_core_set_debug_channel, "Enable/disable debugging on a channel"),
	AST_CLI_DEFINE(group_show_channels, "Display active channels with group(s)"),
	AST_CLI_DEFINE(handle_softhangup, "Request a hangup on a given channel"),
};
```

`handle_chanlist` reads the snapshot cache, as above. `handle_showcalls` reports
only the call counters — `ast_active_calls`, `ast_processed_calls` and uptime —
and touches no channel at all. Three take a reference on a live channel through
`ast_channel_get_by_name`: `handle_showchan` to read it under its lock,
`handle_softhangup` to call `ast_softhangup` on it
([chapter 04](04-state-and-flags.md)), and `handle_core_set_debug_channel` to
flip its frame-counter bits (section 8.9). `group_show_channels` is a fourth
source again — it walks the global group list under
`ast_app_group_list_rdlock` and reads `ast_channel_name(gi->chan)` off the
borrowed channel pointer each entry carries, taking no reference of its own.
Channel groups have no snapshot representation at all.

## 8.9 `core set debug channel`, and why `off` does not work

Frame-level tracing is controlled by the high bit of the channel's `fin` and
`fout` frame counters:

```c
#define	DEBUGCHAN_FLAG  0x80000000

/* XXX not ideal to evaluate x twice... */
#define	FRAMECOUNT_INC(x)	( ((x) & DEBUGCHAN_FLAG) | (((x)+1) & ~DEBUGCHAN_FLAG) )
```

`ast_read` and `ast_write` test it and call `ast_frame_dump`:

```c
	/* High bit prints debugging */
	if (ast_channel_fin(chan) & DEBUGCHAN_FLAG)
		ast_frame_dump(ast_channel_name(chan), f, "<<");
```

at `channel.c:4261` and `:5252` respectively. New channels inherit the bit from
two globals, copied in at `channel.c:864-865`:

```c
unsigned long global_fin, global_fout;
```

The per-channel setter is an `ast_channel_callback` predicate:

```c
static int channel_set_debug(void *obj, void *arg, void *data, int flags)
{
	struct ast_channel *chan = obj;
	struct channel_set_debug_args *args = data;

	ast_channel_lock(chan);

	if (!(ast_channel_fin(chan) & DEBUGCHAN_FLAG) || !(ast_channel_fout(chan) & DEBUGCHAN_FLAG)) {
		if (args->is_off) {
			ast_channel_fin_set(chan, ast_channel_fin(chan) & ~DEBUGCHAN_FLAG);
			ast_channel_fout_set(chan, ast_channel_fout(chan) & ~DEBUGCHAN_FLAG);
		} else {
			ast_channel_fin_set(chan, ast_channel_fin(chan) | DEBUGCHAN_FLAG);
			ast_channel_fout_set(chan, ast_channel_fout(chan) | DEBUGCHAN_FLAG);
		}
		ast_cli(args->fd, "Debugging %s on channel %s\n", args->is_off ? "disabled" : "enabled",
				ast_channel_name(chan));
	}

	ast_channel_unlock(chan);

	return 0;
}
```

**The `off` path is unreachable once debugging is on.** The guard at
`main/cli.c:1602` is true only when at least one of the two flags is *clear*.
Turning debugging on sets both. A subsequent `off` therefore finds both set, the
condition is false, and the whole body is skipped — including the per-channel
`ast_cli` confirmation, so you never see "Debugging disabled on channel …".
Nothing else in the tree clears `DEBUGCHAN_FLAG` on a channel, and
`FRAMECOUNT_INC` deliberately preserves the high bit while incrementing, so the
only way to stop tracing a channel once started is to let the channel go away.

What you do see is misleading. `handle_core_set_debug_channel` ends with an
unconditional line at `main/cli.c:1677`:

```c
	ast_cli(a->fd, "Debugging on new channels is %s\n", args.is_off ? "disabled" : "enabled");
```

That prints for the named-channel form too, where no global was touched. So
`core set debug channel PJSIP/alice-00000001 off` reports "Debugging on new
channels is disabled" and changes neither the channel nor the globals.

`core set debug channel all off` does clear the globals, so *new* channels stop
being traced:

```c
	if (!strcasecmp("all", a->argv[e->args])) {
		if (args.is_off) {
			global_fin &= ~DEBUGCHAN_FLAG;
			global_fout &= ~DEBUGCHAN_FLAG;
		} else {
			global_fin |= DEBUGCHAN_FLAG;
			global_fout |= DEBUGCHAN_FLAG;
		}
		ast_channel_callback(channel_set_debug, NULL, &args, OBJ_NODATA | OBJ_MULTIPLE);
```

but the `ast_channel_callback(channel_set_debug, ...)` at `main/cli.c:1667` runs
into the same dead branch for every existing channel.

## 8.10 `REF_DEBUG` and ao2 leak hunting

When a channel will not go away, the question is who still holds a reference.
`REF_DEBUG` answers it by logging every refcount transition to a file.

Despite the name, it is no longer a compile-time feature. The `REF_DEBUG`
compiler flag now only changes the *default*:

```c
	/* init with buildtime config */
#ifdef REF_DEBUG
	/* The REF_DEBUG compiler flag is now only used to enable refdebug by default.
	 * Support for debugging reference counts is always compiled in. */
	ast_set2_flag64(&ast_options, 1, AST_OPT_FLAG_REF_DEBUG);
#endif
```

The runtime switch is `refdebug` in `asterisk.conf`, parsed at
`main/options.c:326`, setting `AST_OPT_FLAG_REF_DEBUG` (bit 20,
`options.h:81`), tested through the `ast_opt_ref_debug` macro. The logging code
is always compiled in.

The log file is opened once, at startup, with mode `"w"` — so it is truncated on
every restart:

```c
int astobj2_init(void)
{
	char ref_filename[1024];

	if (ast_opt_ref_debug) {
		snprintf(ref_filename, sizeof(ref_filename), "%s/refs", ast_config_AST_LOG_DIR);
		ref_log = fopen(ref_filename, "w");
		if (!ref_log) {
			ast_log(LOG_ERROR, "Could not open ref debug log file: %s\n", ref_filename);
		}
	}

```

It lands at `<astlogdir>/refs`, usually `/var/log/asterisk/refs`.

Before reading one for a channel problem, know the limitation that shapes
everything else: **channel references are untagged.** `ast_channel_ref` and
friends are thin macros over the untagged `ao2_ref`:

```c
#define ast_channel_ref(c) ({ ao2_ref(c, +1); (c); })
```

```c
#define ast_channel_unref(c) ({ ao2_ref(c, -1); (struct ast_channel *) (NULL); })
```

```c
#define ast_channel_cleanup(c) ({ ao2_cleanup(c); (struct ast_channel *) (NULL); })
```

and `ao2_ref` passes `NULL` for the tag:

```c
#define ao2_ref(o,delta)       __ao2_ref((o), (delta), NULL,  __FILE__, __LINE__, __PRETTY_FUNCTION__)
```

`grep ao2_t_ref main/channel.c main/channelstorage_ao2_legacy.c` returns nothing.
The tag column in every channel line of `refs` is therefore empty, and the only
thing identifying a reference is its `file:line:func`. That is usually enough —
but it means you are reading call sites, not intentions, and a leaked reference
taken inside a helper that many callers share will point at the helper, not at
whoever asked for it.

There are five line formats. Construction, carrying the allocation sizes:

```c
	if (ref_log && !(obj->priv_data.options & AO2_ALLOC_OPT_NO_REF_DEBUG)) {
		fprintf(ref_log, "%p,+1,%d,%s,%d,%s,**constructor**%zu**%zu**,%s\n",
			EXTERNAL_OBJ(obj), ast_get_tid(), file, line, func, overhead, data_size, tag ?: "");
		fflush(ref_log);
	}
```

Every ordinary ref and unref, with the resulting count:

```c
		if (ref_log && !(obj->priv_data.options & AO2_ALLOC_OPT_NO_REF_DEBUG)) {
			fprintf(ref_log, "%p,%s%d,%d,%s,%d,%s,%d,%s\n", user_data,
				(delta < 0 ? "" : "+"), delta, ast_get_tid(),
				file, line, func, (int)ret, tag ?: "");
			fflush(ref_log);
		}
```

Two different situations emit the same `**invalid**` marker, which matters
because `refcounter.py` classifies on the marker and cannot tell them apart. The
first is a refcount that has gone negative — an unref past zero, which means
somebody dropped a reference they did not hold (`main/astobj2.c:598-608`):

```c
	/* this case must never happen */
	if (current_value < 0) {
		ast_log(__LOG_ERROR, file, line, func,
			"Invalid refcount %d on ao2 object %p\n", (int)current_value, user_data);
		if (ref_log) {
			/* Log to ref_log even if AO2_ALLOC_OPT_NO_REF_DEBUG */
			fprintf(ref_log, "%p,%d,%d,%s,%d,%s,**invalid**,%s\n",
				user_data, delta, ast_get_tid(), file, line, func, tag ?: "");
			fflush(ref_log);
		}
```

The second fires much earlier in the same function (`main/astobj2.c:511-517`),
when the object pointer resolves to NULL at all — a reference taken on memory
that has already been handed back:

```c
	if (obj == NULL) {
		if (ref_log && user_data) {
			fprintf(ref_log, "%p,%d,%d,%s,%d,%s,**invalid**,%s\n",
				user_data, delta, ast_get_tid(), file, line, func, tag ?: "");
			fflush(ref_log);
		}
		return -1;
	}
```

Either one in a channel's history is a use-after-free, and the `file:line:func`
on the line is where it was noticed, not where the bug is.

A subtlety worth knowing before you trust the script's output: `refcounter.py`
only *classifies* on the marker when the `**invalid**` line is the first record
it has seen for that address (`contrib/scripts/refcounter.py:89-98`). If the
object already has a history — the usual case for a channel, which was
constructed in the same log — the line falls through to the `else` branch at
`:109` and its delta is simply added to the running count. The object is then
reported as `leaked` or `finished` on the arithmetic alone and the marker is
never mentioned. So grep the `refs` file for `**invalid**` yourself; do not wait
for the script to raise it.

And destruction, recording whether the object's lock was held:

```c
	if (ref_log && !(privdataoptions & AO2_ALLOC_OPT_NO_REF_DEBUG)) {
		fprintf(ref_log, "%p,%d,%d,%s,%d,%s,**destructor**lock-state:%s**,%s\n",
			user_data, delta, ast_get_tid(), file, line, func, lock_state, tag ?: "");
		fflush(ref_log);
	}
```

Note that both invalid cases log even when `AO2_ALLOC_OPT_NO_REF_DEBUG` is set —
the negative-refcount one says so in a comment, and the NULL-object one runs
before the options word is reachable at all. The constructor, destructor and
ordinary ref lines honour the flag. It exists mainly to keep container internals
out of the log, and has two users: `main/astobj2_hash.c:212` for a hash bucket
node and `main/astobj2_rbtree.c:899` for an rbtree node.

There is also a tripwire for runaway refs:

```c
#define EXCESSIVE_REF_COUNT		100000

		if (EXCESSIVE_REF_COUNT <= current_value && ret < EXCESSIVE_REF_COUNT) {
			char excessive_ref_buf[100];

			/* We just reached or went over the excessive ref count trigger */
			snprintf(excessive_ref_buf, sizeof(excessive_ref_buf),
				"Excessive refcount %d reached on ao2 object %p",
				(int)current_value, user_data);
			ast_log(__LOG_ERROR, file, line, func, "%s\n", excessive_ref_buf);

			__ast_assert_failed(0, excessive_ref_buf, file, line, func);
```

100,000 refs on one object calls `__ast_assert_failed`, which under `DO_CRASH`
aborts. A slow reference leak on a long-lived channel will eventually hit this
rather than merely consuming memory.

### Reading the log

`contrib/scripts/refcounter.py` splits each line into eight fields — address,
delta, thread id, file, line, function, state, tag — and groups by address,
classifying each object as:

| Class | Meaning |
|---|---|
| finished | constructor through destructor, refcount reached zero cleanly |
| invalid | an `**invalid**` line was the *first* record for the address — either flavour; the script does not distinguish them, and does not apply this class to a marker that arrives mid-history |
| leaked | constructed, never destroyed by end of log |
| skewed | history begins at an unusual refcount — object predates the log |

It defaults to `/var/log/asterisk/refs` and takes `-i`, `-l`, `-n` and `-s` to
suppress each class in turn. For a channel leak, `-i -n -s` leaves only the
leaked objects. Siblings `refstats.py` (peak and total allocation by type) and
`reflocks.py` (lock state at destruction) read the same file.

`astobj2 show stats` and `astobj2 test` are *not* available with `refdebug`
alone — they are compiled only under `AO2_DEBUG`:

```c
#if defined(AO2_DEBUG)
static struct ast_cli_entry cli_astobj2[] = {
	AST_CLI_DEFINE(handle_astobj2_stats, "Print astobj2 statistics"),
	AST_CLI_DEFINE(handle_astobj2_test, "Test astobj2"),
};
#endif /* AO2_DEBUG */
```

## 8.11 Hunting a channel that will not die

This is the one workflow in the chapter that is a procedure rather than a fact,
so it gets its own section; the symptom table below covers everything else.

Start with the counts, because they separate "slow" from "leaked" without
touching a debugger. `core show channels count` gives you
`ast_active_channels()`. `ast_undestroyed_channels()` is not exposed to the CLI,
but if a `core stop gracefully` hangs, that is what it is waiting on
(`main/asterisk.c:1954`). If storage is empty and shutdown still hangs, the
channel is unlinked but its destructor has not finished — look for a blocked
thread with `core show locks`, not for a reference leak. If the channel is still
in storage and `core show channel <name>` still finds it, you have a reference
leak and the counts cannot tell you whose.

For that, turn on `refdebug` in `asterisk.conf`, restart, reproduce, and then:

```sh
cp /var/log/asterisk/refs /tmp/refs.keep
contrib/scripts/refcounter.py -f /tmp/refs.keep -i -n -s
```

`-i -n -s` suppresses the invalid, finished and skewed classes, leaving only
objects that were constructed and never destroyed. Copy the file aside first —
it is reopened `"w"` on every start, so a restart to try something else destroys
your evidence.

What comes back is a list of addresses with their full ref/unref history. To
find the channel among them, get its address: `core show channel <name>` does
not print it, but the `**constructor**` line in `refs` carries the allocation
size, and a channel allocation is distinctive. More practically, work the other
way — grep the leaked histories for `channel.c` or `channelstorage` call sites
and read the unmatched `+1`. Because channel refs are untagged, that `file:line`
is all you get, and if the unmatched ref was taken inside a shared helper you
are looking at the helper, not the caller that forgot to drop it. That is the
point at which the ao2 refcounting internals stop being enough and the next
guide in this series takes over.

## 8.12 Common bugs

| Symptom | Cause |
|---|---|
| A field set on the channel never appears in any AMI/ARI event | The setter does not invalidate a segment — only the twenty sites inside `main/channel_internal_api.c` listed in section 8.3, plus five more elsewhere in the tree (`main/channel.c:7404`, `:7417`, `:8493`, `:9304` and `pbx/pbx_realtime.c:352`), mark anything dirty. The six flat fields on the snapshot (`state`, `amaflags`, `flags`, `softhangup_flags` and the two variable lists) are copied on every rebuild and need no invalidation; a channel field that is neither in a segment nor among those six is not observable at all. |
| A field is invalidated but the event comes much later, attached to an unrelated change | Only five sites publish synchronously (`main/channel_internal_api.c:91`, `:98`, `:288`, `:1153`, `:1352`). Everything else waits for the next publish from any source. |
| Snapshot volume far higher than expected | The `memcmp` suppression at `main/stasis_channels.c:1101-1107` never fires. Every publish call publishes. |
| An inner staging region ends and the outer one stops coalescing | `AST_FLAG_SNAPSHOT_STAGE` (`channel.h:1070`) is one bit, not a depth counter. `_done` clears it unconditionally. |
| Segment contents change under a consumer that cached a segment pointer | Two consecutive snapshots share the segments whose dirty bit was clear. A cached `snapshot->dialplan` can be the *same object* as the next snapshot's — and a cached `snapshot->caller` never is. Cache the snapshot, with a reference, or nothing. |
| Crash dereferencing `old_snapshot` in a subscriber | `old_snapshot` is NULL for a channel's first publish. `new_snapshot` never is, and its segments never are. |
| Hangup never reported to AMI, or reported for the wrong channel | AMI infers hangup from `AST_FLAG_DEAD` in the snapshot flags (`main/manager_channels.c:645-646`), which only arrives because `ast_channel_destructor` sets the flag at `main/channel.c:2201` before publishing at `:2208`. Publishing a final snapshot by any other route loses it. |
| A blob event (`VarSet`, `Hold`, `DTMFBegin`…) arrives with no channel information | Both blob constructors come back empty once `ast_channel_publish_final_snapshot` has unlinked the caches and NULLed `chan->snapshot`, and `create_channel_blob_message` (`main/stasis_channels.c:773`) attaches nothing rather than failing. Section 8.5. |
| `core show channels` lists a channel that `core show channel` says is unknown | The snapshot cache and the channel storage container are separate. The window is between the final snapshot's cache unlink and the destructor's storage unlink. Tab completion uses `ast_channel_cache_all()`, so it can offer a name that cannot be looked up. |
| Row count and "N active channels" disagree in `core show channels` | Rows come from `ast_channel_cache_by_name()` (`main/cli.c:1148`), the count from `ast_active_channels()` (`main/cli.c:1211`). Three counts exist; section 8.8 has the table. |
| `core set debug channel <chan> off` reports success and changes nothing | The guard at `main/cli.c:1602` is false once both `fin` and `fout` carry `DEBUGCHAN_FLAG`, so the whole body — including its per-channel confirmation — is skipped. The "Debugging on new channels is disabled" line at `:1677` prints regardless and is not about that channel. |
| `refs` file empty or missing after a restart | Opened `"w"` at `main/astobj2.c:1175`, so it is truncated at every start; and `refdebug` must be set in `asterisk.conf` — the `REF_DEBUG` build flag only changes the default (`main/options.c:228-233`). |
| `refs` analysis cannot tell two channel references apart | Channel refs are untagged (`include/asterisk/channel.h:3008`, `include/asterisk/astobj2.h:459`); the tag column is always empty. Only `file:line:func` distinguishes them. |
| `refcounter.py` reports an object `invalid` and the history makes no sense | Two unrelated conditions share the `**invalid**` marker: a negative refcount (`main/astobj2.c:598-608`) and a ref on a NULL object (`:511-517`). Check which by reading the log line's delta and the `Invalid refcount` error in the main log. |
| An `**invalid**` line is in the `refs` file but `refcounter.py` never classes the object `invalid` | The class is only assigned when the marker is the first record for that address (`contrib/scripts/refcounter.py:89-98`); mid-history markers fall into the `else` at `:109` and are folded into the delta sum. Grep for `**invalid**` directly. |
| `astobj2 show stats` is an unknown command | It is compiled only under `AO2_DEBUG` (`main/astobj2.c:1150-1155`), which `refdebug` does not enable. |
| A snapshot carries a field value that was never consistent with the others | `ast_channel_publish_snapshot` documents `\pre chan is locked` (`include/asterisk/stasis_channels.h:462`) and asserts nothing, while reading a dozen fields through accessors that lock nothing. A caller that publishes unlocked produces a torn snapshot and no diagnostic. Check the call site holds the lock. |

## 8.13 Rules

| Rule | Why |
|---|---|
| Hold the channel lock across `ast_channel_publish_snapshot`, `ast_channel_stage_snapshot` and `_done` | `\pre chan is locked` on all three (`include/asterisk/stasis_channels.h:426-466`), enforced by nothing. |
| Treat a snapshot and all seven of its segments as immutable | Shared across threads and across snapshots by design (`include/asterisk/stasis_channels.h:143-148`). |
| Never cache a bare segment pointer; take a reference on the whole snapshot | Clean segments are shared with the next snapshot, dirty ones are not. Which is which is not visible to a consumer. |
| Always NULL-check `old_snapshot` in a subscriber; never NULL-check `new_snapshot` or any segment | First publish has no old snapshot; `ast_channel_snapshot_create` fails rather than producing a partial one. |
| Detect channel death from `AST_FLAG_DEAD` on the new snapshot, not from a missing new snapshot | Explicitly documented at `include/asterisk/stasis_channels.h:173-176`. |
| Adding a field to a snapshot segment means updating that segment's creator, its invalidation flag, and every setter that touches the underlying channel field | Three separate edits, in three files, with no compiler help. |
| Pair every `ast_channel_stage_snapshot` with exactly one `_done` on the same channel, and do not nest | One bit, no counter (`include/asterisk/channel.h:1070`). |
| Do not assume an unchanged channel produces no snapshot message | The `memcmp` guard is dead (`main/stasis_channels.c:1101-1107`). `main/rtp_engine.c:2816-2830` is the tree's only working workaround. |
| Mark a tech `AST_CHAN_TP_INTERNAL` to hide its channels from all four event consumers — but not from the caches or the CLI | Filters at `main/manager_channels.c:548`, `main/cdr.c:2176`, `main/cel.c:1039` and, for ARI, the sanitizer at `res/res_stasis.c:2332-2339`. Nothing equivalent in `main/cli.c` except the CDR-variables skip at `:1883`. |
| Use `ast_channel_publish_blob` when you hold the channel and `ast_channel_publish_cached_blob` when you hold only a uniqueid — and expect neither to attach a snapshot after the final one | `chan->snapshot` and both caches are cleared together by `ast_channel_publish_final_snapshot`. |
| Read the live channel with `core show channel`, the cache with `core show channels`, and do not expect them to agree | Different data structures, different moments, no synchronisation between them. |
| Turn on `refdebug` before you need it, and copy `refs` aside before restarting | Truncated `"w"` at every startup (`main/astobj2.c:1175`). |

## Next

That completes the channels guide. [Chapter 01](01-anatomy.md) is the one to
re-read first if any of this felt like it assumed the accessor layer; the
[overview](README.md) has the reading order and the canonical file list.
