# Anki plugin review (issue #648)

## Issue comment

[Comment](https://github.com/ilius/pyglossary/issues/648#issuecomment-4537363821) by @glowinthedark

> From previous delving into the anki format: anki cards can have variable number of fields with arbitrary names and arbitrary types: e.g. plain text, html, css, audio/video resources.
>
> Each deck has its own 'schema', i.e., a deck defines one or more card 'models', and cards in a single deck can use any of the declared models, and each model can define **variable numbers** of fields in whatever order the deck creator decided.
>
> The _only_ way this can be sanely converted is interactively, by parsing the fields for each model and letting the user decide what gets mapped into what, e.g. is this the source language or the target (ie definition/translation)? or pinyin transliteration? or a link to media? or some sort of ID, category or label? Or a comment? or a translation into a _third_ language? (no joke, I've seen decks like these too) Or is it a transliterated or alternative version of the headword?
>
> Normal dictionaries imply key-value pairs — anki does not fit into this schema, and there is absolutely no guarantee that the conversion will not be lossy, unless the deck creator used a single fixed template such as Front + Back, and most shared anki decks don't satisfy this requirement. It's wild and results are unpredictable. If you decide to take the first field as headword and then join the rest into a 'definition' you end up with an unpredictable mess. I.e. it _might_ work with some decks and fail with others.
>
> Anki models and fields are documented here:
>
> - https://github.com/ankidroid/Anki-Android/wiki/Database-Structure
>
> ### tldr summary
>
> The useful data is stored in the sqlite db in the column `notes.flds`; this column can have any number of data 'sub-fields' — it's a single string split with a magic separator character — and the data you want can be located in any position in this pseudo-array, how many fields are there for a given card and what's the name of each field is defined by the model the card is based on, i.e. the column `notes.mid` (model ID).
>
> ⚠️ The fact that all fields are packed into a **_single_** db column should give you an idea why this design had to be used: the card layout can have any number of internal fields (`flds`) which are defined by the model (i.e. 'template' referenced by `notes.mid`). The database schema only sees a single column. Where it gets interesting is that a single anki deck can have N cards using model A which for example has two fields, then X more cards using model B that has 5 fields, and then more model A cards, etc in any order.
>
> ### tldr tldr
>
> _Writing_ an anki deck, ie, creating an sqlite db and packing it to a zip (`.apkg`) is doable and relatively trivial.
>
> **Reading** an existing anki deck to a dictionary will require fuzzy logic parsing all the models ('templates'); extracting the fields in a way that would make sense for any anki deck is non-deterministic.
>
> i.e.
>
> | Format | | Extension | Read | Write |
> | ------------------------------------------------------- | :-: | :-------------: | :--: | :---: |
> | Anki2 (sqlite3) | 🔢 | .apkg | ❌ (non-deterministic!) | ✅ (doable) |

The comment is largely accurate, with two important refinements.

### Confirmed claims

- `notes.flds` is one column holding all fields joined by the magic separator
  `0x1f` (unit separator) — confirmed in `rslib/src/storage/schema11.sql` and
  the AnkiDroid wiki.
- `notes.mid` references the note type ("model"), and each model's fields
  (name + ordinal) are defined there — confirmed. Field *names* are structured
  JSON in `col.models[<mid>].flds[].name`, not fuzzy — parsing is fully
  deterministic.
- Cards per note come from model templates (`ord` in `cards` = template
  ordinal); decks/models/fields/cards are all independent, so a single
  collection mixes models freely — confirmed.
- Media: fields hold text/HTML or `[sound:...]` refs; actual files live in the
  zip. The comment's "css/audio/video" claim is correct.
- Write is doable/trivial — confirmed. A minimal schema-11 collection needs
  `col`, `notes`, `cards`, `graves`, `revlog` (+ indexes); `notes.sfld`
  (sort-field text in an integer-typed column) and `csum` (first 8 hex digits
  of SHA1 of the first field) are the only non-obvious bits.

### Refinement 1 — reading is not "fuzzy", it is a mapping problem

The comment's core point stands: which field is headword vs. definition vs.
pinyin vs. label is a per-model judgment call, so a lossless automatic
conversion is impossible in general. But field names *are* available, so the
correct design is: enumerate each model's named fields and let the user map
them — the issue's "interactive" idea, but feasible non-interactively via
options too.

### Refinement 2 — the current reader has exactly the bug the comment warns about

`reader.py:118` applies one global `word_field` index to every note. In a
mixed-model deck, notes whose model has fewer fields are silently skipped
(`reader.py:127-132`). It never reads `col.models`, so it can't use the
available field names or per-model field counts.

## Current state

- Reader exists (commit `711734ab`), only tables `notes`/`flds` used,
  `word_field` + `include_tags` options.
- Writer: none. `doc/p/anki_apkg.md` says "Write support: No".

## Plan

### Phase 1 — model-aware reader

1. Parse `col.models` JSON at `open()`; build `mid -> [field names]`.
1. In `__iter__`, look up each note's model; if `word_field` is out of range
   for that model, warn naming the model (no silent drops).
1. New option `word_field_name`: select the headword field by name, resolved
   per model (fallback: first non-empty). Index option stays for
   compatibility.
1. Emit a model/field summary as glossary info metadata.

### Phase 2 — writer (`writer.py`)

1. New `Writer`; build schema-11 SQLite (col/notes/cards/graves/revlog +
   indexes) in a temp file.
1. One standard model (Basic, 2 fields) by default; options for `field_names`,
   `model_name`, `deck_name`.
1. `notes`: ids = epoch-ms (+collision counter), random 10-char `guid`,
   `usn=-1`, padded `tags`, `\x1f`-joined `flds`, `sfld` = headword text,
   `csum` = `int(sha1(headword).hexdigest()[:8], 16)`. One `cards` row per
   note (`ord=0`, new).
1. `col` JSON: `models`/`decks`/`dconf`/`conf`/`tags`; zip as
   `collection.anki21`.
1. Register `Writer` in `__init__.py` (per `sql` plugin pattern).

### Phase 3 — tests & docs

1. Extend `tests/g_anki_apkg_test.py`: writer round-trip through the reader,
   plus structural checks (zip members, schema, csum/sfld). Test the
   mixed-model reader case (two models, different field counts).
1. Regenerate `doc/p/anki_apkg.md` via `./scripts/gen`; update
   `doc/releases/`.
1. `ruff format` + `ruff check --fix`.

## Open design question

For the mixed-model default, should the reader pick the headword via each
model's `sortf` when `word_field` is unset (nice default), or stay strictly on
the explicit index?
