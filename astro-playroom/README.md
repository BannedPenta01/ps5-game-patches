# ASTRO's PLAYROOM — Graphics preset patches (PPSA01325, v01.905.000)

Astro's Playroom (PPSA01325, v01.905.000) - Internal 1080p patch for KytyPS5
=============================================================================

WHAT THIS IS
A Riivolution-style (non-destructive) patch: the game files are NEVER modified.
The patch is applied in memory at load time by KytyPS5's built-in game-patch
system (ETAHen/GoldHEN-style JSON, --game-patch), same mechanism as cheat files.

It lowers the game's internal render resolution from 3840x2160 to 1920x1080
to make the game viable on PC handhelds (ROG Ally etc.).

FILE
PPSA01325_01.905.000_1080p.json  (this folder; despite the name it holds
both mods below - enable exactly ONE of them in the Kyty launcher's
patches dialog, or via the "enabled" flags)

MODS (pick exactly ONE - the launcher's patches dialog shows these as
settings checkboxes, like a PC game's graphics options)
1. "Preset: Balanced - 1080p internal" (enabled by default) - 9 writes,
   3840x2160 -> 1920x1080. Best quality/perf balance. Title: ~12.3 fps.
2. "Preset: Performance - 720p internal" (disabled by default) - same 9
   sites rewritten to 1280x720 (0x500/0x2D0). Title: ~15.2 fps (~3.5x stock,
   +24% over 1080p). Verified: 1280x720 main target + full mip chain, no
   1080p/4K targets left, clean boot to the "START A NEW GAME" screen.

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

BENCHMARK (Ryzen Z1 / RADV, 1280x720 window unless noted, KYTY_FPS_LOG=1)
- Stock 4K internal, 15W:      ~4.4 fps title steady, p95 ~240ms
- 1080p patch, 15W:            ~9.7-12.3 fps title (windowed-fullscreen range)
- 1080p patch, 20W (AC prof.): ~13.0 fps title, p95 ~85ms
- 720p patch, 15W:             ~15.2 fps title (same 250 draws/present scene)
- 1080p patch, 25W STAPM test: ~28-29 fps intro/tutorial scenes
- 720p patch, 25W STAPM test:  ~28 fps peak scene, ~22.5 settled scene
- Temps at 25W: CPU 87-90C, GPU 90C (under 95C firmware throttle).
  TDP was restored to stock 15W after testing - raise it yourself to play
  this way (Ally Turbo/manual mode or ryzenadj --stapm-limit=25000).
PATH TO PLAYABLE: 25W sustained + Performance (720p) preset + 720p window
  lands ~22-28 fps across scenes (vs 4.4 stock). 30 locked is not there,
  but heavy scenes roughly triple and light scenes go playable.

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

FUTURE EFFECT OPTIONS (bloom / SSAO / shadows / LOD) - investigated,
not yet shippable. Findings: the engine is a deferred renderer with named
stages (CheckerboardStencil, DecalShadowTex, SSAO_Async, TaaHalton, SSR,
ZPrepass, GBuffer debug) and a named-param/tweak system (HightQuality,
IsHalf, DebugRenderScale, per-effect params created via a common
new+name+type constructor). BUT readers use hashed lookups, so static
analysis finds declarations, not the decision points - there are no safe
patch sites for effect toggles yet. Blind-patching would risk corruption
for zero verified gain. Per-effect work (starting with bloom: fullscreen,
additive, mild visual cost) needs render-stage correlation and is roadmap,
not in this file. The two resolution presets above are the verified,
safe core of the preset.
BLOOM DEEP-DIVE (2026-10-09): fully mapped the chain - stages
FirstDownSample->Blur->Mix->Final (GfxRenderStageBloom*), ping-pong buffers
bloomColor{2,4,8,16,32}{A,B} + bloomInputColor, shader uniform u_bloomMax,
per-stage shader/descriptor blobs, construction sites at VA 0x1DFA3F4 etc.
(via new+name+descriptor constructor 0x1BE23C0). Experiment: zeroed the
FirstDownSample stage-name string in R segment (R pages proven writable via
a no-op GamePatch write) to starve the chain. Result: NO-OP - identical
visuals (title glow intact), identical fps/draws. Stage names are debug
labels only; the render graph does not key on them. A real bloom skip needs
the traversal-time decision point, still open. Traversal analysis: stage
objects carry per-stage u16 slots at +0x28/+0x2A/+0x2C written at
construction from constructor 0x1BE23C0 output - but the parent object
arrives as a caller argument (never stored locally), i.e. heap/runtime
objects with no static address, and readers use hashed registry lookups
with no static xrefs. Static RE cannot reach the skip decision; unlocking
it needs Kyty-side pass-to-command-buffer attribution plus writer RE.

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

THERMALS & POWER (Ryzen Z1 handheld, measured 5-min session)
- Sustained load: CPU ~76-77C, GPU ~76-77C, package power flat at 15.1W.
  Verdict: NOT overheating. 70s are normal operating temps (TjMax ~95C+),
  and the flat 15W line is the platform's own TDP cap (Silent profile)
  doing its job. Modern APUs self-protect in firmware; software cannot push
  them to dangerous levels. Brief 90C+ spikes at boot (shader compile) are
  normal and short.
- No spin-waits found in the emulator: kernel sync is futex-based, waits
  sleep properly, governor already "performance", no RAM pressure (5GB
  resident of 9GB available).
- Eco changes shipped in the custom build: video decode threads capped 8->6
  (decode uses ~150% CPU with headroom to spare; spare threads are worth
  more to the render path under a fixed power budget).
- Helper: add-ons/kytyps5/eco-guard.sh --seconds=N logs CPU/GPU temp +
  package/GPU power and warns after 60s sustained 90C+. Observer only;
  it changes nothing by itself.

RE-PORTING TO A NEW VERSION
1. In the new eboot.bin, search code for B8 imm64 0x87000000F00 and for
   mov r32,imm32 pairs (0xF00 vs 0x780, 0x870 vs 0x438) near resolution
   selectors; confirm packed format via nearby existing 0x43800000780.
2. Offsets in this JSON are guest virtual addresses (RVA, ELF p_vaddr space;
   file_offset = RVA + 0x10430 for the main code segment). Kyty resolves them
   position-independently via anchor-byte search, so only the VALUES and the
   relative layout matter.
