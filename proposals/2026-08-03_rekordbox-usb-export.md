# rekordbox USB Export for CDJ/XDJ Hardware

* **Owners:** @sparkly-quasar
* **Implementation Status:** Partially implemented (prototype). A standalone C++ writer (openboxxx) and
  an in-Mixxx export button exist in a fork, but the writer is still missing structures that players
  require (see How ▸ Format requirements), and nothing has been tested in rekordbox or on CDJ/XDJ
  hardware yet.
* **Related Issues and PRs:**
  * Feature requests: mixxxdj/mixxx#9463, mixxxdj/mixxx#10321 (Device Library export),
    mixxxdj/mixxx#15556 (OneLibrary)
  * Mixxx architecture: mixxxdj/mixxx#12126 (abstract over library exporters), mixxxdj/mixxx#12873
    (earlier discussion of rekordcrate and Rust in Mixxx)
  * Mixxx importer: mixxxdj/mixxx#16895 (hidden `.PIONEER` directory), mixxxdj/mixxx#16121
    (unique `analyze_path`)
  * libdjinterop: xsco/libdjinterop#194 (rekordcrate integration), xsco/libdjinterop#199 (SQLCipher,
    merged), xsco/libdjinterop#196 and xsco/libdjinterop#204 (OneLibrary reading)
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

There is also one policy question: whether Mixxx should ship code that uses OneLibrary's fixed SQLCipher
passphrase. Without it, Mixxx can't reach the newest players. This document sets out the facts and
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
| OneLibrary | None | Holzhaus/rekordcrate#269 (open) | SQLCipher merged (xsco/libdjinterop#199); reading and schema in xsco/libdjinterop#196 and xsco/libdjinterop#204 (open); no writing yet |
| Main concern | Duplicates format work happening elsewhere | Packaging a Rust dependency (raised in xsco/libdjinterop#194) | Depends on progress in two upstream projects |

In mixxxdj/mixxx#12873, Holzhaus suggested keeping C/C++ bindings for rekordcrate in the rekordcrate
repository rather than adding Rust to Mixxx, while Swiftb0y was open to Rust inside Mixxx in the longer
term. The libdjinterop route would keep a single export dependency that Mixxx already has.

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
* The passphrase is already distributed in open-source code, including libdjinterop's open OneLibrary
  pull requests (xsco/libdjinterop#196 and xsco/libdjinterop#204), which use it as the default.
* AlphaTheta hasn't published the format. Comments in mixxxdj/mixxx#15556 report that it has been
  shared with some other vendors and that AlphaTheta appears open to partnerships.
* Without OneLibrary, Mixxx can't export to the CDJ-3000X, XDJ-AZ, OPUS-QUAD or OMNIS-DUO.

**Considerations in favour of shipping it.**

* The purpose is interoperability: writing the user's own library in the format their own hardware
  expects. The encrypted database holds the user's own metadata, not third-party content. US law has an
  interoperability provision in this area (17 U.S.C. §1201(f)); whether it applies is a legal question.
* The code is already landing in libdjinterop, which Mixxx depends on.
* rekordbox itself now writes both formats, so users will increasingly expect both.

**Considerations against.**

* US law also restricts distributing tools that get around encryption (§1201(a)(2)). A key being public
  isn't by itself a defence.
* Mixxx, or downstream packagers such as Linux distributions and app stores, could receive takedown
  requests, and may have their own policies.
* AlphaTheta could change the passphrase or format in a firmware or rekordbox update.
* It could affect a future partnership with AlphaTheta.

**Options.**

1. **Ship OneLibrary support behind a build option**, so Mixxx and each packager can choose. The review
   suggested asking libdjinterop for such an option.
2. **Don't ship OneLibrary writing.** Support Device Library only, and pursue the official specification
   with AlphaTheta.
3. **Ship the OneLibrary code without the passphrase**, and let users supply it themselves (for example
   in preferences). This avoids distributing the key, at some cost to usability.

These aren't mutually exclusive. Asking AlphaTheta for the specification is worthwhile whatever is
decided.

### Format requirements for a stick players accept

Passing a parser isn't enough: parsers are lenient, and players are strict. These are requirements
documented by people who tested on hardware, with openboxxx's current status against each.

**`export.pdb`** (sources: FORMAT.md, Deep-Symmetry/crate-digger#46, PIONEER.md):

| Requirement | openboxxx today |
|---|---|
| Each table's `first_page` is an index page (type 0x64); CDJs ignore a table whose index page is wrong | No index pages |
| `last_page.next_page` equals the table's `empty_candidate` | Both 0 (TODO) |
| File header field at 0x10 is 1 or 5 in real exports (Deep-Symmetry/crate-digger#46 reports it must be 5) | 0 |
| History tables and tables 17/18 contain the rows players expect; empty history pages make players reject the stick | 12 of 20 tables are a single empty page |

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
   an empty `export.pdb`.)
3. **Import into rekordbox.** Useful, but not a reliable stand-in for hardware either way; one
   contributor in mixxxdj/mixxx#10321 reported rekordbox 5 flagging a stick as corrupted that an XDJ
   played fine.
4. **Real hardware.** Device Library: CDJ-2000NXS2, CDJ-3000, XDJ-XZ (or XDJ-1000MK2). OneLibrary:
   CDJ-3000X, XDJ-AZ, and OPUS-QUAD or OMNIS-DUO where available.

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

* [ ] If openboxxx: add index pages, `empty_candidate`, header field 0x10, populated history and 17/18
      tables, the ANLZ path hash with collision handling, and `.EXT` files; replace the round-trip stub.
* [ ] If rekordcrate or libdjinterop: contribute there (for example Holzhaus/rekordcrate#259,
      Holzhaus/rekordcrate#264, xsco/libdjinterop#194), including bindings and packaging.

Verification:

* [ ] Automated invariant tests.
* [ ] Reference sticks from rekordbox 7.2.11+.
* [ ] Hardware matrix with tester results. — refs mixxxdj/mixxx#9463, mixxxdj/mixxx#10321

Licensing:

* [ ] Add a LICENSE file to openboxxx and keep pyrekordbox's MIT notice.

Already done (prototype):

* [x] Reader for Mixxx's `mixxxdb.sqlite` into the export model, run on a real ~2,900-track library.
* [x] `RekordboxExportJob` and export dialog in a Mixxx fork, building desktop installers for macOS,
      Windows and Linux.
* [x] Prototype Device Library writer (openboxxx); not yet player-ready (see Format requirements).
