# XDG Base Directory Support

* **Owners:**
  * `<@owner: GitHub handle>`

* **Implementation Status:** `Not implemented`

* **Related Issues and PRs:**
  * [XDG Base Directory support](https://github.com/mixxxdj/mixxx/issues/8090)

* **Other docs or links:**
  * [XDG Base Directory Specification](https://specifications.freedesktop.org/basedir-spec/latest/)

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

The Mixxx codebase already acknowledges this gap. In
`src/util/cmdlineargs.cpp`, the following comment guards the
Linux/BSD path:

```cpp
// We are not ready to switch to XDG folders under Linux, so keeping $HOME/.mixxx as preferences folder. see #8090
```

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

* **Home directory clutter.** `~/.mixxx` is a visible dotfile in
  the user's home directory. XDG-compliant apps use `~/.config/`,
  `~/.local/share/`, and similar directories, keeping the home
  directory clean.

* **No separation of concerns.** Config files (restorable from
  backup), data files (user-created content like controller
  mappings), cache (regenerable waveform analysis), and state
  (logs, history) are all mixed together. This makes selective
  backup, cleanup, and synchronization harder.

* **Cache cleanup tools cannot help.** System cache cleaners
  (BleachBit, systemd-tmpfiles) operate on `~/.cache/`. Mixxx's
  analysis cache in `~/.mixxx/` is invisible to them.

* **Read-only home directory breaks Mixxx.** Users who set `$HOME`
  to read-only (a practice for testing XDG compliance) cannot run
  Mixxx, since it writes directly to `~/.mixxx`.

* **Inconsistent cross-platform behavior.** macOS and Windows
  already use platform-standard locations via `QStandardPaths`.
  Only Linux and BSD use a hardcoded non-standard path.

## Goals

Goals and use cases for the solution as proposed in [How](#how):

* Separate Mixxx files into the correct XDG categories (config, data,
  state, cache) on Linux.
* Use platform-standard locations on macOS and Windows.
* Preserve backward compatibility for existing installations that use
  `~/.mixxx`.

### Audience

Mixxx developers and packagers.

## Non-Goals

* Automatic migration of existing `~/.mixxx` directories.
* Changing the behavior of the `--settings-path` command-line flag.

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

| Current Path (relative to `~/.mixxx/`) | XDG Category | New Path (Linux default) | Rationale |
|----------------------------------------|-------------|--------------------------|-----------|
| `mixxx.cfg` | config | `~/.config/mixxx/mixxx.cfg` | Main preferences file; deletion resets all settings to defaults |
| `soundconfig.xml` | config | `~/.config/mixxx/soundconfig.xml` | Sound hardware configuration; user must reconfigure audio devices if lost |
| `Custom.kbd.cfg` | config | `~/.config/mixxx/Custom.kbd.cfg` | User keyboard shortcut overrides; configuration by definition |
| `mixxxdb.sqlite` | data | `~/.local/share/mixxx/mixxxdb.sqlite` | Track library database with irreplaceable user metadata (crates, playlists, play counts, ratings) |
| `controllers/` | data | `~/.local/share/mixxx/controllers/` | User-created or user-modified controller mappings; user content |
| `midi/` | data | `~/.local/share/mixxx/midi/` | Legacy controller mappings (pre-1.11.0, still checked); user content |
| `skins/` | data | `~/.local/share/mixxx/skins/` | User-created custom skins; user content |
| `broadcast_profiles/` | data | `~/.local/share/mixxx/broadcast_profiles/` | Streaming profiles containing server credentials; user content with secrets |
| `effects/defaults/` | data | `~/.local/share/mixxx/effects/defaults/` | User-configured effect presets; user content |
| `effects/chains/` | data | `~/.local/share/mixxx/effects/chains/` | User-configured effect chain presets; user content |
| `sandbox.cfg` | data | `~/.local/share/mixxx/sandbox.cfg` | macOS sandbox permission bookmarks; loss requires re-granting filesystem access |
| `effects.xml` | state | `~/.local/state/mixxx/effects.xml` | Current effects chain state (loaded effects per unit); runtime state, not preferences |
| `samplers.xml` | state | `~/.local/state/mixxx/samplers.xml` | Current sampler deck state (loaded samples); runtime state, not preferences |
| `mixxx.log` | state | `~/.local/state/mixxx/mixxx.log` | Current session log; the spec lists "action history (logs)" as state |
| `mixxx.log.1` through `mixxx.log.9` | state | `~/.local/state/mixxx/mixxx.log.1` .. `.9` | Rotated log files; same rationale as current session log |
| `co_dump_*.csv` | state | `~/.local/state/mixxx/co_dump_*.csv` | Developer debug dumps; diagnostic state data |
| `analysis/` | cache | `~/.cache/mixxx/analysis/` | Waveform analysis data; regenerable from audio files (see warning below) |
| `lut/` | cache | `~/.cache/mixxx/lut/` | Vinyl control lookup tables; generated/computed data, regenerable |

**Notes:**

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
library of 10,000 tracks could take 1-3 hours to fully re-analyze.

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

This section will continue with the proposed `MixxxPathResolver` API
surface, platform path mappings, and the legacy detection strategy.

## Alternatives

Alternative approaches will be evaluated here, including keeping the
current single-directory layout and other directory organization schemes.

## Action Plan

* [ ] Define file categorization table mapping every ~/.mixxx entry to
  an XDG category
* [ ] Specify cross-platform path resolution using QStandardPaths
* [ ] Design legacy detection and migration strategy
