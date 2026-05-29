# Hillside46 ZMK Config

ZMK firmware keymap for the Hillside46 split keyboard (nice!nano v2 controller).

> This file is maintained in **English**. Always write and update it in English.

## Key files

- `config/hillside46.keymap` — main file: layers, behaviors, macros
- `config/combos.dtsi` — all combos (included from the keymap)
- `config/hillside46.conf` — Kconfig settings (bluetooth, sleep, RGB, etc.)
- `config/west.yml` — ZMK version (currently pinned to v0.3.0)
- `build.yaml` — build matrix for GitHub Actions (hillside46_left + hillside46_right)

## Layers

Order matters: the bases (DEF_MAC, DEF) are the lowest so that momentary/overlay
layers stack on top of either of them; ADJ is the highest so it overrides NAV_MAC
when reached via NUM+NAV_MAC. DEF_MAC is layer 0 — the keyboard boots into mac mode
by default; Win/Lin requires an ADJ switch.

| # | Name    | Description                                       |
|---|---------|---------------------------------------------------|
| 0 | DEF_MAC | QWERTY base for macOS (boot default)              |
| 1 | DEF     | QWERTY base (Win/Lin), homerow mods               |
| 2 | NUM     | Numbers, symbols, F-keys (shared)                 |
| 3 | NAV     | Navigation, symbols, media (Win/Lin)              |
| 4 | WM      | Window management (Win/Lin: Win+1..9, Win+arrows) |
| 5 | FZ      | FancyZones, snap, virtual desktops (Win)          |
| 6 | NAV_MAC | Navigation for macOS                              |
| 7 | WM_MAC  | Window management (macOS via AeroSpace)           |
| 8 | HK_MAC  | macOS hotkeys (screenshots; leftmost-left thumb)  |
| 9 | ADJ     | Bluetooth, system, OS switching                   |

ADJ is activated via a conditional_layer when NUM and NAV are held simultaneously
(works with both NAV and NAV_MAC — two tri_layer entries: `<2 3>` and `<2 6>` → 9).

## Target OSes (Win/Lin ↔ macOS)

The config keeps two parallel bases:
- **macOS** (default on boot, BT profile 0): layers DEF_MAC / NAV_MAC / WM_MAC / HK_MAC
- **Win/Lin** (BT profile 1): layers DEF / NAV / WM / FZ

Switching is done with keys in the ADJ layer (top row of the left half, above BT_SEL 0/1):
- above BT_SEL 0 → `&to DEF_MAC` (mac base)
- above BT_SEL 1 → `&to DEF` (Win/Lin base)

The BT slot ↔ layer pairing is convention: ZMK's BT_SEL just selects a stored
pairing profile, so the physical Bluetooth slot 0 should be paired with the mac
device (and slot 1 with the Win/Lin device) for the on-boot default to be useful.
If your current pairings are reversed, see "Re-pair BT slots" below.

Modifier convention on mac: `LGUI`=Cmd, `LALT`=Option, `LCTRL`=Ctrl.

The home-row mod order differs between bases (chosen by how often each OS's primary
hotkey modifier is used — it goes on the strongest finger, the index):
- **Win/Lin** (DEF, NAV): `GASC` — pinky→index = Gui · Alt · Shift · **Ctrl**
- **macOS** (DEF_MAC, NAV_MAC): `CASG` — pinky→index = Ctrl · Alt · Shift · **Cmd**

So on mac, Cmd and Ctrl are swapped relative to Win: Cmd moves off the pinky onto
the index (A=Ctrl, S=Opt, D=Shift, F=Cmd, mirrored on the right). Shift and Alt stay
on the same fingers in both OSes. The shared NUM layer was left as-is — its HRM stays
in Win order.

### Known limitations of mac mode
- On boot/battery swap the keyboard always starts on layer 0 (DEF_MAC, mac base) —
  ZMK does not persist a different choice. To use Win/Lin you must press the switch
  in ADJ each time the keyboard powers on.
- Combos are gated by the topmost active layer (`combo_active_on_layer` in ZMK), so on
  the mac base only mac combos fire and on the win base only win combos fire — no conflict.
- Clipboard: Win/Lin — Ctrl (cut/copy/paste), mac — Cmd (LG(X)/LG(C)/LG(V)).
- Language switch: Win/Lin — Punto Switcher (Ctrl+Shift+1/2), mac — Ctrl+Space
  (both combos, positions 15-16 and 19-20).
- Symbol combos and F-keys work on both bases.

### macOS-side dependencies
- **Tab switching** (NAV_MAC, Cmd+Opt+←/→): native in Firefox/Chrome/VSCode.
  For Ghostty add to `~/.config/ghostty/config`:
  `keybind = super+alt+right=next_tab` and `keybind = super+alt+left=previous_tab`.
