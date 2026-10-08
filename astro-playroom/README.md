# ASTRO's PLAYROOM — Internal 1080p patch (PPSA01325, v01.905.000)

Astro's Playroom (PPSA01325, v01.905.000) - Internal 1080p patch for KytyPS5
=============================================================================

WHAT THIS IS
A Riivolution-style (non-destructive) patch: the game files are NEVER modified.
The patch is applied in memory at load time by KytyPS5's built-in game-patch
system (ETAHen/GoldHEN-style JSON, --game-patch), same mechanism as cheat files.

It lowers the game's internal render resolution from 3840x2160 to 1920x1080
to make the game viable on PC handhelds (ROG Ally etc.).

FILE
PPSA01325_01.905.000_1080p.json  (this folder)

USAGE - command line (Batocera / kyty.sh):
  kyty_emulator --game "/userdata/roms/ps5/Astro Playroom.ps5/eboot.bin" \
    --game-patch /userdata/system/patches/astro-playroom-1080p/PPSA01325_01.905.000_1080p.json \
    --screen-width 1280 --screen-height 720 --amd-cpu

  Expected stdout: "Game cheat: matched eboot.bin with source base 0x0"
                   "Successfully applied cheat: Internal 1080p (was 4K) - 9 writes"

USAGE - Kyty launcher GUI (custom builds with community-patch support):
  The file also lives at _Patches/PPSA01325.json next to the emulator, which
  the launcher picks up automatically. Running the game offers to download
  missing patches from https://github.com/BannedPenta01/ps5-game-patches
  (one-time disclaimer, "Patches successfully downloaded.", never asked
  again). The toolbar band-aid button opens per-game patch selection, and the
  patches dialog can download from / link to the repository.

USAGE - plain Kyty launcher / Batocera:
  Copy this JSON to <emulator-dir>/_Patches/PPSA01325.json (exact name), or
  pass it via --game-patch as above. (Patched kyty.sh does this automatically
  by matching the game's TITLE_ID.)

WHAT WAS REVERSE ENGINEERED
- eboot.bin is NOT encrypted: it is a standard PS5 SELF (magic 4F153D1D...,
  12 segments) wrapping a plain x86-64 ELF (FreeBSD ABI v2). No keys needed;
  code disassembles directly with Zydis/Capstone after mapping SELF segments.
- Found 1x `mov rax, 0x87000000F00` (packed width=0xF00/3840,
  height=0x870/2160, VA 0x183EEA7) stored to a render-config struct.
- Found 4 code sites that select 3840x2160 vs 1920x1080 at runtime
  (the game already contains an HD path using 0x780/1920 + 0x438/1080, plus
  existing `movabs ..., 0x43800000780` instances proving the packed format).
  The patch rewrites the immediates of the 4K legs to the HD values, so every
  branch yields 1080p without touching control flow (no jmp/nop changes).

VERIFIED (KytyPS5 instrumented render-target logging + screenshots)
- Before: main target RT 3840x2160 present (~2500 samples over boot).
- After:  no RT 3840x2160 at all; RT 1920x1080 roughly doubled and a full
  mip chain 1920x1080 -> 960x540 -> ... -> 60x33 appears.
- Game boots cleanly to the "ASTRO's PLAYROOM / PRESS ANY BUTTON" title
  screen with no rendering corruption (screenshot verified).

KNOWN LIMITATIONS (honest notes, not marketing)
- One 3840x3240 buffer remains (proven via instrumentation to receive ZERO
  draws and no large compute dispatches: clear/copy scratch, not a shading
  cost center - intentionally untouched). No 3240 immediate exists anywhere
  in code, and the 9 writes here already cover every true resolution
  immediate in the executable (all other 0xF00 hits are buffer sizes, memset
  lengths, struct offsets, or math divisors).
- This lowers GPU fill / render-target memory for the main scene (~4x fewer
  pixels: 8.3M -> 2.1M) but does NOT touch shadows, effects density, or CPU
  load. Expect a large but not miraculous speedup on handheld iGPUs.
- If the game is updated (APP_VER != 01.905.000) the patch will refuse to
  apply (by design: title/version/process are validated) and the offsets
  must be re-ported.
- Presentation/window size (--screen-width/height) is independent; pair this
  with a 720p/1080p window. Kyty's --display-resolution Auto already reports
  HD for sub-4K windows, which complements this patch.

BENCHMARK (Ryzen Z1 / RADV, 1280x720 window, KYTY_FPS_LOG=1, title screen)
- Stock 4K internal:   ~4.4 fps steady, p95 frame ~240ms
- With this patch:     ~12.3 fps steady, p95 frame ~85ms  (~2.8x)
- Per-present draw counts identical (~250), confirming the same scene is
  being rendered faster, not a different scene.

EMULATOR UPDATE - video cutscene path (KytyPS5 source + deployed binary)
- Problem: Astro's boot logo ps_studio_short2.mp4 is HEVC 4K 3840x2160@60.
  Kyty decoded all video single-threaded (ffmpeg default thread_count=1)
  and converted each frame via a temp buffer + second full-frame copy.
- Fix 1 (src/libs/avPlayer.cpp, src/libs/videoDec2Decoder.cpp): enable
  ffmpeg frame+slice threading, count = host cores capped at 8
  (KYTY_VIDEO_THREADS=1 restores legacy single-thread for A/B tests).
- Fix 2 (avPlayer PrepareVideo): convert YUV420P->NV12 directly into the
  guest buffer at its native pitch; removed the per-frame temp allocation
  and the duplicate copy pass.
- Measured: isolated 4K HEVC decode 74fps (1 thread) -> 259fps (8 threads);
  in-emulator convert+copy ~2.1ms/frame. Logo-phase fps roughly unchanged
  (~35-46fps): decode/convert were already faster than the content rate,
  the remaining gap is downstream (texture upload / presentation / game
  pacing).

FOLLOW-UP EXPERIMENTS (2026-10-08)
- Shadow 2048->1024 experiment: NEGATIVE. The most promising candidate
  (three (0x800,0x800) surface setups at VA 0x1C3E207/224/241) was patched
  to 0x400 in a test build: both cheats applied, game booted fine, but the
  2048x2048 depth target persisted unchanged. Those immediates are something
  else (likely buffer sizes). The real shadow allocator was not found, and
  blind-patching 0x800 values is unsafe (85+ sites, mostly byte sizes), so
  NO shadow mod is shipped. The 2048 shadow pass is ~4% of draws / ~9% of
  indices - small even if found.
- Texture-upload profiling: EXONERATED. Instrumented Image::Upload shows
  only ~10 uploads >=8MB over a full 110s boot-to-title run (~110MB total,
  boot logos only) - video frames are NOT re-uploaded per frame. Combined
  with threaded decode (~150% CPU, headroom left), convert+copy ~2.1ms, and
  a pegged guest main thread at title, the video/title limits are game-side
  pacing and per-frame CPU work, not transfer bandwidth.

RE-PORTING TO A NEW VERSION
1. In the new eboot.bin, search code for B8 imm64 0x87000000F00 and for
   mov r32,imm32 pairs (0xF00 vs 0x780, 0x870 vs 0x438) near resolution
   selectors; confirm packed format via nearby existing 0x43800000780.
2. Offsets in this JSON are guest virtual addresses (RVA, ELF p_vaddr space;
   file_offset = RVA + 0x10430 for the main code segment). Kyty resolves them
   position-independently via anchor-byte search, so only the VALUES and the
   relative layout matter.
