# os-schedule-gen

A multi-OS scheduler file generator: write a schedule once and run it on Windows, Linux, and
macOS. It generates correct, safely escaped scheduler files for Windows Task Scheduler, systemd
timers, launchd plists, and crontab lines from one weekly or daily schedule.

It is a command line and a library. The command line renders one schedule into files in a
directory you name; the library hands the same artifacts back as strings and bytes. Neither
installs anything: registering each file with its scheduler is a step you take yourself, so
every platform's files can be generated, read, and tested on any one machine.

## Install

Needs Python 3.12 or newer. Install into a virtual
environment, because Homebrew and Debian Python refuse a
system-wide `pip install`:

```bash
python3 -m venv .venv
# shellcheck source=/dev/null # the activate script exists only once the line above has run
source .venv/bin/activate
pip install '.[test]'
```

The commands below assume the venv is active, so `python`
is the venv's interpreter.

On Windows, create and activate the venv this way instead, in `cmd.exe`. In PowerShell the
second line is `.venv\Scripts\Activate.ps1`, which runs only where scripts are allowed; for
the current window, `Set-ExecutionPolicy -Scope Process RemoteSigned` allows it:

```text
py -3 -m venv .venv
.venv\Scripts\activate
pip install ".[test]"
```

The tool, the library, and the two small libraries vendored beside them use the standard
library alone. The install adds only Hypothesis, which the tests need.

## Quick start

From the repository root:

```bash
mkdir -p out
python os_schedule_gen.py --name nightly-report --time 02:30 \
  --program /usr/bin/python3 --arg /opt/example/report.py \
  --arg=--mode --arg run --output-dir out
```

That writes a daily job to all four targets and prints one path per file written:

```text
out/nightly-report.xml
out/nightly-report.service
out/nightly-report.timer
out/nightly-report.plist
out/nightly-report.crontab
```

Add `--day Sunday` to make it weekly, and `--target systemd --target cron` (repeatable) to write
only some targets. A second run into the same directory exits 1 and writes nothing; add `--force`
to overwrite.

`samples/README.md` has two runs to check by hand against committed output: a weekly schedule
that sets every optional field, and a daily one that sets only the required ones.

## The four outputs, and installing them

Each file is named from `--name`, which is also the job's identity inside every scheduler: the
systemd unit basename, the launchd `Label`, and the task path `\<name>`. Installing is up to you;
these are the usual steps for each.

| File | Scheduler | Where it goes |
| --- | --- | --- |
| `<name>.xml` | Windows Task Scheduler | Imported, from an elevated prompt |
| `<name>.service`, `<name>.timer` | systemd user timer | `~/.config/systemd/user/`, or `$XDG_CONFIG_HOME/systemd/user/` when that is set |
| `<name>.plist` | launchd user agent (macOS) | `~/Library/LaunchAgents/` |
| `<name>.crontab` | cron | Pasted into your crontab |

**Task Scheduler.** The XML is UTF-16 with a byte-order mark, as `schtasks /XML` expects. The
task runs with least privilege, by default as `S4U` (whether or not you are signed in, with no
stored password); `--logon-type InteractiveToken` runs it only inside your signed-in session.
Creating a task needs an elevated prompt:

```text
schtasks /Create /TN "\nightly-report" /XML nightly-report.xml
```

**systemd.** The service is a `oneshot` (it runs and exits) and the timer attaches it to
`timers.target`. `Persistent=true` makes a machine that was asleep or off at the scheduled time
run the job at next boot instead of skipping it. Install it as a user unit, which needs no
elevation, into `~/.config/systemd/user/`, or `$XDG_CONFIG_HOME/systemd/user/` when that is
set:

```bash
mkdir -p ~/.config/systemd/user
cp out/nightly-report.service out/nightly-report.timer ~/.config/systemd/user/
systemctl --user daemon-reload
systemctl --user enable --now nightly-report.timer
```

A user timer fires only while your user's service manager runs; `loginctl enable-linger` keeps it
running when you are not logged in.

**launchd.** A user agent, not a system daemon, so it needs no elevation; it fires only while
you have a session. `RunAtLoad` is false, so loading the agent does not start the job:

```bash
cp out/nightly-report.plist ~/Library/LaunchAgents/
launchctl bootstrap "gui/$(id -u)" ~/Library/LaunchAgents/nightly-report.plist
```

When the plist was generated with `--log-dir`, create that directory before the job first runs.
The tool does not create it, because the plist is often generated on another machine than the
one it runs on. For the same reason, a `$HOME` in `--log-dir` is expanded by the shell on the
generating machine, so a plist meant for another Mac needs that Mac's absolute path spelled out.
Generated on the Mac that runs the job with `--log-dir "$HOME/Library/Logs/nightly-report"`,
for example, the directory is created with:

