# 8. Atomics

Every `file:line` citation in this guide was verified against the `24` branch.
Newer releases and other branches may differ — line numbers drift — so if you
are reading this against a different tree, treat the citations as approximate
and re-grep for the symbol name.

This is the smallest region in the guide and the one with the widest gap
between what is defined and what is used. The whole of it is 129 lines at the
end of `include/asterisk/lock.h` (`:652-780`) — sixteen names, twelve of them
written out twice against two different families of compiler built-in, and a
one-line `#error` for the platform that has neither. Four of the sixteen names
have callers. Twelve have none.

That ratio is the chapter. The layer was written to be a general-purpose
wrapper over the compiler's atomic built-ins, with the full `__atomic`
vocabulary and a `memorder` argument on every macro; what the tree actually
does with it is increment and decrement integers with relaxed ordering, 179
times, and 176 of those go through two helpers that hard-wire the `memorder`
to `__ATOMIC_RELAXED` so the caller never sees it. Everything else in the
region is there for a caller that has not arrived.

It also closes the subsystem, so this chapter ends with an addendum
cataloguing the rough edges the previous seven chapters turned up, each one
re-checked against the tree and cited so you can check it yourself.

## 8.1 The region, and the two arms

The doc comment first, because it is unusually candid about its own design:

```c
/*!
 * \brief Support for atomic instructions.
 *
 * These macros implement a uniform interface to use built-in atomic functionality.
 * If available __atomic built-ins are prefered.  Legacy __sync built-ins are used
 * as a fallback for older compilers.
 *
 * Detailed documentation can be found in the GCC manual, all API's are modeled after
 * the __atomic interfaces but using the namespace ast_atomic.
 *
 * The memorder argument is always ignored by legacy __sync functions.  Invalid
 * memorder arguments do not produce errors unless __atomic functions are supported
 * as the argument is erased by the preprocessor.
 *
 * \note ast_atomic_fetch_nand and ast_atomic_nand_fetch purposely do not exist.
 *       It's implementation was broken prior to gcc-4.4.
 *
 * @{
 */
```

— `include/asterisk/lock.h:652-670`

Three things to take from it. The `__atomic` built-ins are preferred and the
`__sync` built-ins are the fallback — so the region has two complete arms and
your build uses exactly one of them. The `memorder` argument is "always
ignored" by the fallback, and §8.2 is about what that costs. And
`ast_atomic_fetch_nand` and `ast_atomic_nand_fetch` are missing on purpose,
because GCC's implementation of nand was wrong before 4.4 — a compiler released
in April 2009, which the tree is still routing around.

Then the preferred arm:

```c
#if defined(HAVE_C_ATOMICS)
/*! Atomic += */
#define ast_atomic_fetch_add(ptr, val, memorder)  __atomic_fetch_add((ptr), (val), (memorder))
#define ast_atomic_add_fetch(ptr, val, memorder)  __atomic_add_fetch((ptr), (val), (memorder))

/*! Atomic -= */
#define ast_atomic_fetch_sub(ptr, val, memorder)  __atomic_fetch_sub((ptr), (val), (memorder))
#define ast_atomic_sub_fetch(ptr, val, memorder)  __atomic_sub_fetch((ptr), (val), (memorder))

/*! Atomic &= */
#define ast_atomic_fetch_and(ptr, val, memorder)  __atomic_fetch_and((ptr), (val), (memorder))
#define ast_atomic_and_fetch(ptr, val, memorder)  __atomic_and_fetch((ptr), (val), (memorder))

/*! Atomic |= */
#define ast_atomic_fetch_or(ptr, val, memorder)   __atomic_fetch_or((ptr), (val), (memorder))
#define ast_atomic_or_fetch(ptr, val, memorder)   __atomic_or_fetch((ptr), (val), (memorder))

/*! Atomic xor = */
#define ast_atomic_fetch_xor(ptr, val, memorder)  __atomic_fetch_xor((ptr), (val), (memorder))
#define ast_atomic_xor_fetch(ptr, val, memorder)  __atomic_xor_fetch((ptr), (val), (memorder))

#if 0
/* Atomic compare and swap
 *
 * See comments near the __atomic implementation for why this is disabled.
 */
#define ast_atomic_compare_exchange_n(ptr, expected, desired, success_memorder, failure_memorder) \
	__atomic_compare_exchange_n((ptr), (expected), (desired), 0, success_memorder, failure_memorder)

#define ast_atomic_compare_exchange(ptr, expected, desired, success_memorder, failure_memorder) \
	__atomic_compare_exchange((ptr), (expected), (desired), 0, success_memorder, failure_memorder)
#endif
```

— `include/asterisk/lock.h:674-705`

Ten macros in five pairs, one `#if 0`, and nothing else. Each pair is the same
operation in the two orders the GCC built-ins offer: `fetch_add` returns the
value before the addition, `add_fetch` the value after. There is no wrapper
function anywhere — each macro is a direct textual substitution onto a built-in
with the same argument list, so the layer adds a namespace and nothing else.
That is the point of it, and it is why there is no error handling and no
message table worth the name in §8.11.

Then the fallback arm, which is the same ten names over the older built-ins:

```c
#elif defined(HAVE_GCC_ATOMICS)
/*! Atomic += */
#define ast_atomic_fetch_add(ptr, val, memorder)  __sync_fetch_and_add((ptr), (val))
#define ast_atomic_add_fetch(ptr, val, memorder)  __sync_add_and_fetch((ptr), (val))

/*! Atomic -= */
#define ast_atomic_fetch_sub(ptr, val, memorder)  __sync_fetch_and_sub((ptr), (val))
#define ast_atomic_sub_fetch(ptr, val, memorder)  __sync_sub_and_fetch((ptr), (val))

/*! Atomic &= */
#define ast_atomic_fetch_and(ptr, val, memorder)  __sync_fetch_and_and((ptr), (val))
#define ast_atomic_and_fetch(ptr, val, memorder)  __sync_and_and_fetch((ptr), (val))

/*! Atomic |= */
#define ast_atomic_fetch_or(ptr, val, memorder)  __sync_fetch_and_or((ptr), (val))
#define ast_atomic_or_fetch(ptr, val, memorder)  __sync_or_and_fetch((ptr), (val))

/*! Atomic xor = */
#define ast_atomic_fetch_xor(ptr, val, memorder)  __sync_fetch_and_xor((ptr), (val))
#define ast_atomic_xor_fetch(ptr, val, memorder)  __sync_xor_and_fetch((ptr), (val))

#if 0
/* Atomic compare and swap
 *
 * The \a expected argument is a pointer, I'm guessing __atomic built-ins
 * perform all memory reads/writes in a single atomic operation.  I don't
 * believe this is possible to exactly replicate using __sync built-ins.
 * Will need to determine potential use cases of this feature and write a
 * wrapper which provides consistant behavior between __sync and __atomic
 * implementations.
 */
#define ast_atomic_compare_exchange_n(ptr, expected, desired, success_memorder, failure_memorder) \
	__sync_bool_compare_and_swap((ptr), *(expected), (desired))

#define ast_atomic_compare_exchange(ptr, expected, desired, success_memorder, failure_memorder) \
	__sync_bool_compare_and_swap((ptr), *(expected), *(desired))
#endif

#else
#error "Atomics not available."
#endif
```

