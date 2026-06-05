# 360tools

**Everything you need to statically recompile Xbox 360 XBLA games to native PC executables.**

No emulator. No interpreter. No JIT. Just your Xbox 360 game, recompiled to C++ and running natively on x86-64 at full speed.

```
   XBLA Game (PowerPC) ----or---- Xbox 360 ISO
         |                              |
    [ extract_stfs.py ]         [ extract_iso.py ]
         |                              |
         +------------------------------+
         |
    [ ReXGlue SDK ]          <-- analyze, recompile, & build
         |
    Native x86-64 .exe       <-- your game, running on PC
```

## What's In The Box

### `tools/` -- Pre-Recompilation Pipeline

| Script | What It Does |
|--------|-------------|
| **`extract_stfs.py`** | Rips files out of STFS/LIVE/PIRS/CON Xbox 360 packages. Point it at a downloaded XBLA title and it pulls out the XEX and all game assets. Handles edge cases like `start_block=0` entries. |
| **`extract_iso.py`** | Extracts game files from Xbox 360 XDVDFS disc images. Finds the partition automatically (XGD1/XGD2), parses the B-tree directory, and extracts all files. Detects encrypted ISOs and points you to extract-xiso. |
| **`extract_xex_direct.py`** | Brute-force XEX2 extractor that finds XEX2 magic in STFS containers and rebuilds the contiguous data stream. Useful fallback when `extract_stfs.py`'s algorithm fails on unusual packages. |
| **`parse_xex_imports.py`** | Parses XEX2 import tables to identify which kernel/XAM functions the game actually calls. Helps you know which stubs you need to implement. |
| **`xex_info.py`** | Quick XEX2 header dumper -- parses and displays all header fields, security info, and import libraries. Great for initial triage before recompiling. |
| **`extract_switch_tables.py`** | Manually finds PPC switch/jump tables by pattern-matching. ReXGlue handles this natively now, but this script is kept as a fallback for generating TOML overrides if auto-detection fails. |

### `templates/advanced/` -- Advanced Project Scaffold

While the `rexglue init` command provides the basic project structure out-of-the-box, this folder contains battle-tested advanced components you can drop into your project:

- `menu.cpp` / `menu.h` — Win32 native menu bar + ImGui config dialogs (Graphics, Game, Debug, Controls)
- `settings.cpp` / `settings.h` — TOML-based settings persistence via `toml++`
- `stubs.cpp` — Game-specific kernel stub overrides (license bypass, multi-user sign-in, etc.)
- `keyboard_driver.cpp` / `keyboard_driver.h` — Keyboard + XInput merged input driver
- `test_boot.cpp` — Console-mode test harness for isolating crashes

### `docs/` -- How It All Works

| Doc | What It Covers |
|-----|---------------|
| **`rexglue-workflow.md`** | Full step-by-step from "I have an XBLA download" to "I have a running PC executable". |
| **`speed-fix.md`** | The VdSwap frame limiter (Windows `Sleep(16)` actually sleeps 31ms!) and how to fix game speed. |
| **`binary-analysis.md`** | How to analyze an Xbox 360 binary: memory layout, finding entry points, and debugging. |

## Quick Start

### Prerequisites

- **Python 3.8+** with dependencies: `pip install -r requirements.txt`
- **CMake 3.20+**, **Ninja**, **Clang 18+** (clang-cl on Windows)
- **Git**

### The Pipeline

```bash
# 1. Get your game files out of the XBLA package (or ISO)
python tools/extract_stfs.py path/to/XBLA_PACKAGE output_dir/
# -- or for disc-based games --
python tools/extract_iso.py path/to/game.iso output_dir/

# 2. Get the ReXGlue SDK
git clone --recursive https://github.com/rexglue/rexglue-sdk.git tools/rexglue-sdk

# 3. Initialize your project structure
tools/rexglue-sdk/out/install/win-amd64/bin/rexglue init --app_name mygame --app_root my_project/
cd my_project

# 4. Copy the extracted default.xex into my_project/assets/
mkdir assets
cp ../output_dir/default.xex assets/

# 5. Run Codegen to analyze the XEX and generate C++ code
../tools/rexglue-sdk/out/install/win-amd64/bin/rexglue codegen mygame_config.toml

# 6. Build and Run!
cmake -B build -G Ninja -DCMAKE_BUILD_TYPE=Release
cmake --build build --config Release
```

