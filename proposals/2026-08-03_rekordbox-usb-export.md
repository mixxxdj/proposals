# rekordbox USB Export for CDJ/XDJ Hardware

* **Owners:** @sparkly-quasar
* **Implementation Status:** Partially implemented (prototype). A standalone C++ writer (openboxxx) and
  an in-Mixxx export button exist in a fork. The writer's `export.pdb` layout now matches a real
  rekordbox export, but it still lacks the history row, the ANLZ path hash and `.EXT` files (see How ▸
  Format requirements). There is one hardware result so far, from a port of the writer (see
  Verification); openboxxx's own output hasn't been tested in rekordbox or on CDJ/XDJ hardware yet.
* **Related Issues and PRs:**
  * Feature requests: mixxxdj/mixxx#9463, mixxxdj/mixxx#10321 (Device Library export),
    mixxxdj/mixxx#15556 (OneLibrary)
  * Mixxx architecture: mixxxdj/mixxx#12126 (abstract over library exporters), mixxxdj/mixxx#12873
    (earlier discussion of rekordcrate and Rust in Mixxx)
  * Mixxx importer: mixxxdj/mixxx#16895 (hidden `.PIONEER` directory), mixxxdj/mixxx#16121
    (unique `analyze_path`)
  * libdjinterop: xsco/libdjinterop#194 (rekordcrate integration), xsco/libdjinterop#199 (SQLCipher,
    merged), xsco/libdjinterop#196 (OneLibrary reading, merged; superseded xsco/libdjinterop#204),
    xsco/libdjinterop#214 (creating empty OneLibrary databases, open)
  * rekordcrate: Holzhaus/rekordcrate#259 (device export from scratch), Holzhaus/rekordcrate#264
    (ANLZ path calculation), Holzhaus/rekordcrate#269 (Device Library Plus)
  * Format findings: Deep-Symmetry/crate-digger#46
