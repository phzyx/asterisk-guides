# 03 — Memory, lifecycle and ownership

**Target tree:** Asterisk, the `24` branch. Line numbers and quoted code may
differ on newer releases and on other branches; re-grep for the symbol name if
you are reading against a different tree.

This is the file to read twice. Almost every frame-related crash in Asterisk is an
ownership error: a double free, a use-after-free, or a leak caused by
misunderstanding which function takes ownership of what.

## The three ownership bits

`include/asterisk/frame.h:232-237`:

```c
/*! Need the header be free'd? */
#define AST_MALLOCD_HDR		(1 << 0)
/*! Need the data be free'd? */
#define AST_MALLOCD_DATA	(1 << 1)
/*! Need the source be free'd? (haha!) */
#define AST_MALLOCD_SRC		(1 << 2)
```

They are independent. A frame can own its payload but not its header (a static
header pointing at a malloc'd buffer), or its header but not its payload (a
heap header pointing into a driver's receive buffer).

Four combinations you will actually meet:

| `mallocd` | Meaning | Typical origin |
|---|---|---|
| `0` | Owns nothing. Free is a no-op. | Stack frame; driver's per-instance static frame |
| `AST_MALLOCD_HDR` | Owns one allocation covering header + payload + src | `ast_frdup()` output |
| `HDR\|DATA` | Header and payload are separate allocations | Hand-built frames; translator output |
| `HDR\|DATA\|SRC` | All three separate | `ast_frisolate()` output |

### The `AST_MALLOCD_HDR`-only convention of `frdup`

`__ast_frdup()` makes **one** allocation covering everything and then marks only
the header bit. The comment explaining why is worth quoting in full
(`main/frame.c:364-371`):

```c
	/* Even though this new frame was allocated from the heap, we can't mark it
	 * with AST_MALLOCD_HDR, AST_MALLOCD_DATA and AST_MALLOCD_SRC, because that
	 * would cause ast_frfree() to attempt to individually free each of those
	 * under the assumption that they were separately allocated. Since this frame
	 * was allocated in a single allocation, we'll only mark it as if the header
	 * was heap-allocated; this will result in the entire frame being properly freed.
	 */
	out->mallocd = AST_MALLOCD_HDR;
```

**Consequence:** you cannot look at a frame with `mallocd == AST_MALLOCD_HDR` and
conclude its payload is borrowed. It may be a `frdup` output whose payload lives
inside the same block. Never try to steal `data.ptr` out of such a frame and free
it independently.

## Freeing

### `ast_frfree(fr)` — the one you call

A macro over `ast_frame_free(fr, 1)` — the `1` means "allow caching". Use it
everywhere. The only reason to call `ast_frame_free(fr, 0)` directly is when you
know the header must actually go back to the allocator, which the error paths in
`__ast_frisolate()` do (`main/frame.c:256`, `:285`).

### `ast_frame_free(frame, cache)` frees **lists**

`main/frame.c:176-185`:

```c
void ast_frame_free(struct ast_frame *frame, int cache)
{
	struct ast_frame *next;

	while (frame) {
		next = AST_LIST_NEXT(frame, frame_list);
		__frame_free(frame, cache);
		frame = next;
	}
}
```

So `ast_frfree(f)` on a frame that is the head of a chain destroys the **whole
chain**. This is intentional and convenient for translator output, and lethal if
`f` is still linked into a queue you care about. Before freeing one frame out of a
list, null its link:

<!-- illustrative -->
```c
	AST_LIST_NEXT(f, frame_list) = NULL;
	ast_frfree(f);
```

which is exactly what `channel.c` does in several places, e.g.
`main/channel.c:3806` and `:4228`.

### `__frame_free()` step by step

`main/frame.c:122-173`:

1. `if (!fr->mallocd) return;` — borrowed frames are untouched. This is why
   freeing a stack frame or `&ast_null_frame` does not crash.
2. If `mallocd == AST_MALLOCD_HDR` exactly, and `cache` is set, and
   `ast_opt_cache_media_frames` is on, the header goes onto the per-thread cache
   instead of being freed (see below). The format/topology ao2 reference is
   released first.
3. `AST_MALLOCD_DATA` → `ast_free(fr->data.ptr - fr->offset)`. **Note the
   `- fr->offset`.** If you mutated `offset` after allocating, you have corrupted
   the heap.
4. `AST_MALLOCD_SRC` → `ast_free((void *) fr->src)`.
5. `AST_MALLOCD_HDR` → release the format/topology reference, then `ast_free(fr)`.
   **If the header bit is not set, `mallocd` is merely cleared to 0** — the header
   survives, now owning nothing:
   ```c
   	} else {
   		fr->mallocd = 0;
   	}
   ```
   — `main/frame.c:170-172`. This is how a static header can be reused after
   its payload is freed.

### `ast_frame_dtor(f)`

`main/frame.c:187-190` — a plain `ast_frfree()` wrapper with an
`ao2_cleanup`-compatible signature, so frames can be used with
`RAII_VAR` and ao2 destructors:

```c
void ast_frame_dtor(struct ast_frame *f)
{
	ast_frfree(f);
}
```

## Duplicating: `ast_frdup()`

`ast_frdup(f)` → `__ast_frdup(f, __FILE__, __LINE__, __PRETTY_FUNCTION__)`.

`main/frame.c:305-394`. What it does:

- Computes one allocation size (`main/frame.c:316-325`):
  ```c
  	len = sizeof(*out) + AST_FRIENDLY_OFFSET + f->datalen;
  	/* If we have a source, add space for it */
  	/*
  	 * XXX Watch out here - if we receive a src which is not terminated
  	 * properly, we can be easily attacked. Should limit the size we deal with.
  	 */
  	if (f->src)
  		srclen = strlen(f->src);
  	if (srclen > 0)
  		len += srclen + 1;
  ```
  Note the `XXX` warning sitting in the middle of that calculation about
  unterminated `src` strings being an attack surface — do not point `src` at
  non-NUL-terminated memory.
- Tries the per-thread cache first, accepting any cached header whose
  `mallocd_hdr_len >= len` (`main/frame.c:330`).
- Bumps the format or topology reference (`main/frame.c:354-359`).
- Sets `mallocd = AST_MALLOCD_HDR` and `offset = AST_FRIENDLY_OFFSET`.
- Lays out payload at `buf + sizeof(*out) + AST_FRIENDLY_OFFSET` and `src`
  immediately after it.
- Preserves an empty text frame's `data.ptr` (`main/frame.c:373-379`):
  ```c
  	/* Make sure that empty text frames have a valid data.ptr */
  	if (out->datalen || f->frametype == AST_FRAME_TEXT) {
  		out->data.ptr = buf + sizeof(*out) + AST_FRIENDLY_OFFSET;
  		memcpy(out->data.ptr, f->data.ptr, out->datalen);
  	} else {
  		out->data.uint32 = f->data.uint32;
  	}
  ```
- Copies flags, `ts`, `len`, `seqno`, `stream_num` and `delivery`.

**`frdup` does not copy `frame_list`.** The duplicate is a standalone frame.

**`frdup` takes a `const struct ast_frame *`** — it never mutates or consumes its
input. The caller keeps ownership of the original.

## Isolating: `ast_frisolate()`

`main/frame.c:197-303`. Its job (`main/frame.c:192-196`):

> 'isolates' a frame by duplicating non-malloc'ed components (header, src, data).
> On return all components are malloc'ed.

Use it when you have a frame that may be partly borrowed and you need to hold onto
it past the lifetime of whatever lent you the memory — the classic case being a
driver's static frame that you want to keep after `tech->read` returns.

Its three-way behaviour:

1. `mallocd == 0` → delegates straight to `__ast_frdup()`, which is cheaper
   (`main/frame.c:202-207`).
2. All three bits already set → **returns `fr` unchanged**
   (`main/frame.c:209-213`). So `ast_frisolate()` is *not* guaranteed to return a
   new frame.
3. Otherwise it allocates a new header if needed and then, for `src` and `data`
   independently, either **duplicates** them (if the original did not own them) or
   **steals** them (if it did, and we allocated a new header). The steal clears the
   bit on the original — `src` at `main/frame.c:262-268`, `data` at
   `main/frame.c:294-300`:
   ```c
   	} else if (out != fr) {
   		/* Steal the data buffer from the original frame. */
   		out->data = fr->data;
   		memset(&fr->data, 0, sizeof(fr->data));
   		fr->mallocd &= ~AST_MALLOCD_DATA;
   		out->mallocd |= AST_MALLOCD_DATA;
   	}
   ```

**Two rules follow, and both are easy to get wrong:**

- **Always use the return value.** `ast_frisolate(f)` may return `f`, or a new
  frame, or `NULL` on allocation failure.
  <!-- illustrative -->
  ```c
  	/* WRONG */
  	ast_frisolate(f);
  	use(f);

  	/* RIGHT */
  	struct ast_frame *iso = ast_frisolate(f);
  	if (!iso) { ... }
  	use(iso);
  	if (iso != f) ast_frfree(f);
  ```
- **The original may have been gutted.** After a steal, the original frame's
  `data` is zeroed and its ownership bits cleared. It is still safe to
  `ast_frfree()` (that is the point), but its payload is gone.

`main/channel.c:5408` is the canonical in-tree use, rebuilding a frame list after
write-direction audiohooks.

### A real bug at `main/frame.c:222-227`

```c
		if ((fr->frametype == AST_FRAME_VOICE) || (fr->frametype == AST_FRAME_VIDEO) ||
			(fr->frametype == AST_FRAME_IMAGE)) {
			ao2_bump(out->subclass.format);
		} else if (fr->frametype == AST_FRAME_VOICE && fr->subclass.integer == AST_CONTROL_ANSWER) {
			ao2_bump(out->subclass.topology);
		}
```

The `else if` tests `AST_FRAME_VOICE` where every analogous site tests
`AST_FRAME_CONTROL`. Compare `__ast_frdup()` at `main/frame.c:357` and
`__frame_free()` at `main/frame.c:357-359` and `:165`, which both get it right:

```c
	} else if (f->frametype == AST_FRAME_CONTROL && f->subclass.integer == AST_CONTROL_ANSWER) {
		ao2_bump(out->subclass.topology);
	}
```

Worse, the condition is **unreachable**: the first `if` already matched every
`AST_FRAME_VOICE` frame. So `__ast_frisolate()` never bumps the topology
reference on an `AST_CONTROL_ANSWER` frame, while `__frame_free()` *will* release
it — an over-release waiting for the right caller. In practice `ANSWER` frames
reach the heap via `ast_queue_frame()`/`frdup` rather than `frisolate`, which is
why this has not bitten. Do not rely on that.

## The per-thread frame header cache

Gated at `main/frame.c:48-50`:

```c
#if (defined(LOW_MEMORY) || defined(MALLOC_DEBUG)) && !defined(NO_FRAME_CACHE)
#define NO_FRAME_CACHE
#endif
```

So the cache is compiled out entirely under `LOW_MEMORY` or `MALLOC_DEBUG` — which
matters when you are chasing a leak, because the behaviour you are debugging may
differ from production. It is additionally gated at runtime on
`ast_opt_cache_media_frames` (`main/frame.c:130`).

The storage (`main/frame.c:52-77`):

```c
#if !defined(NO_FRAME_CACHE)
static void frame_cache_cleanup(void *data);

/*! \brief A per-thread cache of frame headers */
AST_THREADSTORAGE_CUSTOM(frame_cache, NULL, frame_cache_cleanup);

/*!
 * \brief Maximum ast_frame cache size
 *
 * In most cases where the frame header cache will be useful, the size
 * of the cache will stay very small.  However, it is not always the case that
 * the same thread that allocates the frame will be the one freeing them, so
 * sometimes a thread will never have any frames in its cache, or the cache
 * will never be pulled from.  For the latter case, we limit the maximum size.
 */
#define FRAME_CACHE_MAX_SIZE	10

/*! \brief This is just so ast_frames, a list head struct for holding a list of
 *  ast_frame structures, is defined. */
AST_LIST_HEAD_NOLOCK(ast_frames, ast_frame);

struct ast_frame_cache {
	struct ast_frames list;
	size_t size;
};
#endif
```

The `FRAME_CACHE_MAX_SIZE` cap has a specific rationale, spelled out in the
comment at `main/frame.c:58-66`: in the common case a thread both allocates and
frees its own frames, so its cache stays tiny. But nothing guarantees that. A
producer thread that never frees would grow its cache without bound, and a
consumer thread that never allocates would never pull from its own — so the cap
bounds the first case and accepts the waste in the second.

`ast_frame_header_new()` (`main/frame.c:81-107`) pops the head of the cache,
`memset`s it, and **restores `mallocd_hdr_len`** so the recycled block's true size
is not lost (`main/frame.c:88-97`):

```c
	if ((frames = ast_threadstorage_get(&frame_cache, sizeof(*frames)))) {
		if ((f = AST_LIST_REMOVE_HEAD(&frames->list, frame_list))) {
			size_t mallocd_len = f->mallocd_hdr_len;

			memset(f, 0, sizeof(*f));
			f->mallocd_hdr_len = mallocd_len;
			frames->size--;
			return f;
		}
	}
```

`__ast_frdup()` is choosier: it walks the cache looking for a block big enough for
header *plus* payload *plus* src (`main/frame.c:328-342`), which is why a thread
that handles large frames builds up large cached blocks and keeps reusing them.

Cleanup runs per-thread on exit (`main/frame.c:110-119`).

**Debugging implication:** a cached header is live memory that no frame points at.
Under a leak hunt, expect up to `FRAME_CACHE_MAX_SIZE` headers per thread of
apparently unreferenced allocation, and build with `MALLOC_DEBUG` to make them go
away.

## The ownership rules table

The single most useful thing in this guide. Memorize the right-hand column.

| API | Who owns the input afterwards | Who owns the output |
|---|---|---|
| `ast_read(chan)` / `ast_read_stream()` | n/a | **Caller.** Must `ast_frfree()` — except `&ast_null_frame` |
| `ast_write(chan, fr)` / `ast_write_stream()` | **Caller keeps `fr`.** Always. | n/a |
| `ast_queue_frame(chan, f)` | **Caller keeps `f`** — it is `frdup`'d | the channel's readq |
| `ast_queue_frame_head()` | same | same |
| `ast_frdup(f)` | Caller keeps `f` (it is `const`) | Caller owns the new frame |
| `ast_frisolate(f)` | Caller keeps `f`, but it may be gutted | Caller owns the return, which **may be `f`** |
| `ast_frfree(f)` | consumed, **along with its whole `frame_list` chain** | n/a |
| `ast_translate(path, f, 1)` | **consumed** — translate frees it | Caller owns the output |
| `ast_translate(path, f, 0)` | Caller keeps `f` | Caller owns the output |
| framehook `event_cb` returning `frame` | the core keeps it | the core |
| framehook `event_cb` returning a *different* frame | **the callback must free the old one** | the core |
| `tech->read` / `tech->read_stream` | n/a | **the driver** — core must not free it; it is typically `mallocd == 0` |
| `tech->write` / `tech->write_stream` | the core keeps it | n/a |

### The `ast_write` dance, in detail

`ast_write_stream()` tracks two pointers: `fr`, the caller's frame, and `f`, the
possibly-translated intermediate. At the very end (`main/channel.c:5492-5493`):

```c
	if (f && f != fr)
		ast_frfree(f);
```

It frees **only what it created**. The caller's `fr` is never touched. And the
`default:` case deliberately writes the *original* `fr`
(`main/channel.c:5482-5489`) with this comment:

> Channels do not expect to get NULL as a frame pointer and will segfault

### The `ast_queue_frame` convention

`__ast_queue_frame()` duplicates every frame it is given
(`main/channel.c:1088-1100`):

```c
	/* Build copies of all the new frames and count them */
	AST_LIST_HEAD_INIT_NOLOCK(&frames);
	for (cur = fin; cur; cur = AST_LIST_NEXT(cur, frame_list)) {
		if (!(f = ast_frdup(cur))) {
			if (AST_LIST_FIRST(&frames)) {
				ast_frfree(AST_LIST_FIRST(&frames));
			}
			ast_channel_unlock(chan);
			return -1;
		}

		AST_LIST_INSERT_TAIL(&frames, f, frame_list);
		new_frames++;
```

Which is why so much of the core queues frames that live on the stack —
`main/channel.c:1249-1273` has three in a row. If you *built* a heap frame to
queue it, you must free it yourself afterwards; the queue took a copy.

## Pitfalls, ranked by how often they happen

1. **Freeing `&ast_null_frame`.** Harmless today, wrong always. Check
   `f != &ast_null_frame` before freeing anything that came out of `ast_read()`.
2. **Freeing a frame that is still in a list** — takes the rest of the chain. Null
   the link first.
3. **Freeing the frame you passed to `ast_write()`... twice.** `ast_write()` does
   not consume it; if you also free it you are fine, but if you believed it
   consumed it and freed nothing, you leak; and if you believed it consumed it and
   *also* free it in your error path, you double-free.
4. **Ignoring `ast_frisolate()`'s return value.** See above.
5. **Mutating `offset` on a frame with `AST_MALLOCD_DATA`.** The free path
   subtracts it. Use `AST_FRAME_SET_BUFFER`.
6. **Returning a different frame from a framehook without freeing the old one.**
   The interface contract (`include/asterisk/framehook.h`) is explicit that this
   is your job. See [07-interception.md](07-interception.md).
7. **Getting `ast_translate()`'s `consume` argument wrong.** The read path passes
   `1`, the write path passes `0`. Mixing those up is a leak one way and a
   double-free the other.
8. **`strlen()` on an `AST_FRAME_TEXT` payload.** It is not NUL-terminated. Use
   `datalen`.
9. **Pointing `src` at non-NUL-terminated memory.** `__ast_frdup()` `strlen()`s it
   — see the `XXX` at `main/frame.c:318-321`.
10. **Assuming a heap frame's payload is separately allocated** because
    `AST_MALLOCD_HDR` is set. For `frdup` output it is not.

## Dead API: `ast_frame_enqueue()`

`include/asterisk/frame.h:612` declares:

```c
struct ast_frame *ast_frame_enqueue(struct ast_frame *head, struct ast_frame *f, int maxlen, int dupe);
```

**There is no definition anywhere in the tree.** Calling it will fail to link.
Treat the declaration as vestigial and use `__ast_queue_frame()` or plain
`AST_LIST_*` macros.

---

Next: [04-control-frames.md](04-control-frames.md) — the signalling catalog.