— `include/asterisk/lock.h:707-747`

Read the two arms against each other and the `memorder` parameter is the only
difference that matters. On the `__atomic` side it is passed through. On the
`__sync` side the macro takes three arguments and uses two: `memorder` is
accepted and dropped. The `#else` at `:745` is the whole of the layer's
platform support story — a compiler that has neither family does not get a
degraded build, it gets `#error "Atomics not available."` and no binary at all.

Which of the two arms you get is decided by `configure`, with two link tests
that are nine lines apart:

```c
AC_MSG_CHECKING(for compiler sync operations)
AC_LINK_IFELSE(
[AC_LANG_PROGRAM([], [int foo1; int foo2 = __sync_fetch_and_add(&foo1, 1);])],
AC_MSG_RESULT(yes)
ax_cv_have_gcc_atomics=1
AC_DEFINE([HAVE_GCC_ATOMICS], 1, [Define to 1 if your GCC C compiler provides __sync atomic operations.]),
AC_MSG_RESULT(no)
)

AC_MSG_CHECKING(for compiler atomic operations)
AC_LINK_IFELSE(
[AC_LANG_PROGRAM([], [int foo1; int foo2 = __atomic_fetch_add(&foo1, 1, __ATOMIC_RELAXED);])],
AC_MSG_RESULT(yes)
ax_cv_have_c_atomics=1
AC_DEFINE([HAVE_C_ATOMICS], 1, [Define to 1 if your C compiler provides __atomic operations.]),
AC_MSG_RESULT(no)
)

if test -z $ax_cv_have_c_atomics$ax_cv_have_gcc_atomics; then
	AC_MSG_ERROR([*** Atomic operations are not supported by your compiler.])
fi
```

— `configure.ac:1196-1216`

Note the shape. Both tests always run, so a modern GCC or clang sets
`HAVE_GCC_ATOMICS` *and* `HAVE_C_ATOMICS`; `lock.h:674` tests `HAVE_C_ATOMICS`
first, so the `__atomic` arm wins whenever it is available and the `__sync` arm
is reached only on a compiler that has `__sync_fetch_and_add` but not
`__atomic_fetch_add`. In GCC terms that is a compiler older than 4.7 (2012).
The `AC_MSG_ERROR` at `:1215` fires only if both tests fail.

So on anything you are likely to be building on, the `__sync` arm at
`lock.h:707-747` is dead code, and so is the `#error` below it. The chapter
still reads both, because the `memorder` behaviour of the dead arm is what
determines how the live arm may be called — §8.2.

## 8.2 The `memorder` argument, and the one value the tree passes

The doc comment's phrasing at `lock.h:662-664` is worth reading twice:

> The memorder argument is always ignored by legacy `__sync` functions.
> Invalid memorder arguments do not produce errors unless `__atomic` functions
> are supported as the argument is erased by the preprocessor.

"Erased by the preprocessor" is exact. On the `__sync` arm the macro parameter
`memorder` appears in the parameter list at `lock.h:709` and nowhere in the
replacement text, so whatever the caller wrote is discarded before the compiler
sees a token of it. `ast_atomic_fetch_add(&n, 1, banana)` compiles on that arm.
On the `__atomic` arm the same call fails — but as an undeclared identifier,
which is not what the comment is warning about. The built-in does not require
an integer constant: `__atomic_fetch_add(p, 1, m)` with a runtime `int m`
compiles clean. What a declared-but-invalid value gets you is a warning,
`invalid memory model 99 for '__atomic_fetch_add_4' [-Winvalid-memory-model]`,
and a warning stops the build only under `AST_DEVMODE`, which adds `-Werror`
at `Makefile:211-212`.

So the stated hazard is real but narrower than it reads: code written and
tested on a `__sync` build can fail to compile on an `__atomic` build, and a
`memorder` that is wrong rather than invalid — acquire where you needed
sequential consistency — compiles silently on both and warns on neither.

In this tree the hazard is theoretical, for a reason the comment does not
mention. Every `memorder` argument in Asterisk's own C sources is
`__ATOMIC_RELAXED`. There are five, and this is all of them:

| Site | Call |
|---|---|
| `main/astobj2.c:530` | `ast_atomic_fetch_add(&obj->priv_data.ref_counter, delta, __ATOMIC_RELAXED)` |
| `res/res_aeap/transport.c:91` | `ast_atomic_fetch_add(&transport->connected, 0, __ATOMIC_RELAXED)` |
| `res/res_aeap/transport.c:112` | `ast_atomic_fetch_sub(&transport->connected, 1, __ATOMIC_RELAXED)` |
| `include/asterisk/lock.h:763` | `ast_atomic_fetch_add(p, v, __ATOMIC_RELAXED)` |
| `include/asterisk/lock.h:773` | `ast_atomic_sub_fetch(p, 1, __ATOMIC_RELAXED)` |

The last two are inside the two helpers of §8.6, which is why the other 176
call sites in the tree do not appear here: they go through
`ast_atomic_fetchadd_int` or `ast_atomic_dec_and_test` and never name a
`memorder` at all. A grep for `__ATOMIC_` outside `third-party/` finds no
`__ATOMIC_ACQUIRE`, no `__ATOMIC_RELEASE`, no `__ATOMIC_SEQ_CST` and no
`__ATOMIC_ACQ_REL` anywhere in Asterisk's own code. The only occurrences of
the prefix outside those five are the probe at `configure.ac:1207` and the
same probe in the generated — but git-tracked — `configure` at `:31376`, and
both of those are `__ATOMIC_RELAXED` too.

Relaxed ordering means the atomic operation is indivisible and nothing more: it
orders nothing around it, and the compiler and the processor may move ordinary
loads and stores across it freely. For a reference count that is the textbook
correct choice on the increment — you are only counting — and the textbook
*questionable* choice on the decrement, because the thread that takes a
refcount to zero then proceeds to read and free the object, and relaxed
ordering gives it no guarantee that it sees the writes another thread made
before dropping *its* reference. The usual pattern is a release on the
decrement and an acquire fence before the destructor. §8.9 looks at what
`astobj2.c` does instead.

By contrast, the bundled jansson in `third-party/` does use acquire and release
on its own refcount (`third-party/jansson/source/src/jansson.h:111-113`). The
library Asterisk vendors is more careful about this than Asterisk is.