* **Other docs or links:**
  * Prototype: [openboxxx](https://github.com/sparkly-quasar/openboxxx) (standalone writer) and
    [sparkly-quasar/mixxx `feat/rekordbox-usb-export`](https://github.com/sparkly-quasar/mixxx/tree/feat/rekordbox-usb-export)
    (Mixxx integration)
  * [fragmede/rekordbox-pdb FORMAT.md](https://github.com/fragmede/rekordbox-pdb/blob/ee3bac2f22ca11a5ce61eea35f8cb951c246eaef/FORMAT.md)
    (MIT): write-side `export.pdb` invariants from byte-level diffs of real exports
  * [AnnoyingTechnology PIONEER.md](https://github.com/AnnoyingTechnology/rhythmbox-to-pioneer-xdj-exporter/blob/master/PIONEER.md):
    ANLZ path hash, history tables, hardware notes
  * [AlphaTheta USB notice](https://alphatheta.com/en/information/important-notice-for-customers-using-usb-devices-with-our-dj-equipment/):
    which players read which format
  * Related projects: pyrekordbox (MIT), rekordcrate (MPL-2.0), libdjinterop (LGPL-3.0),
    crate-digger (EPL-2.0 with MPL-2.0/LGPL-3.0 as secondary licences),
    [seabass](https://invent.kde.org/multimedia/seabass) (KDE)

---

## Changes in this revision

This revision incorporates @markst's review. Thank you for it. The main changes:

* **Scope covers both Pioneer USB formats.** OneLibrary moved from a non-goal to an explicit decision
  for maintainers, because the newest players can't read anything else.
* **The writer backend is now an open choice** between three options (openboxxx, rekordcrate, and
  libdjinterop), instead of proposing openboxxx's writer as the answer.
* **Corrected overstated claims** about table coverage, the round-trip verifier, CI, rekordbox 7
  behaviour, and the hardware test matrix.
* **Added the format requirements real players enforce**, with sources and openboxxx's current status
  against each.

**Update, October 2026.** The first hardware result came in, from Sortero, a DJ library app whose
Device Library writer is a port of openboxxx's. An XDJ-RX2 (firmware 1.43) mounted a stick it wrote
but showed none of the tracks, while a plain-folder stick from the same set played. Comparing the
stick table by table with a rekordbox export of the same kind showed the cause: the browse-menu
tables (16, 17 and 18) were empty, so the player had no categories to browse by. Every parser check
had passed. openboxxx now writes those tables and the rest of a real export's page layout, tested
page by page against a real export (see Format requirements). Whether this is enough for the XDJ-RX2
isn't confirmed yet.

> **A note from the author.** I'm a long-time Mixxx user, not a professional C/C++ developer. This
> work grew out of experimenting with AI-assisted ("vibecoding") development, and I'd like to offer it
> back to the community. I have no attachment to *my* code being the answer. If the right outcome is
> that Mixxx builds on rekordcrate or libdjinterop, that's a win. I'm mainly hoping to move
> mixxxdj/mixxx#9463 forward and to help however is most useful, including finding CDJ hardware
> testers.

## TL;DR

Mixxx can import rekordbox USB sticks but can't create them. This proposal adds an "Export to rekordbox
USB" feature, so Mixxx users can play their prepared library on Pioneer/AlphaTheta CDJ and XDJ
hardware, the way Mixxx already exports to Engine DJ.

Pioneer hardware now reads two different USB formats: the older **Device Library** (`export.pdb` plus
ANLZ analysis files) and the newer, encrypted **OneLibrary** (`exportLibrary.db`). The newest players
read only OneLibrary, older players read only Device Library, and rekordbox 7.2.11+ writes both to
every stick. A complete solution should do the same.

The work splits into two parts:

1. **The Mixxx-side export feature**, which doesn't depend on the backend: export dialog and job,
   mapping Mixxx's cues, beatgrids, keys and colours to rekordbox's model (including the MP3 timing
   correction the importer applies), and a hardware test plan. A prototype exists.
2. **The format writer**, which produces bytes a player accepts. Three candidate backends are in
   progress in different places (openboxxx, rekordcrate, libdjinterop). Choosing one is the first
   decision for maintainers.

There is also one policy question: whether Mixxx should ship OneLibrary's fixed SQLCipher passphrase
in its own code, or leave users to supply it. Without it, Mixxx can't reach the newest players. This document sets out the facts and
options but doesn't try to settle that.

## Why

**Motivation and context.** "Export to rekordbox USB" has been an open, confirmed request since 2018
(mixxxdj/mixxx#9463), re-filed more specifically in 2021 (mixxxdj/mixxx#10321), with OneLibrary
support requested in 2025 (mixxxdj/mixxx#15556). It removes the last reason many Mixxx users keep
rekordbox installed: preparing a set in Mixxx but being unable to play it on the club's CDJs.

Which players read which format, according to AlphaTheta's
[USB notice](https://alphatheta.com/en/information/important-notice-for-customers-using-usb-devices-with-our-dj-equipment/)
and [rekordbox FAQ](https://rekordbox.com/en/support/faq/trouble-shooting-7/):

| Format | Players | Encryption |
|---|---|---|
| Device Library (`export.pdb` + `USBANLZ/` ANLZ files) | CDJ-3000, XDJ-XZ, XDJ-RX3, XDJ-RR, XDJ-1000MK2, XDJ-700, CDJ-900NXS2, CDJ-2000NXS2 and older | None |
| OneLibrary, formerly Device Library Plus (`exportLibrary.db`) | CDJ-3000X, XDJ-AZ, OPUS-QUAD, OMNIS-DUO; the FAQ also directs CDJ-1500X and XDJ-AN users to OneLibrary | SQLCipher, fixed passphrase |

Groundwork that already exists in Mixxx:

* A working **Engine DJ exporter** (`Library ▸ Export Library to Engine DJ`), built on libdjinterop
  (pinned to 0.27.1), which shows the pattern for walking the library and writing a foreign database.
* An **exporter-abstraction plan** (mixxxdj/mixxx#12126) for a generic `Library ▸ Export Library…`
  with a per-format interface.
* **Read-side format definitions**: Kaitai-generated `rekordbox_pdb` / `rekordbox_anlz` parsers in
  `lib/rekordbox-metadata/`, used by the rekordbox importer.

**Pitfalls of the current solution.**

* **There is no export path.** A Mixxx user with a CDJ has to redo their preparation in rekordbox.
* **Kaitai can't generate a C++ writer.** Kaitai 0.11 added serialization for Java and Python only;
  the C++ target is still parse-only.
* **Export code is tied to Engine DJ.** The whole export chain sits behind `#ifdef __ENGINEPRIME__`:
  the menu entry, the `Library` signals, `LibraryExporter`, the QML proxy and the crate/playlist context
  menus, and `dlglibraryexport.cpp` includes djinterop directly. A second exporter needs that gating
  split, which overlaps with mixxxdj/mixxx#12126.
* **The importer's mapping can't simply be reversed.** Its helpers live in an anonymous namespace in
  `rekordboxfeature.cpp` and use Qt/Mixxx types. Keys are only read as name strings, so there is no
  key table to invert. On import, MP3 (and M4A on CoreAudio) cue and grid positions are shifted by a
  decoder-dependent offset (`mp3guessenc_timing_shift_case`), so an exporter must apply the inverse.
* **The in-tree PDB parser is limited as a checker.** `rekordbox_pdb.ksy` parses row bodies for only
  12 of the 20 table types, and still uses the `num_rows_large` heuristic that crate-digger replaced in
  [9314a0b](https://github.com/Deep-Symmetry/crate-digger/commit/9314a0b0ac30aa2e1d3796ba72d62dc4b904a902)
  ("Incorporate major PDB table row fix"). Re-vendoring it would help the importer as well.

## Goals

* Export a Mixxx library (tracks, crates and playlists, beatgrids, hot cues, memory cues and loops) to a
  USB stick that Pioneer players can browse and play.
* Cover both USB formats, so one stick works on older and newer players, subject to the OneLibrary
  decision below.
* Keep the format writer behind a backend interface, so the Mixxx-side feature doesn't depend on which
  backend is chosen, and implement (or help shape) the exporter interface from mixxxdj/mixxx#12126.
* Verify against real players, not just parsers: automated checks for the documented format invariants,
  plus a published hardware compatibility matrix.
* Make failures easy to report: an in-app "report an export problem" flow that builds a structural
  diagnostic bundle (counts, sections written, check results, hashes; never audio or full metadata) and
  opens a prefilled GitHub issue for the user to review.

**Audience.** Mixxx users who play on Pioneer/AlphaTheta hardware and keep rekordbox installed only to
export USB sticks; Mixxx maintainers who own the library and export code; and testers with access to
CDJ/XDJ units.

## Non-Goals

* **Reading rekordbox's encrypted application database (`master.db`).** Importing stays via
  `rekordbox.xml` and USB sticks.
* **Settling the legal question in this document.** The OneLibrary section sets out facts and options.
  The decision belongs to the Mixxx maintainers and, where needed, the Mixxx e.V.
* **Waveforms, My Tags, and exporting play history in the first milestone.** These are deferred. The
  history tables themselves must still be written in the form players expect (see Format requirements).
* **Writing a new format library if an existing backend is chosen.** In that case the effort goes into
  that project instead.

## How

### The Mixxx-side export feature

The prototype adds a `RekordboxExportJob` modelled on `EnginePrimeExportJob` (a `QThread` that loads
tracks, crates and playlists and reports progress), plus an export dialog. Whatever backend is chosen,
the Mixxx side needs to:

* **Split export from the Engine DJ build switch**, so rekordbox export can be built with or without
  `ENGINEPRIME`. This is the same work mixxxdj/mixxx#12126 describes, so the two should be coordinated.
  If libdjinterop becomes the rekordbox backend too, the two exporters share that dependency.
* **Own a mapping layer**: Mixxx colours to rekordbox colour IDs, a new Mixxx key to rekordbox key
  table, Mixxx's main cue to the first memory cue, hot cues and loops, the inverse of the importer's
  MP3/M4A timing shift, and `beat_number` aligned to the downbeat rather than counted from the first
  beat (the prototype currently does the latter).
* **Decide how to handle HFS+ sticks.** rekordbox writes `.PIONEER/` instead of `PIONEER/` on HFS+
  media, and Mixxx's importer checks both since mixxxdj/mixxx#16895.

### Writer backends (decision needed)

| | openboxxx | rekordcrate via C/C++ bindings | libdjinterop |
|---|---|---|---|
| Language | C++, Qt-free | Rust with C/C++ bindings | C++ (could wrap rekordcrate or a C++ writer) |
| Already a Mixxx dependency | No | No | Yes (Engine DJ export) |
| Device Library writing | Prototype with known gaps (see below) | Holzhaus/rekordcrate#259 (export from scratch) and Holzhaus/rekordcrate#264 (ANLZ paths), both open | Proposed via rekordcrate in xsco/libdjinterop#194, with a prototype adapter |
| OneLibrary | None | Holzhaus/rekordcrate#269 (open) | SQLCipher merged (xsco/libdjinterop#199); reading merged (xsco/libdjinterop#196), with the passphrase as a required argument; writing still stubbed, with creating empty databases in xsco/libdjinterop#214 (open) as the first piece |
| Main concern | Duplicates format work happening elsewhere | Packaging a Rust dependency (raised in xsco/libdjinterop#194) | Depends on progress in two upstream projects; Mixxx is pinned to 0.27.1, and SQLCipher needs libdjinterop's shared build (see below) |

In mixxxdj/mixxx#12873, Holzhaus suggested keeping C/C++ bindings for rekordcrate in the rekordcrate
repository rather than adding Rust to Mixxx, while Swiftb0y was open to Rust inside Mixxx in the longer
term. The libdjinterop route would keep a single export dependency that Mixxx already has. One packaging
constraint: statically linking SQLCipher alongside an application's own SQLite causes symbol clashes
("SQL logic error"). xsco/libdjinterop#196 hides SQLCipher's symbols in a shared libdjinterop build,
so Mixxx would need to use the shared build.

Other prior art worth reviewing: [seabass](https://invent.kde.org/multimedia/seabass) (KDE, C++/Qt),
which reads and edits existing Device Library and OneLibrary sticks; the hardware-verified Swift and
Rust writer described in Deep-Symmetry/crate-digger#46; and AnnoyingTechnology's Rhythmbox-to-XDJ
exporter.

The Mixxx-side feature works with any of these backends. The openboxxx writer is offered as one option.
Whichever backend is chosen, its tests and the requirements list below can serve as test material.

### OneLibrary encryption (decision needed)

This section is not legal advice. It sets out what is known so maintainers can decide.

**Facts.**

* OneLibrary's database is encrypted with SQLCipher using a fixed passphrase. It's the same on every
  install and isn't tied to a user or licence.
* The passphrase circulates in public tools and write-ups, but libdjinterop doesn't ship it. A
  libdjinterop maintainer asked for it to be removed after informal legal advice that including it
  could mean the library "can be understood to 'break encryption'", and it was removed before
  xsco/libdjinterop#196 merged. The merged reading code (`load_database`, `library::load`) takes the
  passphrase as a required argument. (xsco/libdjinterop#204, which also used it as a default, was
  closed without merging.)
* AlphaTheta hasn't published the format. Comments in mixxxdj/mixxx#15556 report that it has been
  shared with some other vendors and that AlphaTheta appears open to partnerships.
* Without OneLibrary, Mixxx can't export to the CDJ-3000X, XDJ-AZ, OPUS-QUAD or OMNIS-DUO.

**Considerations in favour of shipping it.**

* The purpose is interoperability: writing the user's own library in the format their own hardware
  expects. The encrypted database holds the user's own metadata, not third-party content. US law has an
  interoperability provision in this area (17 U.S.C. §1201(f)); whether it applies is a legal question.
* OneLibrary reading code, without the key, has landed in libdjinterop, which Mixxx depends on.
* rekordbox itself now writes both formats, so users will increasingly expect both.

**Considerations against.**

* US law also restricts distributing tools that get around encryption (§1201(a)(2)). A key being public
  isn't by itself a defence.
* Mixxx, or downstream packagers such as Linux distributions and app stores, could receive takedown
  requests, and may have their own policies.
* AlphaTheta could change the passphrase or format in a firmware or rekordbox update.
* It could affect a future partnership with AlphaTheta.

**Options.**

Since libdjinterop doesn't distribute the key, the choice sits entirely with Mixxx:

1. **Supply the passphrase in Mixxx's own code**, optionally behind a Mixxx build option so each
   packager can choose.
2. **Ship the OneLibrary code without the passphrase**, and let users supply it themselves (for example
   in preferences). This matches libdjinterop's approach and avoids distributing the key, at some cost
   to usability.
3. **Don't ship OneLibrary writing.** Support Device Library only, and pursue the official specification
   with AlphaTheta.

These aren't mutually exclusive. Asking AlphaTheta for the specification is worthwhile whatever is
decided.

### Format requirements for a stick players accept

Passing a parser isn't enough: parsers are lenient, and players are strict. The parsers used to check
output skip the tables a player browses with, which is how an empty browse menu passed every check
(see the update above). These are requirements documented by people who tested on hardware, with
openboxxx's current status against each.

**`export.pdb`** (sources: FORMAT.md, Deep-Symmetry/crate-digger#46, PIONEER.md):

| Requirement | openboxxx today |
|---|---|
| Each table's `first_page` is an index page (type 0x64); CDJs ignore a table whose index page is wrong | Done, as in a real export; tested against one |
| `last_page.next_page` equals the table's `empty_candidate` | Done: every table ends on a blank spare page, which the header names |
| File header field at 0x10 is 1 or 5 in real exports (Deep-Symmetry/crate-digger#46 reports it must be 5) | 5. An XDJ-RX2 rewrote a 0 there to 5 when it read a stick |
| The browse-menu tables (16 `columns`, 17, 18) hold rekordbox's rows; an XDJ-RX2 showed none of a stick's tracks without them | Done, copied from a real export; a test checks the pages byte for byte |
| History tables contain the rows players expect; empty history pages make players reject the stick | Empty tables are now an index page alone, as in a real export, but the `history` table's one row isn't written yet |

**ANLZ analysis files** (sources: PIONEER.md, Holzhaus/rekordcrate#264):

| Requirement | openboxxx today |
|---|---|
| Players compute the `P{XXX}/{XXXXXXXX}` folder from the audio file path with a published hash and ignore `analyze_path`; collisions need handling, and Mixxx's importer requires unique paths (mixxxdj/mixxx#16121) | Folder derived from the track ID |
| `ANLZ0000.EXT` is required on XDJ-XZ, and Mixxx's importer only reads beatgrids when `.EXT` sits next to `.DAT` | `.DAT` only |
| `.2EX` for CDJ-3000 extended data | Not written (phase 2) |

### Verification

1. **Automated checks.** Turn the FORMAT.md invariants into tests, and round-trip output through a
   current parser (re-vendored `rekordbox_pdb.ksy`). openboxxx's PDB round-trip check is still a stub.
2. **Diff against real rekordbox output.** rekordbox 7.2.11+ writes both formats, so a stick it produces
   is a valid reference for both. (An earlier version of this document wrongly said rekordbox 7 writes
   an empty `export.pdb`.) openboxxx's tests now compare its `export.pdb` pages with a real export
   (rekordcrate's demo export, MPL-2.0). For hardware reports, the most useful evidence turned out to
   be the stick's `PIONEER` folder after the player had read it (players write a log there) next to a
   rekordbox export of the same tracks.
3. **Import into rekordbox.** Useful, but not a reliable stand-in for hardware either way; one
   contributor in mixxxdj/mixxx#10321 reported rekordbox 5 flagging a stick as corrupted that an XDJ
   played fine.
4. **Real hardware.** Device Library: CDJ-2000NXS2, CDJ-3000, XDJ-XZ (or XDJ-1000MK2). OneLibrary:
   CDJ-3000X, XDJ-AZ, and OPUS-QUAD or OMNIS-DUO where available. Results so far: XDJ-RX2 (firmware
   1.43), from Sortero's port of the writer, showed no tracks without the browse-menu tables; a stick
   with them is waiting to be tried.

### Migration and compatibility

Nothing changes for existing users or their data. The build changes, though: export code has to be
separated from the Engine DJ switch (see above), so this is not purely additive.

### Licensing

* openboxxx is intended to be GPL-2.0-or-later but doesn't have a LICENSE file yet. Its ANLZ code is
  adapted from pyrekordbox (MIT), whose notice must be kept. Both are in the action plan.
* crate-digger is EPL-2.0 with MPL-2.0 and LGPL-3.0 as secondary licences. Mixxx already links Kaitai
  code generated from its `.ksy` files (`lib/rekordbox-metadata/`).
* rekordcrate is MPL-2.0 and libdjinterop is LGPL-3.0; both are compatible with Mixxx's licence.

### Open questions

1. Which writer backend?
2. What is Mixxx's policy on the OneLibrary passphrase?
3. If OneLibrary is deferred or declined, is a Device-Library-only exporter worth shipping on its own?
   It would reach the large installed base of older players.
4. Should rekordbox export wait for the mixxxdj/mixxx#12126 refactor, or drive it?

## Alternatives

1. **Device Library only, permanently.** Simpler and avoids the encryption question, but excludes
   current players, while rekordbox itself writes both formats.
2. **Generate a writer with Kaitai.** Not available for C++ (see Pitfalls).
3. **Maintain a hard fork of Mixxx.** Splits the community and loses upstream review; only a fallback.
4. **Export `rekordbox.xml` only.** Users would still need rekordbox to put the library on a stick, which
   is the dependency this feature is meant to remove.
5. **Wait for an official OneLibrary specification.** Device Library export doesn't depend on it, so
   there's no need to block that work.

## Action Plan

Decisions:

* [ ] Choose a writer backend.
* [ ] Decide the OneLibrary passphrase policy (maintainers, and the Mixxx e.V. if needed).
* [ ] Agree whether Device Library only is an acceptable first milestone.
* [ ] Contact AlphaTheta about the OneLibrary specification, whatever the above decisions.
      — refs mixxxdj/mixxx#15556

Mixxx side (any backend):

* [ ] Separate export code from `__ENGINEPRIME__`, coordinated with mixxxdj/mixxx#12126.
* [ ] Mapping layer: key table, MP3/M4A timing inverse, downbeat-aligned `beat_number`.
* [ ] Decide HFS+ `.PIONEER/` handling.
* [ ] Re-vendor `rekordbox_pdb.ksy` from current crate-digger (also benefits the importer).
* [ ] Run CI on the fork with pre-commit and static analysis re-enabled (currently disabled there to
      save CI minutes).

Writer:

* [x] openboxxx: index pages, `empty_candidate`, header field 0x10 and the browse-menu tables (16-18),
      tested against a real export.
* [ ] If openboxxx: the `history` table's row, the ANLZ path hash with collision handling, and `.EXT`
      files; replace the round-trip stub.
* [ ] If rekordcrate or libdjinterop: contribute there (for example Holzhaus/rekordcrate#259,
      Holzhaus/rekordcrate#264, xsco/libdjinterop#194), including bindings and packaging.

Verification:

* [ ] Automated invariant tests.
* [ ] Reference sticks from rekordbox 7.2.11+.
* [ ] Hardware matrix with tester results (first: XDJ-RX2, via Sortero). — refs mixxxdj/mixxx#9463,
      mixxxdj/mixxx#10321

Licensing:

* [ ] Add a LICENSE file to openboxxx and keep pyrekordbox's MIT notice.

Already done (prototype):

* [x] Reader for Mixxx's `mixxxdb.sqlite` into the export model, run on a real ~2,900-track library.
* [x] `RekordboxExportJob` and export dialog in a Mixxx fork, building desktop installers for macOS,
      Windows and Linux.
* [x] Prototype Device Library writer (openboxxx); not yet player-ready (see Format requirements).
