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

This section explains why Mixxx should adopt XDG Base Directory paths.
Mixxx currently stores all user data, configuration, caches, and state in a
single `~/.mixxx` directory. This violates the XDG Base Directory
Specification on Linux and misses platform-standard locations on macOS and
Windows.

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
