# XDG Base Directory Support

- **Owners:**
  - Owen Williams <owilliams@mixxx.org>

- **Proposal Status:** `Draft`

- **Implementation Status:** `Not implemented`

- **Related Issues and PRs:**
  - [XDG Base Directory support](https://github.com/mixxxdj/mixxx/issues/8090)

- **Other docs or links:**
  - [XDG Base Directory Specification](https://specifications.freedesktop.org/basedir-spec/latest/)

> TL;DR: Migrate Mixxx from a single ~/.mixxx directory to XDG Base Directory
> compliant paths on Linux, using platform-standard locations on macOS and
> Windows, while preserving backward compatibility for existing installations.

## Why

Mixxx stores all user data in a single `~/.mixxx` directory on Linux
and BSD. The path is set via a hardcoded CMake variable
(`MIXXX_SETTINGS_PATH`) and constructed at startup in
`src/util/cmdlineargs.cpp`. This one directory holds configuration
files, the track database, controller mappings, custom skins, logs,
and waveform analysis cache.

The [XDG Base Directory Specification](https://specifications.freedesktop.org/basedir-spec/latest/)
defines standard locations for user-specific files on Linux and BSD.
Most modern Linux desktop applications follow this spec. Mixxx
violates it by mixing configuration, data, state, and cache in a
single location rather than separating them into the directories the
spec requires.

The spec defines four user-specific base directories:
`$XDG_CONFIG_HOME` (default `~/.config`) for configuration,
`$XDG_DATA_HOME` (default `~/.local/share`) for persistent data,
`$XDG_STATE_HOME` (default `~/.local/state`) for state that persists
between restarts but is not important enough for backup (logs,
history, layout), and `$XDG_CACHE_HOME` (default `~/.cache`) for
non-essential cached data that may be deleted at any time. The core
principle is separation of concerns: users and system tools can
manage each category independently. Note that `$XDG_STATE_HOME` was
added to the spec in 2021, making it the newest of the four.

[Issue #8090](https://github.com/mixxxdj/mixxx/issues/8090)
("replace ~/.mixxx folder with XDG config folders") has been open
since 2015 with 34 comments. The discussion has seen periodic
renewed interest in 2015, 2018, 2021, 2024, and 2025, with
contributors proposing path resolver classes and detect-not-migrate
strategies, but no implementation has landed.

macOS and Windows already use platform-standard locations via
`QStandardPaths::AppLocalDataLocation` (`~/Library/Application
Support/Mixxx/` on macOS, `%LOCALAPPDATA%/Mixxx/` on Windows).
Linux and BSD are the only platforms where Mixxx uses a hardcoded
non-standard path.

### Pitfalls of the current solution

- **Home directory clutter.** `~/.mixxx` is a visible dotfile in
  the user's home directory. XDG-compliant apps use `~/.config/`,
  `~/.local/share/`, and similar directories, keeping the home
  directory clean.

- **No separation of concerns.** Config files (restorable from
  backup), data files (user-created content like controller
  mappings), cache (regenerable waveform analysis), and state
  (logs, history) are all mixed together. This makes selective
  backup, cleanup, and synchronization harder.

- **Cache cleanup tools cannot help.** System cache cleaners
  (BleachBit, systemd-tmpfiles) operate on `~/.cache/`. Mixxx's
  analysis cache in `~/.mixxx/` is invisible to them.

- **Read-only home directory breaks Mixxx.** Users who set `$HOME`
  to read-only (a practice for testing XDG compliance) cannot run
  Mixxx, since it writes directly to `~/.mixxx`.

- **Inconsistent cross-platform behavior.** macOS and Windows
  already use platform-standard locations via `QStandardPaths`.
  Only Linux and BSD use a hardcoded non-standard path.

## Goals

Goals and use cases for the solution as proposed in [How](#how):

- Separate Mixxx files into the correct XDG categories (config, data,
  state, cache) on Linux.
- Use platform-standard locations on macOS and Windows.
- Preserve backward compatibility for existing installations that use
  `~/.mixxx`.

### Audience

Mixxx developers and packagers.

## Non-Goals

- Automatic migration of existing `~/.mixxx` directories.
- Changing the behavior of the `--settings-path` command-line flag.

## How

### File Categorization

The XDG Base Directory Specification defines four user-specific base
directories, each with a distinct purpose:

- **config** (`$XDG_CONFIG_HOME`, default `~/.config`): User preferences
  and settings. If deleted, the application resets to defaults but no
  user data is lost.
- **data** (`$XDG_DATA_HOME`, default `~/.local/share`): User-created
  or user-curated content. If deleted, the user loses irreplaceable
  work.
- **state** (`$XDG_STATE_HOME`, default `~/.local/state`): Information
  that persists between application restarts but is not important
  enough for backup. Logs, history, and runtime layout fall here. If
  deleted, the application restarts cleanly but loses session history.
- **cache** (`$XDG_CACHE_HOME`, default `~/.cache`): Non-essential
  data that can be regenerated from other sources. If deleted, the
  user notices nothing except a temporary performance impact.

The categorization below uses the spec definitions and the
[GNOME deletion test](https://wiki.gnome.org/Initiatives/GnomeGoals/XDGConfigFolders)
heuristic: "if you delete `~/.cache`, no data is lost; if you delete
`~/.config`, preferences reset; `~/.local/share` is user-created
content."

| Current Path (relative to `~/.mixxx/`) | XDG Category | New Path (Linux default)                   | Rationale                                                                                         |
| -------------------------------------- | ------------ | ------------------------------------------ | ------------------------------------------------------------------------------------------------- |
| `mixxx.cfg`                            | config       | `~/.config/Mixxx/mixxx.cfg`                | Main preferences file; deletion resets all settings to defaults                                   |
| `soundconfig.xml`                      | config       | `~/.config/Mixxx/soundconfig.xml`          | Sound hardware configuration; user must reconfigure audio devices if lost                         |
| `Custom.kbd.cfg`                       | config       | `~/.config/Mixxx/Custom.kbd.cfg`           | User keyboard shortcut overrides; configuration by definition                                     |
| `mixxxdb.sqlite`                       | data         | `~/.local/share/Mixxx/mixxxdb.sqlite`      | Track library database with irreplaceable user metadata (crates, playlists, play counts, ratings) |
| `controllers/`                         | data         | `~/.local/share/Mixxx/controllers/`        | User-created or user-modified controller mappings; user content                                   |
| `midi/`                                | data         | `~/.local/share/Mixxx/midi/`               | Legacy controller mappings (pre-1.11.0, still checked); user content                              |
| `skins/`                               | data         | `~/.local/share/Mixxx/skins/`              | User-created custom skins; user content                                                           |
| `broadcast_profiles/`                  | data         | `~/.local/share/Mixxx/broadcast_profiles/` | Streaming profiles containing server credentials; user content with secrets                       |
| `effects/defaults/`                    | data         | `~/.local/share/Mixxx/effects/defaults/`   | User-configured effect presets; user content                                                      |
| `effects/chains/`                      | data         | `~/.local/share/Mixxx/effects/chains/`     | User-configured effect chain presets; user content                                                |
| `effects.xml`                          | state        | `~/.local/state/Mixxx/effects.xml`         | Current effects chain state (loaded effects per unit); runtime state, not preferences             |
| `samplers.xml`                         | state        | `~/.local/state/Mixxx/samplers.xml`        | Current sampler deck state (loaded samples); runtime state, not preferences                       |
| `mixxx.log`                            | state        | `~/.local/state/Mixxx/mixxx.log`           | Current session log; the spec lists "action history (logs)" as state                              |
| `mixxx.log.1` through `mixxx.log.9`    | state        | `~/.local/state/Mixxx/mixxx.log.1` .. `.9` | Rotated log files; same rationale as current session log                                          |
| `co_dump_*.csv`                        | state        | `~/.local/state/Mixxx/co_dump_*.csv`       | Developer debug dumps; diagnostic state data                                                      |
| `analysis/`                            | cache        | `~/.cache/Mixxx/analysis/`                 | Waveform analysis data; regenerable from audio files (see warning below)                          |
| `lut/`                                 | cache        | `~/.cache/Mixxx/lut/`                      | Vinyl control lookup tables; generated/computed data, regenerable                                 |

**Notes:**

- `Mixxx` is capitalized per the XDG spec that uses the exact registered
  application name, which is capitalized.
- `sandbox.cfg` is macOS-only. The path is set unconditionally in
  the source code, but the file is only written on macOS. It still
  needs a category for the cross-platform path resolver.
- `broadcast_profiles/` is placed in data rather than config because
  profiles contain server passwords. Data is safer from casual
  sharing or syncing than config, which users may back up or
  distribute more freely.
- `effects.xml` and `samplers.xml` are state, not config. They
  represent what is currently loaded in each effect unit or sampler
  deck, not user preferences. Applying the GNOME deletion test:
  losing them means effect units and sampler decks reset to defaults
  on next launch. The user does not lose any saved preferences.
- Legacy files from pre-1.7 Mixxx (`mixxxtrack.xml`,
  `mixxxbpmscheme.xml`, etc.) appear only in upgrade code and are
  not created by current Mixxx. They do not need XDG placement.

### Waveform Analysis Cache

The `analysis/` directory stores pre-computed waveform data (waveforms,
beatgrids). This data is regenerable from the original audio files,
making it a cache by the XDG spec definition. However, regeneration is
expensive: several seconds per track on modern hardware, meaning a
library of 10,000 tracks could take many minutes to fully re-analyze
even in parallel.

Users who run cache cleanup tools (BleachBit, `systemd-tmpfiles`,
manual `rm -rf ~/.cache/*`) will trigger a full re-analysis cycle.
This is the single most user-visible consequence of XDG compliance.

The recommendation is to place `analysis/` in cache per the spec. The
alternative (placing it in `$XDG_DATA_HOME`) would be technically
incorrect since the data can be regenerated from source. Instead,
mitigate through documentation warning users that
`~/.cache/mixxx/analysis/` may be cleaned by system tools. Mixxx
should NOT register with `systemd-tmpfiles` or similar cleanup
systems. The cache is valid indefinitely as long as the source audio
files exist.

NOTE: We could decide instead to locate analysis in .local/share if that's
preferred.

### Cross-Platform Path Resolution

Mixxx uses Qt's `QStandardPaths` API to resolve platform-standard
directories for user files. The application sets
`QCoreApplication::setApplicationName("Mixxx")` with no
`organizationName`, so `QStandardPaths` appends just `Mixxx` to each
base path. Note the intentional case difference from the legacy
`~/.mixxx` directory (lowercase m): the new XDG paths use capital-M
`Mixxx` because that is what `QStandardPaths` produces from the
registered application name.

The following table shows which `QStandardPaths` enum resolves each
XDG category on each platform:

| Category | QStandardPaths Enum                                                | Linux                  | macOS                                 | Windows                                     |
| -------- | ------------------------------------------------------------------ | ---------------------- | ------------------------------------- | ------------------------------------------- |
| config   | `ConfigLocation` (Linux) / `AppLocalDataLocation` (macOS, Windows) | `~/.config/Mixxx`      | `~/Library/Application Support/Mixxx` | `C:/Users/<USER>/AppData/Local/Mixxx`       |
| data     | `AppLocalDataLocation`                                             | `~/.local/share/Mixxx` | `~/Library/Application Support/Mixxx` | `C:/Users/<USER>/AppData/Local/Mixxx`       |
| state    | `StateLocation` (Qt 6.7+)                                          | `~/.local/state/Mixxx` | `~/Library/Preferences/Mixxx/State`   | `C:/Users/<USER>/AppData/Local/Mixxx/State` |
| cache    | `CacheLocation`                                                    | `~/.cache/Mixxx`       | `~/Library/Caches/Mixxx`              | `C:/Users/<USER>/AppData/Local/Mixxx/cache` |

On Linux, config and data resolve to two distinct directories
(`~/.config/Mixxx` and `~/.local/share/Mixxx`). On macOS and Windows,
both resolve to the same `AppLocalDataLocation` directory. This means
the config/data split is Linux-only for practical purposes.

The reason is platform-specific:

- **macOS:** `ConfigLocation` resolves to `~/Library/Preferences`
  with no application subdirectory. That directory is designed for
  `.plist` files managed by `NSUserDefaults`, not arbitrary config
  directories. Using it would scatter Mixxx files into a shared
  system directory. The path resolver must return
  `AppLocalDataLocation` (`~/Library/Application Support/Mixxx`) for
  both config and data on macOS.

- **Windows:** `ConfigLocation` and `AppLocalDataLocation` both
  resolve to the same path (`C:/Users/<USER>/AppData/Local/Mixxx`).
  There is no config/data split to exploit. The path resolver returns
  `AppLocalDataLocation` for both.

As a consequence, the path resolver must use `ConfigLocation` only on
Linux. On macOS and Windows, it returns `AppLocalDataLocation` for
both config and data categories.

#### StateLocation Fallback (Qt < 6.7)

Mixxx requires Qt >= 6.2, but `QStandardPaths::StateLocation` was
added in Qt 6.7. Code referencing the `StateLocation` enum will not
compile on Qt 6.2 through 6.6.

The fallback strategy is to use `AppLocalDataLocation` with a `/State`
subdirectory when `StateLocation` is unavailable. This matches Qt's
own internal implementation pattern: on macOS and Windows,
`StateLocation` always resolves to a subdirectory of another standard
path.

The compile-time guard:

```cpp
#if QT_VERSION >= QT_VERSION_CHECK(6, 7, 0)
    return QStandardPaths::writableLocation(QStandardPaths::StateLocation);
#else
    return QStandardPaths::writableLocation(QStandardPaths::AppLocalDataLocation)
           + QStringLiteral("/State");
#endif
```

The fallback produces these resolved paths on Qt < 6.7:

| Platform | Fallback State Path                         |
| -------- | ------------------------------------------- |
| Linux    | `~/.local/share/Mixxx/State`                |
| macOS    | `~/Library/Application Support/Mixxx/State` |
| Windows  | `C:/Users/<USER>/AppData/Local/Mixxx/State` |

On Linux, the fallback path (`~/.local/share/Mixxx/State`) differs
from the XDG-correct location (`~/.local/state/Mixxx`). State files
land inside the data directory rather than in the dedicated state
directory. When Mixxx's minimum Qt version moves to 6.7+, the
fallback can be removed and state files will land in the correct XDG
location. On macOS and Windows, the difference between fallback and
native Qt 6.7+ paths is minimal (different parent directory, same
`/State` suffix), and neither platform has an established convention
for state directories.

#### Flatpak and Snap Path Remapping

Flatpak overrides XDG environment variables inside the sandbox.
Because `QStandardPaths` reads these variables on Linux, path
resolution works transparently with no code changes. Inside a Flatpak
sandbox, the XDG variables resolve to:

| Variable          | Flatpak Sandbox Path                      |
| ----------------- | ----------------------------------------- |
| `XDG_CONFIG_HOME` | `~/.var/app/org.mixxx.Mixxx/config`       |
| `XDG_DATA_HOME`   | `~/.var/app/org.mixxx.Mixxx/data`         |
| `XDG_STATE_HOME`  | `~/.var/app/org.mixxx.Mixxx/.local/state` |
| `XDG_CACHE_HOME`  | `~/.var/app/org.mixxx.Mixxx/cache`        |

The current Flatpak manifest (`org.mixxx.Mixxx.yaml`) uses
`--persist=.mixxx` to map the legacy `~/.mixxx` directory into the
sandbox. When Mixxx switches to XDG paths, this directive should be
removed. Alternatively, it can be retained temporarily to support
legacy detection during the transition period (see Phase 4).

Snap remaps `HOME` to `~/snap/<snap-name>/<revision>/`, so all XDG
paths land inside the Snap sandbox automatically. For example,
`~/.config/Mixxx` becomes `~/snap/mixxx/<rev>/.config/Mixxx`. Mixxx
does not currently have an official Snap package; this is documented
for completeness.

Neither Flatpak nor Snap require code changes beyond the core switch
from hardcoded paths to `QStandardPaths`.

### Legacy Detection and Migration

Mixxx must decide at startup whether to operate in legacy single-directory
mode or XDG split-directory mode. This decision is made once at startup
and is immutable for the duration of the session. There are exactly two
modes:

- **Legacy mode:** All files (config, data, state, cache) reside in a
  single directory.
- **XDG mode:** Files are distributed across config, data, state, and
  cache directories as defined by the platform matrix above.

The following subsections specify how the mode is selected, what the
known legacy paths are, how `--settings-path` interacts with mode
selection, and why no hybrid mode is permitted.

#### Startup Decision Tree

The startup logic follows this flowchart. Each branch terminates with
the selected mode. No further mode switching occurs after this point.

```text
START
  |
  v
Was --settings-path provided?
  |
  YES --> Use that path as single directory (Legacy mode). DONE.
  |
  NO --> What platform?
           |
           LINUX/BSD --> Does ~/.mixxx/ exist?
           |               |
           |               YES --> Use ~/.mixxx/ as single directory
           |               |       (Legacy mode). DONE.
           |               |
           |               NO --> Use XDG split paths:
           |                      ConfigLocation, AppLocalDataLocation,
           |                      StateLocation, CacheLocation.
           |                      (XDG mode). DONE.
           |
           macOS --> Run existing Sandbox::migrateOldSettings() logic.
           |         Result is always a single directory.
           |         (Legacy mode, current behavior unchanged). DONE.
           |
           Windows --> QStandardPaths::AppLocalDataLocation already used.
                       Config and data share one directory.
                       (Legacy mode, current behavior unchanged). DONE.
```

Key observations:

- Only Linux/BSD gains a new code path (the XDG split). macOS and
  Windows behavior is unchanged by this proposal.
- The legacy check is a simple `QDir::exists()` on one known path per
  platform. No file-content inspection is needed.
- Mode determination must happen before any file I/O, specifically
  before logging initialization. The existing
  `initializeSettings()` -> `initializeLogging()` order in
  `CoreServices` is correct and must be preserved.

#### Legacy Paths Per Platform

Each platform has exactly one "most recent" legacy path. This is the
path that the decision tree checks for existence.

| Platform  | Most Recent Legacy Path                | Notes                                                                          |
| --------- | -------------------------------------- | ------------------------------------------------------------------------------ |
| Linux/BSD | `~/.mixxx/`                            | Hardcoded via `MIXXX_SETTINGS_PATH` CMake variable since always                |
| macOS     | `~/Library/Application Support/Mixxx/` | Pre-2.3.0 location before macOS sandbox migration                              |
| Windows   | `C:/Users/<USER>/AppData/Local/Mixxx/` | Current location since Mixxx 1.12.0 via `QStandardPaths::AppLocalDataLocation` |

Older legacy paths (macOS `~/.mixxx/` from pre-1.9.0, Windows
`Local Settings/Application Data/Mixxx/` from pre-1.12.0) are already
handled by existing upgrade code in `upgrade.cpp` and are out of scope
for this proposal.

#### The `--settings-path` Flag

**Current behavior:** The `--settings-path` flag accepts a directory
path and sets `m_settingsPathSet = true`, which gates all automatic
path detection. When this flag is set, macOS sandbox migration
(`Sandbox::migrateOldSettings()`), macOS pre-1.9 legacy detection
(`upgrade.cpp`), and Windows pre-1.12 legacy detection (`upgrade.cpp`)
are all skipped.

**Proposed behavior:** Identical. When `--settings-path` is provided,
all files (config, data, state, cache) go into the specified directory.
No XDG split occurs. No legacy detection runs. This preserves the
profile-switching use case valued by users who maintain multiple Mixxx
configurations.

The deprecated `--settingsPath` (camelCase) form continues to work
identically.

#### All-or-Nothing Mode Selection

Mixxx operates in exactly one of two modes per session. There is no
hybrid.

- **Legacy mode (single directory):** Triggered by any of:
  - `--settings-path` flag provided
  - Legacy directory detected at startup (Linux: `~/.mixxx/` exists)
  - macOS or Windows (current behavior, single directory already)

- **XDG mode (split directories):** Triggered by:
  - Fresh Linux/BSD install with no `~/.mixxx/` directory

There is no per-file fallback between legacy and XDG locations. The
reasons are:

1. **Complexity.** Every file access would need try-legacy-then-XDG
   logic, creating 30+ branch points across the 21 call sites that
   reference `getSettingsPath()`.
2. **Ambiguity.** If `~/.mixxx/mixxx.cfg` exists but
   `~/.mixxx/controllers/` does not, where do new controller mappings
   go? The legacy directory or the XDG data directory?
3. **User confusion.** Some files in the old location, some in the
   new. Users cannot reason about where their data lives.
4. **Testing burden.** 2^N combinations of file presence across two
   location sets makes comprehensive testing impractical.

The all-or-nothing approach matches how macOS sandbox migration already
works: `Sandbox::migrateOldSettings()` moves the entire directory, not
individual files.

#### No Automatic Migration

Mixxx will not automatically copy or move files from the legacy
directory to XDG locations. The rationale:

1. **Beta testing safety.** daschuer (project lead) warned in
   [#8090](https://github.com/mixxxdj/mixxx/issues/8090) (September
   2024): "After testing a beta, the stable version will no longer find
   .mixxx and create a new one: everything gone." If migration moved
   files out of `~/.mixxx/`, reverting to a stable release would find
   an empty legacy directory.

2. **No migration code to maintain.** Copy-on-migrate needs error
   handling for disk full, permissions errors, partial copy rollback,
   and symlink preservation. GIMP's profile migration code is a
   cautionary example of this maintenance burden.

3. **User agency.** Users who want XDG paths can manually move files
   at a time of their choosing, following wiki instructions. Users who
   prefer `~/.mixxx/` keep it indefinitely with no forced change.

4. **Atomic consistency.** If migration fails halfway, files exist in
   two places and Mixxx cannot determine which copy is authoritative.
   The detect-not-migrate approach means files are always in exactly
   one location.

**Precedent:** Firefox 147 (2025) adopted XDG paths with a similar
detect-not-migrate approach for the profile directory. New profiles
use XDG paths; existing profiles stay in `~/.mozilla/firefox/`. This
validates the pattern for a major desktop application with a much
larger user base than Mixxx.

**Community consensus:** acolombier proposed in #8090 to "support both
locations for now, and create in the new XDG compatible location on new
setup only." Krafting responded: "seems like a better idea than copying
data indeed!" daschuer suggested: "don't copy, use the old folder if
found, but default to the new if not." This proposal implements that
consensus.

#### Wiki Migration Guide Outline

A wiki page (not part of this proposal) should provide manual migration
instructions for users who want to move from `~/.mixxx/` to XDG paths.

### Implementation Notes

This section provides implementation guidance for developers working
on the XDG transition. It sketches the proposed API surface,
inventories the affected call sites, and outlines a deprecation bridge
for the existing `getSettingsPath()` API. This is not a specification.
Implementers should adapt the design to fit the codebase as it
evolves.

#### MixxxPathResolver API Sketch

In [issue #8090](https://github.com/mixxxdj/mixxx/issues/8090),
holzhaus proposed (October 2021): "As a first step, we should
introduce a class that allows accessing the different path types
(cache, data, config) and use it everywhere." The class sketched
here follows that approach.

Design principles:

- Return `QDir` not `QString` for directory paths. Callers use
  `QDir::filePath()` to construct full paths, eliminating
  trailing-slash inconsistencies in the current API.
- Immutable after construction. Mode and all directory paths are set
  in the constructor and never change, matching the all-or-nothing
  decision from Legacy Detection.
- No platform ifdefs at call sites. The resolver encapsulates all
  platform logic. Callers call `configDir()` and get the right path
  regardless of platform or mode.
- Constructed once in `CoreServices::initializeSettings()` and
  passed via constructor injection (not a global singleton).

```cpp
// src/util/mixxxpathresolver.h

#pragma once

#include <QDir>
#include <QString>

class MixxxPathResolver {
  public:
    enum class PathMode {
        Legacy,  // All files in one directory
        XDG      // Files split across config/data/state/cache
    };

    // Factory: determines mode from --settings-path flag
    // and legacy detection. Must be called before any file
    // I/O (before logging init).
    static MixxxPathResolver create(
            const QString& settingsPathOverride,
            bool settingsPathSet);

    // Typed directory accessors. Each returns a QDir that
    // is guaranteed to exist (created in constructor if
    // needed).
    QDir configDir() const { return m_configDir; }
    QDir dataDir() const { return m_dataDir; }
    QDir stateDir() const { return m_stateDir; }
    QDir cacheDir() const { return m_cacheDir; }

    PathMode mode() const { return m_mode; }

    // Legacy compatibility: returns dataDir().path() + "/"
    // Deprecated. Use typed accessors instead.
    [[deprecated("Use typed accessors instead")]]
    QString settingsPath() const;

  private:
    MixxxPathResolver(PathMode mode,
                      QDir configDir,
                      QDir dataDir,
                      QDir stateDir,
                      QDir cacheDir);

    PathMode m_mode;
    QDir m_configDir;
    QDir m_dataDir;
    QDir m_stateDir;
    QDir m_cacheDir;
};
```

The `create()` factory method implements the startup decision tree
from Legacy Detection. When `--settings-path` is provided, all four
directories point to the same location. On Linux/BSD with no legacy
`~/.mixxx/` directory, it uses `QStandardPaths` for each category,
including the Qt 6.7 `StateLocation` fallback. On macOS and Windows,
it returns a single directory as today.

A note on `QDir` return types: the current codebase has inconsistent
patterns for path construction. Some call sites use
`QDir::filePath()` (correct), others use string concatenation with
`+` (fragile, depends on trailing slash). Returning `QDir` forces
the correct pattern and eliminates an entire class of path bugs.

#### Affected Call Sites

There are 38 call sites across 21 files that reference
`getSettingsPath()` through two API surfaces:
`CmdlineArgs::getSettingsPath()` (8 call sites) and
`ConfigObject::getSettingsPath()` (30 call sites).

The following table groups call sites by XDG category:

| Category | Call Sites | Key Files                                                                                                      | Accessor      |
| -------- | ---------- | -------------------------------------------------------------------------------------------------------------- | ------------- |
| Config   | 3          | `main.cpp`, `soundmanagerconfig.cpp`                                                                           | `configDir()` |
| Data     | 18         | `mixxxdb.cpp`, `defs_controllers.h`, `skinloader.cpp`, `broadcastsettings.cpp`, `effectchainpresetmanager.cpp` | `dataDir()`   |
| State    | 6          | `effectsmanager.cpp`, `playermanager.cpp`, `dlgdevelopertools.cpp`                                             | `stateDir()`  |
| Cache    | 2          | `analysisdao.cpp`, `vinylcontrolxwax.cpp`                                                                      | `cacheDir()`  |
| Meta     | 9          | `cmdlineargs.cpp`, `coreservices.cpp`, `upgrade.cpp`                                                           | (various)     |

Notable edge cases:

- `Custom.kbd.cfg` is accessed via `pConfig->getSettingsPath()` in
  `coreservices.cpp` but is a config file. It must use
  `configDir()`, not `dataDir()`.
- `legacycontrollermapping.cpp` uses `getSettingsPath()` for string
  replacement (deconstructing paths, not constructing them). This
  call site needs special handling: use `dataDir().path()` as the
  string to match against.
- `wmainmenubar.cpp` and `dlgpreflibrary.cpp` open or display "the
  settings directory" to users. In XDG mode there is no single
  directory. Recommendation: open the data directory as it contains
  the most user-relevant files.
- Trailing slash dependency: the current API appends a trailing
  slash. `QDir::path()` does not. All migrated call sites must use
  `QDir::filePath()` instead of string concatenation.

Before/after examples:

```cpp
// Before (database/mixxxdb.cpp)
QDir(pConfig->getSettingsPath()).absoluteFilePath(kDefaultFileName);
// After
m_pathResolver.dataDir().absoluteFilePath(kDefaultFileName);
```

```cpp
// Before (library/dao/analysisdao.cpp)
QDir analysisDir(m_pConfig->getSettingsPath() + "analysis");
// After
QDir analysisDir(m_pathResolver.cacheDir().filePath("analysis"));
```

## Alternatives

Alternative approaches will be evaluated here, including keeping the
current single-directory layout and other directory organization schemes.
