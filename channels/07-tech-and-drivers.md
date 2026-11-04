# 7. Channel technologies and drivers

Every `file:line` citation in this guide was verified against the `24` branch.
Newer releases and other branches may differ — line numbers drift — so if you
are reading this against a different tree, treat the citations as approximate
and re-grep for the symbol name.

Every channel has a technology. The technology is a `struct ast_channel_tech` —
a table of 36 members, 32 of them function pointers — and it is the entire
contract between the core and a protocol implementation. `main/channel.c` never
knows it is talking SIP, or IAX2, or a DAHDI span. It knows it has a channel,
that `ast_channel_tech(chan)` gives it a vtable, and that some of the slots in
that vtable may be `NULL`.

```
   dialplan / bridging / ARI / AMI          "I want a channel of type PJSIP"
            │
            ▼
   ast_request("PJSIP", caps, ...)          main/channel.c:6373
            │
            ▼
   request_channel()                        :6238
            │   looks up the backends list ───────────┐
            │                                         │
            ▼                                         ▼
   tech->requester(...)                      ┌──────────────────────────┐
            │  the driver allocs its pvt,    │  static AST_RWLIST_HEAD  │
            │  calls ast_channel_alloc(),    │        backends          │
            │  ast_channel_tech_set(),       │  ┌────────────────────┐  │
            │  ast_channel_tech_pvt_set()    │  │ chanlist → tech    │  │
            ▼                                │  │ chanlist → tech    │  │
   ┌───────────────────┐                     │  │ ...                │  │
   │ struct ast_channel│                     │  └────────────────────┘  │
   │   tech      ──────┼────────────────────▶│  ast_channel_register()  │
   │   tech_pvt  ──────┼──▶ sip_pvt          │        :540              │
   └───────────────────┘                     └──────────────────────────┘
            │
            │  from here on, every core operation is a slot lookup:
            ▼
   ast_call → tech->call      ast_read  → tech->read / read_stream
   ast_hangup → tech->hangup  ast_write → tech->write / write_stream
   ast_indicate → tech->indicate        ...and 25 more
```

Three facts carry the chapter:

