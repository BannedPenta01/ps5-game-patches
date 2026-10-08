# KytyPS5 video-decode optimizations

Developed while optimizing ASTRO's PLAYROOM cutscenes. Applies on top of
KytyPS5 `bd4fcda` (ships in the `KytyPS5-2026-10-08` build).

## `kyty-video-decode-optimizations.patch`

Two changes, both in the software video path used by `sceAvPlayer` game
videos and `sceVideodec2`:

1. **Multithreaded decode** (`src/libs/avPlayer.cpp`,
   `src/libs/videoDec2Decoder.cpp`) — codecs were opened with ffmpeg
   defaults (`thread_count=1`). Now frame+slice threading with count =
   host cores capped at 8. `KYTY_VIDEO_THREADS=1` restores legacy
   single-thread behavior for A/B testing.
2. **Zero-copy convert** (`avPlayer PrepareVideo`) — YUV420P->NV12
   conversion now writes directly into the guest buffer at native pitch.
   Removes a per-frame ~12 MB temp allocation and a duplicate full-frame
   copy pass.

## Measured (Ryzen Z1 / RADV)

- Isolated 4K HEVC decode (`ps_studio_short2.mp4`, 3840x2160@60):
  74 fps (1 thread) -> 259 fps (8 threads).
- In-emulator convert+copy: ~2.1 ms/frame.
- In-emulator logo phase: roughly unchanged (~35-46 fps). Decode and
  convert were already faster than the content rate, so the remaining gap
  is downstream (texture upload / presentation / game pacing) — see the
  Astro's Playroom notes. The threading and zero-copy work stands as
  headroom and helps weaker CPUs.

## Apply

```sh
cd kytyps5-src
git apply /path/to/kyty-video-decode-optimizations.patch
# then rebuild kyty_emulator as usual
```
