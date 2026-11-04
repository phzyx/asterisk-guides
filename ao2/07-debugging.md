# 07. Debugging

Six chapters of this guide have been about what ao2 does. This one is about how
you find out what it did.

The surface is larger than it looks, and it is the most commonly misdescribed
part of the subsystem — including inside Asterisk's own tree, where a shipped
script's docstring gets it wrong. The confusion has one root cause: there is
not one debug switch, there are **five**, they are independent, they turn on
completely different things, and two of them have names so similar that
mailing-list posts and commit messages routinely use one when they mean the
other.

So this chapter starts by separating them, then takes each one in turn: the
reference log and its five line formats, the three analysis scripts that ship
in `contrib/scripts`, the statistics and CLI commands `AO2_DEBUG` adds, the
container integrity checks and what they cost, and the compile-time
traversal-state budget chapter 05 ran into and left for here.

Along the way this chapter documents a handful of defects, because the
debugging code is less exercised than the code it instruments: two
`fclose(NULL)` paths in the init and cleanup functions, an off-by-delta in
`refcounter.py`, a substring filter in `refstats.py` that a function named
`*_destructor` walks straight through, four unguarded division sites that make
`refstats.py` traceback on a valid-but-empty log, a dead counter, and the
wrong-on-both-counts docstring mentioned above.

## 7.1 Five switches, not one

| Switch | Kind | What it actually turns on |
|---|---|---|
| `refdebug = yes` in `asterisk.conf` | **runtime**, `[options]` | Opens `$ASTLOGDIR/refs` and makes every refcount change write a line to it. This is the one you almost always want. |
| `REF_DEBUG` | compile-time, menuselect | **Only sets the default value of the runtime flag above.** Nothing else. |
| `AO2_DEBUG` | compile-time, menuselect | Five global statistics counters, five CLI commands, the container registry those commands search, container integrity checks on every link, and five extra container vtable slots. |
| `AST_DEVMODE` | compile-time, `./configure --enable-dev-mode` | Makes `ast_assert` expand to anything at all. Without it, most of ao2's internal sanity checks are no-ops. |
| `DO_CRASH` | compile-time, menuselect under dev mode | Makes a failed assertion abort the process instead of logging and continuing. Its member is in `build_tools/cflags-devmode.xml:17`, which the build only folds into the menuselect tree when `AST_DEVMODE=yes`, so it is not independently selectable. |

The first two are the pair that gets conflated, and the source says so in as
many words. `REF_DEBUG` is consumed in exactly one place in the whole tree:

```c
	/* init with buildtime config */
#ifdef REF_DEBUG
	/* The REF_DEBUG compiler flag is now only used to enable refdebug by default.
	 * Support for debugging reference counts is always compiled in. */
	ast_set2_flag64(&ast_options, 1, AST_OPT_FLAG_REF_DEBUG);
#endif
```

That comment is the entire relationship. "Support for debugging reference
counts is always compiled in" — the ref log is not a special build, it is a
config option and a restart. Chapter 06 noted this in passing when it looked at
the read-after-free on the logging path; here it is the headline. If you have a
production binary from a distribution package and you want a refcount trace,
you do not need to rebuild anything.

Both compile flags live side by side in the menuselect source, which is worth
reading for the display names alone:

```xml
		<member name="REF_DEBUG" displayname="Enable reference count debugging by default">
			<support_level>extended</support_level>
		</member>
		<member name="AO2_DEBUG" displayname="Enable internal Astobj2 debugging">
			<support_level>extended</support_level>
		</member>
```

`REF_DEBUG`'s displayname is literally "Enable reference count debugging **by
default**", which is the clearest statement of this section's point anywhere in
the tree — and the one most people never see, because neither member has a
`description` element at all. Menuselect shows the displayname and nothing more.
Both are `support_level` `extended`, i.e. neither is core-supported.
Neither is on in a default build: the `MENUSELECT_CFLAGS` line of this tree's
`menuselect.makeopts` reads `BUILD_NATIVE OPTIONAL_API DO_CRASH THREAD_CRASH
TEST_FRAMEWORK`.

`AST_DEVMODE` is the quiet one. `ast_assert` has two definitions:

```c
#ifdef AST_DEVMODE
#define ast_assert(a) _ast_assert(a, # a, __FILE__, __LINE__, __PRETTY_FUNCTION__)
```

and, fifteen lines later,

```c
#else
#define ast_assert(a)
```

An empty macro. Every `ast_assert(0)` you will read in this chapter — and there
are several in load-bearing positions — compiles to nothing in a non-dev-mode
build. That matters most in §7.9, where a function's only remaining behaviour
after the `ast_assert` vanishes is to return a success code.

It is worth being concrete about which build you are in, because the answer
flips the meaning of half of this chapter. This tree has `AST_DEVMODE=yes` in
its `makeopts` (`:110`), so every assert here is live, which is also why
`DO_CRASH` appears in its `MENUSELECT_CFLAGS`. A distribution package is the
opposite, and that is the build meant below by "a stock build".

The exception, and it is an important one, is that some of ao2's checks bypass
`ast_assert` and call `__ast_assert_failed` directly. Those fire in *every*
build. §7.5 is about which ones.

## 7.2 Opening the ref log, and two ways to `fclose(NULL)`

The log is a single file-scope `FILE *`:

```c
static FILE *ref_log;
```

Eleven places name it: the declaration, seven `if (ref_log)` guards, the
`fopen`, and two `fclose` calls. Five of the seven guards wrap the `fprintf`
blocks of §7.3 — and those five are the only `fprintf(ref_log` sites in the
entire tree — while the other two are the `lockused` writes of §7.4. The open is
in `astobj2_init`:

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

Three things to notice. The path is fixed — `refs` in the log directory, no
option to change it, which is why every script in `contrib/scripts` defaults to
`/var/log/asterisk/refs`. The mode is `"w"`, so each start truncates: there is
no history across restarts, and if you want to keep a log you copy it before
restarting. And a failed `fopen` only logs an error and falls through, leaving
`ref_log` NULL while `ast_opt_ref_debug` stays set. That last point is the one
that bites.

Immediately after the open comes the first bug:

```c
	ast_register_cleanup(astobj2_cleanup);

	if (container_init() != 0) {
		fclose(ref_log);
		ref_log = NULL;
		return -1;
	}
```

`container_init` returning non-zero takes this path, and the `fclose` is
guarded by neither `ref_log` nor `ast_opt_ref_debug`. With refdebug off — which
is the default — `ref_log` is NULL and this is `fclose(NULL)`. The impact is
limited because `container_init` can only fail when `AO2_DEBUG` is on (§7.10
shows why: with the flag off its body is empty and it unconditionally returns
0), and because an `astobj2_init` failure is fatal anyway. But it is an
unconditional NULL dereference inside glibc on a path that is reachable in
exactly the build you would be using to chase an ao2 bug.

The second one is the reverse mistake, in the cleanup handler registered one
line above:

```c
{
#if defined(AO2_DEBUG)
	ast_cli_unregister_multiple(cli_astobj2, ARRAY_LEN(cli_astobj2));
#endif

	if (ast_opt_ref_debug) {
		fclose(ref_log);
		ref_log = NULL;
	}
}
```

Here the option *is* checked and the pointer is not. So this is
`fclose(NULL)` whenever refdebug is enabled but the `fopen` at `:1175` failed —
a read-only or missing log directory, a permissions problem, an exhausted fd
table. Asterisk logs "Could not open ref debug log file", runs normally, and
then segfaults on shutdown.

Between the two of them the correct guard is `if (ref_log)`, and neither site
uses it. Worth knowing if you ever see a crash in `_IO_fclose` with ao2 on the
stack: it is not your object.

## 7.3 The five line formats