## 8.3 The compare-exchange that is written twice and compiled never

Both arms carry a compare-and-swap, and both are inside `#if 0`. The
`__atomic` version is at `lock.h:695-705` and the comment on it points at the
`__sync` version for the explanation. Here is the explanation:

```c
#if 0
/* Atomic compare and swap
 *
 * The \a expected argument is a pointer, I'm guessing __atomic built-ins
 * perform all memory reads/writes in a single atomic operation.  I don't
 * believe this is possible to exactly replicate using __sync built-ins.
 * Will need to determine potential use cases of this feature and write a
 * wrapper which provides consistant behavior between __sync and __atomic
 * implementations.
 */
#define ast_atomic_compare_exchange_n(ptr, expected, desired, success_memorder, failure_memorder) \
	__sync_bool_compare_and_swap((ptr), *(expected), (desired))

#define ast_atomic_compare_exchange(ptr, expected, desired, success_memorder, failure_memorder) \
	__sync_bool_compare_and_swap((ptr), *(expected), *(desired))
#endif
```

— `include/asterisk/lock.h:728-743`

This is an honest note, and it is the most interesting four lines in the
region. `__atomic_compare_exchange_n` takes `expected` as a *pointer*: on
failure it writes the observed value back through that pointer, so the caller
gets the current value and the failure in one operation.
`__sync_bool_compare_and_swap` takes the expected value directly and returns
only a boolean, so a `__sync` implementation has to dereference `expected`
before the operation and cannot report what it actually found. The two cannot
be made to behave the same way, so rather than ship a wrapper whose semantics
change with the build, the author disabled both and wrote down why.

The consequence is that Asterisk has no compare-and-swap in its atomics layer.
Nothing in the tree calls `ast_atomic_compare_exchange` or
`ast_atomic_compare_exchange_n`, which is unsurprising since neither name
exists after preprocessing. Code that needs a CAS takes a mutex instead, and
`include/asterisk/spinlock.h` — §8.10 — is an unused attempt at the
lighter-weight alternative.

## 8.4 The two flag macros, and their zero callers

```c
/*! Atomic flag set */
#define ast_atomic_flag_set(ptr, val, memorder)   ast_atomic_fetch_or((ptr), (val), (memorder))

/*! Atomic flag clear */
#define ast_atomic_flag_clear(ptr, val, memorder) ast_atomic_fetch_and((ptr), ~(val), (memorder))
```

— `include/asterisk/lock.h:749-753`

Two more names, defined in terms of two of the ten, with no callers anywhere in
the tree. They are worth a paragraph only for what they imply about the
argument: `val` is a bit mask, not a bit number, so
`ast_atomic_flag_set(&flags, 1 << 3, __ATOMIC_RELAXED)` sets bit three and
`ast_atomic_flag_set(&flags, 3, …)` sets bits zero and one. The `~(val)` in
`flag_clear` is parenthesised correctly, so a compound expression works, but
note that `~` on a non-negative signed operand gives you a negative mask —
which is what `fetch_and` wants, and is fine, but will look wrong in a
debugger.

Neither macro returns a useful flag state. Both expand to `fetch_*`, so you get
the whole word as it was before your change and have to mask it yourself.

## 8.5 Sixteen names and one count

The whole region in one table, with the call-site count for each. Counts are
of Asterisk's own `.c` and `.h` files, excluding `third-party/`, excluding the
definitions themselves, and excluding `utils/extconf.c`, which carries a
private fossil copy of the whole layer — definitions and one internal call —
that nothing else in the tree can reach (§8.10).

| Name | Defined at | Call sites |
|---|---|---|
| `ast_atomic_fetch_add` | `lock.h:676`, `:709` | 2 |
| `ast_atomic_add_fetch` | `lock.h:677`, `:710` | 0 |
| `ast_atomic_fetch_sub` | `lock.h:680`, `:713` | 1 |
| `ast_atomic_sub_fetch` | `lock.h:681`, `:714` | 0 |
| `ast_atomic_fetch_and` | `lock.h:684`, `:717` | 0 |
| `ast_atomic_and_fetch` | `lock.h:685`, `:718` | 0 |
| `ast_atomic_fetch_or` | `lock.h:688`, `:721` | 0 |
| `ast_atomic_or_fetch` | `lock.h:689`, `:722` | 0 |
| `ast_atomic_fetch_xor` | `lock.h:692`, `:725` | 0 |
| `ast_atomic_xor_fetch` | `lock.h:693`, `:726` | 0 |
| `ast_atomic_compare_exchange_n` | `lock.h:700`, `:738` — both in `#if 0` | 0 |
| `ast_atomic_compare_exchange` | `lock.h:703`, `:741` — both in `#if 0` | 0 |
| `ast_atomic_flag_set` | `lock.h:750` | 0 |
| `ast_atomic_flag_clear` | `lock.h:753` | 0 |
| `ast_atomic_fetchadd_int` | `lock.h:761-764` | 171 |
| `ast_atomic_dec_and_test` | `lock.h:771-774` | 5 |

Four names carry the entire subsystem, and one of them carries 96 percent of
it. The three uses of the general macros are the two in `res_aeap/transport.c`
and the one in `astobj2.c` that is the refcount itself. The two `#if 0` names
cannot be called even in principle.

Three of the zero rows are not quite unused. `ast_atomic_fetch_or` and
`ast_atomic_fetch_and` are what the two flag macros of §8.4 expand to
(`lock.h:750`, `:753`), and `ast_atomic_sub_fetch` is the body of
`ast_atomic_dec_and_test` (`:773`). The count is of calls from outside the
region, and from outside the region no caller names any of the three.

The 171 `ast_atomic_fetchadd_int` sites by directory: 76 in `main`, 41 in
`res`, 18 in `channels`, 13 each in `tests` and `apps`, 6 in `codecs`, 3 in
`funcs`, 1 in `pbx`. They are spread over 54 files, all of them `.c`. The
densest single file is `main/astobj2.c` with 20, then `res/res_fax_spandsp.c`
with 15 and `res/res_fax.c` with 12.

By delta: 118 sites add 1 — 61 of them written `1` and 57 written `+1`, which
is the same operation spelled two ways — 38 subtract 1, 7 add 0, and exactly
8 pass anything else. The whole of that last group is `main/astobj2.c:317`,
`:534`, `:619`, `:754`, `main/manager.c:8049`, `codecs/codec_dahdi.c:836`,
`apps/app_meetme.c:1594` and `apps/app_voicemail.c:1331`: refcount deltas
threaded through a variable, a byte count, and one ternary. Nothing in the
tree uses an atomic to transfer a batch.

The 38 that subtract 1 are doing by hand what `ast_atomic_dec_and_test` does,
and 37 of them throw the return value away — `main/astobj2.c:314` is the only
one that keeps it, and what it keeps is the return *minus one*, tested `< 0`
at `:315` to detect a write-lock unlock rather than a count reaching zero. The
five sites that use the helper instead are `channels/chan_iax2.c:12232`,
`apps/app_meetme.c:2125` and three places in `apps/app_sla.c`.