- **Cmd+Tab** (DEF_MAC, on the AC-Tab key) switches between apps (tap = the two most
  recent apps). To switch between WINDOWS (like alt-tab on Windows) install the AltTab
  app; windows of a single app — Cmd+` (backtick).
- **Window management** (WM_MAC): driven by [AeroSpace](https://nikitabobko.github.io/AeroSpace/),
  install it (`brew install --cask nikitabobko-tap/aerospace`) and place the project's
  `~/.aerospace.toml`. AeroSpace bindings live on **Hyper = `Ctrl+Opt+Cmd`** (not the
  default `Alt`, which collides with zellij's `Alt+hjkl/-/=/f/i/n/o/p` and several
  Cmd+Opt+* macOS shortcuts). WM_MAC sends Hyper-chords matching the config:

  | Layer keys (left)              | Hyper chord     | AeroSpace action                  |
  |--------------------------------|-----------------|-----------------------------------|
  | W E R / S D F / X C V          | ⌃⌥⌘+7..9/4..6/1..3 | `workspace 7..9 / 4..6 / 1..3` (aligned with NUM layer digits) |
  | `AC-Tab` slot (col 1, mid row) | ⌃⌥⌘+Tab         | `workspace-back-and-forth`        |
  | T / B                          | ⌃⌥⌘+] / ⌃⌥⌘+[   | `workspace next` / `workspace prev` |

  | Layer keys (right)             | Hyper chord     | AeroSpace action                  |
  |--------------------------------|-----------------|-----------------------------------|
  | H J K L (home row)             | ⌃⌥⌘+H/J/K/L     | `focus left / down / up / right`  |
  | Y U I O (row above home)       | ⌃⌥⌘+⇧+H/J/K/L   | `move left / down / up / right`   |
  | M / `,`                        | ⌃⌥⌘+- / ⌃⌥⌘+=  | `resize smart -50 / +50`          |

  AeroSpace has `start-at-login = false` in the shipped config — launch it manually
  (Spotlight → "AeroSpace") or flip the flag if you want auto-start. Without AeroSpace
  running, the Hyper-chords above are no-ops.

- **HK_MAC (mac hotkeys, screenshots)** — held via the **leftmost-left thumb** on DEF_MAC
  (the BSPC slot). Currently maps `Cmd+Shift+4` and `Cmd+Shift+5` over the NUM-layer
  digits 4 and 5 (physical home-row `S` and `D`):
  - `S` → `⌘⇧4` — region screenshot
  - `D` → `⌘⇧5` — screenshot UI
  Layout matches NUM: screenshot keys sit on the same physical positions as digits 4/5,
  so you can think of the layer as "shift+cmd onto the corresponding NUM digit".

### Re-pair BT slots (mac → BT 0, Win → BT 1)

The keymap convention is that BT slot 0 holds the mac pairing and slot 1 holds the
Win/Lin pairing. The keymap change itself does NOT move existing pairings — slots
are stored in the keyboard's flash. If your slots are reversed:

1. From ADJ, select BT slot 0 (`&bt BT_SEL 0`), then `&bt BT_CLR` to clear it.
2. From the currently paired host's OS, "forget" the keyboard in Bluetooth settings.
3. Repeat steps 1–2 for slot 1.
4. Re-pair: BT_SEL 0, then pair to the mac; BT_SEL 1, then pair to Win/Lin.

`&bt BT_SEL N` is on the second row of ADJ (left half) and `&bt BT_CLR` is below the
INS column on the same layer. After re-pairing, the boot default (mac on BT 0) works
without intervention.

## Build

Via GitHub Actions, but **manual only** — the workflow (`.github/workflows/build.yml`)
is configured with `workflow_dispatch`, so a push (to any branch, including `main`)
does **NOT** start a build. Trigger it with:
- web: Actions → "Build" workflow → "Run workflow" → pick the branch;
- CLI: `gh workflow run build.yml --repo GlebYavorski/hillside46 --ref <branch>`.

You can build from any branch (not only `main`), but it must be pushed to `origin`
first. Artifacts (.uf2) are downloaded from the finished run (Actions → run → Artifacts).

### After a firmware-affecting commit (automation rule)

Whenever a commit changes **firmware files** — anything under `config/`
(`hillside46.keymap`, `combos.dtsi`, `hillside46.conf`, `west.yml`) or the build
matrix (`build.yaml`) — automatically follow up with:

1. Push the branch to `origin` (never `upstream`).
2. Trigger the build:
   `gh workflow run build.yml --repo GlebYavorski/hillside46 --ref <branch>`
3. Wait for it to finish, e.g.:
   `gh run watch <run-id> --repo GlebYavorski/hillside46 --exit-status`
4. Download the `.uf2` artifacts into the **fixed** folder `~/Downloads/hillside46-firmware/`,
   wiping it first so the previous build is overwritten:
   ```
   DEST=~/Downloads/hillside46-firmware
   rm -rf "$DEST" && mkdir -p "$DEST"
   gh run download <run-id> --repo GlebYavorski/hillside46 --dir "$DEST"
   ```
   Result: `~/Downloads/hillside46-firmware/firmware/hillside46_{left,right}-nice_nano_v2-zmk.uf2`.

Do **NOT** trigger a build for commits that touch only `CLAUDE.md`, README, or other
non-firmware / auxiliary files.

## Conventions

- Documentation lives directly in code comments
- Every layer must have an ASCII layout diagram in a comment
- This `CLAUDE.md` is written and maintained in **English**

## Git

- `origin` → user's fork: `https://github.com/GlebYavorski/hillside46.git`
- `upstream` → original: `https://github.com/mmccoyd/zmk-config.git`
- Push only to `origin`

## Documentation

- we try to keep the comments in `config/hillside46.keymap` updated
- when adding keymap diagrams use the following letters for modifiers:
  G - Win/Gui/Meta
  A - Alt
  S - Shift
  C - Control
- always exactly in this order. So, for example:
  - for alt+win+h the diagram should be GA-h
  - for shift+ctrl+gui+alt+right it will be GASC-->
  - for alt+5: A-5
