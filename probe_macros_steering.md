# Steering: editing the probing macros

How to work efficiently on the probe/calibration macros in this repo. Read
this before editing any `PROBE*`, `CALIBRATE*`, `FINDCOR`, `COMPZEROPOINT`,
`CHECKPOSITIONALTOLERANCE`, or `PROTECTEDMOVE` macro.

## What actually runs on the machine

- **The machine runs the human-readable named files** (`PROBEBORE`,
  `PROBEXSLOT`, …) called with `G65 "NAME"`. **Edit those.**
- `maker_macros/maker_macro_m###` are **generated** from the named files by
  `build_maker_macros.py` (it rewrites `G65 "NAME"` → `M###`). They are *not*
  what gets called here. Regenerate them only to keep the repo consistent;
  changing them does nothing for the operator.
- Units are inch (`G20`). Feeds, clearances, backoff, and tolerances all come
  from `PROBECONFIG` globals (`@100`–`@116`). Change tuning there, not in each
  macro.

## Call hierarchy — where the logic actually lives

- `PROBECONFIG` is called first by **every** macro; it loads tool data and
  the `@1xx` globals and alarms if the probe tool isn't loaded.
- Wrapper macros delegate and contain **no motion or orientation of their
  own**:
  - `PROBEBORE` → `PROBEXSLOT` + `PROBEYSLOT` (×2 for center then measure)
  - `PROBECIRCULARBOSS` → `PROBEXWEB` + `PROBEYWEB` (×2)
- So when a wrapper "seems to be missing" an `M19`, dwell, error check, etc.,
  **look one level down in the slot/web macro it calls** — that's where the
  real work is. Don't add motion logic to the wrappers.

## Spindle orientation (M19 / M20) — the tricky part

Goal: keep the **same point of the ruby** contacting the part, so the spindle
is oriented per probe direction.

- `M20` = unlock/cancel orientation. `M19 P<angle>` = orient to `<angle>`.
  Bare `M19` (no P) = 0°. **Verified at MDI:** `M20` then `M19 P<angle>`
  works, and the orient holds through a following `G31`.
- **You must `M20` before each new `M19 P` re-orient.** The slot/web macros
  already pair them (`M20` then `M19 P…`) before every probe direction, and
  a final `M20` at the end for safety.
- Direction→angle values are **hardcoded per macro** and are intentionally
  kept as-is (they carry a consistent +90° offset from a pure travel-direction
  convention — that's fine, don't "correct" it). The X macros happen to use
  `M19 P0`; the Y macros use P90/P270. `P0` is fine on this control.

### Known gotcha: a second orient in a macro may not fire

Symptom seen: `PROBEXSLOT` did only **one** orient even though it has two
`M19`s (line ~19 `M19 P180`, line ~53 `M19 P0`). The first fires; the second,
which comes right after a `G01` back-off move, can fail to take effect.

- Root cause is **timing/block sequencing**, not `P0` (P0 works) and not the
  `M19`/`G31` interaction (that holds fine in MDI).
- **Fix pattern under test: add a dwell so the orient settles before the next
  block.** Put it right after the second `M19`:
  ```
  M20
  M19 P0
  G04 X0.5        // let the orient settle before probing
  G31 G91 P2 X[...] F#111
  ```
  If that doesn't do it, move the dwell *before* the `M20` so the preceding
  `G01` fully stops first.

### Dwell syntax (LNC, from the Programming Manual §G04)

- `G04 X<sec>` — seconds, decimal OK, 0.001–99999.999 (e.g. `G04 X0.5`)
- `G04 P<ms>` — milliseconds, **no decimal**, 1–99999999 (e.g. `G04 P500`)
- `G4` == `G04`. `G04` with no time is an exact-stop, not a pause. Prefer the
  `X<sec>` form here to avoid confusion with the `P` on `M19`/`G31` blocks.

## Common macro idioms (so edits match the surrounding code)

- **Probe move:** fast `G31 G91 P2 <axis> F#111` → back off `G01` by `#108`
  → slow `G31 … F#112` → read `R_SKIP[0,20x]` → back off again.
- **Error check after a probe:** `IF[R_SKIP[0,1] == 0] ALARM[...] GOTO1
  END_IF`, where `GOTO1` jumps to the `N1` label near the end and exits. A
  premature-collision check flips the test to `== 1`.
- **WCS write:** guarded by `IF[#1 < 53 || #1 == #0 || #9 > 0]` (skip write),
  else `W_G53G59_COOR` or extended `W_G54EXP_COOR` using `#114 = ROUND[[#1 -
  FIX[#1]] * 10]`.
- **G65 args** map by letter to locals (per each macro's header: `A#1 B#2 C#3
  I#9 Q#17 R#18 S#19`). If you add a new argument, **verify the letter→#var
  mapping on this control** before trusting it — don't assume Fanuc-standard
  letters.

## Verifying changes

- These can only be truly verified at the machine. Isolate behavior in **MDI**
  first (that's how we confirmed `M20;M19 P;G31` holds and that the second
  orient was the failure), then test the macro standalone via `G65 "NAME"`
  before running it through a wrapper.
- Reason about the G-code statically here; the manuals in the repo
  (`LNC-Programming Manual…pdf`, `LinuxMACROManual…pdf`) are the reference for
  code/register syntax — `pdftotext -layout` them and grep.