## 8.6 `ast_atomic_fetchadd_int`, `ast_atomic_dec_and_test`, and the host module

The two helpers are not macros:

```c
/*!
 * \brief Atomically add v to *p and return the previous value of *p.
 *
 * This can be used to handle reference counts, and the return value
 * can be used to generate unique identifiers.
 */
AST_INLINE_API(int ast_atomic_fetchadd_int(volatile int *p, int v),
{
	return ast_atomic_fetch_add(p, v, __ATOMIC_RELAXED);
})

/*!
 * \brief decrement *p by 1 and return true if the variable has reached 0.
 *
 * Useful e.g. to check if a refcount has reached 0.
 */
AST_INLINE_API(int ast_atomic_dec_and_test(volatile int *p),
{
	return ast_atomic_sub_fetch(p, 1, __ATOMIC_RELAXED) == 0;
})
```

— `include/asterisk/lock.h:755-774`

Both are `AST_INLINE_API`, which is the tree's own device for a function that
wants to be inlined into every caller and still have exactly one out-of-line
body somewhere for anything that cannot inline it:

```c
#if !defined(DISABLE_INLINE)

#if !defined(AST_API_MODULE)
#if defined(__clang__) || defined(__GNUC_STDC_INLINE__)
#define AST_INLINE_API(hdr, body) static hdr; static inline hdr body
#else /* if defined(__clang__) */
#define AST_INLINE_API(hdr, body) hdr; extern inline hdr body
#endif
#else /* if !defined(AST_API_MODULE) */
#define AST_INLINE_API(hdr, body) hdr; hdr body
#endif

#else /* defined(DISABLE_INLINE) */

#if !defined(AST_API_MODULE)
#define AST_INLINE_API(hdr, body) hdr;
#else
#define AST_INLINE_API(hdr, body) hdr; hdr body
#endif

#endif
```

— `include/asterisk/inline_api.h:48-68`

Five `#define` sites, four distinct expansion texts — `:57` and `:65` are
character-for-character identical — chosen by three conditions:
`DISABLE_INLINE` at `:48`, `AST_API_MODULE` at `:50` and `:62`, and
`__clang__ || __GNUC_STDC_INLINE__` at `:51`. In an ordinary translation unit
on GCC or clang, `AST_INLINE_API(hdr, body)` becomes `static hdr; static
inline hdr body` — a private copy per object file. In a translation unit that
defines `AST_API_MODULE` before including the header, it becomes `hdr; hdr
body` — an external definition with a real symbol.

Note `#undef AST_API_MODULE` at `:70`. The include guard at `:19-20` means
the body runs once per translation unit, so that `#undef` fires once, on the
first inclusion — and the flag therefore has to be defined before *that*
inclusion, wherever in the include graph it happens to come from. For
`lock.h` in the `asterisk` binary the host is `main/utils.c`, which says so in
the comment on the line above the include:

```c
#define AST_API_MODULE		/* ensure that inlinable API functions will be built in lock.h if required */
#include "asterisk/lock.h"
```

— `main/utils.c:48-49`

`utils/check_expr.c` does the same thing twice over — at `:28-29` before
including `inline_api.h` directly, and again at `:31-32` before `lock.h` — and
by the rule just given only the first of the two does anything. It builds the
`check_expr` developer utility rather than `asterisk`, so inside the daemon
`main/utils.c` is the only host.

So `ast_atomic_fetchadd_int` has a real symbol in `main/utils.o` that nothing
in the tree links against, and a private inlined copy in each of the 54 files
that call it — in more translation units than that, since
`apps/app_voicemail.c` is compiled once per storage backend through its two
symlinks (`apps/Makefile:57-59`). Under the `DISABLE_INLINE` build flag — a
menuselect option that exists to work around a 2011 GCC bug,
`build_tools/cflags.xml:33-37` — the macro collapses to
`hdr;` everywhere except `main/utils.c`, and that unused symbol becomes the
only definition. It is the one configuration in which the helpers are function
calls rather than two instructions.

Both helpers pin the `memorder`. `ast_atomic_fetchadd_int` is
`ast_atomic_fetch_add(p, v, __ATOMIC_RELAXED)` and `ast_atomic_dec_and_test` is
`ast_atomic_sub_fetch(p, 1, __ATOMIC_RELAXED) == 0`; neither takes a `memorder`
parameter, so a caller who needs stronger ordering has to abandon the helper
and use the general macro, which three call sites in the tree do. This is why
§8.2's table is five rows long and not 179.

Note the signatures: both take `volatile int *`. The `volatile` is vestigial.
It predates the `__atomic` rewrite, and it is not what makes these operations
ordered: the built-ins take their ordering from the `memorder` argument, which
§8.2 showed is `__ATOMIC_RELAXED` in every case, so there is no ordering here
for `volatile` to be adding. What the signature does cost you is the `int` — a
counter declared `unsigned int` needs a cast at every call. §8.8 is about the
call sites that supply one.

## 8.7 `fetchadd_int(p, 0)` as an atomic read

Seven call sites add zero:

| Site | What it reads |
|---|---|
| `main/astobj2_container.c:36` | the element count of an ao2 container |
| `main/channel.c:507` | the global `chancount`, the whole body of `ast_undestroyed_channels()` (`:505-508`) |
| `res/res_pjsip_config_wizard.c:312` | the ACL reload counter, inside `IS_ACL_RELOAD_ACTIVE()` |
| `tests/test_astobj2_thrash.c:130` | a grow counter in the thrash loop |
| `tests/test_astobj2_thrash.c:338`, `:340` | an allocation counter, twice in one check |
| `tests/test_hashtab_thrash.c:127` | the same grow counter in the hashtab thrash test |

Adding zero to a counter and taking the return value is an atomic read, and
with no atomic load in the layer's vocabulary it is the only atomic read
available. It works. It is also not free: on the `__atomic` arm
`__atomic_fetch_add(p, 0, __ATOMIC_RELAXED)` is a read-modify-write, which on
x86 compiles to a `lock xadd` — a locked bus operation and a dirtied cache
line, where `__atomic_load_n` would be a plain `mov`. Reading a counter this
way from several threads at once contends as hard as writing it.

The idiom appears nowhere in the doc comment and is not named by a macro of its
own, so each of the seven sites rediscovered it. `res_aeap/transport.c:91` uses
the general macro to do the same thing and explains itself in a comment —
"Avoid using a lock to 'read' the 'connected' variable in order to keep things
slightly more efficient" (`:88-90`).

## 8.8 Three places where the atomic sits next to a plain access

