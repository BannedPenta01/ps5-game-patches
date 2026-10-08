# PS5 Game Patches (KytyPS5)

Non-destructive, Riivolution-style patches for PS5 games running under
[KytyPS5](https://github.com/KytyPS5/KytyPS5). Nothing here modifies game
files — everything is applied in memory at load time.

## Games

| Game | Title ID | Patch | Effect |
|------|----------|-------|--------|
| ASTRO's PLAYROOM (v01.905.000) | PPSA01325 | [1080p internal resolution](astro-playroom/PPSA01325_01.905.000_1080p.json) | 3840x2160 -> 1920x1080 (~2.8x fps on Ryzen Z1 handhelds) |

See each game's folder for details, benchmarks, and usage.

## Patch format

Game patches are [ETAHen/GoldHEN-style JSON](https://github.com/TeeKay87/HEN-Cheats-Collection)
consumed by KytyPS5's built-in game-patch system (`--game-patch <file>` or
the launcher's Patches dialog). Each file validates title ID, app version,
and executable name before applying, so a wrong-version patch refuses to
load instead of corrupting memory.

## Emulator improvements

The [`emulator/`](emulator/) folder holds KytyPS5 source patches developed
alongside these game patches (e.g. multithreaded video decoding for game
cutscenes). See [`emulator/VIDEO_OPTIMIZATIONS.md`](emulator/VIDEO_OPTIMIZATIONS.md).