```bash
mkdir -p "$HOME/Library/Logs/nightly-report"
```

**cron.** The file holds one line. Open your crontab with `crontab -e` and paste it in. See
*Why cron is emit-only* for why nothing writes the crontab for you.

## Removing a job

Each scheduler keeps the job under its `--name`. To remove `nightly-report`:

**Task Scheduler**, from an elevated prompt:

```text
schtasks /Delete /TN "\nightly-report" /F
```

**systemd.** Stop and disable the timer, delete both unit files from the directory they were
copied into, then reload:

```bash
systemctl --user disable --now nightly-report.timer
rm ~/.config/systemd/user/nightly-report.service ~/.config/systemd/user/nightly-report.timer
systemctl --user daemon-reload
```

**launchd.** Unload the agent by its label, then delete the plist. A `--log-dir` directory, if
you created one, is yours to delete as well:

```bash
launchctl bootout "gui/$(id -u)/nightly-report"
rm ~/Library/LaunchAgents/nightly-report.plist
```

**cron.** Open your crontab with `crontab -e` and delete the line you pasted.

## Command line

```text
python os_schedule_gen.py --name NAME --time HH:MM --program PATH --output-dir DIR
                          [--day DAY] [--arg WORD]... [--target TARGET]... [--force]
                          [--description TEXT] [--working-directory DIR]
                          [--path-env PATH] [--log-dir DIR]
                          [--time-limit-minutes MINUTES] [--logon-type TYPE]
                          [--start-date YYYY-MM-DD]
```

| Option | Required | Default | Meaning |
| --- | --- | --- | --- |
| `--name` | yes | n/a | `[A-Za-z0-9][A-Za-z0-9._-]*` only: a leading `-` or `.`, and `.` and `..`, are refused. The systemd unit basename, the launchd `Label`, the task `\<name>`, and every file name. |
| `--time HH:MM` | yes | n/a | The 24-hour time it fires. |
| `--day DAY` | no | none: every day | `Monday` to `Sunday`. Given, the schedule is weekly; omitted, daily. |
| `--program PATH` | yes | n/a | The executable. |
| `--arg WORD` | no | none | One argument word, repeatable, in order. Write a word starting with `-` as `--arg=--mode`. |
| `--output-dir DIR` | yes | n/a | Must already exist. |
| `--target` | no | all four | `task-xml`, `systemd`, `launchd`, or `cron`; repeatable. |
| `--force` | no | off | Overwrite files that already exist. |
| `--description` | no | `Scheduled job <name>` | systemd `Description=` and the task's `Description`. |
| `--working-directory DIR` | no | none | The working directory in every artifact (cron enters it with `cd`); omitted, none is written and each scheduler uses its own default. Must be absolute, in the POSIX or the Windows sense, for every target; systemd and cron need a POSIX one. |
| `--path-env PATH` | no | none: inherit | The job's `PATH` under systemd, launchd, and cron. |
| `--log-dir DIR` | no | none | An absolute directory for launchd's `StandardOutPath` and `StandardErrorPath`, `<DIR>/<name>.out.log` and `.err.log`; omitted, both keys are left out and launchd discards the output. |
| `--time-limit-minutes` | no | 120 | A whole number from 1 to 525600 (one year): the task's `ExecutionTimeLimit`, written `PT<n>M` as given, and systemd's `TimeoutStartSec`, in seconds. |
| `--logon-type` | no | `S4U` | `S4U` or `InteractiveToken`; Task XML only. |
| `--start-date` | no | today | `YYYY-MM-DD`: the task's `StartBoundary` date, "not before". |

`--path-env` matters more than it looks: a scheduler starts a job with a minimal environment, not
your login shell's, so a program under a user-local `bin` directory is not found unless you pass
the `PATH` it needs.

Every target is rendered, and every destination checked, before the first file is written, so a
refusal or an existing file leaves the directory as it was. Each file is written through a
temporary file and an atomic rename, created with mode 0644 less the process umask (0600 under
`umask 077`).

### Exit codes

| Code | Meaning |
| --- | --- |
| 0 | Every file was written. |
| 1 | A destination already exists and `--force` was not given; nothing was written. |
| 2 | Bad input: an option refused by the parser, a malformed `--time`, a name outside the set, a missing `--output-dir`, a relative `--working-directory` or `--log-dir`, a value holding a line break, or a value a format cannot hold (named with its target and option). Also a file that could not be written; the files listed before it were. |

## The library

Import `os_schedule_gen` with its folder on `sys.path`; it puts its own `src/` there. Every
function is pure: no filesystem access, no subprocess, no configuration. The command line's
half is imported only by `build_parser` and `main`, so importing the library for its builders
never loads it.

