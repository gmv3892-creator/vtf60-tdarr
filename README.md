# vtf60 — 60 fps conversion with Apple VTFrameProcessor, built for Tdarr

Two small shell wrappers around [interp-harness][upstream] (from
[wasim-osman/60FPSVideo][upstream]) that turn it into something you can run
unattended over a whole library.

The upstream tool is a native macOS app and a CLI. It is fast — it interpolates
on the ANE — but a single invocation cannot process a full episode, and its raw
output does not line up with the original audio. These wrappers exist to solve
those two problems, and to plug the result into a Tdarr flow.

Measured on a MacBook Air M4 (16 GB): **~26 min for a 47-minute 1080p episode**,
output `hevc / yuv420p10le / 60 fps`, drift **under 0.05 s**.

---

## 1. Why segments

The harness never releases roughly **1.2 MB per output frame**. Measured while
converting a 1080p 23.976 source:

| output frames | resident memory |
|---|---|
| 4 500 | 5.1 GB |
| 10 800 (60 s segment) | 8.6 – 10 GB |
| 12 000, single process | 18 GB, swapping |

A single process dies at roughly **72 000–74 000 output frames** (about eight
minutes of source). Re-creating the framework session inside the process was
tried and measured: it does not help.

So `vtf60` cuts the input into **60-second segments** and runs each one in a
fresh process. Memory returns to a few megabytes between segments, and the
per-process limit never comes into play.

* 120-second segments already peaked at 8.7 GB, so 60 s is the ceiling.
* Segments are kept when a run fails and reused on the next attempt.
* Segments are never run in parallel: one process already peaks near 10 GB.

## 2. Why one extra frame per segment

The harness produces `ceil(2.5 × (N-1))` output frames for `N` input frames:

| input frames | output frames |
|---|---|
| 240 | 598 |
| 241 | 600 |
| 242 | 603 |
| 243 | 605 |

A 10-second segment therefore comes out two frames short, and a naive
concatenation accumulates a gap at every boundary — up to **1.5 s of A/V drift**
over a 46-minute episode.

Two things fix it:

1. request **N+1** source frames per segment, so the output lands on the nominal
   frame count;
2. write an explicit `duration <segment>.000` line into the concat list.
   Without it ffmpeg uses each segment's container length as the offset, which
   re-opens the gap.

Verified: 10 segments of 10 s concatenate to **5 997 frames with zero PTS gaps**.

## 3. Why the drift check

Even with the above, the last segment can end a frame early. `vtf60-tdarr`
measures container length against the last video PTS and, if the difference is
above 0.1 s, rescales the video timestamps with ffmpeg's `setts` bitstream
filter:

    ffmpeg -i in.mkv -map 0 -c copy -bsf:v "setts=pts=PTS*R:dts=DTS*R" out.mkv

`R = container_duration / last_video_pts` is typically 1.0002–1.0006, i.e. the
video is stretched by a few hundredths of a percent. Nothing is re-encoded, so
quality and file size are unchanged.

## 4. Usage

    # direct
    ./vtf60 input.mkv output.mkv

    # with drift correction, in the shape a Tdarr plugin calls it
    ./vtf60-tdarr input.mkv output.mkv --target-fps 60

Environment variables:

| variable | default | meaning |
|---|---|---|
| `VTF_HARNESS` | `interp-harness` | path to the harness binary |
| `VTF_SEGMENT_SECONDS` | `60` | segment length |
| `VTF_WORKERS` | `4` | workers inside a segment |
| `VTF_PARALLEL` | `1` | segments at once (keep at 1) |
| `VTF_DRIFT_LIMIT` | `0.1` | seconds before timestamp correction |
| `VTF_KEEP_TMP` | `0` | keep segments after success |

### Worker count

Measured on 30 s of source, single segment:

| workers | time | memory |
|---|---|---|
| 1 | 36 s | ~3 GB |
| 2 | 17 s | ~6 GB |
| **4** | **14 s** | ~10 GB |
| 6 | 15 s | — |
| 8 | 17 s | — |

Four is the sweet spot; more workers queue on the framework's internal slots.

## 5. Tdarr integration

These wrappers were written for a Tdarr 3.01 flow that converts only files
carrying a `<file>.fps60` marker:

    Input File -> FPS60 Gate + Convert
                    | marker?  no -> Not required
                    | yes
                    v
           vtf60-tdarr <src> <out> --target-fps 60
                    v
           replace original in place, delete marker

Point the plugin's **CLI path** at `vtf60-tdarr` and leave its extra arguments
empty. The node needs one transcode worker, and no other jobs should run at the
same time (memory).

## 6. Caveats

* **File names with a single quote.** `It's a Deal` breaks ffmpeg's concat
  list. Both wrappers escape it; keep that line if you fork this.
* **Test the first job on a short clip.** Throughput and memory depend on
  resolution, bit depth and the encoder profile in use.
* **Not a backup tool.** The wrapper replaces files in place; keep your own
  copies of anything irreplaceable.
* Tested only on macOS with Apple silicon. The ANE path is the whole point of the
  upstream tool; there is no CPU fallback here.

## 7. Requirements

* macOS on Apple silicon
* interp-harness built from source (`swift build -c release`)
* `ffmpeg`, `ffprobe`, `python3`

## 8. License

MIT for these scripts. The harness is a separate project with its own license.

[upstream]: https://github.com/wasim-osman/60FPSVideo
