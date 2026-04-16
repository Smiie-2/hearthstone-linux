# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this project does

`hearthstone-linux` makes the macOS Hearthstone client run natively on Linux by:
1. Downloading the macOS game files via `keg` (a Battle.net CDN tool, included as a git submodule)
2. Substituting Unity's Linux runtime binaries for the macOS ones
3. Building stub `.so` libraries that fake the macOS-only APIs the game depends on (`CoreFoundation`, `OSXWindowManagement`, Blizzard commerce SDK)
4. Building a GTK/WebKit login tool that intercepts the Battle.net OAuth token and AES-encrypts it to disk; the `CoreFoundation` stub reads this `token` file when the game requests auth

## Build commands

Build the stub libraries:
```bash
make -C stubs
```

Build the login tool:
```bash
make -C login
```

Clean build artifacts:
```bash
make -C stubs clean
make -C login clean
```

Full setup (download game, build everything, install desktop entry):
```bash
./craft.sh
```

With a local macOS installation (skips download):
```bash
./craft.sh [<path to /Applications/Hearthstone>] [<Unity path>]
```

## Architecture: how the pieces fit together

### Token flow (the core trick)

The game reads its auth token via macOS `CFPreferencesCopyAppValue` / `CFDataGetBytePtr`. On Linux these don't exist, so `stubs/CoreFoundation.c` provides stub implementations. `CFDataGetBytePtr` simply opens the file `token` in the current directory and returns its bytes — this is the encrypted token written by `login/login.c`.

`login/login.c` opens a GTK window embedding a WebKit view pointed at `https://battle.net/login/?app=wtcg`. It watches each page load URI; when the OAuth redirect URL contains a token (detected by UUID-like pattern with dashes at positions 2 and 35), it AES-CBC-encrypts it using a PBKDF2-derived key (seeded from the system username XOR'd with a static entropy blob) and writes the 48-byte ciphertext to `./token`.

The key derivation uses `login/login.c::getEncryptionKey` and `stubs/CoreFoundation.c` uses the same algorithm — so the token is readable only by the same username that created it.

### `craft.sh` orchestration

`craft.sh` is the entry point for both fresh installs and updates. It:
- Tunes `Bin/Hearthstone_Data/boot.config` for Linux (enables multithreaded rendering and graphics jobs via `tune_boot_config()`)
- Manages a Python venv (for the `keg` submodule) in `./venv/`
- Persists region/locale choices in `.region` and `.locale` files inside the `hearthstone/` directory
- Tracks installed versions in `.version` and `.unity` files
- Downloads and extracts only the Linux playback engine from a full Unity Editor `.tar.xz` (the specific path is `Editor/Data/PlaybackEngines/LinuxStandaloneSupport/Variations/linux64_player_nondevelopment_mono`)
- Transforms the macOS `.app` bundle layout into the flat `Bin/` layout Unity Linux expects
- Generates `client.config` from a template by substituting `REGION` and `LOCALE` placeholders; `Aurora.ClientCheck=false` bypasses the launcher requirement

### Stub libraries

All three stubs in `stubs/` are minimal no-op `.so` files:
- `CoreFoundation.so` — the only non-trivial stub; `CFDataGetBytePtr` reads `./token`
- `libOSXWindowManagement.so` — single no-op `DisableTabBar` function
- `libblz_commerce_sdk_plugin.so` — stubs for the Blizzard commerce SDK (why the shop is closed)

Stubs are placed at paths the Unity plugin loader expects:
- `CoreFoundation.so` → `Bin/Hearthstone_Data/Plugins/System/Library/Frameworks/CoreFoundation.framework/`
- Others → `Bin/Hearthstone_Data/Plugins/`

### Dependencies

- **C++ (login):** `g++`, `libcryptopp`, `gtk+-3.0`, `webkit2gtk-4.1` (fallback: `webkit2gtk-4.0`)
- **C (stubs):** `gcc` only — no extra libraries
- **Python (keg):** Python 3 venv with the `keg` submodule installed into it

## Known constraints

- **keg submodule must be initialized** — run `git submodule update --init --recursive` after cloning; `craft.sh` fails with `realpath: keg/bin/ngdp: No such file or directory` if skipped
- **Regional CDNs 404 on config files** — `eu/us/kr.cdn.blizzard.com` only host game data; config/metadata must use `level3.blizzard.com`. Pass `--cdn` to `ngdp install` (data), not `ngdp fetch` (config)
- **keg downloads are sequential by default** — the patched submodule uses `ThreadPoolExecutor` (32 workers); if reverting keg, expect ~36h install time for ~21,951 loose files
- The in-game shop is non-functional (the commerce SDK stub returns no-ops)
- The `token` file is tied to the username that created it (key derivation XORs username into entropy)
- The game must be launched from inside the `hearthstone/` directory (both `token` and `client.config` are read from the working directory)
- Wayland + NVIDIA may crash the login tool with `Error 71`; workaround: `WEBKIT_DISABLE_DMABUF_RENDERER=1 ./login` or `__NV_DISABLE_EXPLICIT_SYNC=1 ./login`
- **`hearthstone/` is gitignored** — it contains the game installation, not source code. Any persistent changes to game files (e.g. `boot.config`) must be automated in `craft.sh` or they'll be lost on the next update
- **`boot.config` ships from macOS with suboptimal Linux defaults** — `gfx-disable-mt-rendering=1` disables multithreaded rendering; `craft.sh`'s `tune_boot_config()` patches this automatically
- **`./craft.sh` is not cleanly re-runnable** — on a second run against an already-installed, up-to-date game, `check_unity` exits 1 because `[ -d "MonoBleedingEdge" ] && mv ...` is false, and `set -e` kills the script. The game is already set up at that point; the error is cosmetic.
- **`craft.sh` has two install paths** — `./craft.sh` (managed, runs through `ensure_keg` with keg venv) vs `./craft.sh /path/to/Hearthstone` (local, `check_directory` returns early and skips `ensure_keg`). Environment setup that must cover both paths belongs in the function that actually needs it (e.g. `ensure_pillow` in `transform_installation`), not in `ensure_keg`.
- **Git remotes:** `origin` = user's fork (`Smiie-2/hearthstone-linux`), `upstream` = original repo (`0xf4b1/hearthstone-linux`). Always push to `origin`, never directly to `upstream`
