# ReXGlue Workflow

This guide covers the end-to-end process for statically recompiling an Xbox 360 game using the modern ReXGlue SDK.

## Step 1: Extract the Game Files

First, you need to extract the executable (`default.xex`) and the game assets from the Xbox 360 package.

If you have an XBLA container file (STFS/LIVE/PIRS/CON):
```bash
python tools/extract_stfs.py "path/to/XBLA_file" extracted/
```

If you have an Xbox 360 ISO (XDVDFS):
```bash
python tools/extract_iso.py "path/to/game.iso" extracted/
```

## Step 2: Get the ReXGlue SDK

The ReXGlue SDK provides the recompiler and runtime. Clone it into your workspace:

```bash
git clone --recursive https://github.com/rexglue/rexglue-sdk.git tools/rexglue-sdk
```

*(Note: We recommend building the SDK following the instructions in `tools/rexglue-sdk/README.md`, or using a pre-compiled binary release if available. Ensure the `rexglue` executable is in your path or reference it explicitly.)*

## Step 3: Initialize the Project

ReXGlue comes with an `init` command that bootstraps a complete CMake project. Pick a directory name and an app name (e.g., `mygame`):

```bash
rexglue init --app_name mygame --app_root my_project/
cd my_project
```

This generates `CMakeLists.txt`, `src/main.cpp`, `CMakePresets.json`, and `mygame_config.toml`.

## Step 4: Prepare the XEX

Copy your extracted `default.xex` into your project directory:

```bash
mkdir assets
cp ../extracted/default.xex assets/
```

Verify your `mygame_config.toml` has the correct path set:
```toml
file_path = "assets/default.xex"
```

## Step 5: Run Codegen

Run the ReXGlue codegen pipeline to analyze the XEX and generate C++ code. This single command handles VTable discovery, switch table detection, missing instruction handling, and C++ generation.

```bash
rexglue codegen mygame_config.toml
```

This will produce C++ source files inside the `generated/` directory. If it reports any validation errors or missing switch tables, you can manually fix them using `[[switch_tables]]` overrides in your `mygame_config.toml` (you can use our `extract_switch_tables.py` script as a fallback to help find these!).

## Step 6: Build the Project

Once code generation finishes, configure and build the project using CMake:

```bash
cmake -B build -G Ninja -DCMAKE_BUILD_TYPE=Release
cmake --build build --config Release
```

## Step 7: Run!

Run the built executable:

```bash
./build/mygame.exe
```

Congratulations! You now have an Xbox 360 game running natively on your PC. The next step is usually stubbing out specific game hooks or kernel calls, debugging input handling, or refining your config. See the ReXGlue Wiki for more advanced features like Mid-ASM Hooks and ReXCRT overrides.
