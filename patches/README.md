# patches

## ac4-decoder.patch

Adds `libavcodec/ac4dec.c` + `ac4dec_data.h` (AC-4 audio decoder) rebased onto ffmpeg `release/9.0`.

- Original code: Paul B Mahol (`onemda`), 2019, LGPL-2.1+. Submitted to the ffmpeg-devel mailing list, never merged upstream — publicly known reason is patent-risk objections, not code quality.
- **Current source (2026-07-23): [librempeg/librempeg](https://github.com/librempeg/librempeg)**, Mahol's own actively-maintained fork he's continued developing since leaving mainline ffmpeg. Its `ac4dec.c` has 81 commits of fixes beyond the 2019 drop (array sizing bugs, immersive/object-coding fixes, etc.) as of this port. **The fork is GPLv3, not LGPL** — this decoder's license changed accordingly (see file header). That's a materially different, more restrictive condition than the original LGPL-2.1+ drop; on top of the existing Dolby patent risk below, treat this as copyleft-encumbered code.
- The 2019/funnymanva-sourced version was tried first and is **confirmed broken**: it decodes the first (i-)frame of a real-world file correctly but fails every frame after with `sect_cb > 11` (bitstream desync), verified independently via a host-built `ffmpeg`/`ffprobe` from this exact tree against a real 7.1/immersive AC-4 file — reproduced identically on both the 2019 code and, at first, this fork's code too, until it turned out that second test was accidentally re-running a stale cached binary (a broken `Makefile` edit had silently failed the rebuild). Once genuinely rebuilt, librempeg's version decodes the full file cleanly (RMS/peak levels confirm real audio, not silence).
- Porting notes: the two new files applied with zero API drift against `release/9.0` (librempeg tracks upstream closely) except one function (`ff_vector_fmul_reverse_c`, an internal symbol not exposed via header in mainline) — replaced with the equivalent `s->fdsp->vector_fmul_reverse` vtable call, which is what the SIMD-aligned branch already used; the special-cased branch existed purely as a micro-optimization, not for correctness. Same three small registration hunks (`Makefile`, `allcodecs.c`, `kbdwin.h`) as before.
- `ac4_parser.c` also exists upstream (a small `AVCodecParser` for key-frame/sample-rate detection) but isn't included — not needed for decode-only use, and not registered anywhere in this patch.
- Used only by `build-ac4.yml`, which is `workflow_dispatch`-only (never runs on push). Output artifact is named `FFmpeg-aar-ac4`, kept separate from the default `FFmpeg-aar` build.

**AC-4 is an active Dolby-patented codec, and this code is now GPLv3.** Neither condition is satisfied here. Building it for personal/local use on your own device is one thing; distributing the resulting AAR or an APK built with it to anyone else is a real legal exposure (patent + copyleft) the app's author takes on directly. Keep it out of anything Play-published, and be deliberate about who else ever receives a build containing it.

## media3-resample-fix.patch

Patches `androidx/media`'s own `libraries/decoder_ffmpeg/src/main/jni/ffmpeg_jni.cc` (Apache 2.0, no license concern) — a separate, real bug from the decoder itself.

Media3's JNI bridge creates one `SwrContext` (libswresample resampler) on the first decoded frame and caches it in `AVCodecContext.opaque`, reusing it for every subsequent frame with no check that the format it was built for still matches. Fine for codecs with a static output format for the whole stream. **Not fine for AC-4**: `avctx->ch_layout`/`sample_rate`/`sample_fmt` are only known after parsing each frame's own TOC (this codec supports multiple presentations, e.g. an immersive mix plus a stereo fold-down, and can legitimately change format frame to frame). If a later frame's format differs from whatever got cached on frame 1, `swr_convert` silently resamples with stale settings — no error, no crash, just wrong/silent output.

Symptom this fixes: decoder verifiably produces correct, non-silent PCM (confirmed via a host-built `ffmpeg` decoding the same file with RMS/peak levels matching real audio) and Android does open a real `AudioTrack` for it, but nothing audible comes out on-device.

Fix: replaced the raw `SwrContext*` cached in `opaque` with a small `ResampleState` struct that also remembers the channel count/sample rate/sample format the resampler was built for; `decodePacket()` now compares the current frame's format against that cache on every call and rebuilds the `SwrContext` (via `swr_free` + `swr_alloc_set_opts2` + `swr_init`) whenever it differs, instead of only once. `releaseContext()` updated to free the struct correctly.

Applied in `build-ac4.yml` against the exact `androidx/media` `release` branch checkout, after the ffmpeg decoder patch, before building.