Every chapter so far has described `__ao2_ref` in terms of what it does to the
counter. What it writes is a different story: there are five logging sites
producing four distinct line shapes. All of them share a comma-separated prefix
of `addr,delta,thread_id,file,line,function` and differ in the seventh field,
the *state* — and two of the five sites use a byte-identical format string, so a
parser sees four shapes even though there are five ways to get here.

**The normal line.** This is the one you will see millions of:

```c
		if (ref_log && !(obj->priv_data.options & AO2_ALLOC_OPT_NO_REF_DEBUG)) {
			fprintf(ref_log, "%p,%s%d,%d,%s,%d,%s,%d,%s\n", user_data,
				(delta < 0 ? "" : "+"), delta, ast_get_tid(),
				file, line, func, (int)ret, tag ?: "");
			fflush(ref_log);
		}
```

Two subtleties. The `(delta < 0 ? "" : "+")` prefix exists because a negative
delta already prints its own minus sign, so the field reads `+1` or `-1` and
never `+-1`. And the state field is `(int)ret` — which is the value the atomic
returned, i.e. the count *before* this operation:

```c
	/* we modify with an atomic operation the reference counter */
	ret = ast_atomic_fetch_add(&obj->priv_data.ref_counter, delta, __ATOMIC_RELAXED);
	current_value = ret + delta;
```

`ret` is the pre-operation count, `current_value` the post-operation one, and
the log prints `ret`. So a line reading `+1,...,3` means the count went from 3
to 4, and the destruction of an object is the line that reads `-1,...,1`. Get
this backwards and every trace you read is off by one.

Note also the `AO2_ALLOC_OPT_NO_REF_DEBUG` check: an object allocated with that
option is invisible on this path. Chapter 01 covered why — the option exists for
objects allocated so often that logging them drowns everything else.

**The constructor line**, written once per object from `internal_ao2_alloc`
(`main/astobj2.c:684-766`) — the one function chapter 01 found every public
allocator forwarding to, so `__ao2_alloc` at `:768` never writes this itself:

```c
#endif

	if (ref_log && !(obj->priv_data.options & AO2_ALLOC_OPT_NO_REF_DEBUG)) {
		fprintf(ref_log, "%p,+1,%d,%s,%d,%s,**constructor**%zu**%zu**,%s\n",
			EXTERNAL_OBJ(obj), ast_get_tid(), file, line, func, overhead, data_size, tag ?: "");
		fflush(ref_log);
	}
```

The state field is `**constructor**%zu**%zu**` — the ao2 header overhead and
the caller's requested data size, in that order. This is the only line that
carries size information, which is why `refstats.py` can report memory
totals at all and why it refuses to run on a log that lacks it.

**The destructor line**, covered in §7.4, which carries lock-usage information
instead of sizes.

**The bad-pointer line.** Written when the thing you passed is not an ao2
object:

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

State `**invalid**`. Note that `delta` here is printed with plain `%d` — no `+`
prefix — so an invalid line and a normal line have subtly different formats for
the same field. Note too the `user_data` guard: a NULL pointer logs nothing at
all, it just returns -1.

**The negative-refcount line**, when the count went below zero:

```c
	if (current_value < 0) {
		ast_log(__LOG_ERROR, file, line, func,
			"Invalid refcount %d on ao2 object %p\n", (int)current_value, user_data);
		if (ref_log) {
			/* Log to ref_log even if AO2_ALLOC_OPT_NO_REF_DEBUG */
			fprintf(ref_log, "%p,%d,%d,%s,%d,%s,**invalid**,%s\n",
				user_data, delta, ast_get_tid(), file, line, func, tag ?: "");
			fflush(ref_log);
		}
		ast_assert(0);
		/* stop here even if assert doesn't DO_CRASH */
		return -1;
```

The format string here is byte-identical to the bad-pointer one at `:513`, so
the two are indistinguishable in the refs file; you tell them apart by the
`ERROR` line that accompanies this one and not that one. The comment is the
interesting part: this is the one logging site that ignores
`AO2_ALLOC_OPT_NO_REF_DEBUG`. An object that opted
out of logging still gets logged when it breaks. The `ast_assert(0)` is a no-op
outside dev mode, hence the second comment and the explicit `return -1` — the
function declines to carry on even when the assert does nothing.

That is the complete set. Anything in a refs file that is not one of these four
shapes did not come from `main/astobj2.c`.

## 7.4 `lockused`, and the one statistic ao2 collects for free

Chapter 03 §3.7 already took `lockused` apart. This section is here to connect
it to the log rather than to re-derive it. Both lock entry points set it:

```c
	if (ref_log) {
		obj->priv_data.lockused  = 1;
	}
```

That is from `__ao2_lock`; `__ao2_trylock` has the identical three lines at
`:354-356`. The guard is `ref_log`, not `ast_opt_ref_debug` and not a compile
flag — so the field is maintained exactly when the log is open, and the cost
when it is closed is one load and one branch per lock acquisition.

What it is for shows up in the destructor line:

```c
	switch (obj->priv_data.options & AO2_ALLOC_OPT_LOCK_MASK) {
	case AO2_ALLOC_OPT_LOCK_MUTEX:
		obj_mutex = INTERNAL_OBJ_MUTEX(user_data);
		lock_state = obj->priv_data.lockused ? "used" : "unused";
		ast_mutex_destroy(&obj_mutex->mutex.lock);

		ast_free(obj_mutex);
		break;
	case AO2_ALLOC_OPT_LOCK_RWLOCK:
		obj_rwlock = INTERNAL_OBJ_RWLOCK(user_data);
		lock_state = obj->priv_data.lockused ? "used" : "unused";
		ast_rwlock_destroy(&obj_rwlock->rwlock.lock);

		ast_free(obj_rwlock);
		break;
	case AO2_ALLOC_OPT_LOCK_NOLOCK:
		lock_state = "none";
		ast_free(obj);
		break;
	case AO2_ALLOC_OPT_LOCK_OBJ:
		obj_lockobj = INTERNAL_OBJ_LOCKOBJ(user_data);
		lock_state = obj->priv_data.lockused ? "used" : "unused";
		ao2_t_ref(obj_lockobj->lockobj.lock, -1, "release lockobj");

		ast_free(obj_lockobj);
		break;
	default:
		ast_log(__LOG_ERROR, file, line, func,
			"Invalid lock option on ao2 object %p\n", user_data);
		lock_state = "invalid";
		break;
	}

	if (ref_log && !(privdataoptions & AO2_ALLOC_OPT_NO_REF_DEBUG)) {
		fprintf(ref_log, "%p,%d,%d,%s,%d,%s,**destructor**lock-state:%s**,%s\n",
			user_data, delta, ast_get_tid(), file, line, func, lock_state, tag ?: "");
		fflush(ref_log);
	}
```

Three of the four lock options report `used` or `unused` from the flag, and
`NOLOCK` reports `none` because there is nothing to have used. The `default:`
arm's `"invalid"` is dead code: `AO2_ALLOC_OPT_LOCK_MASK` is `3 << 0`
(`include/asterisk/astobj2.h:369`) and all four masked values have case labels,
so chapters 01 and 03 are right that the log can never print
`lock-state:invalid`. The result is appended to the destructor's state field as
`**destructor**lock-state:%s**`.

So for every object that lives and dies while the log is open, Asterisk records
whether a lock function was ever *called* on it. Chapter 03 is careful about
that distinction and it matters here: the write at `:234-236` happens before the
switch and before any acquisition, so the bit records intent, and code that
reaches an object's mutex directly — `main/bridge_basic.c:3021` takes
`ao2_object_get_lockaddr(props)` under a `SCOPED_MUTEX` — never sets it at all.
With that caveat the field still answers a real question — "is this object
paying for a mutex it never uses?" — and `reflocks.py` exists to aggregate it. Allocating with `AO2_ALLOC_OPT_LOCK_NOLOCK` instead of the default
mutex saves the mutex, its initialisation and destruction, and on glibc a
non-trivial slice of the allocation; for an object type instantiated tens of
thousands of times, a run of `unused` in this column is a cheap win.