The layer gives you an indivisible operation on one variable. It cannot give
you anything about the accesses to that variable you did not route through it,
and three places in the tree mix the two on the same field.

**The ao2 refcount answers `ao2_ref(obj, 0)` with a plain read.** Chapter 05 of
the ao2 guide has the reference-counting mechanics; the atomic question is
this:

```c
	/* if delta is 0, just return the refcount */
	if (delta == 0) {
		return obj->priv_data.ref_counter;
	}

	if (delta < 0 && obj->priv_data.magic == AO2_MAGIC && (weakproxy = obj->priv_data.weakptr)) {
		ao2_lock(weakproxy);
	}

	/* we modify with an atomic operation the reference counter */
	ret = ast_atomic_fetch_add(&obj->priv_data.ref_counter, delta, __ATOMIC_RELAXED);
	current_value = ret + delta;
```

— `main/astobj2.c:520-531`

Line `:522` and line `:530` are eight lines apart and read the same field, one
with a plain load and one with an atomic. That plain read is everything
`ao2_ref(obj, 0)` does with the counter — `:501` validates the object and
`:511-518` deals with the null case before it, and then the function returns —
and `ao2_ref(obj, 0)` is how the rest of the tree asks for a refcount, so every
refcount you have ever read out of Asterisk came from a non-atomic load. On
any machine Asterisk runs on a 32-bit aligned load is not going to tear, so
what you lose is not atomicity but ordering — and with the
write side at `__ATOMIC_RELAXED` there was no ordering to lose. The plain read
is as correct as the atomic one here, which is a statement about the atomic one.

**`res_aeap/transport.c` sets its flag without an atomic and clears it with
one.** The comment is the thing to read:

```c
int aeap_transport_is_connected(struct aeap_transport *transport)
{
	/*
	 * Avoid using a lock to 'read' the 'connected' variable in order to
	 * keep things slightly more efficient.
	 */
	return ast_atomic_fetch_add(&transport->connected, 0, __ATOMIC_RELAXED);
}

int aeap_transport_disconnect(struct aeap_transport *transport)
{
	int res;

	SCOPED_MUTEX(rlock, &transport->read_lock);
	SCOPED_MUTEX(wlock, &transport->write_lock);

	if (!aeap_transport_is_connected(transport)) {
		return 0;
	}

	res = transport->vtable->disconnect(transport);

	/*
	 * Even though the transport is locked here use atomics to set the value of
	 * 'connected' since it's possible the variable is being 'read' by another
	 * thread via the 'is_connected' call.
	 */
	ast_atomic_fetch_sub(&transport->connected, 1, __ATOMIC_RELAXED);

	return res;
```

— `res/res_aeap/transport.c:85-114`

The comment at `:107-110` says "use atomics to set the value of `connected`
since it's possible the variable is being 'read' by another thread". The two
places that write the field without an atomic are `:43` in the constructor,
which sets it to 0, and `:62` in `aeap_transport_connect`, which sets it to 1.
Only the clear is atomic, and it is a decrement rather than a store, so it is
a clear only on the assumption that the value is exactly 1 — which `:62`
guarantees and `:43` establishes. `:62` holds both mutexes; `:43` holds
neither, two lines after the second of the two `ast_mutex_init` calls at
`:40-41` and before the transport is reachable by any other thread, which is
the one place that is defensible. The reader deliberately holds neither. So
the atomic on the clear is doing nothing the plain store at `:62` is not
doing, and the asymmetry is the comment's own claim being half-implemented.

**`app_sla.c` casts `unsigned int *` to `int *` seven times.** The fields are
`unsigned int active_stations` and `unsigned int hold_stations`
(`apps/app_sla.c:237`, `:239`); the helpers take `volatile int *`; so each call
site has an explicit `(int *)` in front of the address to silence the
conversion:

```c
	if (ast_atomic_dec_and_test((int *) &trunk_ref->trunk->active_stations) &&
```

— `apps/app_sla.c:1007`

The cast is benign on every platform Asterisk supports, since `int` and
`unsigned int` have the same size and representation for non-negative values,
and these counters do not go negative. What is not benign is what surrounds
them. The same two fields are read without an atomic at `:777` and `:1492` and
assigned without one at `:1010` and `:2093` — and `:1010` is three lines after
the `dec_and_test` at `:1007`, setting `hold_stations = 0` outright while
another thread may be in the atomic increment at `:1487`. Six of the seven
atomic accesses are on those two fields; the seventh, at `:2548`, is on a
third, `num_stations` (`:235`), which nothing else in the file touches
atomically at all. Six atomic accesses and four plain ones, on two fields, in
one file.

One more oddity in the same file: `:2005` reads

```c
		if (ast_atomic_dec_and_test((int *) &trunk_ref->trunk->hold_stations) == 1) {
```

— `apps/app_sla.c:2005`

`ast_atomic_dec_and_test` returns the result of `== 0`, so it is already 0 or
1, and comparing it to 1 is redundant. Of the other four call sites in the
tree, `channels/chan_iax2.c:12232` throws the return value away entirely —
with a comment at `:12231` telling Coverity to stop asking about it — and the
remaining three use it as a plain condition.

## 8.9 What happens if you want ordering

Nothing in the layer offers it. There is no `ast_atomic_thread_fence`, no
`ast_atomic_load`, no `ast_atomic_store`, and the two `compare_exchange` names
are inside `#if 0`. A caller who needs a release on a decrement and an acquire
before touching what it protects has three options in this tree: pass
`__ATOMIC_RELEASE` to `ast_atomic_sub_fetch` directly, which works on the
`__atomic` arm and is silently ignored on the `__sync` arm; call
`__atomic_thread_fence` with no wrapper, which nothing outside `third-party/`
does; or take a mutex, which is what the tree does.

Taking a mutex is not the obviously wrong answer. `ast_mutex_lock` is a
function call into `main/lock.c` (chapter 02) with lock tracking behind it, so
it is expensive by the standards of this chapter — but it is correct, it
shows up in `core show locks` on a build that has `DEBUG_THREADS` and does
not have `LOW_MEMORY` — the whole tracking and CLI block sits inside one
`#ifdef DEBUG_THREADS`, `main/utils.c:978` to `:1586`, with a second
`#if !defined(LOW_MEMORY)` around the CLI handler at `:1551-1585` and the
registration at `:2657` — and the deadlock-avoidance machinery of chapter 06
knows about it. An atomic does none of those things
and is invisible to every diagnostic in the subsystem. Nothing in `core show
locks`, `core show threads` or `DETECT_DEADLOCKS` can see an atomic operation,
and there is no counter anywhere of how many have been performed.

That is the practical rule this chapter ends on: the atomics layer is for
counters, and for counters it is the right tool. For anything where one
thread's writes have to become visible to another in a particular order, the
layer as it stands will not help you, and reaching for it because it looks
cheaper than a mutex is how you get a bug that no Asterisk diagnostic can show
you.

