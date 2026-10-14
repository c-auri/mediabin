# conversion commands

Spec for the three commands that replace `deweb`. Written against the conventions in `design.md`; where it settles something `design.md` leaves open, that is called out at the end.

## What replaces what

`deweb` did two unrelated conversions, over the whole current directory, deleting each original on success. It is replaced by three commands that each do one conversion, take the files they are given, and write next to the input without touching it.

```
mb fromwebp <file>...   webp    -> jpg or png (animated webp -> mp4)
mb fromgif  <file>...   gif     -> mp4
mb togif    <file>...   video   -> gif
```

Deletion is no longer part of any of them. `rm` is the follow-up, which is both safer and finer grained: the batch that just ran is visible in the shell history above it, and nothing is removed by a command whose job was to write a file.

The pairing is that `from<format>` means "get this file out of a bad format", with the target implied by what the file actually contains, and `to<format>` means "produce this format", with the source being anything ffmpeg can read. `gif` appears on both sides because it is a bad container and a useful deliverable at the same time.

## Arguments

Every one of the three takes one or more input paths:

```
mb fromwebp shot.webp
mb fromwebp a.webp b.webp c.webp
mb fromwebp *.webp
mb fromwebp img/*.webp
```

The glob is the shell's work, so the third and fourth forms cost nothing to support beyond looping over `"$@"`. Three details do cost something:

1. A glob that matches nothing arrives as its own literal text, since the interactive shell has no `nullglob` set. `mb fromwebp *.webp` in a directory with no webp files must report that `*.webp` does not exist, in those words, rather than handing the string to ImageMagick and printing whatever comes back.
1. `mb togif *` is a thing people type. An input the command cannot handle is skipped with a warning and counted as a failure, not treated as a fatal error that abandons the rest of the batch.
1. Output paths are built from the input path, not the working directory, so `mb fromwebp img/*.webp` writes into `img/`.

No arguments at all is a usage error: print usage to stderr and exit 2.

## Output paths

The output goes next to its input, under the input's own name with the extension replaced:

```
img/shot.webp  ->  img/shot.jpg
my.photo.gif   ->  my.photo.mp4
screenshot     ->  screenshot.mp4
CLIP.MP4       ->  CLIP.gif
```

Everything after the last dot in the basename is the extension and is dropped; a basename with no dot keeps all of it and gains one. The new extension is always lowercase, whatever case the input used. This is the rule `rotate` already implements, and splitting on the last dot rather than the first is precisely what `convertwebp` got wrong.

`-o FILE` overrides the derived path. It is only meaningful for a single input, so combining it with more than one is a usage error: exit 2 without converting anything. With `-o` the path is used exactly as given, extension included.

An output that resolves to the input itself is a per-file failure, and `-force` does not override it. The extension rule alone cannot produce that, but `-o` can, and so can a file whose extension disagrees with its contents.

## Batch semantics

A batch keeps going after a failure. Aborting on the first bad file is wrong for a command whose normal invocation is a glob over a directory: it leaves the run half done at an unpredictable point, and re-running it has to redo the work that already succeeded.

So each file is attempted independently, failures are reported as they happen, and a summary line closes the run when more than one file was given:

```
3 converted, 1 failed
```

Each converted file reports one line, in the shape `rotate` already uses at `commands/rotate:250`:

```
wrote img/shot.jpg (1920x1080, 340K)
wrote clip.mp4 (640x480, 1.2M)
wrote clip.gif (480x270, 15 fps, 4.2M)
```

Dimensions and size are read back off the finished file, so the line describes what was actually written rather than what was asked for. `togif` adds the frame rate because that is its one knob.

Exit status is 0 when every file succeeded, 1 when any file failed, and 2 for a usage error or a missing dependency, which are conditions that stop the command before it converts anything. Usage errors are no arguments at all, an unknown flag, and `-o` with more than one input.

The exit code is what makes the deletion follow-up correct. `mb fromwebp *.webp && rm *.webp` deletes only when every conversion in the batch worked, which is the guarantee the old inline `&& rm` gave per file, now given per batch and opted into by hand.

## Overwriting

Not deleting the input is not the same as being safe. Writing `shot.jpg` when `shot.jpg` already exists is the one destructive act left, and it is the likely one: it happens on the second run of a batch that partly failed the first time.