One trap: the state field is built from `privdataoptions`, a local copy, not
from `obj->priv_data.options`. Chapter 06 traced why — the object's memory is
freed inside the `switch` you just read, so reading the real field afterwards
would be a use-after-free. The local copy fixes the destructor frame. Chapter 06
also showed that the *caller's* frame has the same bug and no such copy.

## 7.5 The checks that fire in every build

Most of ao2's internal assertions are `ast_assert` and therefore absent outside
dev mode. Seven call sites are not: three in `main/astobj2.c` (`:212`, `:219`,
`:586`) and the four container `v_table` validity checks at
`main/astobj2_container.c:108`, `:251`, `:569` and `:765` that chapter 04
catalogued. The two below are the ones most likely to be the first sign of a bug
in *your* code rather than in ao2's.

Pointer validation is cheap and unconditional:

```c
int internal_is_ao2_object(void *user_data)
{
	struct astobj2 *p;

	if (!user_data) {
		return 0;
	}

	p = INTERNAL_OBJ(user_data);

	return !p || IS_AO2_MAGIC_BAD(p) ? 0 : 1;
}
```

A magic-number check, nothing more. When it fails, the complaint is loud:

```c
void log_bad_ao2(void *user_data, const char *file, int line, const char *func)
{
	struct astobj2 *p;
	char bad_magic[100];

	if (!user_data) {
		__ast_assert_failed(0, "user_data is NULL", file, line, func);
		return;
	}

	p = INTERNAL_OBJ(user_data);
	snprintf(bad_magic, sizeof(bad_magic), "bad magic number 0x%x for object %p",
		p->priv_data.magic, user_data);
	__ast_assert_failed(0, bad_magic, file, line, func);
}
```

`__ast_assert_failed` directly — so in a stock build, passing a non-ao2 pointer
to any ao2 function produces an `ERROR[...] FRACK!, Failed assertion bad magic
number 0x... for object 0x... (0)` in your log, the `FRACK!` wrapper coming from
`main/utils.c:2856-2859`, with the file, line and function of the *caller*. The NULL
case is distinguished and reported as "user_data is NULL". Whether the process
then dies depends on `DO_CRASH`, which is the fifth switch from §7.1 and is on
in this tree's build.

The other unconditional check is the excessive-reference tripwire:

```c
		if (EXCESSIVE_REF_COUNT <= current_value && ret < EXCESSIVE_REF_COUNT) {
			char excessive_ref_buf[100];

			/* We just reached or went over the excessive ref count trigger */
			snprintf(excessive_ref_buf, sizeof(excessive_ref_buf),
				"Excessive refcount %d reached on ao2 object %p",
				(int)current_value, user_data);
			ast_log(__LOG_ERROR, file, line, func, "%s\n", excessive_ref_buf);

			__ast_assert_failed(0, excessive_ref_buf, file, line, func);
		}
```

`EXCESSIVE_REF_COUNT` is 100000. The `ret < EXCESSIVE_REF_COUNT` half of the
condition is what makes this fire once rather than on every subsequent ref: it
tests that the *previous* count was below the threshold, so only the crossing
triggers. This is the mechanism chapter 06 found doing double duty — a weak
proxy's deliberate reference cycle is caught not by this but by a separate
`current_value == 1` test a few lines up.

Both of these together are why "I need a debug build to find ao2 problems" is
false twice over: the log needs a config option, and the two loudest checks need
nothing at all.

## 7.6 `refcounter.py`

Three scripts ship in `contrib/scripts`, not the two the chapter outline
promised: `refcounter.py` (244 lines), `refstats.py` (180) and `reflocks.py`
(118). They read the same file and answer different questions.

`refcounter.py` answers "what leaked". Its parser is one line of real work:

```python
    tokens = line.strip().split(',', 7)
    if len(tokens) < 8:
        print("ERROR: ref debug line '%s' contains fewer tokens than "
              "expected: %d" % (line.strip(), len(tokens)))
        return None

    processed_line = {'addr': tokens[0],
                      'delta': tokens[1],
                      'thread_id': tokens[2],
                      'file': tokens[3],
                      'line': tokens[4],
                      'function': tokens[5],
                      'state': tokens[6],
                      'tag': tokens[7],
                      }
    return processed_line
```

`split(',', 7)` with maxsplit 7 is deliberate: the tag is the last field and
user-supplied, so it may contain commas, and capping the split keeps it intact.
Anything with fewer than eight fields is reported and skipped.

The object-state machine is where the interesting behaviour lives:

```python
            if obj not in current_objects:
                current_objects[obj] = {'log': [], 'curcount': 1}
                if 'constructor' in parsed_line['state']:
                    # This is the normal expected case
                    pass
                elif 'invalid' in parsed_line['state']:
                    invalid = True
                    current_objects[obj]['curcount'] = 0
                    if options.invalid:
                        invalid_objects.append((obj, current_objects[obj]))
                elif 'destructor' in parsed_line['state']:
                    current_objects[obj]['curcount'] = 0
                    if options.skewed:
                        skewed_objects.append((obj, current_objects[obj]))
                else:
                    current_objects[obj]['curcount'] = int(
                        parsed_line['state'])
                    if options.skewed:
                        skewed_objects.append((obj, current_objects[obj]))
            else:
                current_objects[obj]['curcount'] += int(parsed_line['delta'])
```

Read the branches. If the first line seen for an address is a constructor, the
count starts at 1 and all is well. If it is `**invalid**`, the object is flagged
invalid and zeroed. If it is a destructor, the object is flagged *skewed* —
reasonable, since you are seeing a death with no birth, which means the log
started mid-life.

The `else` is the bug. It fires when the first line for an address is a normal
ref line, which is the common case for every object that already existed when
logging began. It seeds `curcount` from `int(parsed_line['state'])` — and §7.3
established that the state field on a normal line is the count *before* the
operation. So the seeded count omits this line's delta: an object first seen on
a `+1` line at count 3 is seeded as 3 when it is actually at 4. The `else` on
the next line, for every subsequent sighting, does apply the delta correctly.
Only the seeding is wrong, and only on the skewed path — but the consequence is
more than cosmetic. Seeding one delta *low* means `curcount` reaches `<= 0` one
unref early, so the object takes the finalisation branch below and is moved into
`finished_objects` and deleted from `current_objects` **while it is still
alive**. A genuinely leaked object can be reported as Finalized rather than
Leaked. The fix is one `+ int(parsed_line['delta'])`.

Three lines later the state field is normalised away:

```python
            if 'constructor' in parsed_line['state']:
                parsed_line['state'] = '**constructor**'
            elif 'destructor' in parsed_line['state']:
                parsed_line['state'] = '**destructor**'
```

The pre-operation count is discarded before the line is stored for display.
This is why `refcounter.py` output shows you `**constructor**` and
`**destructor**` and never the numbers that §7.3's off-by-one warning was
about — and it is also why you cannot cross-check the tool's arithmetic from its
own output.

When a count reaches zero the object is finalised, and the sub-zero case gets
its own annotation:

```python
            if current_objects[obj]['curcount'] <= 0:
                if current_objects[obj]['curcount'] < 0:
                    current_objects[obj]['log'].append(
                        "[%s] %s:%s %s: %s %s - [%s]" % (
                            parsed_line['thread_id'],
                            parsed_line['file'],
                            parsed_line['line'],
                            parsed_line['function'],
                            "+0",
                            "Object abnormally finalized",
                            "**implied destructor**"))
                    # Highlight the abnormally finished object in the
                    # invalid section as well as reporting it in the normal
                    # finished section.
                    if options.invalid:
                        invalid_objects.append((obj, current_objects[obj]))
                if not invalid and options.normal:
                    finished_objects.append((obj, current_objects[obj]))
                del current_objects[obj]
```

An object that goes negative is reported in *both* the invalid and the normal
sections, with a synthetic `**implied destructor**` entry appended. That is
intentional duplication: the object's history is what you need, and burying it
in one section only would hide it.

Finally, the exit code is a bitmask, which makes the script usable in CI:

```python
        if options.invalid and len(invalid_objects):
            print_objects(invalid_objects, "Invalid Referenced")
            ret_code |= 4

        if options.leaks and len(leaked_objects):
            print_objects(leaked_objects, "Leaked")
            ret_code |= 1

        if options.skewed and len(skewed_objects):
            print_objects(skewed_objects, "Skewed")
            ret_code |= 2

        if options.normal:
            print_objects(finished_objects, "Finalized")
```

Invalid sets bit 2 (4), leaked sets bit 0 (1), skewed sets bit 1 (2). `Finalized`
sets nothing — a clean run of normally-finished objects still exits 0 even
though it printed pages of output. That makes the exit code usable as a gate,
with one adjustment: bit 1 fires on any log that began mid-life, which is nearly
every real log, so test `$? & ~2` rather than `$? != 0`. Bit 2 is the one to
alarm on.

Two smaller notes. The default path is `/var/log/asterisk/refs`:

```python

    parser.add_option("-f", "--file", action="store", type="string",
                      dest="filepath", default="/var/log/asterisk/refs",
                      help="The full path to the refs file to process")
    parser.add_option("-i", "--suppress-invalid", action="store_false",
```

— hard-coded, and wrong on any install with a non-default `astlogdir`, so `-f`
is usually necessary. And `main` is called as `main(sys.argv)` with
`parse_args(argv)` at `:204`, passing `argv[0]` as a positional argument;
`optparse` ignores it, but it means the script's positional handling is not what
it looks like.

## 7.7 `refstats.py` and `reflocks.py`

`refstats.py` answers "what is memory going to". It needs only the constructor
and destructor lines, and filters for them like this:

```python
    with open(filename, 'r') as ref_file:
        for line in ref_file:
            if 'constructor' not in line and 'destructor' not in line:
                continue
            # The line format is:
            # addr,delta,thread_id,file,line,function,state,tag
            # Only addr, file, line, function, state are used by reflocks.py
            tokens = line.strip().split(',', 7)
            addr = tokens[0]
            state = tokens[6]
            if 'constructor' in state:
                split_state = state.split("**")
                if len(split_state) < 4:
                    print("File does not contain object size information", file=sys.stderr)
                    sys.exit(1)
```

The filter at the top tests `line`, the whole line, for the substrings
`constructor` and `destructor`. The state field is not read until `:69`. That is
a fast first pass, and it is also a correctness hole: the tree has more than a
hundred and fifty distinct functions whose names end in `_destructor`, and the
function name is field six of every line. A `-1` ref line taken inside
`ao2_reg_destructor` — which is exactly what `main/astobj2_container.c:948-955`
does — passes this filter, reaches the `if 'constructor' in state` test, fails
it, and falls through to the destructor branch, where it is read as the death of
an object that is still alive. The tool's own comment on the next line says
"Only addr, file, line, function, state are used by reflocks.py", which is true
of `reflocks.py` and slightly odd to find inside `refstats.py`.

One gate sits in between, at `:92-95`: an address with no recorded constructor
is skipped with a `continue`. So the miscount only reaches objects whose
constructor line is in the same log — and for those it is worse than one bad
count, because the `del objects[addr]` on the following line discards the entry,
so the object's *real* destructor line is later skipped by that same gate and
its lock-state datum is lost entirely.

The object-type key is built from the construction site, not the type:

```python
                obj_type = '%s:%s:%s' % (tokens[3], tokens[4], tokens[5])
                if obj_type not in object_types:
                    object_types[obj_type] = {
                        'used': 0,
                        'unused': 0,
                        'none': 0
                    }
```

`file:line:function` of the constructor. That is the right granularity in
practice — ao2 has no type information, and the allocation site is the closest
available proxy — but it means two call sites allocating the same struct are
counted separately, and a refactor that moves a line renumbers your
history.

Note the `'none': 0` key. `refstats.py` initialises it and never increments it
anywhere. It is dead, and the reason is interesting: this script detects
lock-free object types by a different route, at `:132-134`, where a type with
neither `used` nor `unused` is treated as having no locking at all. The `'none'`
key is a vestige of sharing this dictionary shape with `reflocks.py`, which does
use it.

The reporting block has a sharper problem:

```python

    print("Total usage statistics:")
    print("%20s: %d" % ("Count", total['count']))
    print("%20s: %d" % ("Total Memory (k)", total['totalmem'] / 1024))
    print("%20s: %d (%.2f%%)" % ("Overhead (k)", total['overhead'] / 1024, total['overhead'] * 100.0 / total['totalmem']))
    print("%20s: %d" % ("User Data (k)", total['user_data'] / 1024))
    print("")
    print("Peak usage statistics:")
    print("%20s: %d" % ("Count", peak['count']))
    print("%20s: %d" % ("Total Memory (k)", peak['totalmem'] / 1024))
    print("%20s: %d (%.2f%%)" % ("Overhead (k)", peak['overhead'] / 1024, peak['overhead'] * 100.0 / peak['totalmem']))
    print("%20s: %d" % ("User Data (k)", peak['user_data'] / 1024))
```

`total['overhead'] * 100.0 / total['totalmem']` divides by a value that is zero
when no constructor lines were found — and there are four such divisions, at
`:114`, `:120`, `:142` and `:147`. A refs file that is valid but has no
constructors in it is easy to produce: enable refdebug, let an idle system run
briefly, and every object was constructed before logging started. The result is
a `ZeroDivisionError` traceback, because the script's only guard is:

```python
    try:
        process_file(options)
    except (KeyboardInterrupt, SystemExit, IOError):
        print("File processing cancelled", file=sys.stderr)
        return -1
```

`KeyboardInterrupt`, `SystemExit`, `IOError` — and `ZeroDivisionError` is none
of those.

`reflocks.py` is the smallest of the three and answers the §7.4 question: which
object types pay for locks they never take. It reads the `lock-state` suffix:

```python
                if '**lock-state:unused**' in state:
                    object_types[obj_type]['unused'] += 1
                elif '**lock-state:used**' in state:
                    object_types[obj_type]['used'] += 1
                elif '**lock-state:none**' in state:
                    object_types[obj_type]['none'] += 1
```

This is the script that uses the `'none'` bucket. Its docstring, however, is
wrong twice over:

```python
 This file will process a log file created by Asterisk
 that was compiled with REF_DEBUG and DEBUG_THREADS.
```

Neither flag is required. `REF_DEBUG` is the compile-time default-setter of
§7.1, and the log it needs comes from the runtime option; and `DEBUG_THREADS`
has nothing to do with `lockused`, which §7.4 showed is maintained
unconditionally whenever `ref_log` is non-NULL. A reader following this
docstring would rebuild Asterisk twice over to get data a config edit would have
produced. It is the clearest single piece of evidence that the five switches are
genuinely confusing: this is in-tree documentation, written by someone with the
source in front of them.

There is also a stray semicolon at `:68`, visible in the second line here:

```python
    for (allocator, info) in object_types.items():
        stats = [];
```

Harmless in Python, and a small sign of how much review this corner gets.