1. **Only four slots are universal.** Across the twelve technology tables in
   `channels/`, exactly four callbacks are implemented by all twelve:
   `requester`, `call`, `hangup` and `write`. Five of the 32 are implemented by
   none of the twelve, and one of those five has no implementation anywhere in a
   default build. See [§7.8](#78-what-a-driver-actually-has-to-implement).
2. **An absent slot degrades silently, and inconsistently.** There is no
   validation pass over a tech table at registration time — `ast_channel_register`
   checks the *name* for duplicates and nothing else. Every absent slot is
   discovered at the call site, and the call sites do not agree on what to do:
   some log a warning, some return failure, several return *success*, and
   `request_channel` returns `NULL` with no diagnostic whatsoever. See
   [§7.6](#76-the-rest-of-the-slots-and-what-happens-when-they-are-absent).
3. **The vtable is const member by member, not as a whole,** and the
   qualification is uneven: 29 of the 36 members are `const`, seven are not. 27
   of the 32 callbacks are `* const`, five are plain function pointers, and
   `capabilities` and `properties` are plain mutable fields. `main/message.c:756`
   writes to `properties` on a live, in-use tech table at runtime, through a
   `const` pointer's worth of nothing. See [§7.1](#71-the-vtable).

This chapter assumes [chapter 01](01-anatomy.md) for the accessor layer,
[chapter 02](02-lifecycle.md) for `ast_channel_alloc` and `ast_hangup`, and
[chapter 05](05-io-and-waiting.md) for the fd vector that a driver's `read`
callback is dispatched from.

## 7.1 The vtable

`struct ast_channel_tech` is declared in `include/asterisk/channel.h:648-865`.
Unlike `struct ast_channel`, it is *not* opaque: the full definition is public,
because every channel driver has to fill one in.

It opens with four data members:

```c
struct ast_channel_tech {
	const char * const type;
	const char * const description;

	struct ast_format_cap *capabilities;  /*!< format capabilities this channel can handle */

	int properties;         /*!< Technology Properties */
```

`type` is the string the dialplan uses (`PJSIP/alice` → `type` is `"PJSIP"`) and
the key `ast_channel_register` dedupes on. `description` appears only in CLI
output. `capabilities` is an ao2-refcounted format capability set that the
request path matches the caller's request against; it is the driver's
responsibility to allocate and populate it, usually in `load_module`. `properties`
is a bitfield of four flags, covered below.

The remaining 32 members are callbacks. Grouped by what they do:

| Group | Slots |
|---|---|
| Creating a channel | `requester` `:673`, `requester_with_stream_topology` `:692` |
| Call setup and teardown | `call` `:721`, `answer` `:727`, `answer_with_stream_topology` `:740`, `hangup` `:724` |
| Media in | `read` `:754`, `read_stream` `:767`, `exception` `:785` |
| Media out | `write` `:770`, `write_stream` `:773`, `write_video` `:806`, `write_text` `:809` |
| Signalling | `indicate` `:791`, `send_digit_begin` `:703`, `send_digit_end` `:710`, `transfer` `:803`, `early_bridge` `:788` |
| Text and images | `send_text` `:776`, `send_text_data` `:864`, `send_image` `:779`, `send_html` `:782` |
| State providers | `devicestate` `:695`, `presencestate` `:696` |
| Options | `setoption` `:797`, `queryoption` `:800` |
| Dialplan integration | `func_channel_read` `:815`, `func_channel_write` `:821`, `pre_call` `:861` |
| Housekeeping | `fixup` `:794`, `get_pvt_uniqueid` `:824`, `cc_callback` `:847` |

### The const discipline, and where it stops

Members are `const`-qualified individually rather than by qualifying the struct,
which is unusual enough to be worth reading carefully. 29 of the 36 are
qualified; seven are not. Most of the callbacks are declared as
`(* const name)` — the *pointer* is const, so a slot cannot be reassigned after
the initialiser even through a non-`const` table:

```c
	int (* const devicestate)(const char *device_number);	/*!< Devicestate call back */
	int (* const presencestate)(const char *presence_provider, char **subtype, char **message); /*!< Presencestate callback */
```

That holds for 27 of the 32 callbacks. Five are plain function pointers with no
`const`: `func_channel_read` `:815`, `func_channel_write` `:821`,
`get_pvt_uniqueid` `:824`, `cc_callback` `:847` and `pre_call` `:861`. Those
five are mutable at runtime. There is no reason given in the source for the
difference; they are members added late, and `send_text_data` `:864` — added
later still — *is* `const`, so it is not a clean chronological story either.

The remaining two unqualified members are the data fields `capabilities` and
`properties`. That matters, because `ast_channel_tech()` returns
`const struct ast_channel_tech *` and therefore *looks* immutable at every call
site in the tree, while the underlying tables are not. Of the twelve tables under
`channels/`, **none** is declared `const` — they are all plain
`static struct ast_channel_tech`. Thirteen tables in the tree are `const`, but
all thirteen are either core-internal or test and mock techs; `ast_kill_tech`
(`main/channel.c:435`) is the only one that is both `const` and `extern`.

`main/message.c` exploits the gap. `msg_chan_tech_hack` (`main/message.c:340`)
is a tech table whose `properties` field is modified, at runtime, every time a
message channel is created:

```c
	if (ast_opt_hide_messaging_ami_events) {
		msg_chan_tech_hack.properties |= AST_CHAN_TP_INTERNAL;
	}

	ast_channel_tech_set(chan, &msg_chan_tech_hack);
```

This is a global mutation driven by a config option, performed under no lock, on
a table that other threads may be reading through a `const` pointer at the same
moment. Two things keep it benign: `ast_opt_hide_messaging_ami_events` does not
change after load so the write is idempotent, and this particular table is never
handed to `ast_channel_register` at all — its own doc comment says so
(`main/message.c:333-339`: "This will not be registered as we never want
anything to try to create Message channels other than internally in this file"),
so the only readers are message channels created by that file. It is still a
clear demonstration that the `const` on the accessor's return type is
documentation and not enforcement: every reader in the tree holds that table
through a pointer the compiler believes is immutable.

### `ast_channel_tech_set` and its one assertion

The setter is one of the few accessors in `main/channel_internal_api.c` that
does anything beyond assignment (`:642-643`):

```c
void ast_channel_tech_set(struct ast_channel *chan, const struct ast_channel_tech *value)
{
	if (value->read_stream || value->write_stream) {
		ast_assert(value->read_stream && value->write_stream);
	}

	chan->tech = value;
}
```

This is the only structural validation of a tech table anywhere in the tree, and
it is an `ast_assert` — compiled out entirely outside `AST_DEVMODE`
(`include/asterisk/utils.h:779`). The rule it encodes is that `read_stream` and
`write_stream` come as a pair: a driver that implements one must implement the
other. [§7.5](#75-media-dispatch-and-the-multistream-fork) covers why.

Note what it does *not* check: that `type` is non-`NULL`, that `requester` exists,
that `write` exists for a non-multistream driver, or that `capabilities` has been
allocated. A tech table with every slot `NULL` registers successfully and sets
successfully.

### Properties

```c
/*! \brief ast_channel_tech Properties */
enum {
	/*!
	 * \brief Channels have this property if they can accept input with jitter;
	 * i.e. most VoIP channels
	 */
	AST_CHAN_TP_WANTSJITTER = (1 << 0),
	/*!
	 * \brief Channels have this property if they can create jitter;
	 * i.e. most VoIP channels
	 */
	AST_CHAN_TP_CREATESJITTER = (1 << 1),
	/*!
	 * \brief Channels with this particular technology are an implementation detail of
	 * Asterisk and should generally not be exposed or manipulated by the outside
	 * world
	 */
	AST_CHAN_TP_INTERNAL = (1 << 2),
	/*!
	 * \brief Channels have this property if they implement send_text_data
	 */
	AST_CHAN_TP_SEND_TEXT_DATA = (1 << 3),
};
```

Only one of the four carries real weight.

`AST_CHAN_TP_WANTSJITTER` and `AST_CHAN_TP_CREATESJITTER` are read in exactly one
place in the tree, `main/abstract_jb.c:176-183`, which decides whether to put a
jitterbuffer between two channels. Five drivers set at least one of them:
`chan_unistim.c:741`, `chan_iax2.c:1382`, `chan_pjsip.c:133`, `chan_motif.c:411`
and `addons/chan_ooh323.c:153` — `chan_iax2` sets only `WANTSJITTER`, the other
four set both.

`AST_CHAN_TP_INTERNAL` is the load-bearing one. It means "this channel is an
implementation detail; do not show it to the outside world", and it is honoured
by every external-facing subsystem: AMI (`main/manager_channels.c:548`,
`main/manager_bridges.c:576`), CDR (`main/cdr.c:2176`), CEL (`main/cel.c:1039`),
the CLI channel list (`main/cli.c:1883`) and ARI (`res/res_stasis.c:2335` and
`:2344`). A tech that forgets it leaks internal plumbing into every event
stream and every billing record.

`AST_CHAN_TP_SEND_TEXT_DATA` means "I implement `send_text_data`". It is read at
exactly one site, `main/channel.c:4809`, and set by exactly one driver,
`chan_pjsip`. It is redundant with the slot it advertises —
[§7.6](#76-the-rest-of-the-slots-and-what-happens-when-they-are-absent) shows
the call site checking both the flag *and* the pointer — but a driver that sets
the slot without the flag will find its callback never invoked.

## 7.2 Registration and the `backends` list

A driver publishes itself by calling `ast_channel_register` with a pointer to its
table, normally from `load_module`, and withdraws with `ast_channel_unregister`
from `unload_module`. The registry is a module-static read-write list in
`main/channel.c`:

```c
int ast_channel_register(const struct ast_channel_tech *tech)
{
	struct chanlist *chan;

	AST_RWLIST_WRLOCK(&backends);

	AST_RWLIST_TRAVERSE(&backends, chan, list) {
		if (!strcasecmp(tech->type, chan->tech->type)) {
			ast_log(LOG_WARNING, "Already have a handler for type '%s'\n", tech->type);
			AST_RWLIST_UNLOCK(&backends);
			return -1;
		}
	}

	if (!(chan = ast_calloc(1, sizeof(*chan)))) {
		AST_RWLIST_UNLOCK(&backends);
		return -1;
	}
	chan->tech = tech;
	AST_RWLIST_INSERT_HEAD(&backends, chan, list);

	ast_debug(5, "Registered handler for '%s' (%s)\n", chan->tech->type, chan->tech->description);

	ast_verb(5, "Registered channel type '%s' (%s)\n", chan->tech->type, chan->tech->description);

	AST_RWLIST_UNLOCK(&backends);

	return 0;
}
```

Three things to notice. The duplicate check is `strcasecmp`, so `type` is
case-insensitive and `"pjsip"` cannot coexist with `"PJSIP"`. The wrapper
struct is a bare `struct chanlist` holding only the tech pointer — the registry
does not copy the table, so the table must outlive its registration, which is why
every one of them is `static` storage in the module. And insertion is at the
head, so the list order is reverse registration order; `core show channeltypes`
output is therefore ordered by module load order, backwards.

Unregistration removes and frees the wrapper under the write lock:

```c
void ast_channel_unregister(const struct ast_channel_tech *tech)
{
	struct chanlist *chan;

	ast_debug(5, "Unregistering channel type '%s'\n", tech->type);

	AST_RWLIST_WRLOCK(&backends);

	AST_RWLIST_TRAVERSE_SAFE_BEGIN(&backends, chan, list) {
		if (chan->tech == tech) {
			AST_LIST_REMOVE_CURRENT(list);
			ast_free(chan);
			ast_verb(5, "Unregistered channel type '%s'\n", tech->type);
			break;
		}
	}
	AST_LIST_TRAVERSE_SAFE_END;

	AST_RWLIST_UNLOCK(&backends);
}

```

Lookup by name is `ast_get_channel_tech`:

```c
const struct ast_channel_tech *ast_get_channel_tech(const char *name)
{
	struct chanlist *chanls;
	const struct ast_channel_tech *ret = NULL;

	AST_RWLIST_RDLOCK(&backends);

	AST_RWLIST_TRAVERSE(&backends, chanls, list) {
		if (!strcasecmp(name, chanls->tech->type)) {
			ret = chanls->tech;
			break;
		}
	}

	AST_RWLIST_UNLOCK(&backends);

	return ret;
}

/*! \brief Gives the string form of a given hangup cause */
const char *ast_cause2str(int cause)
```

Note that it returns the tech pointer *after* dropping the lock. The pointer is
into the module's static storage, so it stays valid as long as the module stays
loaded — and nothing in the signature or the call sites holds a module reference.
`request_channel` has a sharper version of the same problem; see
[§7.3](#73-requesting-a-channel).

### The CLI surface

Two CLI commands read the list, registered together at `main/channel.c:391-394`.
`core show channeltypes` (`:262`) prints a table with four capability columns:

```c

	AST_RWLIST_RDLOCK(&backends);
	AST_RWLIST_TRAVERSE(&backends, cl, list) {
		ast_cli(a->fd, FORMAT, cl->tech->type, cl->tech->description,
			(cl->tech->devicestate) ? "yes" : "no",
			(cl->tech->presencestate) ? "yes" : "no",
			(cl->tech->indicate) ? "yes" : "no",
			(cl->tech->transfer) ? "yes" : "no");
		count_chan++;
	}
	AST_RWLIST_UNLOCK(&backends);

```

`core show channeltype <type>` (`:327`) prints ten lines, nine of which are slot
probes plus one for `capabilities`. Both commands are built the same way —
`(cl->tech->slot) ? "yes" : "no"` — so the CLI is, in effect, a partial dump of
which vtable slots a driver filled in. Between them they cover nine of the 32
callbacks: `devicestate`, `presencestate`, `indicate`, `transfer`,
`send_digit_begin`, `send_digit_end`, `send_html`, `send_image` and `send_text`.
There is no CLI view of `read`/`write`, `fixup`, the stream callbacks or the
properties bitfield.

The lookup in `core show channeltype` is a prefix match in the wrong direction:

```c
		if (!strncasecmp(cl->tech->type, a->argv[3], strlen(cl->tech->type)))
			break;
	}
```

`strlen(cl->tech->type)` is the length of the *registered* name, not of the
user's argument, so any argument that begins with a registered type matches it.
`core show channeltype PJSIPnonsense` prints the PJSIP entry. Harmless in a
diagnostic command, but it is the opposite of what the code reads as.

## 7.3 Requesting a channel

Everything that creates an outbound channel goes through one static function.
The two public entry points are thin wrappers:

```c
struct ast_channel *ast_request(const char *type, struct ast_format_cap *request_cap, const struct ast_assigned_ids *assignedids, const struct ast_channel *requestor, const char *addr, int *cause)
{
	return request_channel(type, request_cap, NULL, assignedids, requestor, addr, cause);
}

struct ast_channel *ast_request_with_stream_topology(const char *type, struct ast_stream_topology *topology, const struct ast_assigned_ids *assignedids, const struct ast_channel *requestor, const char *addr, int *cause)
{
	return request_channel(type, NULL, topology, assignedids, requestor, addr, cause);
}
```

`request_channel` opens by looking up the type:

```c
static struct ast_channel *request_channel(const char *type, struct ast_format_cap *request_cap, struct ast_stream_topology *topology,
	const struct ast_assigned_ids *assignedids, const struct ast_channel *requestor, const char *addr, int *cause)
{
	struct chanlist *chan;
	struct ast_channel *c = NULL;
	int res;
	int foo;

	if (!cause)
		cause = &foo;
	*cause = AST_CAUSE_NOTDEFINED;

	if (AST_RWLIST_RDLOCK(&backends)) {
		ast_log(LOG_WARNING, "Unable to lock technology backend list\n");
		return NULL;
	}

	AST_RWLIST_TRAVERSE(&backends, chan, list) {
		if (strcasecmp(type, chan->tech->type)) {
			continue;
		}

		break;
	}

	AST_RWLIST_UNLOCK(&backends);

	if (!chan) {
		ast_log(LOG_WARNING, "No channel type registered for '%s'\n", type);
		*cause = AST_CAUSE_NOSUCHDRIVER;
		return NULL;
	}
```

**The list is unlocked at `:6263`, and `chan->tech` is dereferenced from `:6272`
onwards.** `ast_channel_unregister` `ast_free`s that `chanlist` entry under the
write lock. So between `:6263` and the
`requester_with_stream_topology` call at `:6280`, or the `requester` call at
`:6339`, there is a window in which a module unload can free the node this
function is about to read. The window is small and unloading a channel driver
with calls in flight is already discouraged, but it is real, and it was
introduced deliberately: `2b22c3c84b` ("channel: Add ability to request an
outgoing channel with stream topology", April 2017) hoisted the unlock to
`:6263` and deleted the two in-body unlocks that had been there before.

The hoist left the lock scope hard to read, and a later author misread it. The
function still contains a *second* unlock, on an error path, releasing a lock the
function no longer holds:

```c

		/* find the best audio format to use */
		tmp_cap = ast_format_cap_alloc(AST_FORMAT_CAP_FLAG_DEFAULT);
		if (!tmp_cap) {
			AST_RWLIST_UNLOCK(&backends);
			return NULL;
		}
```

That unlock at `:6298` is not residue — `2b22c3c84b` removed every in-body
unlock. `git blame` attributes `:6298` to `6d980de282` ("channel: Set up calls
without audio (text+video), again", January 2021), which introduced the whole
`if (!tmp_cap)` block almost four years later. The author wrote a correct-looking
error path for a lock the function had stopped holding in 2017. Reading the two
commits in order is the clearest evidence available that the lock scope here is
genuinely confusing, which is the real reason the use-after-free window above is
worth taking seriously. (The same error path also leaks `tmp_converted_cap` when
a topology-to-caps conversion has happened.)

### Negotiating caps against topology

Once the tech is found, the function reconciles the two ways a caller can
describe what it wants. A modern multistream driver gets a
`struct ast_stream_topology`; a legacy driver gets a `struct ast_format_cap`.
`request_channel` accepts either from its caller and converts:

```c
	/* Allow either format capabilities or stream topology to be provided and adapt */
	if (chan->tech->requester_with_stream_topology) {
		struct ast_stream_topology *tmp_converted_topology = NULL;

		if (!topology && request_cap) {
			/* Turn the requested capabilities into a stream topology */
			topology = tmp_converted_topology = ast_stream_topology_create_from_format_cap(request_cap);
		}

		c = chan->tech->requester_with_stream_topology(type, topology, assignedids, requestor, addr, cause);

		ast_stream_topology_free(tmp_converted_topology);
	} else if (chan->tech->requester) {
		struct ast_format_cap *tmp_converted_cap = NULL;
		struct ast_format_cap *tmp_cap;
		RAII_VAR(struct ast_format *, tmp_fmt, NULL, ao2_cleanup);
		RAII_VAR(struct ast_format *, best_audio_fmt, NULL, ao2_cleanup);
		struct ast_format_cap *joint_cap;

		if (!request_cap && topology) {
			/* Turn the request stream topology into capabilities */
			request_cap = tmp_converted_cap = ast_stream_topology_get_formats(topology);
		}
```

If the driver is legacy, the caps branch additionally runs
`ast_translator_best_choice` against `chan->tech->capabilities` and fails the
request with `AST_CAUSE_BEARERCAPABILITY_NOTAVAIL` if no audio format can be
bridged — but only if the request contains audio at all. The check is inside
`if (!ast_format_cap_empty(tmp_cap))` (`:6303`), which the source annotates
"Avoid this check for calls without audio, like text+video calls", so a
text-and-video request skips format negotiation entirely. The topology branch
does no such check in either case: a multistream driver is trusted to negotiate
for itself.

There is no final `else`. A tech table with neither `requester` nor
`requester_with_stream_topology` falls through both branches with `c` still
`NULL` and `*cause` still at its initial `AST_CAUSE_NOTDEFINED`, and returns:

```c

	if (!c) {
		return NULL;
	}
```

No warning, no error, no cause code that distinguishes this from any other
failure. It is the quietest way to misconfigure a driver in the subsystem.

### What the driver must do inside `requester`

The contract is not written down anywhere; it is whatever the tail of
`request_channel` (`:6343-6371`) assumes. On success the driver must have called
`ast_channel_alloc` (which locks the new channel and registers it in channel
storage — see [chapter 02](02-lifecycle.md)), `ast_channel_tech_set` with its own
table, and `ast_channel_tech_pvt_set` with its private state. `request_channel`
then copies the requestor's call ID onto the new channel and applies security
requirements:

```c

	if (requestor) {
		ast_callid callid;

		ast_channel_lock_both(c, (struct ast_channel *) requestor);

		/* Set the newly created channel's callid to the same as the requestor. */
		callid = ast_channel_callid(requestor);
		if (callid) {
			ast_channel_callid_set(c, callid);
		}

		ast_channel_unlock(c);
		ast_channel_unlock((struct ast_channel *) requestor);
	}

	if (set_security_requirements(requestor, c)) {
		ast_log(LOG_WARNING, "Setting security requirements failed\n");
		ast_hangup(c);
		*cause = AST_CAUSE_BEARERCAPABILITY_NOTAVAIL;
		return NULL;
	}

	/* no need to generate a Newchannel event here; it is done in the channel_alloc call */
	return c;
```

Note `ast_hangup(c)` on the security-requirements failure path: once `requester`
returns a channel, the core owns it, and a late failure is reported as a
`NULL` return *after* the driver's `hangup` callback has already run.

## 7.4 Calling and answering

`ast_call` is the thinnest wrapper over a slot in the file:

```c
int ast_call(struct ast_channel *chan, const char *addr, int timeout)
{
	/* Place an outgoing call, but don't wait any longer than timeout ms before returning.
	   If the remote end does not answer within the timeout, then do NOT hang up, but
	   return anyway.  */
	int res = -1;
	/* Stop if we're a zombie or need a soft hangup */
	ast_channel_lock(chan);
	if (!ast_test_flag(ast_channel_flags(chan), AST_FLAG_ZOMBIE) && !ast_check_hangup(chan)) {
		if (ast_channel_tech(chan)->call)
			res = ast_channel_tech(chan)->call(chan, addr, timeout);
		ast_set_flag(ast_channel_flags(chan), AST_FLAG_OUTGOING);
	}
	ast_channel_unlock(chan);
	return res;
}
```

Three observations. The channel is locked across the callback, which is what the
slot's doxygen promises ("The channel is locked when called") and which means
`tech->call` must not block for long and must not take any lock that orders above
the channel lock — see [chapter 03](03-references-and-locking.md). `res` stays at
its initialiser `-1` when the driver has no `.call`, so a missing slot is
reported as a failed call rather than as a programming error. And
`AST_FLAG_OUTGOING` is set unconditionally inside the `if`, including when
`tech->call` failed or was absent — which matters, because the answer path keys
off that flag.

`ast_pre_call` (`:6463`) is the one slot with a genuine fallback: a driver that
does not implement it gets the dialplan subroutine executed by the core instead.

```c
int ast_pre_call(struct ast_channel *chan, const char *sub_args)
{
	int (*pre_call)(struct ast_channel *chan, const char *sub_args);

	ast_channel_lock(chan);
	pre_call = ast_channel_tech(chan)->pre_call;
	if (pre_call) {
		int res;

		res = pre_call(chan, sub_args);
		ast_channel_unlock(chan);
		return res;
	}
	ast_channel_unlock(chan);
	return ast_app_exec_sub(NULL, chan, sub_args, 0);
}
```

The callback is read out into a local *before* dispatch, because `pre_call` is
one of the five non-`const` slots and the fallback must not be taken on a torn
read. That the core bothers is a small piece of evidence that the missing `const`
on those five is understood to be a hazard.

### The answer path

`ast_raw_answer_with_stream_topology` is where a tech's answer callback is
dispatched, and it is the clearest example in the subsystem of a missing slot
being treated as success:

```c
int ast_raw_answer_with_stream_topology(struct ast_channel *chan, struct ast_stream_topology *topology)
{
	int res = 0;
	SCOPE_TRACE(1, "%s\n", ast_channel_name(chan));

	ast_channel_lock(chan);

	/* You can't answer an outbound call */
	if (ast_test_flag(ast_channel_flags(chan), AST_FLAG_OUTGOING)) {
		ast_channel_unlock(chan);
		return 0;
	}

	/* Stop if we're a zombie or need a soft hangup */
	if (ast_test_flag(ast_channel_flags(chan), AST_FLAG_ZOMBIE) || ast_check_hangup(chan)) {
		ast_channel_unlock(chan);
		return -1;
	}

	/*
	 * Mark when incoming channel answered so we can know how
	 * long the channel has been up.
	 */
	set_channel_answer_time(chan);

	ast_channel_unlock(chan);

	switch (ast_channel_state(chan)) {
	case AST_STATE_RINGING:
	case AST_STATE_RING:
		ast_channel_lock(chan);
		if (ast_channel_tech(chan)->answer_with_stream_topology) {
			res = ast_channel_tech(chan)->answer_with_stream_topology(chan, topology);

		} else if (ast_channel_tech(chan)->answer) {
			res = ast_channel_tech(chan)->answer(chan);
		}
		ast_setstate(chan, AST_STATE_UP);
		ast_channel_unlock(chan);
		break;
	case AST_STATE_UP:
		/* Fall through */
	default:
		ast_debug(2, "Skipping answer, since channel state on %s is %s\n", ast_channel_name(chan), ast_state2str(ast_channel_state(chan)));
		break;
	}

	ast_indicate(chan, -1);

	return res;
}
```

If neither `answer_with_stream_topology` nor `answer` is present, `res` stays `0`
and `ast_setstate(chan, AST_STATE_UP)` still runs. The channel is answered as
far as the core, the dialplan, CDR and every Stasis subscriber are concerned,
with no protocol-level answer having been sent. For `chan_rtp`'s media channels,
which have no `answer` and are never in `AST_STATE_RINGING`, that is correct
behaviour. For a driver that merely forgot the slot, it is a call that the far
end never sees answered.

Two more details worth knowing. The early return for `AST_FLAG_OUTGOING` is
unconditional and returns `0` — you cannot answer a channel you dialled, and the
attempt is not an error. And the `switch` on `ast_channel_state(chan)` happens
*after* `ast_channel_unlock(chan)` at `:2663`, so the state that selects the
branch is read without the lock and re-confirmed by nothing; the lock is
re-taken inside the `RINGING`/`RING` arm before the callback.

`ast_raw_answer` is the single-stream wrapper (`:2690`). `__ast_answer` (`:2695`)
wraps *that* with a media-drain step — it calls `ast_waitfor` for
`MAX(delay, 500)` ms and queues the frames it reads onto a local list so that
media arriving during answer is not lost — and `ast_answer` (`:2805`) is
`__ast_answer(chan, 0)`.

## 7.5 Media dispatch and the multistream fork

Every tech has two possible media interfaces, and which one it implements changes
the core's behaviour rather than just the destination of the call.

| | single-stream | multistream |
|---|---|---|
| read | `read` | `read_stream` |
| write | `write` | `write_stream` |
| frame carries | one implicit stream per media type | `f->stream_num` |
| topology | synthesised by the core from formats | owned by the driver |

The `read` slot's own doxygen says a multistream driver "should not implement
this callback", and `ast_channel_tech_set`'s assertion
([§7.1](#71-the-vtable)) enforces the pairing. Two techs in the tree take the
multistream path, and both omit `read` entirely: `chan_pjsip_tech`
(`channels/chan_pjsip.c:110`) and `local_tech` (`main/core_local.c:185`). Of the
twelve drivers under `channels/`, `chan_pjsip` is the only one.

The read dispatch is in `__ast_read`:

```c
	} else {
		if (ast_test_flag(ast_channel_flags(chan), AST_FLAG_EXCEPTION)) {
			if (ast_channel_tech(chan)->exception)
				f = ast_channel_tech(chan)->exception(chan);
			else {
				ast_log(LOG_WARNING, "Exception flag set on '%s', but no exception handler\n", ast_channel_name(chan));
				f = &ast_null_frame;
			}
			/* Clear the exception flag */
			ast_clear_flag(ast_channel_flags(chan), AST_FLAG_EXCEPTION);
		} else if (ast_channel_tech(chan) && ast_channel_tech(chan)->read_stream) {
			f = ast_channel_tech(chan)->read_stream(chan);

			/* This channel driver supports multistream so the stream_num on the frame is valid, the only
			 * thing different is that we need to find the default stream so we know whether to invoke the
			 * default stream logic or not (such as transcoding).
			 */
			if (f && (f->frametype == AST_FRAME_VOICE || f->frametype == AST_FRAME_VIDEO)) {
				stream = ast_stream_topology_get_stream(ast_channel_get_stream_topology(chan), f->stream_num);
				default_stream = ast_channel_get_default_stream(chan, ast_format_get_type(f->subclass.format));
			}
		} else if (ast_channel_tech(chan) && ast_channel_tech(chan)->read) {
			f = ast_channel_tech(chan)->read(chan);

			/* Since this channel driver does not support multistream determine the default stream this frame
			 * originated from and update the frame to include it.
			 */
			if (f && (f->frametype == AST_FRAME_VOICE || f->frametype == AST_FRAME_VIDEO)) {
				stream = default_stream = ast_channel_get_default_stream(chan, ast_format_get_type(f->subclass.format));
				if (!stream) {
					ast_frfree(f);
					f = &ast_null_frame;
				} else {
					f->stream_num = ast_stream_get_position(stream);
				}
			}
		}
		else
			ast_log(LOG_WARNING, "No read routine on channel %s\n", ast_channel_name(chan));
```

Read that in order. The `exception` arm comes first, selected by
`AST_FLAG_EXCEPTION` rather than by which slots exist, and it is the only one of
the three with a diagnostic when its slot is absent (`:3746`). Then
`read_stream`, whose frames are trusted to carry a valid `stream_num`. Then
`read`, whose frames do not, so the core looks up the default stream for the
frame's media type and stamps `stream_num` itself — and drops the frame if the
driver handed up a format with no negotiated stream. Then the `else` at `:3778`,
whose `LOG_WARNING` at `:3779` fires whenever the exception flag is clear and
*both* `read_stream` and `read` are `NULL`. Whether the tech has an `exception`
slot has nothing to do with reaching it.

The write side funnels through a two-line helper:

```c
static int tech_write(struct ast_channel *chan, struct ast_stream *stream,
			struct ast_stream *default_stream, struct ast_frame *frame)
{
	if (ast_channel_tech(chan)->write_stream) {
		return stream ? ast_channel_tech(chan)->write_stream(
			chan, ast_stream_get_position(stream), frame) : 0;
	}

	return ((stream == default_stream) && ast_channel_tech(chan)->write) ?
		ast_channel_tech(chan)->write(chan, frame) : 0;
}
```

`tech_write` returns `0` — success — when there is no slot to call, and also when
a single-stream driver is handed a frame for a stream that is not its default.
That second case is deliberate: it is how a non-multistream driver silently
discards media for streams it cannot represent.

`ast_write_stream`'s own `switch` bypasses the helper for several frame types,
and one arm does not null-check at all:

```c
	default:
		/* At this point, fr is the incoming frame and f is NULL.  Channels do
		 * not expect to get NULL as a frame pointer and will segfault.  Hence,
		 * we output the original frame passed in. */
		CHECK_BLOCKING(chan);
		res = ast_channel_tech(chan)->write(chan, fr);
		ast_clear_flag(ast_channel_flags(chan), AST_FLAG_BLOCKING);
		break;
	}
```

The `default:` arm calls `tech->write` unconditionally. Every other dispatch
site in the file guards. All twelve drivers under `channels/` implement `write`,
so no real call leg reaches it — but `write` is not only a driver's slot. Several
techs the core registers itself have no `write` at all, `surrogate_tech`
(`main/channel.c:697`, registered at `:8192`) among them, and so do
`res/res_calendar.c:752`, `res/res_cliexec.c:42` and `main/pbx_variables.c:988`.
Reaching this arm needs a frame type that falls past every named `case`, so it is
not an everyday path, but it is the one slot whose absence is a segfault rather
than a silent no-op, and the assertion in `ast_channel_tech_set` does not cover
it.

## 7.6 The rest of the slots, and what happens when they are absent

The pattern established above holds throughout: the absence of a slot is handled
at the call site, and the call sites disagree. This is the catalogue for the
slots the core dispatches directly — all of them from `main/channel.c` except
`send_image`:

| Slot | Dispatched at | Absent → |
|---|---|---|
| `requester_with_stream_topology` | `:6280` | falls back to `requester` |
| `requester` | `:6339` | `NULL` return, `AST_CAUSE_NOTDEFINED`, **no log** (`:6343`) |
| `call` | `:6490` | `-1`, but `AST_FLAG_OUTGOING` still set |
| `answer_with_stream_topology` | `:2670` | falls back to `answer` |
| `answer` | `:2673` | **`0` — success**; state forced to `AST_STATE_UP` |
| `hangup` | `:2610` | skipped silently |
| `read` | `:3763` | `LOG_WARNING "No read routine on channel %s"` (`:3779`) |
| `read_stream` | `:3752` | falls back to `read` |
| `exception` | `:3744` | `LOG_WARNING` (`:3746`), `&ast_null_frame` |
| `write` | `:5164` | **`0` — success**; except `:5487`, which segfaults |
| `write_stream` | `:5159` | falls back to `write` for the default stream only |
| `write_video` | `:5312` | `ast_write_video` returns `0` before dispatching |
| `write_text` (from `ast_sendtext_data`) | `:4804` | falls through to `send_text_data`, then `send_text` |
| `write_text` (from `ast_write_stream`) | `:5289` | **`0` — success**, no fall-through |
| `send_text_data` | `:4816` | falls through to `send_text` |
| `send_text` (from `ast_sendtext_data`) | `:4821` | `ast_debug(1, ...)` and `-1` |
| `send_text` (from `ast_write_stream`) | `:5292` | **`0` — success**, no diagnostic |
| `send_html` | `:6650` | `-1`; `ast_channel_supports_html` returns `0` |
| `send_image` | `main/image.c:165` | `-1`; `ast_supports_images` returns `0` |
| `indicate` | `:4548` | `-1` → core falls back to its own tone generation |
| `send_digit_begin` | `:4928` | `0`, before any channel state is touched |
| `send_digit_end` | `:4952` | `0`, but core tone generation is stopped |
| `transfer` | `:6529` | `0`, meaning "not supported" rather than "failed" |
| `early_bridge` | `:7454` | `-1` |
| `fixup` | `:7239`, `:7245` | skipped silently (see [chapter 06](06-masquerade.md)) |
| `setoption`, `queryoption` | `:7472`, `:7492` | `errno = ENOSYS`, `-1` |
| `pre_call` | `:6472` | core runs `ast_app_exec_sub` instead |

The two `write_text` rows and the two `send_text` rows are the clearest single
illustration of the thesis: the same slot, absent, means `-1` with a debug line
when the text arrived through `ast_sendtext_data` and `0` with no diagnostic when
it arrived as a frame through `ast_write_stream`. Nothing in the slot's
declaration hints at either.

A few of these are worth looking at directly, because the shape of the fallback
*is* the subsystem's design.

`indicate` is the only slot with a full core-side substitute. A driver
that cannot signal ringing in-protocol gets ringback generated as audio:

```c
	if (ast_channel_tech(chan)->indicate) {
		/* See if the channel driver can handle this condition. */
		res = ast_channel_tech(chan)->indicate(chan, _condition, data, datalen);
	} else {
		res = -1;
	}

	if (!res) {
		/* The channel driver successfully handled this indication */
		res = 0;
		return res;
	}

	/* The channel driver does not support this indication, let's fake
	 * it by doing our own tone generation if applicable. */

```

Note that the fallback triggers on *any* non-zero return, so a driver that
returns `-1` from `indicate` for a condition it does not recognise gets the
core's tone generator for free. That is the documented way to implement partial
indication support.

The text path is a four-way cascade in one `if`/`else if` chain, and it is where
`AST_CHAN_TP_SEND_TEXT_DATA` is read:

```c
	if (ast_channel_tech(chan)->write_text
		&& (ast_strlen_zero(content_type) || ast_begins_with(content_type, "text/"))
		&& (ast_format_cap_has_type(ast_channel_nativeformats(chan), AST_MEDIA_TYPE_TEXT))) {
		struct ast_frame f;
		/* T.140 payload does not include the null byte terminator */
		size_t body_len = strlen(body);

		/* Process as T.140 text (moved here from ast_sendtext() */
		memset(&f, 0, sizeof(f));
		f.src = "DIALPLAN";
		f.subclass.format = ast_format_t140;
		f.frametype = AST_FRAME_TEXT;
		f.datalen = body_len;
		f.mallocd = AST_MALLOCD_DATA;
		f.data.ptr = ast_strdup(body);
		if (f.data.ptr) {
			res = ast_channel_tech(chan)->write_text(chan, &f);
		} else {
			res = -1;
		}
		ast_frfree(&f);
	} else if ((ast_channel_tech(chan)->properties & AST_CHAN_TP_SEND_TEXT_DATA)
		&& ast_channel_tech(chan)->send_text_data) {
		/* Send enhanced message to a channel driver that supports it */
		ast_debug(1, "Sending TEXT_DATA from '%s' to %s:%s %s\n",
			ast_msg_data_get_attribute(msg, AST_MSG_DATA_ATTR_FROM),
			ast_msg_data_get_attribute(msg, AST_MSG_DATA_ATTR_TO),
			ast_channel_name(chan), body);
		res = ast_channel_tech(chan)->send_text_data(chan, msg);
	} else if (ast_channel_tech(chan)->send_text
		&& (ast_strlen_zero(content_type) || ast_begins_with(content_type, "text/"))) {
		/* Send the body of an enhanced message to a channel driver that supports only a char str */
		ast_debug(1, "Sending TEXT to %s: %s\n", ast_channel_name(chan), body);
		res = ast_channel_tech(chan)->send_text(chan, body);
	} else {
		ast_debug(1, "Channel technology does not support sending content type '%s' on channel '%s'\n",
			S_OR(content_type, "text/plain"), ast_channel_name(chan));
		res = -1;
```

The flag and the pointer are both tested at `:4809-4810`. A driver that sets
`send_text_data` but omits the property flag silently falls through to the
`send_text` arm and loses every message attribute except the body.

The DTMF pair looks at first like another inconsistency and is not, but reading
it correctly requires the whole of both functions. `ast_senddigit_begin` treats
an absent slot as success and returns before touching any channel state:

```c
	if (!ast_channel_tech(chan)->send_digit_begin)
		return 0;

	ast_channel_lock(chan);
	ast_channel_sending_dtmf_digit_set(chan, digit);
	ast_channel_sending_dtmf_tv_set(chan, ast_tvnow());
	ast_channel_unlock(chan);

	if (!ast_channel_tech(chan)->send_digit_begin(chan, digit))
		return 0;
```

`ast_senddigit_end` initialises `res` to `-1`, but that value never reaches the
caller:

```c
int ast_senddigit_end(struct ast_channel *chan, char digit, unsigned int duration)
{
	int res = -1;

	if (ast_channel_tech(chan)->send_digit_end)
		res = ast_channel_tech(chan)->send_digit_end(chan, digit, duration);

	ast_channel_lock(chan);
	if (ast_channel_sending_dtmf_digit(chan) == digit) {
		ast_channel_sending_dtmf_digit_set(chan, 0);
	}
	ast_channel_unlock(chan);

	if (res && ast_channel_generator(chan))
		ast_playtones_stop(chan);

	return 0;
}
```

The function returns `0` unconditionally. `res` is a local flag whose only
consumer is the `ast_playtones_stop` call: a non-zero `res` means "the driver
did not signal the end of this digit itself", which is exactly the case in which
the core's own tone generator is the thing producing the digit and must be
stopped. That is the same convention `ast_senddigit_begin` uses — a non-zero
return from `send_digit_begin` is what makes it fall through to
`ast_playtones_start` rather than return early. The pair is coherent; what is
easy to misread is the `int res = -1` sitting at the top of a function that
cannot return it.

Both slots' doxygen notes "The channel is not locked when this function gets
called", in contrast to `call`, `setoption`, `queryoption` and `pre_call`, whose
notes promise the lock is held. Those notes are the only locking contract a
driver author gets, and they live in the header rather than in any document.

`ast_channel_early_bridge` is a special case: it requires not merely that both
channels have the slot, but that it is the *same function pointer*, which in
practice means the same driver:

```c
/*! \brief Bridge two channels together (early) */
int ast_channel_early_bridge(struct ast_channel *c0, struct ast_channel *c1)
{
	/* Make sure we can early bridge, if not error out */
	if (!ast_channel_tech(c0)->early_bridge || (c1 && (!ast_channel_tech(c1)->early_bridge || ast_channel_tech(c0)->early_bridge != ast_channel_tech(c1)->early_bridge)))
		return -1;

	return ast_channel_tech(c0)->early_bridge(c0, c1);
}
```

Exactly one tech in the tree fills the slot in — `addons/chan_ooh323.c:167`,
`.early_bridge = ast_rtp_instance_early_bridge` — and `addons/` is not built by
default. In a stock build, therefore, no registered tech has the slot and this
function returns `-1` unconditionally; even in a build with `chan_ooh323`
enabled, the same-pointer requirement means it can only ever succeed between two
ooh323 channels.

## 7.7 Hangup and `tech_pvt` ownership

`tech_pvt` is a bare `void *` in the private struct, set and read through
`ast_channel_tech_pvt_set` and `ast_channel_tech_pvt`. The core never
dereferences it, never copies it, and never frees it. The setter's only side
effect is snapshot invalidation:

```c
void *ast_channel_tech_pvt(const struct ast_channel *chan)
{
	return chan->tech_pvt;
}
void ast_channel_tech_pvt_set(struct ast_channel *chan, void *value)
{
	chan->tech_pvt = value;
	if (value != NULL) {
		ast_channel_snapshot_invalidate_segment(chan, AST_CHANNEL_SNAPSHOT_INVALIDATE_BASE);
	}
}
```

(Note the asymmetry: setting a non-`NULL` pvt invalidates the base snapshot
segment, setting `NULL` does not. See [chapter 01](01-anatomy.md) for why a
setter invalidates at all.)

The driver owns that pointer for the channel's whole life, and the one place the
core gives it a chance to release it is `tech->hangup`, dispatched from
`ast_hangup`:

```c
	if (ast_test_flag(ast_channel_flags(chan), AST_FLAG_BLOCKING)) {
		ast_log(LOG_WARNING, "Hard hangup called by thread LWP %d on %s, while blocked by thread LWP %d in procedure %s!  Expect a failure\n",
			ast_get_tid(), ast_channel_name(chan), ast_channel_blocker_tid(chan),
			ast_channel_blockproc(chan));
		ast_assert(0);
	}

	if (ast_channel_tech(chan)->hangup) {
		ast_channel_tech(chan)->hangup(chan);
	}

	ast_channel_unlock(chan);

	ast_cc_offer(chan);
```

Two constraints follow. The callback runs with the channel locked, so it must
not block. And it is the *last* driver code to run on that channel: the ao2
destructor that follows does not call into the tech at all. A driver that does
not clear `tech_pvt` in `hangup` leaves a dangling pointer in a channel object
that may still be referenced — which is why `kill_hangup`
([§7.9](#79-the-cores-own-technologies)) exists and does nothing else:

```c
static int kill_hangup(struct ast_channel *chan)
{
	ast_channel_tech_pvt_set(chan, NULL);
	return 0;
}
```

`tech->hangup` is also the callback a masquerade can reach unexpectedly, because
after `channel_do_masquerade` the zombie carries the *other* driver's tech and
pvt. [Chapter 06](06-masquerade.md) covers the `fixup` protocol that makes that
survivable.

## 7.8 What a driver actually has to implement

There are 44 `struct ast_channel_tech` initialisers in the tree:

| Location | Count | What they are |
|---|---|---|
| `tests/` | 16 | unit-test mocks |
| `channels/` | 12 | the real channel drivers |
| `main/` | 7 | `local`, the three core-internal techs, and three mocks and hacks |
| `res/` | 4 | snoop, calendar, CLI exec, a parking test |
| `apps/` | 3 | voicemail, and ConfBridge's announce and record channels |
| `addons/` | 2 | `chan_mobile`, `chan_ooh323` — not built by default |

The twelve under `channels/` are the ones that answer the question. This is which
of the 32 callbacks each fills in:

| callback | audiosocket | announce | record | console | dahdi | iax2 | motif | pjsip | mcast&nbsp;rtp | ucast&nbsp;rtp | unistim | websocket |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| `requester` | x | x | x | x | x | x | x | x | x | x | x | x |
| `requester_with_stream_topology` | | | | | | | | x | | | | |
| `devicestate` | | | | | x | x | | x | | | | |
| `presencestate` | | | | | | | | | | | | |
| `send_digit_begin` | | x | x | x | x | x | x | x | | | x | |
| `send_digit_end` | x | x | x | x | x | x | x | x | | | x | x |
| `call` | x | x | x | x | x | x | x | x | x | x | x | x |
| `hangup` | x | x | x | x | x | x | x | x | x | x | x | x |
| `answer` | | | | x | x | x | x | x | | | x | |
| `answer_with_stream_topology` | | | | | | | | | | | | |
| `read` | x | x | x | x | x | x | x | | x | x | x | x |
| `read_stream` | | | | | | | | x | | | | |
| `write` | x | x | x | x | x | x | x | x | x | x | x | x |
| `write_stream` | | | | | | | | x | | | | |
| `send_text` | | x | x | x | x | x | x | x | | | x | |
| `send_image` | | | | | | x | | | | | | |
| `send_html` | | x | x | | | x | | | | | | |
| `exception` | | x | x | | x | | x | x | | | | |
| `early_bridge` | | | | | | | | | | | | |
| `indicate` | | x | x | x | x | x | x | x | | | x | |
| `fixup` | | x | x | x | x | x | x | x | | | x | |
| `setoption` | | x | x | | x | x | | | | | | |
| `queryoption` | | x | x | | x | x | | x | | | | |
| `transfer` | | | | | | x | | x | | | | |
| `write_video` | | x | x | | | x | x | | | | | |
| `write_text` | | | | | | | | | | | | |
| `func_channel_read` | | | | | x | x | | x | | | | |
| `func_channel_write` | | | | | x | | | | | | | |
| `get_pvt_uniqueid` | | | | | | | | x | | | | |
| `cc_callback` | | | | | x | | | | | | | |
| `pre_call` | | | | | | | | | | | | |
| `send_text_data` | | | | | | | | x | | | | |
| **total** | **6** | **15** | **15** | **11** | **18** | **19** | **13** | **20** | **5** | **5** | **11** | **6** |

Reading that table across:

- **Four callbacks are universal:** `requester`, `call`, `hangup`, `write`. Those
  are the floor.
- **Five are implemented by none of the twelve:** `presencestate`,
  `answer_with_stream_topology`, `early_bridge`, `write_text` and `pre_call`.
  Two of them do have implementations elsewhere in the tree: `presencestate` at
  `main/presencestate.c:415` (a test tech) and `early_bridge` at
  `addons/chan_ooh323.c:167` (not built by default). The core call sites for all
  five run normally — `ast_pre_call`, `ast_channel_early_bridge`,
  `ast_raw_answer_with_stream_topology` and the rest are reached on ordinary
  calls — it is only the callbacks behind them that are never entered, so each
  site takes its fallback or its "no slot" branch every single time.
- **Five are implemented by `chan_pjsip` alone among the twelve:**
  `requester_with_stream_topology`, `read_stream`, `write_stream`,
  `get_pvt_uniqueid` and `send_text_data`. `local_tech` (`main/core_local.c:185`)
  is the only other multistream tech in the tree, so the multistream half of the
  vtable has two consumers in total and only one of them is a channel driver.
- **Three have a single implementation each:** `cc_callback` and
  `func_channel_write` in `chan_dahdi`, `send_image` in `chan_iax2`.

### The minimal driver

`channels/chan_rtp.c` is the answer to "what is the smallest usable tech table".
Both of its techs fill five slots — the four universal ones plus `read`:

```c
/* Multicast channel driver declaration */
static struct ast_channel_tech multicast_rtp_tech = {
	.type = "MulticastRTP",
	.description = "Multicast RTP Paging Channel Driver",
	.requester = multicast_rtp_request,
	.call = rtp_call,
	.hangup = rtp_hangup,
	.read = rtp_read,
	.write = rtp_write,
};

/* Unicast channel driver declaration */
static struct ast_channel_tech unicast_rtp_tech = {
	.type = "UnicastRTP",
	.description = "Unicast RTP Media Channel Driver",
	.requester = unicast_rtp_request,
	.call = rtp_call,
	.hangup = rtp_hangup,
	.read = rtp_read,
	.write = rtp_write,
```

That is enough to allocate a channel, place it, carry audio both ways and tear it
down. Everything else in the vtable is optional in the literal sense that the
core has a path for its absence.

What the two `chan_rtp` techs give up by stopping at five slots is worth
spelling out, because it is the practical content of [§7.6](#76-the-rest-of-the-slots-and-what-happens-when-they-are-absent):
no `answer` means `ast_raw_answer` reports success without telling the far end;
no `indicate` means every ringing, progress and hold indication becomes
core-generated tone or nothing; no `fixup` means the channel cannot safely be
masqueraded if its pvt caches the owner pointer; no `queryoption` means
`ast_channel_queryoption` fails with `ENOSYS` for every option.

## 7.9 The core's own technologies

Four tech tables in `main/channel.c` and `main/message.c` exist to make the
channel abstraction work on things that are not calls.

`ast_kill_tech` is a tech whose entire job is to make a channel hang up
immediately. Its five callbacks all refuse to do anything:

```c
/*!
 * \brief Kill the channel channel driver technology descriptor.
 *
 * \details
 * The purpose of this channel technology is to encourage the
 * channel to hangup as quickly as possible.
 *
 * \note Used by DTMF atxfer and zombie channels.
 */
const struct ast_channel_tech ast_kill_tech = {
	.type = "Kill",
	.description = "Kill channel (should not see this)",
	.read = kill_read,
	.exception = kill_exception,
	.write = kill_write,
	.fixup = kill_fixup,
	.hangup = kill_hangup,
};
```

It is the only `const` tech table that is also `extern` — declared at
`include/asterisk/channel.h:868` so that other translation units can install it.
Except that **no other translation unit does.** A tree-wide grep for
`ast_kill_tech` finds exactly two hits: the definition above and that `extern`
declaration. The doxygen note "Used by DTMF atxfer and zombie channels" is
stale — the atxfer code that used it was removed, and the zombie path in
`channel_do_masquerade` swaps techs between the two channels rather than
installing this one. It is dead code with a comment that documents a dead caller.

`surrogate_tech` is the opposite: no callbacks at all, one property.

```c
/*! \brief Channel technology used to extract a channel from a running application. The
 * channel created with this technology will be immediately hung up - most external
 * applications won't ever want to see this.
 */
static const struct ast_channel_tech surrogate_tech = {
	.type = "Surrogate",
	.description = "Surrogate channel used to pull channel from an application",
	.properties = AST_CHAN_TP_INTERNAL,
};
```

It exists so that an external interface can get a `struct ast_channel *` handle
on something running inside an application, and the `AST_CHAN_TP_INTERNAL`
property is what keeps it out of AMI, ARI, CDR and CEL. It still appears in
`core show channels` and in both snapshot caches —
[chapter 08](08-snapshots-and-debugging.md) has the four filters and what each
one actually suppresses. It is registered at `main/channel.c:8192` and
unregistered at `:8125`.

`null_tech` is a placeholder used once, at `main/channel.c:927`, for the dummy
channels that `ast_dummy_channel_alloc` hands out:

```c
static const struct ast_channel_tech null_tech = {
	.type = "NULL",
	.description = "Null channel (should not see this)",
};
```

And `msg_chan_tech_hack` (`main/message.c:340`) is the message-queue channel
discussed in [§7.1](#71-the-vtable) — the table that is mutated at runtime, and
the one of the four that is deliberately never registered at all.

The lesson these four carry is that `struct ast_channel_tech` is not only a
driver interface. It is also the mechanism by which the core fabricates channels
that have no protocol behind them at all; a table of zero callbacks is a
legitimate thing to install, and installing one does not require registering the
tech under a name.

## 7.10 Common bugs

| Symptom | Cause |
|---|---|
| `Dial()` fails with no log line and cause 0 | the tech has neither `requester` nor `requester_with_stream_topology`; `request_channel` returns `NULL` silently (`main/channel.c:6343-6345`) |
| Call reports answered, far end hears nothing | no `answer` slot — `ast_raw_answer_with_stream_topology` returns `0` and forces `AST_STATE_UP` anyway (`:2669-2676`) |
| `No read routine on channel ...` | neither `read` nor `read_stream`; the only slot absence the read path logs (`:3779`) |
| `Exception flag set on '...', but no exception handler` | a driver set an exception fd (see [chapter 05](05-io-and-waiting.md)) without an `exception` slot (`:3746`) |
| Media silently vanishes on one stream | a single-stream driver handed a frame for a non-default stream; `tech_write` returns `0` without writing (`:5163`) |
| `ast_assert` fires in `ast_channel_tech_set` on a dev build | `read_stream` and `write_stream` not implemented as a pair (`main/channel_internal_api.c:642-643`) |
| Enhanced message attributes lost, body delivered | `send_text_data` implemented without `AST_CHAN_TP_SEND_TEXT_DATA`; the cascade falls through to `send_text` (`:4809-4821`) |
| Internal channels appearing in AMI/CDR/ARI | tech missing `AST_CHAN_TP_INTERNAL` |
| Crash in the driver's `read` after a masquerade | pvt caches the owner channel and the tech has no `fixup`; see [chapter 06](06-masquerade.md) |
| Dangling `tech_pvt` after hangup | `hangup` callback did not `ast_channel_tech_pvt_set(chan, NULL)`; nothing else will |
| Deadlock inside `tech->call` or `tech->setoption` | those slots are dispatched with the channel lock held, so any lock the callback takes is acquired *after* the channel lock; taking one that elsewhere in the tree is held *before* the channel lock inverts the order |
| Use-after-free unloading a channel driver under load | `request_channel` drops the `backends` read lock at `:6263` and dereferences the freed `chanlist` node afterwards |

## 7.11 Rules

| Rule | Why |
|---|---|
| Fill in `requester`, `call`, `hangup` and `write` | they are the only four slots all twelve drivers under `channels/` implement; without `requester` no channel can be created at all, and the other three have no core substitute that does anything useful |
| Allocate `capabilities` before registering | `request_channel` runs `ast_translator_best_choice` against it for every legacy request that contains audio (`main/channel.c:6303`) |
| Implement `read_stream` and `write_stream` together or neither | `ast_channel_tech_set` asserts it; mixing them is undefined in the read path |
| Set `AST_CHAN_TP_INTERNAL` on any tech that is not a real call leg | it is the single flag that keeps a channel out of AMI, ARI, CDR and CEL — though not out of the snapshot caches or `core show channels` |
| Clear `tech_pvt` in `hangup` | `hangup` is the last driver code to run; the destructor does not call the tech |
| Implement `fixup` if the pvt caches the owner channel pointer | masquerade moves the pvt to a different channel object |
| Keep `tech->call`, `tech->setoption`, `tech->queryoption` and `tech->pre_call` short and lock-free | all four are dispatched with the channel lock held |
| Do not assume `tech->send_digit_begin`/`send_digit_end` run locked | their doxygen says the opposite, and `ast_senddigit_begin` dispatches outside the lock |
| Return non-zero from `indicate` for conditions you do not handle | that is what triggers the core's tone-generation fallback |
| Never write to a tech table once any channel is using it | `ast_channel_tech()` hands every reader a `const` pointer, so no reader expects it to change; `main/message.c:756` gets away with it only because that table is never registered and its write is idempotent |
| Keep the table in `static` storage | the registry stores the pointer, not a copy |
| Treat the slots' doxygen locking notes as the contract | there is no other statement of it anywhere in the tree |

## Next

[Chapter 08](08-snapshots-and-debugging.md) covers channel snapshots and the
Stasis coupling, the
`core show channel` family, `REF_DEBUG` and ao2 leak hunting, and the debugging
recipes for the whole subsystem.