An existing output is a per-file failure with a message naming the file, unless `-force` is given, which is the flag `rotate` already uses for the same purpose.

## fromwebp

The one command of the three with a real decision in it. Three different things ship as `.webp`, and the right output differs for each:

1. **Lossy, no alpha.** The common case, a photo saved by a website. jpg.
1. **Lossless, or any alpha channel.** Screenshots, logos, UI captures. jpg is wrong twice here: it flattens transparency onto a background nobody chose, and it adds generation loss to an image that had none. png.
1. **Animated.** Not a still image at all. ImageMagick will write `shot-0.jpg`, `shot-1.jpg` and so on, one file per frame, which is never what was wanted. This produces mp4, but it cannot reuse the `fromgif` path: ffmpeg's webp decoder handles single still images only and fails outright on an animation with `image data not found`, and none of the libwebp tools that would help (`anim_dump`, `webpmux`, `dwebp`) are installed. ImageMagick reads the frames correctly, so the animation goes through two stages instead: `magick <input> -coalesce` writes the frames into a `mktemp -d` as png, and ffmpeg encodes that sequence with the same settings `fromgif` uses. `magick` can write the mp4 by itself, but it discards the source timing: a three frame source came back as 21 frames at 25 fps.

The third case stays in `fromwebp` rather than being refused. `from<format>` means "get this file out of a bad format", and the target following from what the file actually contains is the whole point of the name; refusing would leave animated webp as the one format in the set with no command that takes it.

The branch is decided by probing the input before converting:

```
magick identify -format '%n|%A|%Q\n' "$input" | head -1
```

giving frame count, alpha type and quality. Checked against real files written by `magick` 7.1.2, the three fields read like this:

```
lossy, no alpha        1|Undefined|92
lossy, alpha           1|Blend|92
lossless, no alpha     1|Undefined|100
animated               3|Undefined|92
```

So: frame count above 1 selects the mp4 path, alpha reported as anything other than `Undefined` selects png, and quality 100 selects png as the signal for a lossless source.

That last one is the shaky part of the rule. `%Q` is not read off the webp header; it is what ImageMagick infers, and it infers 92 for every lossy file regardless of the quality it was actually encoded at. The one thing it does distinguish is lossless, which reads 100, so the rule works. But a lossy file encoded at quality 100 also reads 100 and lands in png too. That misfire costs disk space and nothing else, which is the right direction for the error to point.

`-jpg` and `-png` override the choice for a whole batch. They do not override the animated case, which has no still-image answer to give.

The format field matters as much as the other three. `magick identify` succeeds on any image it can read, so without checking `%m` for `WEBP` a png handed to `fromwebp` is cheerfully converted to jpg rather than skipped.

Frame timing for an animation comes from `magick identify -format '%T'`, which reports centiseconds per frame. The rate is taken from the first frame and applied to all of them, passed to ffmpeg as the exact fraction `-framerate 100/<delay>` so no float arithmetic is involved. A source whose frames have different delays is converted at the first frame's rate with a warning saying so; getting that exactly right needs the concat demuxer and a per-frame duration list, which is not worth it until a real file needs it. A delay of 0 is legal in the format and means "as fast as possible", so it is clamped to 10cs the way browsers do.

The jpg encode uses a fixed quality (92) rather than trying to match the source, since webp and jpeg quality scales do not mean the same thing.

## fromgif

The conversion `deweb` already did, unchanged except for where it gets its inputs and that it no longer deletes:

```
ffmpeg -y -i "$gif" \
    -v warning \
    -stats \
    -movflags faststart \
    -pix_fmt yuv420p \
    -vf "scale=trunc(iw/2)*2:trunc(ih/2)*2" \
    "$mp4"
```

`-pix_fmt yuv420p` for player compatibility, the `scale` expression to force even dimensions because libx264 rejects odd ones, `faststart` to put the metadata at the front. The `-y` becomes conditional on `-force`, since the overwrite rule above now governs it.

The input must actually be a gif, by the probe below rather than by its extension. Anything else is skipped with a warning and counted as a failure, so `mb fromgif *` in a mixed directory converts the gifs and leaves everything else alone instead of quietly re-encoding a mov.

## togif