## 7.8 What `AO2_DEBUG` adds: counters and commands

The second compile flag is the one that actually changes the build. Start with
the statistics, which live in the private header:

```c
#if defined(AO2_DEBUG)
#define AO2_DEVMODE_STAT(stat)	stat
#else
#define AO2_DEVMODE_STAT(stat)
#endif	/* defined(AO2_DEBUG) */
```

`AO2_DEVMODE_STAT` is the idiom: wrap a statistics-only expression in it and the
expression disappears in a normal build. Chapter 04 met it in the container
code without explaining it. The counters themselves:

```c
#ifdef AO2_DEBUG
struct ao2_stats {
	volatile int total_objects;
	volatile int total_mem;
	volatile int total_containers;
	volatile int total_refs;
	volatile int total_locked;
};
extern struct ao2_stats ao2;
```

Five `volatile int`s in one global struct, declared inside the `#ifdef` so that
referring to them outside an `AO2_DEBUG` build is a compile error rather than a
link error. The definition is a single line in `main/astobj2.c`:

```c
#ifdef AO2_DEBUG
struct ao2_stats ao2;
#endif
```

The `volatile` is decoration. All seventeen mutation sites go through
`ast_atomic_fetchadd_int`, so these five do not race and do not undercount — and
not one of them is touched by `AO2_DEVMODE_STAT` or by a bare `++`. That racy
idiom is applied elsewhere: to the per-container `nodes` counter
(`main/astobj2_container.c:134`, `main/astobj2_hash.c:179`,
`main/astobj2_rbtree.c:810`) and to the rbtree's per-case `stats` array, both of
which are covered by the container's write lock instead. `total_containers` is
the easiest of the five to audit: incremented once per container allocation in
each backend (`main/astobj2_hash.c:1061`, `main/astobj2_rbtree.c:2016`) and
decremented once in the shared destructor (`main/astobj2_container.c:650`), so
it is balanced.

What reads them is the first of the five CLI commands:

```c
static char *handle_astobj2_stats(struct ast_cli_entry *e, int cmd, struct ast_cli_args *a)
{
	switch (cmd) {
	case CLI_INIT:
		e->command = "astobj2 show stats";
		e->usage = "Usage: astobj2 show stats\n"
			   "       Show astobj2 show stats\n";
		return NULL;
	case CLI_GENERATE:
		return NULL;
	}
	ast_cli(a->fd, "Objects    : %d\n", ao2.total_objects);
	ast_cli(a->fd, "Containers : %d\n", ao2.total_containers);
	ast_cli(a->fd, "Memory     : %d\n", ao2.total_mem);
	ast_cli(a->fd, "Locked     : %d\n", ao2.total_locked);
	ast_cli(a->fd, "Refs       : %d\n", ao2.total_refs);
	return CLI_SUCCESS;
}
```

`astobj2 show stats`, five lines of output. And the registration array:

```c
static struct ast_cli_entry cli_astobj2[] = {
	AST_CLI_DEFINE(handle_astobj2_stats, "Print astobj2 statistics"),
	AST_CLI_DEFINE(handle_astobj2_test, "Test astobj2"),
};
#endif /* AO2_DEBUG */
```

Two commands here — `astobj2 show stats` and `astobj2 test`. The second is a
self-test that builds containers, exercises callbacks, and then does this on
purpose:

```c
	ast_verbose("now you should see an error and possible assertion failure messages:\n");
	ao2_t_ref(&i, -1, "");	/* i is not a valid object so we print an error here */
```

`&i` is the address of a stack `int`. That is a deliberate trip of §7.5's
`log_bad_ao2`, and with `DO_CRASH` compiled in — which this tree has — `astobj2
test` will abort Asterisk. The `ast_verbose` on the line above is the only
warning you get, and it says "possible assertion failure messages", which
undersells it. Do not run this command on anything you care about.

The other three commands are in `main/astobj2_container.c` and share the array
name:

```c
static struct ast_cli_entry cli_astobj2[] = {
	AST_CLI_DEFINE(handle_cli_astobj2_container_dump, "Show container contents"),
	AST_CLI_DEFINE(handle_cli_astobj2_container_stats, "Show container statistics"),
	AST_CLI_DEFINE(handle_cli_astobj2_container_check, "Perform a container integrity check"),
};
#endif	/* defined(AO2_DEBUG) */
```

Two distinct file-scope arrays, both called `cli_astobj2`, in two translation
units. Legal, since both are `static`, and confusing if you grep. `astobj2
container dump`, `astobj2 container stats` and `astobj2 container check` are
registered here; `astobj2 show stats` and `astobj2 test` are registered in
`main/astobj2.c`. Five commands, two arrays, one name.

## 7.9 Four public debug functions, two of which lie

Here is the trap that makes `AO2_DEBUG` worth understanding rather than just
enabling. The five container debug functions are declared in the *public*
header, at `include/asterisk/astobj2.h:1458`, `:1470`, `:1482`, `:1495` and
`:1503`, with no conditional compilation around the declarations. They are
defined unconditionally too. What is conditional is their **bodies**.

```c
void ao2_container_dump(struct ao2_container *self, enum search_flags flags, const char *name, void *where, ao2_prnt_fn *prnt, ao2_prnt_obj_fn *prnt_obj)
{
	if (!is_ao2_object(self) || !self->v_table) {
		prnt(where, "Invalid container\n");
		ast_assert(0);
		return;
	}

	if (!(flags & OBJ_NOLOCK)) {
		ao2_rdlock(self);
	}
	if (name) {
		prnt(where, "Container name: %s\n", name);
	}
#if defined(AO2_DEBUG)
	if (self->v_table->dump) {
		self->v_table->dump(self, where, prnt, prnt_obj);
	} else
#endif	/* defined(AO2_DEBUG) */
	{
		prnt(where, "Container dump not available.\n");
	}
	if (!(flags & OBJ_NOLOCK)) {
		ao2_unlock(self);
	}
}
```

`ao2_container_dump` is the honest one: with `AO2_DEBUG` off the `#if` removes
the vtable call and the `else` branch prints "Container dump not available." You
asked for a dump, you got a clear refusal.

```c
void ao2_container_stats(struct ao2_container *self, enum search_flags flags, const char *name, void *where, ao2_prnt_fn *prnt)
{
	if (!is_ao2_object(self) || !self->v_table) {
		prnt(where, "Invalid container\n");
		ast_assert(0);
		return;
	}

	if (!(flags & OBJ_NOLOCK)) {
		ao2_rdlock(self);
	}
	if (name) {
		prnt(where, "Container name: %s\n", name);
	}
	prnt(where, "Number of objects: %d\n", self->elements);
#if defined(AO2_DEBUG)
	prnt(where, "Number of nodes: %d\n", self->nodes);
	prnt(where, "Number of empty nodes: %d\n", self->nodes - self->elements);
	/*
	 * XXX
	 * If the max_empty_nodes count gets out of single digits you
	 * likely have a code path where ao2_iterator_destroy() is not
	 * called.
	 *
	 * Empty nodes do not harm the container but they do make
	 * container operations less efficient.
	 */
	prnt(where, "Maximum empty nodes: %d\n", self->max_empty_nodes);
	if (self->v_table->stats) {
		self->v_table->stats(self, where, prnt);
	}
#endif	/* defined(AO2_DEBUG) */
	if (!(flags & OBJ_NOLOCK)) {
		ao2_unlock(self);
	}
}
```

`ao2_container_stats` is partly honest: "Number of objects" is outside the
`#if`, so you always get the element count, and you silently do not get the node
count, the empty-node count, the maximum empty-node watermark, or the backend's
own statistics. Worth reading that `XXX` comment while you are here — it is the
best single sentence in the tree about why chapter 05's rule on calling
`ao2_iterator_destroy` matters. Empty nodes are a correctness non-issue and a
performance one, and `Maximum empty nodes` in single digits is the stated
pass mark.

