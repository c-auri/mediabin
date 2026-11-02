# mediabin design

## What this is

Personal shell wrappers around ffmpeg and ImageMagick, collected into one directory so they can live in version control and share a naming scheme. The `bin` in the name does double duty: a grab bag of tools, and the fact that the dispatcher is symlinked into `~/.local/bin`.

These are editing utilities, not playback. Every command takes file paths on the command line and either transforms a file and writes the result to disk, or reports on what a file contains. Playback is a distinct category of operation on video and images, and it is the one thing that does not belong here: a command that opens a window belongs in `~/.local/bin` as a standalone executable instead.

## Naming

A name should hint at what the command produces. That is the whole rule: there is nothing to say about length or parts of speech, and `info` reads as clearly as `resize` does.

| old         | new      | backend    |
|-------------|----------|------------|
| `ffrotate`  | `rotate` | ffmpeg     |
| `gif-loop`  | `loop`   | ffmpeg     |
| `cnvrt`     | `deweb`  | both       |

The names can be plain and literal because nothing here plays anything: with no player in the set, `loop` cannot be read as a playback mode nor `rotate` as "play this rotated". That is what the exclusion above buys, and the reason it is worth holding to.

`deweb` covers webp to jpg exactly and gif to mp4 loosely, since gif is more an awkward legacy format than a web one. It names the blast radius rather than the mechanism, and still undersells it: the command sweeps the whole current directory and deletes each original after a successful conversion. The old name was vowel-stripped to dodge `convert`, ImageMagick's own binary. Nothing in `commands/` is ever on `PATH`, so that pressure is gone. `fromwebp`, `fromgif` and `togif` replace it: each does one conversion, takes the files it is given, and deletes nothing.

The conversion commands pair up by prefix. `from<format>` means "get this file out of a bad format", with the target implied by what the file actually contains. `to<format>` means "produce this format", with the source being anything ffmpeg can read. `gif` appears on both sides because it is a bad container and a useful deliverable at the same time.

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

### Files in, files out

Commands that write files share one shape for arguments, output, failure and exit status.

**Arguments.** One or more input paths, so `mb togif *.mp4` and `mb togif img/*.mp4` cost nothing beyond looping over `"$@"`. No arguments at all is a usage error. A glob that matches nothing arrives as its own literal text, since the interactive shell has no `nullglob`, and is reported as not existing, in those words, rather than handed to ffmpeg. Flags are single-dash multi-character, following ffmpeg rather than GNU, and cannot be bundled. `--` ends options so a file named `-thing.webp` can still be passed, and every path handed to ffmpeg or `magick` gets a `./` in front if it starts with a dash.

**Output paths.** The output goes next to its input, so `img/shot.webp` writes into `img/`, not the working directory. Everything after the last dot in the basename is the extension and is replaced; a basename with no dot gains one, and a dotfile keeps its whole name. A command that keeps the container, like `mute`, adds a suffix before the extension instead. `-o FILE` overrides the derived path, is used exactly as given, and is a usage error with more than one input.

**Overwriting.** Nothing deletes its input. Removing originals is a separate step gated on the exit status, `mb fromwebp *.webp && rm *.webp`, which deletes only when every conversion in the batch worked. Writing over an existing output is the one destructive act left, and the likely one: it happens on the second run of a batch that partly failed the first time. An existing output is a per-file failure unless `-force` is given. An output that resolves to the input itself is a per-file failure that `-force` does not override; the extension rule cannot produce one, but `-o` can.

**Batches.** A batch keeps going after a failure. Aborting on the first bad file is wrong for a command whose normal invocation is a glob: it leaves the run half done at an unpredictable point. An input the command cannot handle is skipped with a warning and counted as a failure, so `mb togif *` in a mixed directory converts what it can. Each success prints one line, read back off the finished file so it describes what was written rather than what was asked for, and a summary closes any run of more than one file:

```
wrote clip.gif (480x270, 15 fps, 4.2M)

3 converted, 1 failed
```

