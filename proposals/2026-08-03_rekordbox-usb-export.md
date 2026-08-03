# rekordbox USB Export (PDB/ANLZ) for CDJ/XDJ hardware

* **Owners:** @sparkly-quasar (openboxxx)
* **Implementation Status:** Partially implemented. Standalone writer proven and verified against
  independent parsers; in-Mixxx exporter working in a fork that compiles + links on macOS
  (Intel + Apple Silicon), Windows, and Linux via CI. Pending rekordbox-desktop import
  (verification tier 3) and physical CDJ/XDJ playback (tier 4).
* **Related Issues and PRs:**
  * mixxxdj/mixxx#9463 — original "export crates/cues/beatgrids to rekordbox USB" request
  * mixxxdj/mixxx#10321 — "Rekordbox USB Export (PDB/ANLZ)"
  * mixxxdj/mixxx#12126 — "Abstract over library exporters" (the architectural hook)
  * mixxxdj/mixxx#15556 — "Add OneLibrary support" (explicitly *out of scope* here; see Non-Goals)
  * mixxxdj/mixxx#13293 — updated rekordbox Kaitai definitions (import-side)
* **Other docs or links:**
  * Mixxx fork with the working exporter:
    [sparkly-quasar/mixxx `feat/rekordbox-usb-export`](https://github.com/sparkly-quasar/mixxx/tree/feat/rekordbox-usb-export)
  * Downloadable tri-platform CI builds:
    [openboxxx Releases](https://github.com/sparkly-quasar/openboxxx/releases)
  * openboxxx design docs + component status table: `libopenboxxx-export/README.md`; plus
    architecture, export-design, legal, research-findings (linked inline)
  * pyrekordbox (Dylan Jones, MIT) — ANLZ/PDB reference implementation
  * Deep Symmetry `crate-digger` / `dysentery` `.ksy` specs (format documentation)
  * rekordcrate (Jan Holthuis, MPL-2.0) — Rust PDB parser/serializer

---

> **A note from the author.** I'm a long-time Mixxx user, not a professional C/C++ developer. This
> exporter grew out of experimenting with AI-assisted ("vibecoding") development, and getting a real
> stick to round-trip through the parsers felt like a genuinely worthwhile use of that time — so I'd
> like to offer it back to the community as a gift. I have no attachment to *my* code being the
> answer; if the right outcome is that this informs a rekordcrate-based path (or anything else the
> maintainers prefer), that's a win. I'm mainly hoping to move #9463 forward and to help however is
> most useful, including finding CDJ hardware testers.

## TL;DR

Mixxx can *import* rekordbox libraries but cannot *export* to the USB format that
Pioneer/AlphaTheta CDJ and XDJ hardware reads. This proposal adds a **rekordbox USB
exporter**: a writer for the unencrypted `export.pdb` (DeviceSQL) and per-track `ANLZ`
(`.DAT`/`.EXT`) analysis files, plus the `PIONEER/` USB directory layout and audio-file
copy — the same kind of "export my whole library to foreign DJ hardware" flow Mixxx
already ships for Engine DJ / Engine Prime.

The core byte-writer is built as a **standalone, Qt-free C++ library** (`libopenboxxx-export`)
so the hard, fiddly serialization can be tested in isolation, with Mixxx integration as a thin
adapter modeled on the existing `EnginePrimeExportJob`. A **round-trip verifier** (parse our
own output back with Mixxx's in-tree Kaitai parsers and diff it) is a first-class feature, so a
DJ learns whether a stick is good *before* they leave for the gig.

Scope is deliberately narrowed to the **legally clean, widest-reach** target: the classic
unencrypted PDB/ANLZ format. The newer encrypted OneLibrary / Device Library Plus format
(#15556) is explicitly a non-goal here for legal reasons (see Non-Goals).

**This is not a paper design.** The standalone writer emits a full `export.pdb` (all 20 standard
tables + the 8-colour palette) and per-track ANLZ, every output cross-checked against two
independent real-format parsers and diffed field-for-field against genuine rekordbox files. An
in-Mixxx "Export Library to rekordbox USB" button already exists in a fork that compiles and links
on macOS (Intel + Apple Silicon), Windows, and Linux via CI — downloadable builds are on the
[Releases](https://github.com/sparkly-quasar/openboxxx/releases) page. What remains is the real
proof: importing into rekordbox desktop and playing on physical CDJ/XDJ hardware — testers welcome.

## Why

**Motivation and context.** "Export to rekordbox USB" has been an open, confirmed wishlist item
since 2018 (#9463), re-filed more specifically in 2021 (#10321), with no assignee, no branch, and
no merged PR. It is one of the most-requested missing features because it removes the last reason a
Mixxx user is forced back into rekordbox: preparing a set in open software but being unable to play
it on the club's CDJs. Mixxx already reads rekordbox libraries; the export direction is the missing
half.

Much of the groundwork already exists in-tree, which makes the gap bounded rather than green-field:

* A **working Engine DJ / Engine Prime exporter** (`Library ▸ Export Library to Engine Prime`)
  establishes the exact pattern for walking the Mixxx library and writing a foreign DJ database.
* An **exporter-abstraction plan (#12126)** already intends to replace the Engine-specific menu item
  with a generic `Library ▸ Export Library…` and a common per-format interface — the natural socket
  for a rekordbox exporter.
* **rekordbox format definitions are already in-tree**: the importer uses Kaitai-generated
  `rekordbox_pdb` / `rekordbox_anlz` parsers (`lib/rekordbox-metadata/`, kept current by #13293), so
  the *read* side of PDB/ANLZ is already described.
* A **mature importer (since v2.3)** already works out the Mixxx↔rekordbox field mapping (hotcues,
  memory cues, loops, colors, keys) in reverse; the exporter reuses that mapping the other direction.

**Pitfalls of the current solution.**

* **There is no export path at all** — a Mixxx user with a CDJ has to redo their prep in rekordbox.
* **Kaitai Struct can only READ.** Its C++ target generates parsers, not serializers, so Mixxx can
  parse PDB/ANLZ today but has no generated code to *write* them. The single biggest chunk of
  net-new code — a hand-written PDB/ANLZ serializer — simply does not exist yet in the project.
* **No abstraction to plug into yet.** The exporter interface envisioned in #12126 hasn't been built,
  so today a new exporter would have to bolt onto the Engine-specific plumbing.

## Goals

* Add a **rekordbox USB exporter** that produces a stick a real CDJ/XDJ can browse and play from:
  library, playlists, per-track beatgrids, and hot/memory cues.
* Deliver the serializer as a **standalone, dependency-light C++ library** with its own tests, so the
  risky byte-layout work is verifiable independently of a Mixxx build and the eventual Mixxx PR is
  mostly adapter + dialog + CMake option.
* Ship a **round-trip verifier** as a first-class trust feature: parse our own USB back with the
  in-tree Kaitai parsers and report fidelity ("playlists 12/12, hotcues OK, beatgrids OK") before the
  DJ relies on it.
* Land as an **upstream contribution**, implementing (and helping shape) the generic exporter
  interface of #12126 rather than a one-off menu item — and, ideally, retiring #9463 / #10321.
* Provide an **in-app "report an export bug" flow** that opens a prefilled GitHub issue from a
  structural diagnostic manifest (counts, sections written, verifier results, hashes — never the
  user's audio or full metadata), so every beta tester becomes a hardware-compatibility data source.

**Audience.** Mixxx users who play on Pioneer/AlphaTheta CDJ/XDJ hardware and currently keep
rekordbox around solely to export USB sticks; Mixxx maintainers who own the library-export subsystem;
and beta testers with access to real CDJ/XDJ units.

## Non-Goals

* **OneLibrary / Device Library Plus (`exportLibrary.db`, #15556) is out of scope — for legal, not
  just effort, reasons.** That database is SQLCipher-encrypted (256-bit AES); writing it means
  circumventing an access-control measure with an extracted key, which is precisely the DMCA §1201
  exposure the project is structured to avoid. It should be pursued **only** via an official
  AlphaTheta spec/partnership, never with a recovered key. This proposal writes **only** the
  unencrypted classic PDB/ANLZ format. (See openboxxx `docs/legal.md`.)
* **Reading the encrypted rekordbox 6/7 app database (`master.db`).** The sanctioned import path is
  the user's own `rekordbox.xml`; we never decrypt their app DB.
* **Full analysis fidelity in the first milestone.** Waveforms (`PWAV`/`PWV3`/`PWV4`/`PWV5`), extended
  `.EXT` cues with colors + labels (`PCO2`), `.2EX`, `PVBR`, My-Tags (`exportExt.pdb`), and history
  playlists are deferred to a follow-up phase (they are additive ANLZ sections and don't touch the
  PDB). The first milestone is a *playable* stick with a blank on-screen waveform, not zero playback.
* **Bundling or shipping any AlphaTheta-extracted key or crate-digger (EPL) code.** Format knowledge
  is taken from public specs and permissively-licensed reference implementations only (see How ▸
  Licensing).

## How

**Overview.** A standalone C++ library takes a plain intermediate model and emits bytes; Mixxx
integration is a thin adapter that fills that model from the library.

```text
Mixxx library (tracks + cues + beatgrids + crates/playlists [+ waveforms, phase 2])
        │
        ▼   ExportModel  (plain structs — no Qt)
  ┌──────────────────────────────────────────────────────────────────────┐
  │  libopenboxxx-export  (standalone, Qt-free)                           │
  │   mapping/  Mixxx-model → rekordbox-model (colors, keys, ms↔frame,…)  │
  │   pdb/      export.pdb writer   — LITTLE-endian DeviceSQL pages       │
  │   anlz/     ANLZ .DAT/.EXT      — BIG-endian tagged sections          │
  │   usb/      PIONEER/ dir layout, ANLZ path assignment, audio copy     │
  │   verify/   round-trip: parse our output w/ in-tree Kaitai + diff     │
  │   diag/     structural diagnostic bundle (feeds in-app bug report)    │
  └──────────────────────────────────────────────────────────────────────┘
        │
        ▼
  USB stick:  /PIONEER/…  +  copied audio files
```

The Mixxx-side adapter is a `RekordboxExportJob` that mirrors the existing `EnginePrimeExportJob`:
a `QThread` with `loadIds/loadTrack/loadCrate/loadPlaylist` marshalling and
`jobMaximum/jobProgress/completed/failed` signals, driving an export dialog, all behind a
`REKORDBOX_EXPORT` CMake option. This is the concrete implementation of the exporter abstraction
discussed in #12126, so the proposal both uses and helps define that interface. (The fork also adds
a small "Cue Sheet to Tracklist" utility alongside it.) The fork compiles and links on macOS
(Intel + Apple Silicon), Windows, and Linux under CI today.

**Cross-platform by construction.** Byte layout is always explicit, never host-dependent: PDB integers
written explicitly little-endian, ANLZ integers explicitly big-endian (separate helpers, never
shared); strings encoded by hand per `device_sql_string` rules (short-ASCII mangling or `0x90`
UTF-16**LE** for PDB, UTF-16**BE** for ANLZ paths/labels — no `std::wstring`/`wchar_t`);
USB-internal paths always forward-slashed regardless of host OS; no FAT/exFAT-specific code (the DJ
formats the stick). This is why macOS/Linux/Windows are all supported from day one, which also
maximizes the beta-tester (hardware data) pool.

**Testing and verification.** A four-tier ladder, cheapest first:

1. **Round-trip self-check** — write a stick, parse it back with the in-tree Kaitai parsers, confirm
   it reproduces the intended model. Free; runs on every build in CI.
2. **Diff vs a real rekordbox stick** — structural field comparison against a genuine
   rekordbox-produced stick, to catch layout mistakes the parser tolerates.
3. **Import into rekordbox desktop** — behavioral smoke test. ⚠️ *Not* equivalent to hardware (the
   desktop app is more forgiving and may re-analyze). Standardize on **rekordbox 5** for this tier:
   a rekordbox 7 stick writes `export.pdb` as an empty compatibility shell and puts the real library
   in the encrypted OneLibrary DB, so rb7 is a misleading oracle for classic PDB.
4. **Real CDJ/XDJ hardware** — the only true proof, run at milestones. Launch matrix: CDJ-2000NXS2
   (legacy baseline), CDJ-3000/3000X (strict/modern), XDJ-AZ (modern all-in-one).

Tiers 1–2 are exposed as a **library API** (`verify/`) and reused by the in-app verifier so the DJ
gets the same check. **Current status: tiers 1–2 are green** — exports complete without errors and
round-trip through both independent parsers. **Tiers 3–4 (rekordbox-desktop import and physical
CDJ/XDJ playback) are the outstanding proof and have not yet been run**; this is the main thing
hardware testers can help unblock.

**Migration and downtime.** None — this is purely additive. It introduces a new
export path (generalizing the existing Engine Prime menu item) behind a `REKORDBOX_EXPORT` CMake
option, in the same spirit as the existing `ENGINEPRIME`/`__ENGINEPRIME__` gate. No existing data,
schema, or user workflow changes;
nothing is removed. Existing rekordbox *import* is untouched.

**Licensing.** The module is **GPLv2** (Mixxx-compatible). ANLZ layout is *ported* from **pyrekordbox**
(MIT → GPLv2-compatible; attributed). PDB layout is implemented from the **crate-digger `.ksy` spec**
(EPL-1.0) used as a *format description*, not copied code — the same clean-room-ish posture by which
Mixxx's existing rekordbox importer was built (a file format is not itself copyrightable; a specific
code expression is). crate-digger's EPL code is never linked. rekordcrate (MPL-2.0) is
GPL-compatible and is discussed as an alternative below.

**Known unknowns / open questions.**

* `track_row` "always X" magic constants and `bitmask` semantics — written from reference values;
  worth confirming behaviorally on hardware.
* All 20 standard PDB tables (including the library-independent `columns` / `unknown_17` /
  `unknown_18` / `history` browse/sort menu tables) are now written, and output round-trips through
  both parsers. The remaining unknown is purely behavioral: whether CDJ firmware is satisfied with the
  exact contents — resolved only at tiers 3–4.
* The **ANLZ-path hash quirk** (flagged by a prior community exporter): confirm whether newer firmware
  requires a specific folder-path hash before assuming free choice of `USBANLZ` folder names.
* PDB page conventions and `first_page`/`last_page` semantics — pass the parsers; validate on hardware.

## Alternatives

1. **Use rekordcrate (Rust) for PDB export instead of a C++ writer.** rekordcrate (Jan Holthuis) is
   a maintainer-suggested path and already models the PDB format in Rust with serialization in view.
   *Objection to this proposal:* "why hand-write a C++ serializer when a Rust one exists?"
   *Counter-arguments:* (a) Mixxx is C++/CMake with no Rust in the build today, so adopting
   rekordcrate means introducing a Rust toolchain + FFI boundary into the core build — a larger
   architectural commitment than the exporter itself; (b) rekordcrate covers PDB but not the full
   ANLZ writer path, USB layout, verifier, and diagnostic bundle this proposal needs, so it is a
   partial solution regardless; (c) a Qt-free C++ library keeps the serializer inside the language
   and build system maintainers already use. **This is genuinely an open decision for maintainers**,
   and the standalone-library structure here is deliberately arranged so the writer could be swapped
   for a rekordcrate-backed one without disturbing the Mixxx adapter — the adapter/dialog/CMake work
   is reusable either way. Maintainer direction on C++-writer vs Rust-via-rekordcrate is explicitly
   solicited.
2. **Generate a serializer from Kaitai instead of hand-writing.** Rejected: Kaitai's C++ write
   support is experimental/partial and not viable today; the read parsers stay Kaitai-generated, the
   writers are hand-written.
3. **Hard-fork Mixxx and add export out-of-tree.** Rejected as the primary path: forking splits the
   community and forfeits Mixxx's test base, review, and credibility. Upstreaming into the existing
   export subsystem is preferred; a fork is a fallback only if upstream cannot accept the work.
4. **Export `rekordbox.xml` only, and let users import that into rekordbox.** Insufficient: XML round-
   trips metadata but does not produce a USB a CDJ can play from standalone — it still requires the
   user to open rekordbox and re-export to USB, which is the exact dependency this feature removes.
5. **Target OneLibrary because it's the "newest, sanctioned" format.** Rejected on legal grounds — it
   is encrypted; see Non-Goals. Counterintuitively the newest format is the one we must *not*
   reverse-engineer key-in-hand.

## Action Plan

* [x] Survey the state of the Mixxx export effort and confirm the in-tree hooks (Engine DJ exporter,
      `lib/rekordbox-metadata/`, #12126). — openboxxx `docs/mixxx-export-status.md`,
      `research-findings.md`
* [x] **Phase 0 — standalone PDB/ANLZ writer + verifier, proven.** `libopenboxxx-export` emits a full
      `export.pdb` (all 20 standard tables + 8-colour palette) and per-track ANLZ (beat grids + hot/
      memory cues); every output is cross-checked against two independent parsers (pyrekordbox for
      ANLZ, a crate-digger-spec Kaitai parser for PDB) and diffed field-for-field against genuine
      rekordbox ANLZ files.
* [x] **Phase 1 reader path.** `openboxxx_from_mixxx` reads a real `mixxxdb.sqlite` into the export
      model with no Mixxx build required — validated on a ~2,900-track library.
* [x] **Phase 1 in-Mixxx exporter, working in a fork.** "Export Library to rekordbox USB" button —
      `RekordboxExportJob` + dialog mirroring the Engine DJ exporter, behind a `REKORDBOX_EXPORT`
      CMake option (plus a "Cue Sheet to Tracklist" tool). Compiles + links on macOS (Intel + Apple
      Silicon), Windows, and Linux via CI; downloadable builds on the
      [Releases](https://github.com/sparkly-quasar/openboxxx/releases) page. — refs mixxxdj/mixxx#12126
* [ ] **Verification tier 3 — rekordbox desktop import.** Confirm a generated stick imports cleanly
      into rekordbox 5 (the classic-PDB target; not rb7, which prefers the encrypted OneLibrary DB).
* [ ] **Verification tier 4 — real CDJ/XDJ hardware.** The true proof, on the launch matrix
      (CDJ-2000NXS2, CDJ-3000/3000X, XDJ-AZ). Hardware testers very welcome. — refs mixxxdj/mixxx#9463,
      mixxxdj/mixxx#10321
* [ ] Resolve the residual behavioral unknowns at tiers 3–4 (ANLZ-path hash quirk, `track_row` magic
      constants, exact firmware expectations for the menu tables).
* [ ] Propose/align the shared exporter interface upstream so this lands as a contribution, not a
      fork. — refs mixxxdj/mixxx#12126
* [ ] Add the in-app "report an export bug" flow (structural diagnostic bundle → prefilled GitHub
      issue; no telemetry server; user reviews before it leaves the machine).
* [ ] Publish a hardware compatibility matrix with beta-tester results.
* [ ] Phase 2 (follow-up, additive): waveforms and extended `.EXT`/`PCO2` cues with colors + labels,
      `.2EX` for CDJ-3000, `PVBR`.