```python
import os_schedule_gen as osg

weekly = osg.WeeklySchedule(weekday="Sunday", hour=3, minute=15)
service, timer = osg.build_systemd_units(
    weekly, "/usr/bin/python3", "/opt/example/backup.py", "/usr/local/bin:/usr/bin:/bin",
    unit_basename="weekly-backup", description="Weekly backup", args=("--mode", "run"),
)
```

| Name | Meaning |
| --- | --- |
| `WeeklySchedule(weekday, hour, minute)` | One weekly instant; every scheduler's notation is a property derived from it. |
| `DailySchedule(hour, minute)` | One instant that fires every day. |
| `parse_schedule(day, time_hhmm)` · `parse_daily_time(time_hhmm)` | The schedule from a `--day`/`--time` pair or a `--time` alone; `None` when the time is malformed. |
| `DAYS` | The weekday names, in the order Task Scheduler spells them. |
| `build_task_xml(schedule, python, script, start_date, *, task_name, description, args, working_directory, execution_limit, logon_type)` | Least-privilege Task Scheduler XML for a weekly or daily schedule. |
| `render_task_xml(*, task_name, description, triggers, logon_type, command, arguments, working_directory, execution_limit)` | Any least-privilege task, from a list of triggers. |
| `WeeklyAt` · `DailyAt` · `RepeatEvery` · `AtLogon` · `TaskTrigger` | The triggers `render_task_xml` renders, and their union. |
| `LOGON_TYPES` | `("S4U", "InteractiveToken")`. |
| `build_systemd_units(schedule, python, script, path_value, *, unit_basename, description, args, working_directory, time_limit_seconds)` | The `.service` text, then the `.timer` text. |
| `build_launchd_plist(schedule, python, script, state_dir, path_value, *, label, args, run_at_load, every_hour, working_directory, log_paths)` | The LaunchAgent plist, as bytes. |
| `build_cron_line(schedule, python, script, path_value, *, args, working_directory)` | The crontab line. |
| `systemd_unit_dir(home, environ=None)` · `launchd_plist_path(home, label)` | Where a systemd user unit and a user LaunchAgent plist live. |
| `select_backend(platform, *, has_systemd)` · `requires_elevation(backend)` · `BACKEND_*` | Which scheduler a platform installs into, and whether installing there needs elevation. |
| `build_parser()` · `main(argv=None)` | The command line's parser, and its entry point returning the exit code. |

*script* may be `None`, for a program run with its arguments alone; otherwise it is the second
command word, and the working directory defaults to its folder. A *path_value* of `""` leaves the
job's `PATH` inherited. Every keyword has a default, and the defaults describe one example job,
a weekly maintenance run: the task `\Maintenance\WeeklySweep`, the unit `weekly-sweep`, the
launchd label `local.weekly-sweep`, the arguments `--mode run`, and a two-hour time limit. A caller
scheduling anything else passes its own.

`render_task_xml` holds one `Settings` block for every task: it skips a run that would overlap
the one in progress, runs a missed start as soon as it can, and ignores battery state.
`RepeatEvery(minutes, start)` repeats every 1 to 1440 minutes indefinitely, and
`AtLogon(user_id)` fires when one `DOMAIN\user` signs in.

`build_launchd_plist(every_hour=True)` keeps only the schedule's minute, so the agent fires at
that minute of every hour, for a job that decides for itself whether a run is due. It is a
calendar trigger rather than `StartInterval` because launchd coalesces calendar firings missed
during sleep into one on wake, while it simply misses an interval firing that falls during
sleep.

## The output describes the target, never the host

Each artifact is a function of its target platform alone, so generating it on another system
gives the same bytes. Four places hold that rule, and each was a real defect or a near miss:

| Where | What goes wrong without it |
| --- | --- |
| `PurePosixPath` in the systemd and launchd working directories | Plain `Path` follows the host; generating on Windows emitted `\tmp\spaced dir`, which real systemd rejected as "path is not absolute" |
| `PureWindowsPath` in the Task Scheduler working directory | The mirror image: generating on Linux would put forward slashes into a Windows-only file |
| Fixed ASCII day names for systemd | A `strftime("%a")` render follows `LC_TIME` and can emit a localized day systemd cannot parse |
| An escape per format | systemd doubles `%`, cron backslashes it, and a plist needs neither, so one shared escape would be wrong for two of the three |

## What it refuses, and how it escapes

A path is data, and each scheduler must read it back as exactly that path.

- **Schedules.** A weekday outside `DAYS`, an hour outside 0 to 23, a minute outside 0 to 59,
  or a `bool` hour or minute raises `ValueError`. The Task XML uses the weekday as an element
  *name*, which no XML escape covers.
- **Line breaks.** A value holding a newline, a carriage return, or a NUL raises `ValueError` in
  the systemd, cron, and Task Scheduler builders: no escape keeps one inside a unit-file value or
  a crontab line, and it could start a key, or a job, of its own.
