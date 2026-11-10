# Changelog

Notable changes to this project, newest first.

## 0.2.1 - 2026-11-10

### Changed

- The license is now two files. `LICENSE` holds the GNU AGPL v3 text exactly as the Free
  Software Foundation publishes it, so GitHub and other tools recognize it. `NOTICE` holds
  the copyright line and the "or any later version" grant. The terms are unchanged:
  AGPL-3.0-or-later.

## 0.2.0 - 2026-11-02

### Fixed

- `build_launchd_plist` and `launchd_plist_path` now raise `ValueError` for a label outside
  the set a systemd unit name is held to (letters, digits, `.`, `_`, and `-`, starting with a
  letter or digit). A label such as `../../etc/x` used to produce a plist path outside
  `~/Library/LaunchAgents`. The command line was not affected: `--name` was already held to
  that set.
- `build_task_xml` now raises `ValueError` for a `task_name` that is not a task path: a
  leading `\`, then `\`-separated segments, each in that same set. A `..` segment, an empty
  segment or markup such as `<Evil/>` used to be accepted. The command line was not affected,
  since it builds the name from `--name`.

### Changed

- `--help` no longer prints `(default: False)` beside `--force`. When `--force` is omitted,
  the parsed value is now `None` rather than `False`; a caller of `build_parser` that tests
  it for truth sees no difference.
- The vendored `help_formatter` gains `UnsetHidingFormatter`, which this project does not
  use.

### Documentation

- The README has an install section that creates a virtual environment, with the Windows
  commands beside the POSIX ones, and the test instructions use it.
- A new README section removes a job from each scheduler: Task Scheduler, a systemd user
  timer, a launchd agent, and a crontab line.
- The launchd steps now show creating the `--log-dir` directory, which the tool does not
  create, and say where a `$HOME` in that option expands.
- Comments, docstrings, the `--help` description, and the vendored libraries' documents were
  reworded for readers new to the project.
- `pyproject.toml` no longer carries configuration for checks that do not ship with this
  repository.

## 0.1.0 - 2026-10-27

- Initial release.