## 8.10 Two files that are not part of `asterisk`

Two files in the tree implement atomics and neither one is part of `asterisk`.

**`include/asterisk/spinlock.h` is 488 lines that nothing includes.** It dates
from 2014 and its premise is exactly the gap §8.9 just described:

```c
/*! \file
 * \brief Spin Locks.
 *
 * In some atomic operation circumstances the __atomic calls are not quite
 * flexible enough but a full fledged mutex or rwlock is too expensive.
 *
 * Spin locks should be used only for protecting short blocks of critical
 * code such as simple compares and assignments.  Operations that may block,
 * hold a lock, or cause the thread to give up it's timeslice should NEVER
 * be attempted in a spin lock.
 *
 * Because spinlocks must be as lightweight as possible, there are no
 * recursion or deadlock checks.
 *
 */
```

— `include/asterisk/spinlock.h:19-33`

What follows is seven implementations of a five-function API —
`ast_spinlock_init`, `_lock`, `_trylock`, `_unlock`, `_destroy` — selected by a
chain of `#if`s: `__sync_lock_test_and_set` (`:68-103`), hand-written x86
assembly (`:112-158`), ARM with a separate path for ARMv7 and later
(`:167-239`), SPARC (`:248-311`), `pthread_spinlock_t` (`:321-350`), OS X
`OSSpinLock` (`:361-394`), and a `pthread_mutex_t` fallback (`:404-434`),
below which sits `#error "No spinlock implementation could be found."` at
`:437`. A grep of the tree for `spinlock.h` or `ast_spinlock` finds exactly one
file: itself.

Two details are worth having. The OS X arm tests `HAVE_OSX_ATOMICS`, a macro
`configure` never defines — it is not in `include/asterisk/autoconfig.h.in` and
not in `configure.ac` — so that arm is unreachable even if the header were
included. And the `pthread_mutex_t` fallback at `:404-434` calls
`pthread_mutex_init`, `pthread_mutex_lock`, `pthread_mutex_trylock`,
`pthread_mutex_unlock` and `pthread_mutex_destroy` by name, every one of which
chapter 01 §1.6 showed `lock.h` poisons with a `#define` to an
`use_ast_mutex_*_instead_of_*` identifier. `spinlock.h` does not include
`lock.h`, so on its own it compiles; include both in one translation unit, with
no `pthread_spinlock_t` on the platform, and the fallback does not build. The
header has never been in a position to find that out.

**`utils/extconf.c` carries a fossil copy of the pre-2014 implementation.**
`extconf.c` is a 5,674-line standalone extraction of Asterisk's config parser,
linked into exactly one developer utility — `aelparse`, at `utils/Makefile:153`
— and it includes `asterisk/inline_api.h` but not
`asterisk/lock.h`. So it brings its own atomics, and they are the ones Asterisk
had before the `__atomic` rewrite:

```c
/*
 * Initial support for atomic instructions.
 * For platforms that have it, use the native cpu instruction to
 * implement them. For other platforms, resort to a 'slow' version
 * (defined in utils.c) that protects the atomic instruction with
 * a single lock.
 * The slow versions is always available, for testing purposes,
 * as ast_atomic_fetchadd_int_slow()
 */

#if defined(HAVE_OSX_ATOMICS)
#include "libkern/OSAtomic.h"
#endif

/*! \brief Atomically add v to *p and return * the previous value of *p.
 * This can be used to handle reference counts, and the return value
 * can be used to generate unique identifiers.
 */

#if defined(HAVE_GCC_ATOMICS)
AST_INLINE_API(int ast_atomic_fetchadd_int(volatile int *p, int v),
{
	return __sync_fetch_and_add(p, v);
})
#elif defined(HAVE_OSX_ATOMICS) && (SIZEOF_INT == 4)
AST_INLINE_API(int ast_atomic_fetchadd_int(volatile int *p, int v),
{
	return OSAtomicAdd32(v, (int32_t *) p);
})
#elif defined(HAVE_OSX_ATOMICS) && (SIZEOF_INT == 8)
AST_INLINE_API(int ast_atomic_fetchadd_int(volatile int *p, int v),
{
	return OSAtomicAdd64(v, (int64_t *) p);
#elif defined (__i386__) || defined(__x86_64__)
AST_INLINE_API(int ast_atomic_fetchadd_int(volatile int *p, int v),
{
	__asm __volatile (
	"       lock   xaddl   %0, %1 ;        "
	: "+r" (v),                     /* 0 (result) */
	  "=m" (*p)                     /* 1 */
	: "m" (*p));                    /* 2 */
	return (v);
})
#else
static int ast_atomic_fetchadd_int_slow(volatile int *p, int v)
{
	int ret;
	ret = *p;
	*p += v;
	return ret;
}
AST_INLINE_API(int ast_atomic_fetchadd_int(volatile int *p, int v),
{
	return ast_atomic_fetchadd_int_slow(p, v);
})
#endif
```

— `utils/extconf.c:570-625`

Note the comment at `:570-577`: a "slow version (defined in utils.c) that
protects the atomic instruction with a single lock", "always available, for
testing purposes, as `ast_atomic_fetchadd_int_slow()`". No such function exists
in `main/utils.c`, or anywhere outside this file. The slow version here, at
`:614-619`, is `ret = *p; *p += v; return ret;` with no lock of any kind — it is
not atomic, it is not protected, and the comment describing it is the only
remaining trace of the version that was.

In practice none of it is reached. `configure` guarantees `HAVE_GCC_ATOMICS` on
any compiler that can build Asterisk at all — §8.1 — so the first branch wins
and the inline assembly, the OS X branches and the non-atomic fallback are all
dead. Which is fortunate, because two of them do not compile. The
`SIZEOF_INT == 8` OS X branch of `ast_atomic_fetchadd_int` at `:600-602` opens
`AST_INLINE_API(…, {` and reaches the `#elif` at `:603` without its closing
`})`, and `ast_atomic_dec_and_test` has the identical defect at `:641-643`,
running into the `#else` at `:644`. Both sit under
`#elif defined(HAVE_OSX_ATOMICS) && (SIZEOF_INT == 8)` — `:599` and `:640` —
and `HAVE_OSX_ATOMICS` is the macro `configure` does not define.

## 8.11 Messages

The atomics layer emits nothing. There is no `ast_log` call, no
`ast_assert`, and no counter anywhere in the region — an atomic operation
cannot fail, so there is nothing to report, and no instrumentation was added
for the ones that succeed.

The two diagnostics in the region are both compile-time, and both are in the
`#else` arm of a chain that `configure` has already decided cannot be taken:

| Diagnostic | Origin | Fires when |
|---|---|---|
| `#error "Atomics not available."` | `include/asterisk/lock.h:746` | Neither `HAVE_C_ATOMICS` nor `HAVE_GCC_ATOMICS` is defined. `configure.ac:1214-1216` fails the build first, with `*** Atomic operations are not supported by your compiler.` |
| `#error "No spinlock implementation could be found."` | `include/asterisk/spinlock.h:437` | No `AST_SPINLOCK_TYPE` was selected. Unreachable by construction: the `pthread_mutex` fallback is guarded `#if !defined(AST_SPINLOCK_TYPE)` at `:404` and sets the macro at `:405`, and `:436` is character-for-character the same test, so it is always false by the time control reaches it — and nothing includes the header anyway |

If you are looking for a way to watch atomic activity at run time, there is
none, and §8.9 is the reason to care: this is the one part of the threading
subsystem with no introspection at all.

## 8.12 Rules

| Rule | Why |
|---|---|
| Use `ast_atomic_fetchadd_int` for counters and nothing else | It is 171 of the tree's 179 atomic call sites, it hard-wires `__ATOMIC_RELAXED`, and relaxed is the right answer for a count (§8.6) |
| Do not expect the `memorder` argument to do anything | Every `memorder` in Asterisk's own sources is `__ATOMIC_RELAXED`, and on a `__sync` build the argument is erased by the preprocessor before the compiler sees it (§8.2) |
| Never write a `memorder` you have not tested on an `__atomic` build | An invalid one compiles silently on the fallback arm; on the preferred arm it is `-Winvalid-memory-model`, which fails the build only under `AST_DEVMODE` (`lock.h:662-664`, §8.2) |
| Read a counter with `ast_atomic_fetchadd_int(p, 0)`, knowing what it costs | There is no atomic load in the layer, so this is the idiom — but it is a `lock xadd`, not a `mov`, and it contends like a write (§8.7) |
| Do not mix a plain access and an atomic one on the same field | Of the three places in the tree that do, one is saved by relaxed ordering having nothing to lose, one is a comment half-implemented, and the third — `apps/app_sla.c` — is simply racy (§8.8) |
| Do not reach for atomics to get ordering | There is no fence, no load, no store and no compare-exchange — the two that are written are inside `#if 0` (§8.3, §8.9) |
| Take a mutex when correctness depends on ordering | It is more expensive and it is visible to `core show locks` on a `DEBUG_THREADS`, non-`LOW_MEMORY` build and to `DETECT_DEADLOCKS`, which is independent of both; an atomic is invisible to all three, and to every other diagnostic in the subsystem (§8.9) |
| Do not use `include/asterisk/spinlock.h` | Nothing includes it, one of its seven arms tests a macro `configure` never defines, and its `pthread_mutex` fallback names the functions `lock.h` poisons (§8.10) |
| Cast with care if your counter is not an `int` | The helpers take `volatile int *`; the seven `(int *)` casts in `apps/app_sla.c` are benign, and the plain reads and writes beside them are the actual problem (§8.8) |
| Do not read `utils/extconf.c` as current | It carries the pre-`__atomic` implementation, including a "slow" fallback that is not atomic and a comment pointing at a function that does not exist (§8.10) |

## Addendum — rough edges in the threading code

This guide is not a bug tracker, and nothing here should be read as a list of
open issues — some of these are deliberate, some are harmless, and all of them
may be gone by the time you read this. They are collected because each one is a
place where the code does not do what the surrounding prose would lead you to
expect, which makes them worth knowing before you write against these
interfaces.

Each is present on the `24` branch at the lines cited, in rough order of how
likely it is to matter to you. Check your own tree before acting on any of
them — the point of the citations is that you can.

### 1. Two of the production deadlock-avoidance macros are not statements

```c
#define AO2_DEADLOCK_AVOIDANCE(obj) \
	ao2_unlock(obj); \
	usleep(1); \
	ao2_lock(obj);

#define CHANNEL_DEADLOCK_AVOIDANCE(chan) \
	ast_channel_unlock(chan); \
	usleep(1); \
	ast_channel_lock(chan);
```

— `include/asterisk/lock.h:476-484`

Chapter 06 §6.6 has the detail. `DEADLOCK_AVOIDANCE` itself is a `do { … }
while (0)` at `:486-495` and behaves; these two are three bare statements with
no wrapper, so `if (x) AO2_DEADLOCK_AVOIDANCE(obj);` unlocks conditionally and
then sleeps and relocks unconditionally, with no diagnostic. The tree has
exactly six call sites between the two macros — `main/stasis.c:496`,
`main/channel.c:2555`, `channels/chan_dahdi.c:8098`, `:8865`,
`addons/chan_mobile.c:1119` and `:1180` — and every one of them is already
inside a braced block, so none of them trips the hazard today. It is the
seventh that the shape of the macro is waiting for.

### 2. `DLA_UNLOCK` and `DLA_LOCK` are one macro cut in half

Chapter 06 §6.7. Under `DEBUG_THREADS` the pair at `lock.h:417-422` and
`:436-450` opens a scope in the first macro and closes it in the second, so
anything between them is inside a block; without `DEBUG_THREADS` the
definitions at `:497-499` open no scope at all. A `break` or `continue` written
between the two therefore binds to a different construct depending on the build
flags.

### 3. Unlocking an rwlock you do not hold is silent in a stock build

```c
	res = pthread_rwlock_unlock(&t->lock);

#ifdef DEBUG_THREADS
	if (res) {
		log_mutex_error(canlog, "%s line %d (%s): Error releasing rwlock: %s\n",
				filename, line, func, strerror(res));
		DO_THREAD_CRASH;
	}
#endif /* DEBUG_THREADS */
```

— `main/lock.c:853-861`

Chapter 03 §3.7. The rwlock path returns whatever `pthread_rwlock_unlock` gave
it, which on glibc is `0`. There *is* ownership tracking under `DEBUG_THREADS`
— `main/lock.c:814-845` walks `lt->thread_id[]` looking for the calling thread
and logs `rwlock '%s' freed more times than we've locked!` at `:842` — but the
test it hangs that on is `if (lock_found && --lt->reentrancy < 0)` at `:841`.
A thread unlocking an rwlock it never held never sets `lock_found`, so it
skips the decrement and the message together and gets `0` back. The one case
that most deserves a diagnostic is the one case that produces none.

### 4. `ast_cond_wait`'s ownership check falls through

Chapter 04 §4.5. The check at `main/lock.c:604-630` logs when the calling
thread does not hold the mutex it is about to wait on, and then continues into
the wait rather than returning. What it continues into is `lt->reentrancy = 0`
at `:626`. That record is per-lock, not per-thread, so a thread that was never
the owner has just zeroed the recursion depth belonging to the thread that is.
The `ast_suspend_lock_info` call at `:629` is not the culprit: it marks only
the calling thread's own slot and returns early if it does not find one
(`main/utils.c:1218-1243`, early return at `:1238`).

