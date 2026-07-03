# Hyper-hold on inner thumbs (replacing WM_MAC layer activation)

## Problem

The two inner thumb keys on `def_mac_layer` (key positions 41 and 42) currently
both run `&lt WM_MAC RETURN`: hold activates the `WM_MAC` layer (a fixed set of
Hyper chords for AeroSpace), tap sends Enter. The fixed chord layout is too
rigid — it only covers the specific binds baked into `WM_MAC`, and doesn't
generalize to other Hyper-driven tools (e.g. OmniWM) or arbitrary AeroSpace
rebinds.

## Goal

Replace the layer-activation hold with a raw Hyper modifier (Ctrl+Opt+Cmd)
hold: while held, any other key pressed becomes `Hyper+key`, using whatever
that key already emits on the active layer (base QWERTY, or NUM/NAV if held
simultaneously with another thumb). This lets the user press *any* Hyper
hotkey their WM tool defines, without needing a dedicated ZMK layer to
remap physical positions to it.

The `WM_MAC` layer itself is **not deleted** — it's kept for possible future
use — but nothing in the keymap will activate it anymore.

## Design

### New macro

```c
#define HYPER_HOLD LC(LA(LGUI))   // Ctrl+Opt+Cmd as a bare modifier, no extra key
```

Placed next to the existing `#define HYPER(k) LC(LA(LG(k)))` macro. Same
modifier composition, but with no trailing keycode — used as the "hold"
parameter of a hold-tap so the physical key just presses/releases the three
modifiers.

### New behaviors

```c
sk_hyper: sticky_key_hyper {
    compatible = "zmk,behavior-sticky-key";
    label = "SK_HYPER";
    #binding-cells = <1>;
    bindings = <&kp>;
    release-after-ms = <5000>;
    quick-release;
};

hyper_ht: hyper_hold_tap {
    compatible = "zmk,behavior-hold-tap";
    label = "HYPER_HT";
    #binding-cells = <2>;
    flavor = "balanced";
    tapping-term-ms = <200>;
    bindings = <&kp>, <&sk_hyper>;
};
```

- `sk_hyper` is a dedicated sticky-key instance (separate from the generic
  `&sk` used by `ss_cw`/caps-word) so its 5s timeout doesn't affect other
  sticky-key uses.
- `hyper_ht` is a custom hold-tap whose hold action is a plain `&kp` (holds
  the modifier for as long as the physical key is held) and whose tap
  action is `&sk_hyper` (applies the modifier to the next keypress, or
  clears after 5s of inactivity). `flavor`/`tapping-term-ms` match the
  project's existing `&mt` convention (`balanced`, 200ms).

### Keymap changes (`def_mac_layer`, positions 41 & 42)

Both currently read `&lt WM_MAC RETURN`.

- **Left inner thumb (41):** `&mt HYPER_HOLD RETURN` — hold = hold Hyper,
  tap = Enter (unchanged from today).
- **Right inner thumb (42):** `&hyper_ht HYPER_HOLD HYPER_HOLD` — hold =
  hold Hyper, tap = one-shot Hyper (sticky, 5s timeout).

This is intentionally asymmetric: only the right thumb gets the one-shot-tap
behavior; the left thumb keeps its existing Enter tap.

### Known behavior change

Previously, `WM_MAC` remapped physical keys to fixed chords (e.g. W/E/R on
the top row → `Hyper+7/8/9`, aligned with the NUM layer's digit positions).
With the raw-hold approach, holding the thumb no longer switches layers, so
a key pressed while holding Hyper emits whatever it already emits on the
currently active layer (base QWERTY unless another layer, e.g. NUM, is held
concurrently on the opposite thumb). Reaching a digit chord now requires
holding NUM and the Hyper thumb together, same as it would without any
layer. This is the intended tradeoff: generality over a fixed, pre-mapped
chord set.

### Documentation updates

- `def_mac_layer`'s ASCII diagram and prose comment: relabel the two inner
  thumb cells (`WM/ENT` / `WM/ENT`) to `HYP/ENT` (left) and `HYP/OSH`
  (right), and update the thumb-row description.
- `wm_mac_layer` block: add a short note that it is currently unbound (no
  key in the keymap activates it) — kept for potential future rebinding.
- `CLAUDE.md`:
  - Layers table row for `WM_MAC`: note it's currently unbound.
  - "Window management (WM_MAC)" subsection: mark the Hyper-chord table as
    describing the dormant `WM_MAC` layer (historical reference for anyone
    who re-wires it later), and add a short new subsection documenting the
    inner-thumb Hyper hold / one-shot mechanism as the current way to drive
    AeroSpace/OmniWM.

## Out of scope

- No changes to the Win/Lin `WM` layer or `def_layer` thumbs.
- No changes to AeroSpace's `~/.aerospace.toml` or any app-side config —
  that's on the user to set up Hyper-based binds for whatever they need.
- No deletion of the `WM_MAC` layer or its `#define`.
