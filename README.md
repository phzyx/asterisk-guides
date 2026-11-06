# Asterisk Internals Guides

Source-level guides to the inner workings of Asterisk, written for a C developer
working inside the tree.  These guides are meant only to provide general knowledge.
They can and will become obsolote over time.  I will do my best to update them
occasionally, but please keep in mind these are a side project that I built to
help my own understanding and in no way constitute anything official.

## Guides

| Guide | Covers | Target tree |
|---|---|---|
| [frames/](frames/README.md) | `struct ast_frame` end to end: anatomy, frame types, memory ownership, control frames, the channel read/write path, translation and timing, framehooks/audiohooks/bridging, debugging recipes | Asterisk `24` |
| [channels/](channels/README.md) | `struct ast_channel` end to end: the opaque struct and its accessor layer, lifecycle, references vs locks, state and flags, I/O and waiting, masquerade, channel drivers, snapshots and Stasis coupling, CLI and `REF_DEBUG` debugging | Asterisk `24` |
| [ao2/](ao2/README.md) | ao2 objects end to end: the hidden header and four allocation layouts, refcounting and destruction, the four locking modes, containers and their vtable, traversal and iterators, weak proxies, `REF_DEBUG` and `AO2_DEBUG` | Asterisk `24` |
| [threading/](threading/README.md) | The lock layer end to end: the macro-to-function wrapper indirection, `ast_mutex_t` as a struct and the renamed-field ABI device, mutexes and rwlocks in `main/lock.c`, condition variables, lock tracking and `core show locks`, deadlock avoidance and ordering, thread creation, atomics | Asterisk `24` |

See the [roadmap](ROADMAP.md) for what's planned next and why in that order.

## Conventions

Every guide targets the Asterisk `24` branch and cites source as `file:line`.
Newer releases and other branches may differ: line numbers drift, and the files
these guides cite most heavily — `main/channel.c` above all — change by hundreds
of lines between branches. If you are reading against a different checkout,
treat citations as approximate and re-grep for the symbol name. Specific commits
are deliberately not pinned; `24` is the reference point and the guides stand
against it.

Quoted code is quoted, not paraphrased: every fenced block is a whitespace-exact
excerpt of the file and lines cited beside it. The handful of blocks that are
*not* quotations — a deliberate anti-pattern, a skeleton with invented names —
are marked in the source with an `<!-- illustrative -->` comment so that the
distinction is checkable rather than a matter of trust.

## The site

These guides are published with [MkDocs Material](https://squidfunk.github.io/mkdocs-material/)
to <https://phzyx.github.io/asterisk-guides/>. Every push to `main` rebuilds via
`.github/workflows/pages.yml`.

Guides are authored in place (`frames/*.md`), not under a `docs/` directory;
`mkdocs.yml` sets `docs_dir: .` and the `same-dir` plugin is what allows that.
Nav titles are declared in `mkdocs.yml`, so adding a guide means adding its files
and one `nav:` entry.

To preview locally:

```sh
python3 -m venv .venv
./.venv/bin/pip install -r requirements-docs.txt
./.venv/bin/mkdocs serve          # http://127.0.0.1:8000
./.venv/bin/mkdocs build --strict # what CI runs; fails on a broken internal link
```

### A note on the toolchain

The MkDocs ecosystem split in 2026: MkDocs 1.x is unmaintained and closed to
outside contributions, MkDocs 2.0 is a backward-incompatible rewrite with no
plugin system, and the Material team is building a separate successor
([Zensical](https://zensical.org)) while keeping Material on critical maintenance
until **May 5, 2027**. This setup is deliberately pinned to
`mkdocs-material>=9.7.5`, which caps `mkdocs<2`, so a MkDocs 2.0 release cannot
break the build. When this needs revisiting, the two candidates are Zensical
(drop-in for MkDocs 1.x, plugin support still landing) and
[ProperDocs](https://properdocs.org) (a 1.x continuation fork). The only thing in
this repo that would need attention either way is the `same-dir` plugin.