And then the one that matters:

```c
int ao2_container_check(struct ao2_container *self, enum search_flags flags)
{
	int res = 0;

	if (!is_ao2_object(self) || !self->v_table) {
		/* Sanity checks. */
		ast_assert(0);
		return -1;
	}
#if defined(AO2_DEBUG)
	if (!self->v_table->integrity) {
		/* No integrity check available.  Assume container is ok. */
		return 0;
	}

	if (!(flags & OBJ_NOLOCK)) {
		ao2_rdlock(self);
	}
	res = self->v_table->integrity(self);
	if (!(flags & OBJ_NOLOCK)) {
		ao2_unlock(self);
	}
#endif	/* defined(AO2_DEBUG) */
	return res;
}
```

Trace the non-`AO2_DEBUG` path. `res` is initialised to 0. The sanity check
fails only for a non-ao2 pointer or a missing vtable. The entire `#if` block —
the lock, the integrity call, the unlock — is gone. So the function returns
**0, which the CLI command renders as "OK", without having examined the
container at all.** Not an error, not a "not available", not a -1. A clean pass.
The doxygen is weaker than the word "contract" would suggest —
`include/asterisk/astobj2.h:1479-1480` promises only "\retval 0 on success" —
but "success" with nothing checked is the same lie told more quietly.

`ao2_container_register` has the same shape:

```c
int ao2_container_register(const char *name, struct ao2_container *self, ao2_prnt_obj_fn *prnt_obj)
{
	int res = 0;
#if defined(AO2_DEBUG)
	struct ao2_reg_container *reg;

	reg = ao2_t_alloc_options(sizeof(*reg) + strlen(name), ao2_reg_destructor,
		AO2_ALLOC_OPT_LOCK_NOLOCK, "Container registration object.");
	if (!reg) {
		return -1;
	}

	/* Fill in registered entry */
	ao2_t_ref(self, +1, "Registering container.");
	reg->registered = self;
	reg->prnt_obj = prnt_obj;
	strcpy(reg->name, name);/* safe */

	if (!ao2_t_link(reg_containers, reg, "Save registration object.")) {
		res = -1;
	}

	ao2_t_ref(reg, -1, "Done registering container.");
#endif	/* defined(AO2_DEBUG) */
	return res;
}
```

`res = 0` at the top, everything else inside the `#if`, so in a normal build
this function registers nothing and reports success. Nine call sites in the tree
call it — `main/cdr.c:4737` and `:4744`, `main/stasis_state.c:358`,
`main/channelstorage_ao2_legacy.c:380`, `main/stasis_cache.c:979`,
`main/stasis.c:1950`, `main/bridge.c:5771`, and `apps/app_voicemail.c:16282` and
`:16299`. Three of them check the return value — `main/stasis_cache.c:979`,
`apps/app_voicemail.c:16282` and `:16299` — and in a normal build that check can
never fire. The other six, `main/bridge.c:5771` among them, are right to ignore
it, for the wrong reason.

The consequence for `ao2_container_check` is sharper, because it is the kind of
function you reach for in a unit test. `tests/test_astobj2.c` calls it; so could
your module. In a stock build it is an unconditional `return 0`. If you are
writing a test that asserts container integrity, it passes on every build that
does not have `AO2_DEBUG` on — which is every build anyone ships. The CLI
command has the same hole and is at least visible about it:

```c
	name = a->argv[3];
	reg = ao2_t_find(reg_containers, name, OBJ_SEARCH_KEY, "Find registered container");
	if (reg) {
		ast_cli(a->fd, "Container check of '%s': %s.\n", name,
			ao2_container_check(reg->registered, 0) ? "failed" : "OK");
		ao2_t_ref(reg, -1, "Done with registered container object.");
```

`"failed" : "OK"`. In an `AO2_DEBUG` build that is a real answer. The command
only exists in an `AO2_DEBUG` build, so this particular site is safe — but the
function it calls is public, and the next caller may not be so lucky.

One further way `ao2_container_check` can return 0 without checking, and this
one survives even under `AO2_DEBUG`: the `!self->v_table->integrity` early
return at `:866-869`, for a backend with no integrity function. Neither shipped
backend is such a backend — but §7.12 shows the vtable is quite willing to have
holes.

## 7.10 The container registry

The three container commands need a way to name a container, since ao2
containers are anonymous pointers. That is what registration is for, and the
registry is itself an ao2 container:

```c
int container_init(void)
{
#if defined(AO2_DEBUG)
	reg_containers = ao2_t_container_alloc_list(AO2_ALLOC_OPT_LOCK_RWLOCK,
		AO2_CONTAINER_ALLOC_OPT_DUPS_REPLACE, ao2_reg_sort_cb, NULL,
		"Container registration container.");
	if (!reg_containers) {
		return -1;
	}

	ast_cli_register_multiple(cli_astobj2, ARRAY_LEN(cli_astobj2));
	ast_register_cleanup(container_cleanup);
#endif	/* defined(AO2_DEBUG) */

	return 0;
}
```

A sorted list with `AO2_ALLOC_OPT_LOCK_RWLOCK` and —
note — `AO2_CONTAINER_ALLOC_OPT_DUPS_REPLACE`. Chapter 04's duplicate policies
all over again, and the choice here means **registering a second container under
an existing name silently evicts the first**. No warning, no error return; the
old registration's reference is dropped and `astobj2 container dump <name>` now
shows you a different container. The comparison callback at `:910-944` uses
`strcasecmp` and `strncasecmp`, so names are case-insensitive too: registering
`"channels"` after `"Channels"` is a collision.

With `AO2_DEBUG` off, the whole body vanishes and `container_init`
unconditionally returns 0 — which is the detail §7.2 needed to bound the first
`fclose(NULL)` bug.

Registration itself was quoted in §7.9. Two details in it are worth a second
look. `sizeof(*reg) + strlen(name)` is exactly right, not off by one, because
`struct ao2_reg_container` ends in a `name[1]` flexible-array-style member whose
one byte is the terminator. And the error handling is pure ao2 idiom: take `+1`
on `self` up front, and if `ao2_t_link` fails, drop the registration object and
let `ao2_reg_destructor` at `:948-955` release that reference on the way out.
No unwinding by hand.

Cleanup has one ordering quirk:

```c
static void container_cleanup(void)
{
	ao2_t_ref(reg_containers, -1, "Releasing container registration container");
	reg_containers = NULL;

	ast_cli_unregister_multiple(cli_astobj2, ARRAY_LEN(cli_astobj2));
}
```

`reg_containers` is released and NULLed *before* the CLI commands are
unregistered. Between those two statements, a CLI thread in `astobj2 container
dump` would call `ao2_t_find(NULL, ...)`. In practice `ast_register_cleanup`
handlers run late in a shutdown where the CLI is already gone, so this is
theoretical — but the two statements are in the wrong order, and swapping them
would cost nothing.

## 7.11 What integrity checking costs

`ao2_container_check` in an `AO2_DEBUG` build calls the backend's `integrity`
function, and those are not cheap: `hash_ao2_integrity` is
`main/astobj2_hash.c:849-998`, a hundred and fifty lines, and
`rb_ao2_integrity` is `main/astobj2_rbtree.c:1810-1974`, a hundred and
sixty-five. Both walk the entire container, verifying every node's links,
ordering and object count. Here is the hash one's local state, which tells you
the shape of the walk:

```c
static int hash_ao2_integrity(struct ao2_container_hash *self)
{
	int bucket_exp;
	int bucket;
	int count_obj;
	int count_total_obj;
	int count_total_node;
	void *obj_last;
	struct hash_bucket_node *node;
	struct hash_bucket_node *prev;
	struct hash_bucket_node *next;

	count_total_obj = 0;
	count_total_node = 0;
```

Per-bucket and total counters, plus `obj_last` for the ordering check. It is a
full traversal of every bucket and every node.

Now find where it gets called from:

```c
#if defined(AO2_DEBUG)
		if (ao2_container_check(self, OBJ_NOLOCK)) {
			ast_log(LOG_ERROR, "Container integrity failed before insert.\n");
		}
#endif	/* defined(AO2_DEBUG) */

		/* Insert the new node. */
		switch (self->v_table->insert(self, node)) {
		case AO2_CONTAINER_INSERT_NODE_INSERTED:
			node->is_linked = 1;
			ast_atomic_fetchadd_int(&self->elements, 1);
#if defined(AO2_DEBUG)
			AO2_DEVMODE_STAT(++self->nodes);
			if (self->v_table->link_stat) {
				self->v_table->link_stat(self, node);
			}
#endif	/* defined(AO2_DEBUG) */
			/* Fall through */
		case AO2_CONTAINER_INSERT_NODE_OBJ_REPLACED:
#if defined(AO2_DEBUG)
			if (ao2_container_check(self, OBJ_NOLOCK)) {
				ast_log(LOG_ERROR, "Container integrity failed after insert or replace.\n");
			}
#endif	/* defined(AO2_DEBUG) */
			res = 1;
```

Twice per *successful* `ao2_link`: once before the insert, and once on the
`NODE_OBJ_REPLACED` label, which `NODE_INSERTED` falls through to. A rejected
insert (`:148-150`) pays only the first. So in an `AO2_DEBUG` build, linking into
a container is not O(1) amortised — it is O(n), and filling a container is
**O(n²)**. For a container of a few dozen entries that is invisible. For
`channels` on a busy system it is not.

The test suite knows:

```c
/*!
 * \brief The number of iteration of testloop to be performed.
 * \note
 * In order to keep the elapsed time sane, if AO2_DEBUG is defined in menuselect,
 * only 25000 iterations are performed.   Otherwise 100000.
 */
#ifdef AO2_DEBUG
#define ITERATIONS 25000
#else
#define ITERATIONS 100000
```

A quarter of the iterations under `AO2_DEBUG`, with "in order to keep the
elapsed time sane" as the stated reason. That comment is the measured cost of
this section, written down by someone who hit it.

The unlink side is cheaper and tracks the watermark instead:

```c
	if (flags & AO2_UNLINK_NODE_DEC_COUNT) {
		ast_atomic_fetchadd_int(&container->elements, -1);
#if defined(AO2_DEBUG)
		{
			int empty = container->nodes - container->elements;

			if (container->max_empty_nodes < empty) {
				container->max_empty_nodes = empty;
			}
			if (container->v_table->unlink_stat) {
				container->v_table->unlink_stat(container, node);
			}
		}
#endif	/* defined(AO2_DEBUG) */
	}
```

No integrity call *here* — just the empty-node bookkeeping §7.9's `XXX` comment
was about, plus the vtable statistics hook. The removal still pays, though,
because an unlink drops the node's last reference and the node *destructors* run
integrity walks of their own: one before deletion in the hash backend
(`main/astobj2_hash.c:171-176`) and both before and after in the rbtree
(`main/astobj2_rbtree.c:855-867`), each skipped only once the container is
already `destroying`. Draining a container is O(n²) too. Which brings us to the
one piece of `AO2_DEBUG` that is not symmetric between the backends.

## 7.12 The vtable asymmetry

`struct ao2_container_methods` has eight base members and five more under
`AO2_DEBUG`: `link_stat`, `unlink_stat`, `dump`, `stats` and `integrity`. The
hash backend fills all five:

```c
static const struct ao2_container_methods v_table_hash = {
	.alloc_empty_clone = (ao2_container_alloc_empty_clone_fn) hash_ao2_alloc_empty_clone,
	.new_node = (ao2_container_new_node_fn) hash_ao2_new_node,
	.insert = (ao2_container_insert_fn) hash_ao2_insert_node,
	.traverse_first = (ao2_container_find_first_fn) hash_ao2_find_first,
	.traverse_next = (ao2_container_find_next_fn) hash_ao2_find_next,
	.iterator_next = (ao2_iterator_next_fn) hash_ao2_iterator_next,
	.destroy = (ao2_container_destroy_fn) hash_ao2_destroy,
#if defined(AO2_DEBUG)
	.link_stat = hash_ao2_link_node_stat,
	.unlink_stat = hash_ao2_unlink_node_stat,
	.dump = (ao2_container_display) hash_ao2_dump,
	.stats = (ao2_container_statistics) hash_ao2_stats,
	.integrity = (ao2_container_integrity) hash_ao2_integrity,
#endif	/* defined(AO2_DEBUG) */
};
```

The red-black tree backend fills three:

```c
static const struct ao2_container_methods v_table_rbtree = {
	.alloc_empty_clone = (ao2_container_alloc_empty_clone_fn) rb_ao2_alloc_empty_clone,
	.new_node = (ao2_container_new_node_fn) rb_ao2_new_node,
	.insert = (ao2_container_insert_fn) rb_ao2_insert_node,
	.traverse_first = (ao2_container_find_first_fn) rb_ao2_find_first,
	.traverse_next = (ao2_container_find_next_fn) rb_ao2_find_next,
	.iterator_next = (ao2_iterator_next_fn) rb_ao2_iterator_next,
	.destroy = (ao2_container_destroy_fn) rb_ao2_destroy,
#if defined(AO2_DEBUG)
	.dump = (ao2_container_display) rb_ao2_dump,
	.stats = (ao2_container_statistics) rb_ao2_stats,
	.integrity = (ao2_container_integrity) rb_ao2_integrity,
#endif	/* defined(AO2_DEBUG) */
};
```

`link_stat` and `unlink_stat` are missing. Both call sites are NULL-guarded —
you have already read the unlink one at `main/astobj2_container.c:64-66`, and the
link one is at `:135-137` in the block quoted in §7.11 — so this is a
deliberate asymmetry rather than a latent crash. The reason is visible in what
the rbtree backend collects instead:

```c
#if defined(AO2_DEBUG)
	struct {
		/*! Fixup insert left cases 1-3 */
		int fixup_insert_left[3];
		/*! Fixup insert right cases 1-3 */
		int fixup_insert_right[3];
		/*! Fixup delete left cases 1-4 */
		int fixup_delete_left[4];
		/*! Fixup delete right cases 1-4 */
		int fixup_delete_right[4];
		/*! Deletion of node with number of children (0-2). */
		int delete_children[3];
	} stats;
#endif	/* defined(AO2_DEBUG) */
```

Per-case counters for the red-black rebalancing: three insert-fixup cases each
way, four delete-fixup cases each way, and a histogram of how many children the
deleted node had. These are maintained inside the tree algorithms themselves,
where the cases are distinguishable, not at the generic link/unlink boundary
where the hash backend counts bucket depth. Two backends, two things worth
counting, one vtable that accommodates both by letting either half be NULL.

Chapters 04 and 05 found the same pattern in the base half of the vtable:
`traverse_cleanup` is filled by neither backend, and its call site is guarded
the same way. The vtable is a menu, not a contract.

## 7.13 The traversal-state budget

Chapter 04 §4.10 took this apart and chapter 05 summarised it. This section is
here to sit it beside the other compile-time machinery, and to add the one thing
neither chapter said. The number:

```c
/*! Allow enough room for container specific traversal state structs */
#define AO2_TRAVERSAL_STATE_SIZE	100
```