### 5. `ast_pthread_create_background` means nothing in a default build

```c
int ast_background_stacksize(void)
{
#if !defined(LOW_MEMORY)
	return AST_STACKSIZE;
#else
	return AST_STACKSIZE_LOW;
#endif
}
```

— `main/utils.c:1650-1657`

Chapter 07 §7.2 and §7.3. `ast_background_stacksize()` returns
`AST_STACKSIZE_LOW` only under `LOW_MEMORY`; otherwise it returns
`AST_STACKSIZE`. The non-background macros pass a literal `0`
(`include/asterisk/utils.h:626`, `:630`), and `main/utils.c:1684-1685`
substitutes `AST_STACKSIZE` for that `0` before the call is made, so the size
that reaches `pthread_attr_setstacksize` is the same either way. The
`stacksize ? stacksize : AST_STACKSIZE` ternary inside the call at `:1687` is
dead code: the substitution two lines above has already made the condition
true. Two of the four creation macros are indistinguishable from the other two
in every stock build, and the name promises a smaller stack you do not get.

### 6. The creation helpers write to the caller's attribute object

Chapter 07 §7.4. `ast_pthread_create_stack` sets inherit-sched and stack size
on whatever `pthread_attr_t` it was handed (`main/utils.c:1672-1688`), and
`ast_pthread_create_detached_stack` additionally writes the detach state
(`:1720`). A caller who configures an attribute object and reuses it does not
get the object back as they configured it, and a stack size they set is
overwritten whenever the macro passes `0`.

### 7. Four failure modes inside thread creation return success

Chapter 07 §7.11. A failed `pthread_attr_setstacksize`, a failed `ast_malloc`
of the `struct thr_arg` at `main/utils.c:1691`, a failed name allocation at
`:1695`, a failed `ast_calloc` in `ast_register_thread` at
`main/asterisk.c:424`, and `dummy_start` returning early at
`main/utils.c:1616` are all logged or stepped over, and `ast_pthread_create*`
returns `0`. The only trace of the allocation failures is
the generic `Memory Allocation Failure in function %s at line %d of %s` from
`main/astmm.c:70-71`, which names the allocation and not the consequence.

### 8. `really deep reentrancy!` is reliably followed by a false underflow

Chapter 02 §2.6. At `AST_MAX_REENTRANCY` the push at `main/lock.c:361-373`
logs at `:371` and skips the increment at `:369`, but the matching pop at
`:486-490` does not know that and decrements anyway. The damage is not
immediate: the counter is one short for each suppressed push, so the unlocks
drain it normally down to zero, and it is unlock number `AST_MAX_REENTRANCY +
1` that first takes `--lt->reentrancy` below zero and reports a `mutex '%s'
freed more times than we've locked!` at `:487` that did not happen. Because
`:489` resets the counter to `0` after each report, every remaining unlock in
the chain repeats it: *k* suppressed pushes produce *k* messages.

### 9. `pthread_create` is poisoned everywhere except Linux

```c
#ifndef __linux__
#define pthread_create __use_ast_pthread_create_instead__
#endif
```

— `include/asterisk/lock.h:648-650`

Chapter 07 §7.1. Every other name in the poison list at `lock.h:628-650` is
unconditional or guarded only for Cygwin. This one is guarded `#ifndef
__linux__`, so the guard rail that enforces the whole thread-creation
convention is switched off on the platform almost every Asterisk build targets.

### 10. `ooh323cDriver.c` redefines the background stack size for itself

```c
#undef AST_BACKGROUND_STACKSIZE
#define AST_BACKGROUND_STACKSIZE 768 * 1024
```

— `addons/ooh323cDriver.c:25-26`

Chapter 07 §7.10. The `#undef` and redefinition sit above three
`ast_pthread_create*_background` call sites (`:169`, `:201`, `:206`) that read
exactly like the other 58 in the tree, and they get 768 KiB regardless of
`LOW_MEMORY`. It and `res/res_corosync.c` are the only two files that override
the layer's stack size.

### 11. Five `strerror(errno)` sites can print `Success` after a failure

Chapter 07 §7.9. `main/utils.c:1687` and `:1720` — and `:1680` too on Linux
and FreeBSD, where the `#if` at `:1672-1682` lets it in — assign
`pthread_attr_set*` return values into `errno` as scratch space, including `0`
on the success path, so the five call sites that report a failed thread
creation with `strerror(errno)` — `main/tcptls.c:360-364` and `:912-916`,
`main/asterisk.c:1588-1590`, `channels/sig_pri.c:9131-9139`,
`res/res_aeap/aeap.c:352-355` — are reading a variable the layer wrote for its
own purposes.

### 12. A poisoned tracking slot hides a lock that is correctly held

Chapter 05 §5.6. `ast_remove_lock_info` never clears the slot it vacates. It
`memmove`s the tail of the array down over the removed entry when the entry
was not the last, then decrements `num_locks` (`main/utils.c:1308-1314`) —
and that is all. No field beyond the new `num_locks` is written, so every one
of them still describes a lock that has been released. The reuse path then
writes the next lock straight into that slot at `:1130-1143` without touching
`suspended` or `last_unlocked` — the `memset` at `:1127` runs only on the
`pending == -1` branch guarded at `:1121` — so the new entry can inherit
`suspended = 1` from the dead one, and `ast_dump_locks` skips it at `:1507`.
The symptom is a missing `core show locks` entry for a lock somebody is
holding.

### 13. No condition variable in the tree ever has its clock set

Chapter 04 §4.7. `ast_cond_init` passes the attribute pointer straight through
(`main/lock.c:540`), every caller passes `NULL`, and nothing calls
`pthread_condattr_setclock`. So every `ast_cond_timedwait` in Asterisk is on
`CLOCK_REALTIME`, and a step in the wall clock moves every timed wait in the
process.

### 14. `strerror(res)` is passed where the lock name is expected

```c
	if (res) {
		log_mutex_error(canlog, "%s line %d (%s): Error waiting on condition mutex '%s'\n",
				   filename, lineno, func, strerror(res));
		DO_THREAD_CRASH;
```

— `main/lock.c:636-639`

Chapter 04 §4.5. The format string's first `%s` is the lock name and the
argument is a system error string, so the line reports a `pthread` error in
place of the mutex it happened to. The same substitution appears at `:702`.

### 15. The `core show locks` legend lists the columns backwards

Chapter 05 §5.9. The first legend line at `main/utils.c:1493` matches the row
format at `:1380-1389` field for field. The second, at `:1494`, ends
`(attempts, times locked)` — and the row emits `times_locked` first and
`times_lock_attempted` second (`:1396-1397`). Only that trailing pair is
reversed, which is enough: the two counts that tell you whether a lock is
contended are the two you will read the wrong way round.

---

## Back to the top

[README.md](README.md) — index and the one-page mental model.