Video to gif, for anything ffmpeg can read that is not already a gif. The naive `ffmpeg -i in.mp4 out.gif` quantises against a fixed 216-colour palette and looks like it, so this generates a palette from the source in the same pass:

```
ffmpeg -i "$input" -filter_complex \
    "fps=$fps,split[a][b];
     [a]palettegen[p];[b][p]paletteuse" \
    "$output"
```

Dimensions and aspect ratio are left exactly as the source has them. Only when `-width` is given does a scale stage appear, between `fps` and `split`:

```
scale=$width:-1:flags=lanczos,
```

Single pass with `split` rather than a palette written to a temp file. The two-pass form only earns its temp file when `palettegen=stats_mode=diff` is wanted for a mostly-static source, which is a refinement to add if a real clip needs it, not a default.

Both knobs are exposed, because a gif with the wrong frame rate or the wrong width is the whole failure mode of the format:

```
-fps N       frames per second (default 15)
-width N     scale to this width, height following the aspect ratio (default: no scaling)
```

Size is the thing that surprises people here, so the closing line per file reports it:

```
wrote clip.gif (480x270, 15 fps, 4.2M)
```

A gif input is nonsensical here and is skipped as a wrong input type, the same way `fromgif` skips a non-gif. A file with no video stream at all is skipped too, which is the check that makes `mb togif *` behave. A still image is not an error: it converts to a single-frame gif without comment.

## Conventions

All three follow `design.md`: executable, `#!/usr/bin/env bash`, `set -euo pipefail`, a `summary=` line the dispatcher reads, usage text built from `prog=${MB_PROG:-${0##*/}}`, and `-h`.

Flags are single-dash multi-character, following ffmpeg and `rotate` rather than GNU. They cannot be bundled, which is fine at this size. `--` ends options so a file named `-thing.webp` can still be passed.

Dependencies are checked once before the batch starts, not per file, and a missing one exits 2 without converting anything. `fromgif` and `togif` each require both `ffmpeg` and `ffprobe`.

Input type is decided by one ffprobe call, reading the video stream rather than the container:

```
ffprobe -v error -select_streams v:0 -show_entries stream=codec_name -of csv=p=0 "$input"
```

It prints `gif` for a gif, the codec name for anything else with a picture in it (`h264`, `png`, `webp`), and nothing at all for a file with no video stream, which is also what an unreadable file gives. Container-level `format_name` is no use for this: it reports `mov,mp4,m4a,3gp,3g2,mj2` for an mp4 and `png_pipe` for a png. The call exits 0 in the empty case, so the test is whether the output is empty, not whether the command failed.

`fromgif` requires that codec to be `gif`. `togif` requires it to be present and not `gif`.

The ImageMagick backend is `magick`, ImageMagick 7, at `~/.local/bin/magick`. There is an older 6.9.11 at `/usr/bin/convert`, which is what `deweb` picked up by being unspecific, but IM7 deprecates the `convert` name and the two differ in what `%A` reports and in how they handle a multi-frame file. `fromwebp` checks for `magick` by name and exits 2 if it is missing, rather than falling back to `convert` and quietly probing with different semantics than this spec describes.

## Loose ends

**Overlap with `loop`.** Noted, to revisit later, not a blocker for writing these three. `loop` normalises a gif to a lossless mkv as its first stage, which is `fromgif` with a different codec, and then boomerangs it. Two commands quietly sharing a first step is how the collection starts drifting, and it is also the second candidate for the `lib/` that `design.md` says to create only once something is actually shared. Neither script changes on account of it for now; when `loop` is next touched, decide then whether the shared stage moves.

**The naming rule in `design.md`.** Settled in favour of the names. The "three short one-word verbs, four to six letters" line was a description of the three scripts that happened to be there, not a constraint anything was chosen against, and holding to it would cost clarity for no benefit. `design.md` should be edited to say what the names are for, that a command's name says what it does to a file, rather than to prescribe a shape they must fit.

**`rotate` exits 1 on a usage error.** The convention above is 2, and `rotate` predates it: `commands/rotate:145-149` prints usage to stderr and exits 1 when given no input. Worth changing when `rotate` is next touched. `design.md` says nothing about exit codes at all and should record the convention.

**The `deweb` keep flag.** No longer an open question. It was the only guardrail available against a command that deleted, and nothing deletes now.
