# Asterisk Internals Guides

Source-level guides to the inner workings of Asterisk, written for a C developer
working inside the tree.

These live outside any Asterisk checkout on purpose, so they can be versioned and
extended independently of the source they describe.

## Guides

| Guide | Covers | Target tree |
|---|---|---|
| [frames/](frames/README.md) | `struct ast_frame` end to end: anatomy, frame types, memory ownership, control frames, the channel read/write path, translation and timing, framehooks/audiohooks/bridging, debugging recipes | Asterisk 22.2.0 (`a0db68c190`) |

## Conventions

Every guide pins the exact tree it was written against and cites source as
`file:line`. Line numbers drift between releases — if you are reading a guide
against a different checkout, treat citations as approximate and re-grep for the
symbol name. Quoted code blocks are verbatim from the pinned tree.
