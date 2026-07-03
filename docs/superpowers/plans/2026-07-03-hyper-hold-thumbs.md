# Hyper-Hold on Inner Thumbs Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the WM_MAC layer-activation hold on `def_mac_layer`'s two inner thumbs with a raw Hyper (Ctrl+Opt+Cmd) modifier hold, so any Hyper hotkey works through them instead of a fixed chord set.

**Architecture:** Add one macro (`HYPER_HOLD`, mods-only Hyper) and two new hold-tap-adjacent behaviors (`sk_hyper`, a sticky-key with a 5s timeout; `hyper_ht`, a hold-tap whose hold is a plain `&kp` and whose tap is `sk_hyper`). Rewire the two inner thumb keys on `def_mac_layer` to use these. Update the in-file ASCII diagrams/comments and `CLAUDE.md` to describe the new mechanism and mark `WM_MAC` as dormant (kept, unreferenced).

**Tech Stack:** ZMK devicetree keymap (`config/hillside46.keymap`), Zephyr devicetree hold-tap/sticky-key behaviors, GitHub Actions build (manual trigger only).

## Global Constraints

- Spec: `docs/superpowers/specs/2026-07-03-hyper-hold-thumbs-design.md`.
- Only these two files are touched: `config/hillside46.keymap`, `CLAUDE.md`.
- Do **not** delete the `WM_MAC` layer, its `#define`, or the `wm_mac_layer` block — keep them, just unreferenced.
- Only `def_mac_layer` position 41 (left inner thumb) and position 42 (right inner thumb) change. `def_layer`'s `WM` thumbs (Win/Lin) are untouched.
- Left thumb (41): hold = Hyper (`HYPER_HOLD` = `LC(LA(LGUI))`), tap = `RETURN` (unchanged).
- Right thumb (42): hold = Hyper, tap = one-shot Hyper via a dedicated sticky-key (`release-after-ms = <5000>`, `quick-release`) — separate from the existing generic `&sk` (used by `ss_cw`) so its timeout doesn't affect caps-word.
- New hold-tap `hyper_ht`: `flavor = "balanced"`, `tapping-term-ms = <200>` (matches this file's `&mt` convention set at the top of the file).
- This project has no unit-test suite and no local ZMK/Zephyr toolchain — "tests" in this plan are syntax/consistency checks (brace balance, grep for exact strings) run locally with `bash`/`python3`, not a real devicetree compile.
- **Never trigger a GitHub Actions build automatically.** Per `CLAUDE.md`, firmware-file changes require asking the user "готов билдить?" before pushing/building — this happens once, after all tasks are done (see "After implementation" at the end of this plan), not per task.
- **Show the diff and wait for the user's "ok" before creating any commit** (project convention from `CLAUDE.md`). Every "commit" step below means: run `git diff`, show it, wait for explicit approval, then commit — never commit unprompted.

---

## Task 1: Add `HYPER_HOLD` macro and the `sk_hyper` / `hyper_ht` behaviors

**Files:**
- Modify: `config/hillside46.keymap:37` (macro section)
- Modify: `config/hillside46.keymap:141-148` (behaviors section, after `cw_ht`)

**Interfaces:**
- Produces: macro `HYPER_HOLD` (usable anywhere a `&kp`/`&mt`-style modifier argument is expected); behavior `sk_hyper` (`#binding-cells = <1>`, takes one modifier/keycode param); behavior `hyper_ht` (`#binding-cells = <2>`, takes `(hold_param, tap_param)`, both consumed by Task 2).

- [ ] **Step 1: Add the `HYPER_HOLD` macro**

Read `config/hillside46.keymap` and find this exact block (currently lines 35-37):

```c
/* Hyper = Ctrl+Opt+Cmd (mac AeroSpace) / Ctrl+Alt+Win (FancyZones) - same three mods.
 * Use HYPER(key) instead of spelling LC(LA(LG(key))) by hand. */
#define HYPER(k) LC(LA(LG(k)))
```

Replace it with:

```c
/* Hyper = Ctrl+Opt+Cmd (mac AeroSpace) / Ctrl+Alt+Win (FancyZones) - same three mods.
 * Use HYPER(key) instead of spelling LC(LA(LG(key))) by hand. */
#define HYPER(k) LC(LA(LG(k)))
/* Hyper as a bare modifier (no extra keycode) - for hold-tap "hold" params that
 * should just press/release the three mods, e.g. `&mt HYPER_HOLD RETURN`. */
#define HYPER_HOLD LC(LA(LGUI))
```

- [ ] **Step 2: Add the `sk_hyper` and `hyper_ht` behaviors**

In the same file, find the `cw_ht` behavior block and the closing brace of the
`behaviors { ... }` block right after it (currently lines 141-149):

```c
                cw_ht: behavior_mo_cw {
                    compatible = "zmk,behavior-hold-tap";
                        label = "cw_ht";
                        #binding-cells = <2>;
                        flavor = "hold-preferred";
                        tapping-term-ms = <200>;
                        bindings = <&mo>, <&ss_cw>;
                };           
        };
```

Replace it with (adds two new behaviors between `cw_ht` and the closing
`};` of the `behaviors` block; leave everything else in that block as-is):

```c
                cw_ht: behavior_mo_cw {
                    compatible = "zmk,behavior-hold-tap";
                        label = "cw_ht";
                        #binding-cells = <2>;
                        flavor = "hold-preferred";
                        tapping-term-ms = <200>;
                        bindings = <&mo>, <&ss_cw>;
                };
                // Dedicated sticky-key for Hyper one-shot (separate from the
                // generic &sk used by ss_cw, so this 5s timeout doesn't affect
                // caps-word's sticky-shift behavior).
                sk_hyper: sticky_key_hyper {
                        compatible = "zmk,behavior-sticky-key";
                        label = "SK_HYPER";
                        #binding-cells = <1>;
                        bindings = <&kp>;
                        release-after-ms = <5000>;
                        quick-release;
                };
                // Hold = hold the modifier (plain &kp); tap = one-shot the
                // modifier (sk_hyper). Both params of a call site should be
                // the same modifier expression, e.g. `&hyper_ht HYPER_HOLD HYPER_HOLD`.
                hyper_ht: hyper_hold_tap {
                        compatible = "zmk,behavior-hold-tap";
                        label = "HYPER_HT";
                        #binding-cells = <2>;
                        flavor = "balanced";
                        tapping-term-ms = <200>;
                        bindings = <&kp>, <&sk_hyper>;
                };
        };
```

- [ ] **Step 3: Verify the macro and behaviors were added exactly once**

Run:
```bash
grep -c "^#define HYPER_HOLD LC(LA(LGUI))$" config/hillside46.keymap
grep -c "sk_hyper: sticky_key_hyper {" config/hillside46.keymap
grep -c "hyper_ht: hyper_hold_tap {" config/hillside46.keymap
```
Expected: each command prints `1`.

- [ ] **Step 4: Verify brace balance (no unbalanced `{`/`}` introduced)**

Run:
```bash
python3 -c "
s = open('config/hillside46.keymap').read()
o, c = s.count('{'), s.count('}')
assert o == c, f'brace mismatch: {{={o} }}={c}'
print(f'OK: braces balanced ({o} each)')
"
```
Expected: `OK: braces balanced (N each)` for some equal N — must not raise `AssertionError`.

- [ ] **Step 5: Show diff and commit**

```bash
git diff config/hillside46.keymap
```
Show this diff to the user and wait for their explicit "ok". Once approved:
```bash
git add config/hillside46.keymap
git commit -m "$(cat <<'EOF'
keymap: add HYPER_HOLD macro and sk_hyper/hyper_ht behaviors

Groundwork for driving AeroSpace/OmniWM via a raw Hyper hold instead
of the fixed WM_MAC chord layer.
EOF
)"
```

---

## Task 2: Rewire `def_mac_layer`'s inner thumbs to use Hyper hold

**Files:**
- Modify: `config/hillside46.keymap:199-214` (`def_mac_layer` comment + bindings)

**Interfaces:**
- Consumes: macro `HYPER_HOLD`, behaviors `sk_hyper`/`hyper_ht` (from Task 1).

- [ ] **Step 1: Update the `def_mac_layer` comment block**

Find this exact comment (currently lines 199-208):

```c
/* QWERTY macOS (boot default, layer 0) | G-Tab=Cmd+Tab (switch to last app; tap toggles 2 most recent)
 * thumbs left: ZJ/BSPC (zellij, leftmost) · APP/TAB (app tabs+splits) · NUM · WM_MAC (window mgmt)
 * thumbs right: WM_MAC · NAV_MAC · APP/BSPC (app, mirror) · ZJ/DEL (zellij, mirror)
 * HRM mac (Cmd<->Ctrl swapped vs Win): A=Ctrl S=Opt D=Shift F=Cmd  /  J=Cmd K=Shift L=Opt ;=Ctrl  (Cmd on index, Ctrl on pinky)
 * -------------------------------------------------------------------------------------------------------------------------------------
 * |  `    |   Q   |   W   |   E   |   R   |   T   |-----------------------------------|   Y   |   U   |   I   |   O   |   P   |  [/]  |
 * | G-Tab |   A   |   S   |   D   |   F   |   G   |-----------------------------------|   H   |   J   |   K   |   L   |   ;   |   '   |
 * |   -   |   Z   |   X   |   C   |   V   |   B   |  ESC  |-------------------|RA/INS |   N   |   M   |   ,   |   .   |   /   |   \   |
 *                                 |ZJ/BSPC|APP/TAB|NUM/SPC|WM/ENT |WM/ENT |NAV/CW |APP/BSP|ZJ/DEL |
 */
```

Replace it with:

```c
/* QWERTY macOS (boot default, layer 0) | G-Tab=Cmd+Tab (switch to last app; tap toggles 2 most recent)
 * thumbs left: ZJ/BSPC (zellij, leftmost) · APP/TAB (app tabs+splits) · NUM · Hyper hold/Enter tap
 * thumbs right: Hyper hold/one-shot tap (5s) · NAV_MAC · APP/BSPC (app, mirror) · ZJ/DEL (zellij, mirror)
 * Inner thumbs hold = raw Hyper (Ctrl+Opt+Cmd) modifier, no layer switch - press any Hyper
 * hotkey (AeroSpace/OmniWM) while held. Right thumb tap = one-shot Hyper (sticky, 5s idle timeout).
 * WM_MAC layer (below) is no longer wired to any key; kept for possible future rebinding.
 * HRM mac (Cmd<->Ctrl swapped vs Win): A=Ctrl S=Opt D=Shift F=Cmd  /  J=Cmd K=Shift L=Opt ;=Ctrl  (Cmd on index, Ctrl on pinky)
 * -------------------------------------------------------------------------------------------------------------------------------------
 * |  `    |   Q   |   W   |   E   |   R   |   T   |-----------------------------------|   Y   |   U   |   I   |   O   |   P   |  [/]  |
 * | G-Tab |   A   |   S   |   D   |   F   |   G   |-----------------------------------|   H   |   J   |   K   |   L   |   ;   |   '   |
 * |   -   |   Z   |   X   |   C   |   V   |   B   |  ESC  |-------------------|RA/INS |   N   |   M   |   ,   |   .   |   /   |   \   |
 *                                 |ZJ/BSPC|APP/TAB|NUM/SPC|HYP/ENT|HYP/OSH|NAV/CW |APP/BSP|ZJ/DEL |
 */
```

- [ ] **Step 2: Rewire the thumb bindings**

Find this exact line (currently line 213):

```c
                                &lt ZJ BSPC &lt APP TAB &lt NUM SPACE &lt WM_MAC RETURN     &lt WM_MAC RETURN &cw_ht NAV_MAC 0 &lt APP BSPC &lt ZJ DEL
```

Replace it with:

```c
                                &lt ZJ BSPC &lt APP TAB &lt NUM SPACE &mt HYPER_HOLD RETURN     &hyper_ht HYPER_HOLD HYPER_HOLD &cw_ht NAV_MAC 0 &lt APP BSPC &lt ZJ DEL
```

- [ ] **Step 3: Verify no `&lt WM_MAC` remains and the new bindings are present exactly once**

Run:
```bash
grep -c "&lt WM_MAC" config/hillside46.keymap
grep -o "&mt HYPER_HOLD RETURN" config/hillside46.keymap | wc -l
grep -o "&hyper_ht HYPER_HOLD HYPER_HOLD" config/hillside46.keymap | wc -l
```
Expected: first command prints `0`; the other two each print `1` (with possible
leading whitespace from `wc -l`, e.g. `       1`).

- [ ] **Step 4: Verify brace balance again**

Run the same check as Task 1 Step 4:
```bash
python3 -c "
s = open('config/hillside46.keymap').read()
o, c = s.count('{'), s.count('}')
assert o == c, f'brace mismatch: {{={o} }}={c}'
print(f'OK: braces balanced ({o} each)')
"
```
Expected: `OK: braces balanced (N each)`, no `AssertionError`.

- [ ] **Step 5: Show diff and commit**

```bash
git diff config/hillside46.keymap
```
Show this diff to the user and wait for their explicit "ok". Once approved:
```bash
git add config/hillside46.keymap
git commit -m "$(cat <<'EOF'
keymap: switch def_mac_layer inner thumbs to raw Hyper hold

Left thumb: hold = Hyper, tap = Enter (unchanged). Right thumb:
hold = Hyper, tap = one-shot Hyper (5s). Replaces the fixed WM_MAC
chord layer with a generic modifier hold usable with any AeroSpace
or OmniWM hotkey.
EOF
)"
```

---

## Task 3: Mark `WM_MAC` as dormant in comments and update `CLAUDE.md`

**Files:**
- Modify: `config/hillside46.keymap:315-318` (`wm_mac_layer` comment)
- Modify: `CLAUDE.md:31` (layers table)
- Modify: `CLAUDE.md:84-101` (macOS-side dependencies section)

**Interfaces:** None (documentation only; no new code interfaces).

- [ ] **Step 1: Mark `wm_mac_layer`'s comment as dormant**

In `config/hillside46.keymap`, find this exact comment (currently lines 315-318):

```c
                wm_mac_layer {
/* WM_MAC (macOS window management) - AeroSpace control via Hyper (GAC = Ctrl+Opt+Cmd) chords
 * left: workspaces 1-9 numpad aligned with NUM layer's digits | H-Tab back-and-forth (over Cmd+Tab) | T/G/B = H-[ prev / H-] next / H-0 workspace 0
 * right: focus on home row (H-HJKL) | move above home row (Y/U/I/O = HS-HJKL) | resize M/, = H-- / H-= | layout: N = tiles (H-/), . = accordion (H-,) | ; (pinky) = HS-; -> AeroSpace service mode
```

Replace it with:

```c
                wm_mac_layer {
/* WM_MAC (macOS window management) - DORMANT: no key in this keymap activates this
 * layer anymore. Superseded by the raw Hyper hold on def_mac_layer's inner thumbs
 * (see that layer's comment). Kept for possible future rebinding; the chord table
 * below is only accurate if this layer is wired to a key again.
 * AeroSpace control via Hyper (GAC = Ctrl+Opt+Cmd) chords
 * left: workspaces 1-9 numpad aligned with NUM layer's digits | H-Tab back-and-forth (over Cmd+Tab) | T/G/B = H-[ prev / H-] next / H-0 workspace 0
 * right: focus on home row (H-HJKL) | move above home row (Y/U/I/O = HS-HJKL) | resize M/, = H-- / H-= | layout: N = tiles (H-/), . = accordion (H-,) | ; (pinky) = HS-; -> AeroSpace service mode
```

(The rest of that comment block — the ASCII diagram lines below — stays unchanged.)

- [ ] **Step 2: Update the layers table in `CLAUDE.md`**

Find this exact line (currently `CLAUDE.md:31`):

```markdown
| 7 | WM_MAC  | Window management (macOS via AeroSpace)           |
```

Replace it with:

```markdown
| 7 | WM_MAC  | Window management (macOS via AeroSpace) — dormant, see below |
```

- [ ] **Step 3: Rewrite the "Window management" bullet in `CLAUDE.md`**

Find this exact block (currently `CLAUDE.md:84-101`):

```markdown
- **Window management** (WM_MAC): driven by [AeroSpace](https://nikitabobko.github.io/AeroSpace/),
  install it (`brew install --cask nikitabobko-tap/aerospace`) and place the project's
  `~/.aerospace.toml`. AeroSpace bindings live on **Hyper = `Ctrl+Opt+Cmd`** (not the
  default `Alt`, which collides with zellij's `Alt+hjkl/-/=/f/i/n/o/p` and several
  Cmd+Opt+* macOS shortcuts). WM_MAC sends Hyper-chords matching the config:

  | Layer keys (left)              | Hyper chord     | AeroSpace action                  |
  |--------------------------------|-----------------|-----------------------------------|
  | W E R / S D F / X C V          | ⌃⌥⌘+7..9/4..6/1..3 | `workspace 7..9 / 4..6 / 1..3` (aligned with NUM layer digits) |
  | `AC-Tab` slot (col 1, mid row) | ⌃⌥⌘+Tab         | `workspace-back-and-forth`        |
  | T / G                          | ⌃⌥⌘+[ / ⌃⌥⌘+]   | `workspace prev` / `workspace next` (G is lower → next, easier to reach) |

  | Layer keys (right)             | Hyper chord     | AeroSpace action                  |
  |--------------------------------|-----------------|-----------------------------------|
  | H J K L (home row)             | ⌃⌥⌘+H/J/K/L     | `focus left / down / up / right`  |
  | Y U I O (row above home)       | ⌃⌥⌘+⇧+H/J/K/L   | `move left / down / up / right`   |
  | M / `,`                        | ⌃⌥⌘+- / ⌃⌥⌘+=  | `resize smart -50 / +50`          |
  | `;` (right pinky home row)     | ⌃⌥⌘⇧+;          | `mode service` (enter service binding mode) |

  AeroSpace has `start-at-login = false` in the shipped config — launch it manually
  (Spotlight → "AeroSpace") or flip the flag if you want auto-start. Without AeroSpace
  running, the Hyper-chords above are no-ops.
```

Replace it with:

```markdown
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

  AeroSpace has `start-at-login = false` in the shipped config — launch it manually
  (Spotlight → "AeroSpace") or flip the flag if you want auto-start. Without AeroSpace
  running, Hyper chords are no-ops.

  The `WM_MAC` layer below is dormant — no key in the keymap activates it anymore,
  since the inner thumbs now send a raw Hyper hold/one-shot instead of switching to
  this layer. It's kept for possible future rebinding. Its fixed Hyper-chord table
  (accurate only if the layer is wired to a key again):

  | Layer keys (left)              | Hyper chord     | AeroSpace action                  |
  |--------------------------------|-----------------|-----------------------------------|
  | W E R / S D F / X C V          | ⌃⌥⌘+7..9/4..6/1..3 | `workspace 7..9 / 4..6 / 1..3` (aligned with NUM layer digits) |
  | `AC-Tab` slot (col 1, mid row) | ⌃⌥⌘+Tab         | `workspace-back-and-forth`        |
  | T / G                          | ⌃⌥⌘+[ / ⌃⌥⌘+]   | `workspace prev` / `workspace next` (G is lower → next, easier to reach) |

  | Layer keys (right)             | Hyper chord     | AeroSpace action                  |
  |--------------------------------|-----------------|-----------------------------------|
  | H J K L (home row)             | ⌃⌥⌘+H/J/K/L     | `focus left / down / up / right`  |
  | Y U I O (row above home)       | ⌃⌥⌘+⇧+H/J/K/L   | `move left / down / up / right`   |
  | M / `,`                        | ⌃⌥⌘+- / ⌃⌥⌘+=  | `resize smart -50 / +50`          |
  | `;` (right pinky home row)     | ⌃⌥⌘⇧+;          | `mode service` (enter service binding mode) |
```

- [ ] **Step 4: Verify the doc edits landed**

Run:
```bash
grep -ci "dormant" config/hillside46.keymap CLAUDE.md
grep -c "HYP/ENT\|HYP/OSH" CLAUDE.md
grep -c "&lt WM_MAC\|&mo WM_MAC" config/hillside46.keymap
```
Expected: first command shows at least `1` for each file; second shows `2`
(`grep -c` counts matching lines, and `HYP/ENT` and `HYP/OSH` land on two
separate wrapped lines in the replacement text); third command reports `0`.

- [ ] **Step 5: Re-run the brace balance check on the keymap**

```bash
python3 -c "
s = open('config/hillside46.keymap').read()
o, c = s.count('{'), s.count('}')
assert o == c, f'brace mismatch: {{={o} }}={c}'
print(f'OK: braces balanced ({o} each)')
"
```
Expected: `OK: braces balanced (N each)`, no `AssertionError`.

- [ ] **Step 6: Show diff and commit**

```bash
git diff config/hillside46.keymap CLAUDE.md
```
Show this diff to the user and wait for their explicit "ok". Once approved:
```bash
git add config/hillside46.keymap CLAUDE.md
git commit -m "$(cat <<'EOF'
docs: mark WM_MAC layer dormant, document Hyper-hold thumbs

WM_MAC is kept in the keymap but no longer wired to any key. CLAUDE.md
now documents the inner-thumb Hyper hold/one-shot as the current way
to drive AeroSpace/OmniWM, with the old WM_MAC chord table kept as
historical reference.
EOF
)"
```

---

## After implementation

This plan only touches `config/hillside46.keymap` (a firmware file) and
`CLAUDE.md`. Per the project's build-trigger rule, do **not** push the branch
or run the build workflow on your own. Once all three tasks are committed,
ask the user explicitly: **"готов билдить?"** If they confirm, follow the
build procedure in `CLAUDE.md` ("Commit cadence and build trigger (rule)"):
push to `origin`, trigger `gh workflow run build.yml --repo GlebYavorski/hillside46 --ref <branch>`,
watch it, then download the `.uf2` artifacts into `~/Downloads/hillside46-firmware/`
(wiping that folder first).

Before flashing, the user should manually verify on hardware:
- Left inner thumb: hold → Hyper mods engage (test with a known AeroSpace bind); tap → Enter.
- Right inner thumb: hold → Hyper mods engage; tap once → next keypress gets Hyper applied; tap once and wait 5s without pressing anything → Hyper clears (next keypress is unaffected).
