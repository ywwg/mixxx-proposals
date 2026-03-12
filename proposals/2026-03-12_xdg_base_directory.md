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

* Current pitfalls will be documented here.

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

This section will describe the implementation approach, including the
proposed `MixxxPathResolver` API surface, platform path mappings, and
the legacy detection strategy.

## Alternatives

Alternative approaches will be evaluated here, including keeping the
current single-directory layout and other directory organization schemes.

## Action Plan

* [ ] Define file categorization table mapping every ~/.mixxx entry to
  an XDG category
* [ ] Specify cross-platform path resolution using QStandardPaths
* [ ] Design legacy detection and migration strategy
