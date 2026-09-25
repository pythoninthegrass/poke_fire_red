# AGENTS.md

This file provides guidance to AI agents when working with code in this repository.

## What this is

This is `pokefirered`, a [pret](https://pret.github.io/) decompilation of Pokémon FireRed and LeafGreen. The project disassembles/decompiles the original GBA ROMs and rebuilds them, matching byte-for-byte, from C (with `agbcc`, the original compiler), ARM/Thumb assembly, and asset source files. It builds six ROM targets: FireRed, LeafGreen, each with their base/rev1/rev10 (Switch Online) revisions.

## Build commands

Requires `agbcc` cloned as a sibling directory and installed into this repo (`./install.sh ../pokefirered` from within `agbcc`), plus `devkitARM`/`gba-dev` and `libpng`. Full setup is in `INSTALL.md`.

```bash
make                          # build pokefirered.gba (default target)
make -j$(sysctl -n hw.ncpu)   # parallel build (macOS; use nproc on Linux)
make leafgreen                # build pokeleafgreen.gba
make firered_rev1              # FireRed 1.1
make leafgreen_rev1           # LeafGreen 1.1
make firered_switch            # FireRed Switch Online rev
make compare                  # build + verify byte-identical to the original ROM (sha1 check)
make compare_leafgreen        # same, for LeafGreen
make modern                   # build with a modern arm-none-eabi-gcc toolchain instead of agbcc
make clean                    # clean build output, tools, generated files, assets
make tidy                     # clean without touching built tools
```

`GAME_VERSION` (`FIRERED`/`LEAFGREEN`), `GAME_REVISION` (`0`/`1`/`10`), and `MODERN`/`COMPARE` env vars drive the version-specific targets in `Makefile`/`config.mk` directly if a canned target doesn't fit.

There is no unit test suite — correctness is verified by `make compare*`, which checks the built ROM's sha1 against the known-good original (see `*.sha1` files at the repo root). CI (`.github/workflows/build.yml`) runs `make all syms` for all six version/revision combos plus a `modern` build on every push/PR.

## Architecture

This is **not** a conventional application codebase — it's a reconstructed disassembly whose file layout mirrors the original ROM's data/code segments, glued together by a large custom Make build system.

- **`src/`** — decompiled C source, one file roughly per original compilation unit (battle system, overworld, menus, field effects, etc.). File names reflect the ROM's internal organization, not a designed module system.
- **`include/`** — headers for `src/`, plus struct/data layouts reverse-engineered from the ROM.
- **`asm/`** — hand-written ARM/Thumb assembly (macros in `asm/macros`) for code not yet decompiled to C, or that must stay in asm.
- **`data/`** — assembly-format tables and scripts: `data/maps/<MapName>/` (one dir per game map, each with `map.json`, `scripts.inc`, `text.inc`), `data/scripts/` (overworld event scripts, in the `.inc` script-command format compiled by `tools/preproc`), `data/battle_*` (battle scripts/AI), plus tilesets, sound data, and text.
- **`constants/`** — `.inc` assembly constants shared between C, asm, and script sources (game states, species/move/item IDs, etc.) — the closest thing to a shared "enum" layer across the whole project.
- **`graphics/`, `sound/`** — raw asset sources (PNG/`.pal` tiles, MIDI-like `.s` sound data) compiled by dedicated tools into GBA-native formats.
- **`tools/`** — custom C build tools invoked by the Makefile (`gbagfx` for graphics conversion, `mapjson`/`jsonproc` for map data, `preproc` for script preprocessing, `scaninc` for dependency scanning, `mid2agb`/`wav2agb` for audio, `gbafix` for ROM header fixup, `bin2c`, `ramscrgen`). These are built from source on first `make` invocation.
- **Build orchestration**: `Makefile` is the entry point; `*_rules.mk` files (`graphics_file_rules.mk`, `map_data_rules.mk`, `json_data_rules.mk`, `spritesheet_rules.mk`, `tileset_rules.mk`) and `audio_rules.mk` define the pattern rules that turn `data/`/`graphics/`/`sound/` sources into intermediate build artifacts under `build/`. `ld_script*.ld` are the linker scripts (one per revision) that place everything at the correct ROM addresses.
- **`sym_*.txt` files** — manually maintained lists of symbols placed in specific memory regions (EWRAM/IWRAM/common/bss) that the linker can't infer, keyed by build revision.

### Working with maps and scripts

Maps live under `data/maps/<MapName>/`. Map layout/connections/events are in `map.json` (edited with [porymap](https://github.com/huderlem/porymap), not by hand), NPC/trigger scripts are in `scripts.inc` (event-script assembly, commands defined via `data/script_cmd_table.inc` and documented in `include/constants/event_object_movement*.h` / battle/field script headers), and map-specific text is in `text.inc`.

### Matching decompilation conventions

When decompiling or modifying `src/*.c`, the goal is typically that `make compare` still passes — i.e. the compiled output matches the original ROM byte-for-byte with `agbcc`. Changes to C code that alter codegen (even functionally-equivalent refactors) can break the match; verify with `make compare` (or `make compare_<version>`) after non-trivial `src/`/`include/` changes, not just a plain `make`.