For detailed guidance, see `docs/rexglue-workflow.md`.

## Battle-Tested Fixes

These patterns have been discovered, debugged, and proven across multiple projects:

### VdSwap Frame Limiter
Windows `Sleep(16)` actually sleeps ~31ms due to 15.6ms timer granularity. The fix uses `QueryPerformanceCounter` spin-loop for precise 16.667ms frame pacing. Without this, your game runs at half speed. See `docs/speed-fix.md`.

### Native ReXGlue Handling
Older static recompilation approaches required manual fixes for timebase scaling (`__rdtsc()`), manual VTable discovery, and safety macro overrides. The modern **ReXGlue SDK** handles all of these natively:
- **Timebase Scaling:** ReXGlue directly translates `mftb` to `QueryGuestTickCount()`.
- **Safe Dispatch:** ReXGlue handles vtable C++ indirect calls natively.
- **Instruction Support:** ReXGlue supports Altivec/VMX vector instructions natively out of the box.

### ROV vs RTV Render Path
If you're getting white screens with certain render target formats (especially `k_2_10_10_10_FLOAT` + 4xMSAA), switch to the ROV (Rasterizer Ordered Views) path. ROV uses pixel shader interlock for EDRAM emulation and handles these edge cases correctly.

## Projects Built With These Tools

| Game | Repo | Status |
|------|------|--------|
| **The Simpsons Arcade** (XBLA, 2012) | [simpsonsarcade](https://github.com/sp00nznet/simpsonsarcade) | Playable -- full speed, audio, input, graphics |
| **Vigilante 8 Arcade** (XBLA) | [vig8](https://github.com/sp00nznet/vig8) | Playable -- 90 FPS, split-screen multiplayer, 79 shaders |
| **Guitar Hero II** (Xbox 360, 2007) | [gh2](https://github.com/sp00nznet/gh2) | Playable -- gameplay, audio, scoring, keyboard input working. Guitar controller support in progress |
| **Crazy Taxi** (XBLA, 2010) | [ctxbla](https://github.com/sp00nznet/ctxbla) | Playable -- D3D12 rendering, keyboard + XInput, arcade mode. In-game audio (XMA) in progress |

## The Stack

This pipeline stands on the shoulders of some incredible projects:

- **[ReXGlue SDK](https://github.com/rexglue/rexglue-sdk)** -- The static recompiler and runtime that provides everything the Xbox 360 OS gave games: kernel, D3D12 GPU backend, XMA audio, input, threading.
- **[Xenia](https://github.com/xenia-project/xenia)** -- The Xbox 360 emulator whose foundational GPU and kernel implementation powers ReXGlue.
- **[SIMDE](https://github.com/simd-everywhere/simde)** -- SIMD Everywhere, used to translate Altivec/VMX vector instructions to SSE/AVX/NEON.
- **[toml++](https://github.com/marzer/tomlplusplus)** -- TOML config parsing for the settings system.
- **[Dear ImGui](https://github.com/ocornut/imgui)** -- The in-game overlay UI for settings, debug info, and controller config.

## Want to Recomp a Game?

Pick an XBLA title. The delisted ones especially -- those games deserve to be preserved and playable. 
The tools in this repo will get you from "I have an XBLA download" to "I have generated C++ code" very quickly. The real work is in the runtime -- implementing the game-specific stubs, fixing rendering quirks, and getting audio/input working. But ReXGlue handles most of the heavy lifting.

**Every game you recomp is a game preserved forever.**

## Credits

This repository is a modernized, ReXGlue-native fork of the original `360tools` project. Massive thanks and credit to **[sp00nznet](https://github.com/sp00nznet)** for creating the original pipeline, scripts, and discovering the foundational fixes that make static recompilation of Xbox 360 games possible!

## License

Tools and scripts in this repo are provided under the MIT License unless otherwise noted in individual files. `extract_stfs.py` contains code derived from work by Rene Ladan under the 2-clause BSD license.
