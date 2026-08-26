# mediabin design

## What this is

Personal shell wrappers around ffmpeg and ImageMagick, collected into one directory so they can live in version control and share a naming scheme. The `bin` in the name does double duty: a grab bag of tools, and the fact that the dispatcher is symlinked into `~/.local/bin`.

Every command takes a file, transforms it, and writes the result to disk. That uniformity is the defining property of the collection and the reason it can share one entrypoint.

## Naming

Three short one-word verbs, four to six letters, tab-completing after two characters.

| old         | new      | backend    |
|-------------|----------|------------|
| `ffrotate`  | `rotate` | ffmpeg     |
| `gif-loop`  | `loop`   | ffmpeg     |
| `cnvrt`     | `deweb`  | both       |

The names can be plain and literal because nothing here plays anything: with no player in the set, `loop` cannot be read as a playback mode nor `rotate` as "play this rotated". That is what the exclusion above buys, and the reason it is worth holding to.

`deweb` covers webp to jpg exactly and gif to mp4 loosely, since gif is more an awkward legacy format than a web one. It names the blast radius rather than the mechanism, and still undersells it: the command sweeps the whole current directory and deletes each original after a successful conversion. The old name was vowel-stripped to dodge `convert`, ImageMagick's own binary. Nothing in `commands/` is ever on `PATH`, so that pressure is gone.

## Architecture

A single dispatcher named `mb` is the only thing that reaches `PATH`. Subcommands are ordinary executables in `commands/`, invoked as `mb rotate -cw clip.mp4`.

```
mediabin/
  mb                 # dispatcher
  commands/
    rotate loop deweb
  lib/               # shared helpers, only once something is actually shared
  design.md
  README.md
```

`~/.local/bin/mb` is a symlink to `mediabin/mb`. Since `~/.local/bin` is already on `PATH`, that is the entire installation: no `PATH` edit, no new directory entry, no chance of a subcommand name colliding with a system binary.

### Dispatcher behaviour

1. Resolve its own directory through the symlink with `readlink -f`, so `commands/` is found no matter where `mb` is invoked from.
1. Reject any subcommand name containing `/` before touching the filesystem, so `mb ../../something` cannot escape `commands/`.
1. `mb <cmd> [args...]` execs `commands/<cmd>` with the remaining arguments. Using `exec` means the subcommand's exit code, signals and stdio are the dispatcher's, with no wrapper process left in the tree.
1. `mb help` prints the command list on stdout and exits 0, so it can be piped and grepped. Each line is a name plus the `summary` variable read out of the script, without executing it.
1. Bare `mb` prints the same list on stderr and exits 1, because invoking with no arguments is a usage error.
1. `mb help <cmd>` is rewritten to `commands/<cmd> -h`. This form gets tried first by most people, so it is worth the three lines. `mb <cmd> -h` already works for free, since arguments pass straight through.
1. An unknown subcommand prints an error plus the available names on stderr and exits 127, matching the shell's own "command not found".

### Conventions every subcommand follows

Executable, `#!/usr/bin/env bash`, `set -euo pipefail`, and support for `-h`.

A one-line `summary` variable near the top, which the script's own usage text prints as its first line:

```bash
summary="rotate a video 90 degrees, with a preview mode"

function usage
{
  cat <<EOF
$prog: $summary

Usage:
  ...
EOF
}
```

`mb help` reads it back out with `sed -n 's/^summary="\(.*\)"$/\1/p'`, so one string feeds both the listing and the command's own `-h`, and the two cannot drift. Adding a script to `commands/` is the same act as adding it to the help; a hardcoded list inside `mb` would rot the first time a command is added in a hurry.

Reading the variable rather than executing `<cmd> --help` to collect it is deliberate. Building the listing by running every script would oblige each one to short-circuit its help before any setup, dependency check or `source`, forever, and would take the whole listing down when one script fails to parse: `mb help` would break exactly when it is being used to find out what is wrong. Startup cost is not the reason; three subshells is about five milliseconds.

Usage text derives the program name from the environment rather than from `$0`: `prog=${MB_PROG:-${0##*/}}`, with the dispatcher exporting `MB_PROG="mb rotate"` before the exec. Without this, `rotate -h` under the dispatcher would tell the reader to run `rotate -cw input.mp4`, which is not a command that exists. The fallback keeps the output correct when a script is run directly out of `commands/` during debugging.

## Open questions

**Interface consistency across the three.** The scripts disagree about how to take input, because they were written separately. `rotate` uses single-dash multi-character flags, following ffmpeg's convention rather than GNU's. `loop` takes two positional arguments plus `FPS` and `CRF` as environment variable overrides. `deweb` takes nothing at all and operates on the current directory. Under a single entrypoint these read as one tool with three inconsistent interfaces, and every script added from here widens the gap. Worth settling a house style before the fourth one lands; the flag convention is the expensive part to change later.

**A `lib/` for shared helpers.** `rotate` carries its own `die()` and `run()`. Nothing else needs them yet. Extract only when a second script does, following the `git-lib` precedent next door.

**A `deweb` keep flag.** `deweb` deletes originals after a successful conversion. A `-k` flag would cover the cases where that is not wanted, and is the only real guardrail available: a naming scheme cannot warn about deletion.

## Still to do

1. Write `mb`.
1. Bring each script up to the conventions above, one pass per file. `rotate` and `deweb` are still `#!/bin/bash`; `deweb` uses `set -e` and needs `set -euo pipefail`, keeping its `shopt -s nullglob`, which is what makes the loops skip when nothing matches. All three need the `summary` variable printed from their usage text, and `deweb` has no usage function at all yet. `rotate` also switches from `prog=${0##*/}` to the `MB_PROG` form.
1. Fix the usage text in `loop`, which hardcodes the string `gif-loop.sh` in three places including its examples.
1. Symlink `~/.local/bin/mb` to `mediabin/mb` and verify each command still runs.
1. Write `README.md`: the three commands, one line each, plus the installation symlink and the conventions a fourth script would need to follow.

## Future
1. Tab completion is a nice-to-have. Not in the scope of initial setup but worth working on eventually.