- **Task XML controls.** Any C0 control character other than tab, or DEL, raises `ValueError`
  naming the keyword: XML 1.0 cannot carry the C0 controls at all. `build_task_xml` also
  holds its `task_name` to a task path: a leading `\`, then `\`-separated segments in the
  set a unit name is held to, so a `..` segment, an empty one, or markup is refused.
- **systemd.** Each `ExecStart=` word is double-quoted, with `\` and `"` backslash-escaped, `%`
  doubled (a specifier) and `$` doubled (a variable). `Environment=` is quoted the same way but
  for `$`, which has no meaning there. `WorkingDirectory=` takes the rest of its line unquoted,
  so it gets only `%` doubled, and a directory that is relative, ends in `\`, or starts or ends
  in whitespace raises `ValueError`: systemd reads a leading `~` as the home directory and a
  leading `-` as "ignore if missing", joins a line ending in `\` to the next, and strips a
  value's surrounding whitespace. `Description=` follows the same unquoted rules. A unit name
  outside `[A-Za-z0-9._-]`, or not starting with a letter or digit, raises `ValueError`.
- **launchd.** The label is held to the same set as a unit name, in `build_launchd_plist` and
  `launchd_plist_path` alike: it names the plist file, and a `/` or `..` in it would put that
  file outside `LaunchAgents`.
- **cron.** Each value is one double-quoted `sh` word, with `$`, a backtick, `"`, and `\`
  backslash-escaped, then a backslash before each `%`, which crontab(5) otherwise reads as a
  newline. Not `shlex.quote`: cron keeps a backslash before anything but `%`, so a `\%` inside
  single quotes could not survive it. The captured `PATH` is an inline `PATH=... command` prefix,
  never a standalone `PATH=` line, which would change every job below it.
- **Arguments** are written bare only when they are a non-empty run of ASCII letters, digits, and
  `_ . / : = + , @ -`, which no format reads specially. Any other argument takes its format's
  quoting: systemd's and cron's above, and in the Task XML the Windows command-line form
  (`CommandLineToArgvW`): double-quoted, with a run of backslashes doubled before a `"` and at the
  end.
- **Batch files.** A `.bat` or `.cmd` program is not run by `CommandLineToArgvW`: `cmd.exe`
  parses its command line and treats `=`, `,`, and `;` as delimiters, so `--level=3` arrives as
  two words. The guarantee that each argument arrives as exactly the word given covers programs
  that split their command line the Microsoft C runtime's way, not batch files.
- **`%NAME%` in Task Scheduler.** Task Scheduler expands environment references in a task's
  command, arguments, and working directory, and no escape for that exists. A value holding one
  reaches the task as written, and the scheduler may expand it.

The tests check the meaning, not the text. They read each systemd unit back as INI after joining
a line that ends in an unescaped `\` to the next, as systemd does, then decode each field by a
model of systemd's rules for it. Each cron line goes through a model of cron's `%` rule, then a
real `sh`, which parses it and prints its words back under a `PATH` holding no command at all.
The Task XML's arguments are decoded by a model of `CommandLineToArgvW`, itself checked against
the standard library's `subprocess.list2cmdline`.

## Why cron is emit-only

A `.timer` or a `.plist` is a file per job, wholly written by this tool, so installing or
removing one touches only what it wrote. Installing a cron job is a read-modify-write of one
shared file that also holds your unrelated jobs, so a bug there would destroy entries this tool
never wrote. That asymmetry, not any difference in what cron can express, is why cron gets a line
to paste.

## Running the tests

From the repository root, with the venv from *Install* active:

```bash
python test_os_schedule_gen.py
python stdio_utf8/test_stdio_utf8.py
python help_formatter/test_help_formatter.py
```

Each test file runs its folder's suites and fails if a suite yields fewer tests than its floor.
A `# reliability: ignore[RLnnn]` comment in the source is a waiver for a reliability check run
before each release, with its reason on the same line; nothing here reads it.
Nothing is installed or scheduled: the command-line tests write into temporary directories, and
the cron tests run only `sh` builtins.

## Layout

| Path | What it holds |
| --- | --- |
| `os_schedule_gen.py` | the library's public surface, the builders, and the command line's parser |
| `src/` | the schedules, the escapes, the Task Scheduler renderer, the host facts, and the command line's work |
| `tests/` | the suites, one per concern |
| `test_os_schedule_gen.py` | runs every suite in `tests/`, and checks that each one was found |
| `samples/` | a weekly and a daily run to check by hand, with their expected files |
| `stdio_utf8/` | a small library that keeps the command line's output UTF-8 whatever the locale says |
| `help_formatter/` | the `--help` formatter, showing each option's live default |
| `export-record.json` | a generated record of the export that produced this tree: its source commit, its version, and a content hash per file. Regenerated with each export, never edited by hand |

## License

AGPL-3.0-or-later; see `LICENSE`.
