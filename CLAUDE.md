# CLAUDE.md

Macros and post processor for a Syil X7 3-axis mill running an
**Advantech-LNC** control. The repo root holds hand-written `G65 "NAME"`
macros (probing, tool loading, calibration), the Fusion 360 post
(`syil_lnc_toolpath.cps`), and the control manuals (PDFs).

## Read before you edit

- **Probe / calibration macros** (`PROBE*`, `CALIBRATE*`, `FINDCOR`,
  `COMPZEROPOINT`, `CHECKPOSITIONALTOLERANCE`, `PROTECTEDMOVE`): read
  [probe_macros_steering.md](probe_macros_steering.md) first — call hierarchy,
  M19/M20 spindle-orient rules, the dwell gotcha, and code idioms.
- **Control specifics** (logins, FTP, parameter numbers, 4th-axis drive
  settings): [control_notes.md](control_notes.md).

## Must not get wrong

- **The machine runs the named macro files** (`PROBEBORE`, `PROBEXSLOT`, …),
  called with `G65 "NAME"`. **Edit those.** The `maker_macros/maker_macro_m###`
  files are *generated* from them by `build_maker_macros.py` — editing the
  generated files does nothing for the operator.
- Units are inch (`G20`). Probe tuning (feeds, clearances, backoff,
  tolerances) lives in `PROBECONFIG` globals `@100`–`@116`, not in each macro.
- These can only be truly verified **at the machine** — isolate behavior in
  MDI, then test a macro standalone via `G65 "NAME"`, then through its wrapper.
  Reason about the G-code statically here; the repo PDFs are the syntax
  reference (`pdftotext -layout` + grep).

## Maker macros

After editing a named macro, regenerate the `maker_macros/` copies for repo
consistency: `python build_maker_macros.py` (CI also does this — see
`.github/`).
