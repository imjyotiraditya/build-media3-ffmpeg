# patches

## ac4-decoder.patch

Adds `libavcodec/ac4dec.c` + `ac4dec_data.h` (AC-4 audio decoder) rebased onto ffmpeg `release/9.0`.

- Original code: Paul B Mahol (`onemda`), 2019, LGPL-2.1+. Submitted to the ffmpeg-devel mailing list, never merged upstream — publicly known reason is patent-risk objections, not code quality.
- Sourced from [funnymanva/ffmpeg-with-ac4](https://github.com/funnymanva/ffmpeg-with-ac4) (`ffmpeg_ac4.patch`, targeting ffmpeg 6.1), rebased here by hand: the two new files (`ac4dec.c`, `ac4dec_data.h`) applied as-is, the three small registration hunks (`Makefile`, `allcodecs.c`, `kbdwin.h`) redone manually against current `release/9.0` since context had drifted.
- Used only by `build-ac4.yml`, which is `workflow_dispatch`-only (never runs on push). Output artifact is named `FFmpeg-aar-ac4`, kept separate from the default `FFmpeg-aar` build.

**AC-4 is an active Dolby-patented codec.** This decoder has no license from Dolby. Building it for personal/local use is one thing; shipping the resulting AAR in a published, distributed app is a real legal exposure the app's author takes on directly. Keep it out of anything Play-published.
