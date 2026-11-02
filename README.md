# Asterisk Internals Guides

Source-level guides to the inner workings of Asterisk, written for a C developer
working inside the tree.

These live outside any Asterisk checkout on purpose, so they can be versioned and
extended independently of the source they describe.

## Guides

| Guide | Covers | Target tree |
|---|---|---|
| [frames/](frames/README.md) | `struct ast_frame` end to end: anatomy, frame types, memory ownership, control frames, the channel read/write path, translation and timing, framehooks/audiohooks/bridging, debugging recipes | Asterisk 22.2.0 (`a0db68c190`) |

See the [roadmap](ROADMAP.md) for what's planned next and why in that order.

## Conventions

Every guide pins the exact tree it was written against and cites source as
`file:line`. Line numbers drift between releases — if you are reading a guide
against a different checkout, treat citations as approximate and re-grep for the
symbol name. Quoted code blocks are verbatim from the pinned tree.

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