**Exit status.** 0 when every file succeeded, 1 when any failed, 2 for a usage error or a missing dependency, which stop the command before it touches any file. Dependencies are checked once before the batch, not per file.

**Probing input type.** Decided by one ffprobe call on the video stream rather than the container, whose `format_name` reports `mov,mp4,m4a,3gp,3g2,mj2` for an mp4 and `png_pipe` for a png:

```
ffprobe -v error -select_streams v:0 -show_entries stream=codec_name -of csv=p=0 "$input"
```

It prints `gif` for a gif, the codec name for anything else with a picture in it, and nothing for a file with no video stream or an unreadable one. It exits 0 in that empty case, so the test is whether the output is empty.

**ImageMagick is `magick`.** ImageMagick 7, at `~/.local/bin/magick`. The 6.9.11 at `/usr/bin/convert` differs in what `%A` reports and in how it handles a multi-frame file, so a command needing ImageMagick checks for `magick` by name and exits 2 if it is missing, rather than falling back and probing with different semantics.

## Command notes

Decisions behind individual commands that are not visible from the code.

**fromwebp.** An animated webp cannot reuse the `fromgif` path: ffmpeg's webp decoder handles still images only and fails on an animation with `image data not found`, and none of the libwebp tools that would help (`anim_dump`, `webpmux`, `dwebp`) are installed. `magick` reads the frames correctly but discards the source timing when it writes mp4 itself: a three-frame source came back as 21 frames at 25 fps. So frames go through a temp directory as png and ffmpeg encodes them at a rate taken from the first frame's delay. A source whose delays vary gets that rate with a warning; doing it exactly needs the concat demuxer and a per-frame duration list, which is not worth it until a real file needs it. Animations stay in `fromwebp` rather than being refused, since refusing would leave animated webp as the one format with no command that takes it. The jpg quality is fixed at 92 rather than matched to the source, since webp and jpeg quality scales do not mean the same thing.

**togif.** Single pass with `split` rather than a palette written to a temp file. The two-pass form only earns its temp file when `palettegen=stats_mode=diff` is wanted for a mostly-static source, which is a refinement to add when a real clip needs it. Dimensions are untouched unless `-width` is given.

## Open questions

**Older scripts against the house style.** "Files in, files out" settles the interface for new commands, but the scripts that predate it still disagree. `loop` takes two positional arguments plus `FPS` and `CRF` as environment variable overrides. `deweb` takes nothing at all and operates on the current directory. `rotate` exits 1 rather than 2 on a usage error. Each is worth bringing in line when it is next touched.

**Overlap with `loop`.** `loop` normalises a gif to a lossless mkv as its first stage, which is `fromgif` with a different codec, before boomeranging it. Two commands quietly sharing a first step is how the collection starts drifting. When `loop` is next touched, decide whether the shared stage moves.

**A `lib/` for shared helpers.** `rotate` carries its own `die()` and `run()`. Nothing else needs them yet. Extract only when a second script does, following the `git-lib` precedent next door.

## Still to do

1. Write `mb`.
1. Bring each script up to the conventions above, one pass per file. `rotate` and `deweb` are still `#!/bin/bash`; `deweb` uses `set -e` and needs `set -euo pipefail`, keeping its `shopt -s nullglob`, which is what makes the loops skip when nothing matches. All three need the `summary` variable printed from their usage text, and `deweb` has no usage function at all yet. `rotate` also switches from `prog=${0##*/}` to the `MB_PROG` form.
1. Fix the usage text in `loop`, which hardcodes the string `gif-loop.sh` in three places including its examples.
1. Symlink `~/.local/bin/mb` to `mediabin/mb` and verify each command still runs.
1. Write `README.md`: the three commands, one line each, plus the installation symlink and the conventions a fourth script would need to follow.

## Future
1. Tab completion is a nice-to-have. Not in the scope of initial setup but worth working on eventually.
