# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

A ZMK firmware **config repo** for the Kinesis Advantage 360 Pro keyboard (adv360). This is not the ZMK firmware source itself — it's the `config/` overlay (keymap, board defs, west manifest) that gets built against a Kinesis-maintained fork of ZMK (`refil/zmk`, branch `adv360-z3.5-2`, see `config/west.yml`). `west update` pulls that fork down at build time; it isn't vendored in this repo.

## Build Commands

Firmware is built in a Docker/Podman container (`zmk-build-arm:stable` image) via `west`, orchestrated by the `Makefile`:

- `make` — build firmware for both halves (left + right), with Clique/ZMK Studio support (`-S studio-rpc-usb-uart -DCONFIG_ZMK_STUDIO=y`) on the left half only.
- `make left` — build only the left half.
- `make clean_firmware` — remove built `.uf2` files from `firmware/`.
- `make clean_image` — remove the built Docker image.
- `make clean` — both of the above.

Output lands in `firmware/<timestamp>-<commit>-<left|right>-clique.uf2`. Podman is preferred over Docker if both are installed (see `DOCKER :=` detection in `Makefile`); SELinux volume flags (`:z`) are added automatically on non-Darwin hosts.

There are no linters or test suites in this repo — "correctness" here means the devicetree/Kconfig compiles under `west build`, which requires the full Docker toolchain. There is no fast way to validate keymap syntax without running a build.

### CI

`.github/workflows/build.yml` runs on every push/PR and builds two artifact sets in parallel jobs: `build` (no Clique/Studio) and `build-clique` (with Clique/Studio, matching the local `make` default). Both do `west init -l config && west update && west zephyr-export` then `west build` for each half, caching `modules/ tools/ zephyr/ bootloader/ zmk/` keyed on `config/west.yml`'s hash.

## Architecture

- **`config/adv360.keymap`** is the actual keymap source — layers, key bindings, and the `hm` (homerow mods) hold-tap behavior are all defined here. `config/adv360_left.keymap` and `config/adv360_right.keymap` are just `#include "adv360.keymap"` wrappers; per-side differences come from the board defconfig, not the keymap.
- **`config/macros.dtsi`** holds standalone macro behaviors (quote/brace/paren/bracket pair-insertion macros, etc.), `#include`d into the `behaviors` block of `adv360.keymap`.
- **`config/version.dtsi`** is generated, not hand-edited — `bin/get_version_local.sh` (local builds) / `bin/get_version.sh` (CI) regenerate it before each build from the current date, branch name, and short commit hash, encoding them as a `&kp` macro (`macro_ver`) bound to Mod+V on the keyboard. `make` runs `git checkout config/version.dtsi` afterward to discard the generated file so it never gets committed.
- **`config/boards/arm/adv360/`** is the Zephyr board definition (devicetree `.dts`/`.dtsi`, `Kconfig*`, `adv360_left_defconfig` / `adv360_right_defconfig`). Per-side Kconfig flags live here, e.g. `CONFIG_ZMK_HID_KEYBOARD_EXTENDED_REPORT` and `CONFIG_BT_BAS` — both need updating in *both* left and right defconfig files to take effect symmetrically.
- **`config/info.json`** and **`config/keymap.json`** are editor-facing metadata (physical layout + keymap) consumed by [Nick Coutsos's keymap editor](https://nickcoutsos.github.io/keymap-editor/) — a GUI alternative to hand-editing the `.keymap`/`.dtsi` files. These two JSON files and `adv360.keymap` must stay in sync; see `UPGRADE.md` for the specific failure mode (stray `&none` entries) when they drift, and note the stricter syntax rules there (comma-separated + quoted in JSON, space-separated and unquoted in `.keymap`).
- **Layer LED colors**: the active layer is indicated by RGB LEDs per the table in `README.md` ("Layer colors") — layer 0 is always off, layers 1-8 show the same color on both halves, and above that the right half cycles independently before rolling over into the left half's color. Relevant when adding/reordering layers in `adv360.keymap`.
- **`assets/key-positions.md`**: canonical matrix key-position reference, needed for anything position-indexed (e.g. combos).
- **Versioning scheme**: `YYYYMMDD-XXXX-YYYYYY` (date, first 4 branch chars, short commit hash) — typed out live on the keyboard via Mod+V, and also used as the firmware artifact filename prefix in both local and CI builds.

## Known Gotchas

- Going from V2.0-based to V3.0-based configs can hit a merge conflict in `adv360.keymap` and `keymap.json` from removed unused keys/matrix positions — see `UPGRADE.md` for the exact fix (do not just blindly accept one side).
- `settings-reset.uf2` at the repo root is a flashable reset image, not a build artifact — don't delete/regenerate it as part of normal firmware builds.
