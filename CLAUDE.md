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
| 2 | NUM     | Numpad digits — shared by both bases, left half   |
| 3 | NAV     | Navigation, media (Win/Lin) — right half only     |
| 4 | WM      | Window management (Win/Lin: Win+1..9, Win+arrows) |
| 5 | FZ      | FancyZones, snap, virtual desktops (Win)          |
| 6 | NAV_MAC | Navigation for macOS — right half only            |
| 7 | APP     | App tabs + splits (mac; 2nd-from-edge left thumb) |
| 8 | ADJ     | Bluetooth, system, OS switching                   |

ADJ is activated via a conditional_layer when the NUM and NAV keys are held
simultaneously — two tri_layer entries, one per base: `<NUM NAV>` and
`<NUM NAV_MAC>` → `ADJ`. `if-layers`/`then-layer` use the `#define` layer names,
not raw indices, so inserting a layer no longer means hand-editing them.

**NUM is shared by both bases** — there is no NUM_MAC any more. It used to exist purely
because the right home row carried the OS-specific Ctrl/Cmd pair on positions 19 and 22;
those slots are `&trans` since the symbol cleanup, so the correct mod order arrives from
whichever base is active. NAV still needs its mac twin (NAV_MAC): their right halves
genuinely differ (Home/End vs Cmd+arrow, Ctrl+arrow vs Option+arrow).

### Symbols and F-keys live in combos only

Every special symbol (`!@#$%^&*()` `]`) and every F-key (F1-F12) has exactly **one**
definition site: `config/combos.dtsi`. NUM (then still NUM/NUM_MAC) and NAV/NAV_MAC carried verbatim
copies of them; those copies are gone. Never re-add a symbol or an F-key to a layer — change
the combo instead.

What deliberately stays on NUM as *numpad math*, not as "special symbols": `=` `:` `- /`
`+ *` `KP.`.

Cost of the rule, worth remembering: ZMK matches a combo against the **highest active layer**
(`zmk_keymap_highest_layer_active()`), and the symbol combos are registered for `DEF DEF_MAC`
only. So while a layer thumb is held, symbol combos do not fire — typing `(` in the middle of
a number means releasing the thumb first. Extending the combos' `layers` list to cover NUM is
**not** a fix: on NUM the left half is a numpad, and the vertical combo pairs land exactly on
the digit pairs 4/1, 5/2, 6/3, so digit rolls would misfire as `@#$`.

After the cleanup NUM occupies only the left half of the board and NAV only the right half.
Merging them into one layer was considered and rejected for now: it would break the `<NUM NAV>`
conditional that reaches ADJ, put digits where NAV's left home-row mods live, and — since NAV
and NAV_MAC genuinely differ — duplicate the numpad across two OS variants again, undoing the
NUM_MAC merge.

### Physically unused keys

**Key positions 38 and 45 — the outermost thumb of each half — do not exist for
practical purposes.** Their keycaps have been removed from the board, so the switches
cannot be pressed. Never assign a binding there; treat those two slots as absent
hardware, not as free real estate.

Thumb numbering, outer → inner: left `38 39 40 41`, right `42 43 44 45` (so 41 and 42
are the two inner thumbs that carry Hyper, and 38 / 45 are the dead outer ones).

`def_mac_layer` already has `&none` on both. `def_layer` (Win/Lin) still carries
`&lt_t WM BSPC` on 38 and `&mt LALT DEL` on 45 — both **unreachable**; they predate the
keycap removal. Consequences on the Win base: WM is reachable only via the two inner
RETURN thumbs, and Delete currently has no key at all. Not yet resolved.

## Target OSes (Win/Lin ↔ macOS)