One hundred bytes, used as an `alloca` size in the generic traversal function at
`main/astobj2_container.c:315`. The generic code does not know which backend it
is traversing, so it reserves a fixed buffer and lets the backend's
`traverse_first` interpret it as whatever state struct that backend needs. The
obvious hazard is a backend whose state outgrows the budget, silently
overflowing a stack buffer on every container operation.

The defence is a trick:

```c
struct hash_traversal_state_check {
	/*
	 * If we have a division by zero compile error here then there
	 * is not enough room for the state.  Increase AO2_TRAVERSAL_STATE_SIZE.
	 */
	char check[1 / (AO2_TRAVERSAL_STATE_SIZE / sizeof(struct hash_traversal_state))];
};
```

and the rbtree backend has the same construct at `:120-126`. An array whose
size is `1 / (100 / sizeof(state))`. If the state fits in 100 bytes, the inner
division is at least 1 and the outer division is defined; if it does not, the
inner division is 0 and the whole thing is a division by zero in a constant
expression, which is a compile error. The comment says exactly that. Neither
`*_check` struct is ever instantiated — their existence is the entire point.

Chapter 04 measured the two state structs: `struct hash_traversal_state`
(`main/astobj2_hash.c:79-92`) is **32 bytes** and `struct rbtree_traversal_state`
(`main/astobj2_rbtree.c:111-118`) is **24**. So `100 / 32` is 3 and `100 / 24` is
4, and `1 / 3` and `1 / 4` are both **0**. The array is `char check[0]`.

A zero-length array is not standard C — it is a GNU extension, which Asterisk
relies on elsewhere and which GCC and Clang both accept without complaint at
Asterisk's warning levels. So the check works, but not quite the way the comment
implies: it does not produce an array of size 1 in the good case and an error in
the bad one. It produces an array of size 0 in the good case, and only becomes a
hard error when the state actually exceeds 100 bytes.

Here is the part neither earlier chapter drew out. Because integer division
truncates twice, the array's size measures nothing. At 32 bytes the bound is
`1 / 3` = 0; at 51 bytes it is `1 / 1` = 1; at 99 bytes it is still `1 / 1` = 1.
So `check` is `char check[0]` for a small state and `char check[1]` for a large
one, with no values in between, and the only transition the trap detects is
crossing 100 itself. That is the right behaviour for what the macro guards — but
it means you cannot read remaining headroom off the construct, and a backend
whose state grew from 32 to 99 bytes would sail through while tripling its
`alloca`.

Current headroom: 68 bytes for the hash backend, 76 for the rbtree. Both have
room for several more pointers. If you add a field to either state struct and
the build fails with "division by zero" in a struct declaration, this section is
why, and the fix is to raise `AO2_TRAVERSAL_STATE_SIZE` — not to shrink your
state.

## 7.14 In practice

Starting from "something is leaking", the sequence is:

Set `refdebug = yes` in the `[options]` section of `asterisk.conf` and restart.
Do not rebuild anything. Confirm `$ASTLOGDIR/refs` exists and is growing. Note
that the restart truncated any previous one.

Reproduce, then stop, then copy `refs` somewhere safe before the next restart
destroys it.

Run `contrib/scripts/refcounter.py -f /path/to/refs` and read the exit code.
Bit 0 means leaks, bit 1 means skewed objects whose history began before logging
did, bit 2 means genuinely invalid refcounts or bad pointers. Treat bit 2 as the
priority: that is a bug now, not a leak later.

If the Leaked section is large, `refstats.py -f refs` will tell you which
allocation sites account for the memory, and `reflocks.py -f refs` will tell you
which object types are carrying unused locks. Both want constructor lines, so
both are most useful on a log that spans a full object lifetime.

Only reach for `AO2_DEBUG` when the question is about a *container* rather than
an object — "is this container corrupt", "how deep are the buckets", "how many
empty nodes has a leaked iterator left behind". Budget for the O(n²) link cost
before you put that build under load.

| Rule | Where it comes from |
|---|---|
| The ref log is a runtime option, not a build flag | `main/options.c:325-326` parses it, `main/astobj2.c:1169-1179` acts on it |
| `REF_DEBUG` only changes the default of the runtime flag | `main/options.c:230-232`, `build_tools/cflags.xml:15-17` |
| The state field on a normal ref line is the count *before* the operation | `main/astobj2.c:530-531`, `:592` |
| Copy `refs` before restarting; the open mode is `"w"` | `main/astobj2.c:1175` |
| Expect five logging sites but only four line shapes; only the constructor line carries sizes | `main/astobj2.c:756-762`, `:589-593`, `:661-664`, and the identical pair `:512-515` / `:602-606` |
| An object with `AO2_ALLOC_OPT_NO_REF_DEBUG` is invisible except when its count goes negative | `main/astobj2.c:589`, `:602-604` |
| `lockused` is maintained whenever the log is open, with no extra flag — and records a lock *call*, not an acquisition | `main/astobj2.c:234-236`, `:354-356`; chapter 03 §3.7 |
| The destructor line can never print `lock-state:invalid` | `include/asterisk/astobj2.h:369`, `main/astobj2.c:628-659` |
| A bad ao2 pointer is reported in every build, dev-mode or not | `main/astobj2.c:206-220` |
| Seven ao2 assertion sites bypass `ast_assert` and fire in every build | `main/astobj2.c:212`, `:219`, `:586`; `main/astobj2_container.c:108`, `:251`, `:569`, `:765` |
| Crossing 100000 references is reported once, in every build | `main/astobj2.c:577-587` |
| `ast_assert` is a no-op outside a dev-mode build | `include/asterisk/utils.h:763-764`, `:778-779` |
| In a stock build `ao2_container_check` returns "OK" without checking | `main/astobj2_container.c:856-880` |
| In a stock build `ao2_container_register` returns success without registering | `main/astobj2_container.c:958-983` |
| Registering a duplicate container name silently replaces, case-insensitively | `main/astobj2_container.c:1184-1186`, `:910-944` |
| Under `AO2_DEBUG`, every successful `ao2_link` walks the whole container twice, and every unlink's node destructor walks it once more (hash) or twice (rbtree) | `main/astobj2_container.c:122-146`, `main/astobj2_hash.c:171-176`, `main/astobj2_rbtree.c:855-867`, `tests/test_astobj2.c:2019-2022` |
| Never run `astobj2 test` on a system you care about | `main/astobj2.c:1140` |
| `reflocks.py`'s docstring is wrong about both flags it names | `contrib/scripts/reflocks.py:5` vs `main/options.c:230`, `main/astobj2.c:234` |
| `refstats.py` tracebacks on a valid log with no constructor lines | `contrib/scripts/refstats.py:114`, `:120`, `:142`, `:147`, `:172` |
| `refstats.py` can miscount a ref taken inside a `*_destructor` function | `contrib/scripts/refstats.py:62` |
| `refcounter.py` seeds a skewed object's count one delta low | `contrib/scripts/refcounter.py:104-105` vs `main/astobj2.c:530` |
| `fclose(ref_log)` is unguarded on two paths | `main/astobj2.c:1184`, `:1163-1164` |
| Raise `AO2_TRAVERSAL_STATE_SIZE` if a state struct outgrows 100 bytes; the check is a floor, not a fit | `main/astobj2_container_private.h:60`, `main/astobj2_hash.c:94-100` |

## Next

That is the ao2 guide. Seven chapters: objects and allocation, references and
locking, destructors and lifetime, containers, traversal and iterators, weak
proxies and global objects, and debugging. The recurring theme has been that ao2
is a small amount of code with a large amount of implied contract, most of it
documented only by the single call site that depends on it — and that the
header and the implementation disagree often enough that the implementation is
the only thing worth trusting.