The config keeps two parallel bases:
- **macOS** (default on boot, BT profile 0): layers DEF_MAC / NAV_MAC / APP
- **Win/Lin** (BT profile 1): layers DEF / NAV / WM / FZ
- **shared by both**: NUM (numpad) and ADJ

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
on the same fingers in both OSes. NUM and NAV/NAV_MAC do **not** repeat this split: their
home-row slots are `&trans`, so the mods fall through to whichever base is active and the
order is correct for free. That is what let NUM_MAC be deleted entirely.

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
- **Window management**: driven by [AeroSpace](https://nikitabobko.github.io/AeroSpace/),
  install it (`brew install --cask nikitabobko-tap/aerospace`) and place the project's
  `~/.aerospace.toml`. AeroSpace bindings live on **Hyper = `Ctrl+Opt+Cmd`** (not the
  default `Alt`, which collides with zellij's `Alt+hjkl/-/=/f/i/n/o/p` and several
  Cmd+Opt+* macOS shortcuts).

  Hyper is reached via the two inner thumbs on `def_mac_layer` (`HYP/ENT` left slot,
  `HYP/OSH` right slot — see that layer's keymap comment for exact positions): hold
  either one to hold the Hyper modifier for as long as needed and press any AeroSpace
  (or OmniWM) hotkey defined in `~/.aerospace.toml` — no dedicated ZMK layer required.
  The right thumb's tap is a one-shot Hyper (sticky: applies to the next keypress, or
  clears after 5s idle) for single-chord use without holding.

  **Fixed (root cause found):** the one-shot tap used to do nothing on real hardware
  because `sk_hyper` lacked `ignore-modifiers`. ZMK's stock `&sk` sets that flag; a
  hand-rolled sticky-key node does not (devicetree booleans default to false). Without
  it, the sticky key's own generated key-down slips past the "don't catch my own press"
  filter in `behavior_sticky_key.c` — that filter compares `SELECT_MODS(param1)` with
  `ev->implicit_modifiers`, but for `LC(LA(LGUI))` the keycode itself is a modifier, so
  ZMK routes `LCTRL|LALT` into *explicit* modifiers and leaves the implicit set empty.
  The sticky key therefore treats its own activation as "another key was pressed" and
  releases the mods the instant the thumb comes up. `ignore-modifiers` (plus `lazy`, so
  the mods are only held around the next keypress) fixes it — see `sk_hyper`'s comment
  in the keymap. Not yet re-confirmed on hardware.

  AeroSpace has `start-at-login = false` in the shipped config — launch it manually
  (Spotlight → "AeroSpace") or flip the flag if you want auto-start. Without AeroSpace
  running, Hyper chords are no-ops.

  The inner thumbs send a raw Hyper hold/one-shot rather than switching to a
  dedicated ZMK layer, so window-management chords are defined entirely in the WM
  config (AeroSpace/OmniWM) and can be changed there without reflashing.

- **APP (app-level tabs + splits)** — held via the **2nd-from-edge left thumb** (TAB slot)
  or, mirrored, the **right BSPC thumb** on DEF_MAC. Hold with the hand opposite the keys you
  press: left thumb for the right-hand split keys, right thumb for the left-hand tab keys.
  The outermost thumbs (left/right) are disabled. Gives cross-hand control of
  tabs/splits across Firefox, kitty, Obsidian, Linear, DBGate using a single chord per action.
  No app-config edits except Obsidian (below).
  - Left half (tabs, works in all 5 apps): numpad `Cmd+1..9` = go-to tab (aligned with NUM
    digits); T/G = prev/next tab (`Ctrl+Shift+Tab` / `Ctrl+Tab`); B = new tab (`Cmd+T`);
    `G-Tab` slot = `Cmd+W` close tab.
  - Pinky col (screenshots): `S` slot (Q) = `Cmd+Shift+5` (screenshot UI); `A` = `Cmd+Ctrl+Shift+4`
    (region → clipboard, no file — the Ctrl modifier forces clipboard regardless of the macOS
    "Save to" setting). (This replaces the old screenshot-only layer.)
  - Right half (splits — kitty always; Obsidian focus after the binds below): home row
    `Cmd+Opt+hjkl` = directional split focus; Y/U/I/O = `Cmd+Opt+Shift+hjkl` move (kitty);
    M/`,` = `Cmd+Opt+-` / `Cmd+Opt+=` resize (kitty); N/`.` = `Cmd+D` / `Cmd+Shift+D`
    v/h-split (kitty). These match the user's existing `kitty.conf` binds.
  - **Caveat:** `Cmd+Opt+H` is macOS "Hide Others" globally; only press APP hjkl while
    kitty or Obsidian is focused (they intercept the chord before the system sees it).
  - **Obsidian setup (one-time):** Settings → Hotkeys → search "Focus on tab group" and
    assign left/down/up/right to `Cmd+Opt+H/J/K/L`. Go-to-tab (`Cmd+1..8`), next/prev tab,
    and `Cmd+T`/`Cmd+W` are already native. No other app needs configuration.

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

### Commit cadence and build trigger (rule)

**Commits:** each logical change can land as its own commit — feel free to commit
incrementally rather than batching unrelated edits. Always show the diff first and
wait for the user's "ok" before creating the commit.

**Builds: NEVER trigger automatically.** Even when a commit changes **firmware files**
(anything under `config/` — `hillside46.keymap`, `combos.dtsi`, `hillside46.conf`,
`west.yml` — or the build matrix `build.yaml`), do **NOT** push and run the workflow
on your own. Instead, after the commit, ask the user explicitly: *"готов билдить?"*
(or equivalent). The user often stacks several logical commits before flashing, so
they decide when to spend a build.

When the user confirms they're ready to build:

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

Commits that touch only `CLAUDE.md`, README, or other non-firmware / auxiliary
files never need a build at all — don't even ask.

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

### Never cite a bare key position

The user does not memorise key-position numbers. **Whenever a reply mentions one or
more key positions, it MUST include an ASCII map of the board with those positions
marked** — no exceptions, not even for a single position, not even if the same map was
printed earlier in the conversation.

Use this template (positions on top, the DEF_MAC legend underneath each slot), and
replace the slots under discussion with a marker such as `**` or the proposed binding:

```
       0    1    2    3    4    5                          6    7    8    9   10   11
       `    Q    W    E    R    T                          Y    U    I    O    P    [
      12   13   14   15   16   17                         18   19   20   21   22   23
    G-Tab   A    S    D    F    G                          H    J    K    L    ;    '
      24   25   26   27   28   29   30              31   32   33   34   35   36   37
       -    Z    X    C    V    B   ESC           RA/INS   N    M    ,    .    /    \
                         38   39   40   41      42   43   44   45
                        (--) TAB  SPC  ENT     HYP   CW  BSPC (--)
```

`38` and `45` are the physically removed outer thumbs — always draw them as `(--)`.

When the discussion is about the Russian ЙЦУКЕН layer, add the Cyrillic letter next to
the Latin one for the keys under discussion (e.g. `[` = Х, `]` = Ъ, `;` = Ж, `'` = Э),
since the user's motivation is usually the Russian legend, not the Latin one.
