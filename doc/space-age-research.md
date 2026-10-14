# Research: Adding Factorio Space Age items to factompiler

Date: 2026-10 (research session). All claims below were verified against the
installed venv, the pinned dependency source, and live GitHub/wube data.

## TL;DR

factompiler **already supports Space Age** for everything in Factorio 2.0:
all game-data knowledge (entities, signals, footprints) is delegated to the
`factorio-draftsman` dependency, whose pinned fork carries **Factorio 2.0.72**
data. The only real gap is *freshness*: the fork lags upstream by 45 commits
and misses a handful of **2.1.x** items. The recommended fix is to point the
dependency at upstream `redruin1/factorio-draftsman` (4.0.1), which was
verified to work with factompiler unchanged.

## How factorio data reaches the compiler

- `pyproject.toml` pins:
  `factorio-draftsman @ git+https://github.com/Snagnar/factorio-draftsman.git@main`
  (installed in `.venv`: v3.2.0, fork commit `89a7fcc`, 2026-01-29).
- factompiler never hardcodes the item/entity catalog. It queries:
  - `draftsman.data.signals` → `raw`, `type_of`, `add_signal`
    (`dsl_compiler/src/common/signal_registry.py`, `lowering/lowerer.py`)
  - `draftsman.data.entities` → `raw` via `EntityDataHelper.get_footprint()`
    (`dsl_compiler/src/common/entity_data.py`)
  - `draftsman.entity.new_entity()` for emission.
- draftsman's data is a set of pickles (`draftsman/data/*.pkl`) generated from
  the official **`wube/factorio-data`** repo (game Lua prototypes) by running
  the Factorio data lifecycle in Lua (`draftsman update` /
  `draftsman.environment.update.update_draftsman_data`). Space Age is included
  by default (`owned_dlc=["space-age"]`).

## Current state (verified)

- Installed data = Factorio **2.0.72** (fork's `factorio-data` submodule pin,
  Oct 2025): 1767 signals, 158 entities — includes `fusion-reactor`,
  `asteroid-collector`, `linked-belt`, `space-platform-hub`, quality signals,
  `space-science-pack`, `rocket-fuel`, etc.
- End-to-end smoke tests (in `temp/`):
  - `Signal fuel = ("rocket-fuel", 0); ... place("small-lamp", ...)` compiles
    to a valid blueprint with `version` 2.0.0 and `"quality": "normal"` in
    constant-combinator filters.
  - `place("fusion-reactor")` / `place("asteroid-collector")` (large Space Age
    footprints) compile and place correctly.
- `lowering/lowerer.py::_infer_signal_category` already understands 2.0
  prototype types: `item`, `fluid`, `entity`, `space-location`,
  `asteroid-chunk`, `quality`.

## Gaps found

1. **Data freshness (the main gap).** Current game is 2.1.21 (wube/factorio-data
   master). Diffing wube master against the installed draftsman data shows 2.1
   items missing from factompiler's signal validation:
   - `satellite` (base, 2.1)
   - `landing-pad-unloading-bay` (space-age, 2.1)
   - `electric-energy-interface-equipment` (base, hidden equipment item)
   ("parameter-" hits in a naive scan are regex artifacts, not real items.)
2. **Regeneration blocker in the fork.** `draftsman update` (3.2.0) crashes on
   2.1.x game data during the `data-updates` stage:
   `defines.constant` is nil (base/data-updates.lua line 59). Upstream fixed
   this class of lifecycle bugs in 4.x (see upstream changelog: "global unpack
   missing from data stage", `parse_energy` fixes, etc.). So the fork cannot
   simply bump its submodule to 2.1 and regenerate; it must first sync to
   upstream 4.x.
3. **`place()` prototype validation.** Unknown entity names in `place(...)`
   silently fall back to a 1×1 footprint during layout (the only signal is an
   `UnknownEntityWarning` raised by draftsman's `new_entity` during emission,
   called from `entity_emitter.py`). A Space Age entity
   missing from the data would be laid out with the wrong footprint.

## Dependency topology (verified via GitHub API)

- `Snagnar/factorio-draftsman` is a pure mirror of
  `redruin1/factorio-draftsman`: compare fork@89a7fcc...upstream/main →
  upstream **ahead_by 45, behind_by 0** (no fork-unique commits; the
  "selector combinator count_signal" fix is already upstream).
- The fork has dependabot configured for the `factorio-data` submodule (daily),
  but merged submodule bumps don't help until the pkl data is regenerated,
  which requires upstream's fixed lifecycle.
- Upstream 4.0.1 data pickles already contain `landing-pad-unloading-bay`
  (and track newer wube data); its Python API surface (`signals.raw`,
  `type_of`, `add_signal`, `entities.raw`, `new_entity`) is unchanged for
  everything factompiler uses.

## Options

### Option A (recommended): depend on upstream `redruin1/factorio-draftsman`

- Verified feasible: installed upstream 4.0.1 into a scratch venv
  (`temp/dv4`), ran factompiler against it:
  - Smoke compile of a program using the 2.1-only signal
    `landing-pad-unloading-bay` succeeds end-to-end.
  - `pytest dsl_compiler/src/semantic/tests dsl_compiler/src/emission/tests -n auto`
    → **296 passed** with zero code changes.
- Steps:
  1. Change the pyproject pin to
     `factorio-draftsman @ git+https://github.com/redruin1/factorio-draftsman.git@<tag>`
     (prefer a tag, e.g. `4.0.1`, over `@main` for reproducibility).
  2. Reinstall into `.venv`, run the full test suite (`pytest -n auto`).
  3. Keep the Snagnar fork around only if future bot-specific patches appear.
- Freshness thereafter: upstream tracks `wube/factorio-data` itself; new
  Factorio items (2.1/2.2/…) arrive with upstream releases — no factompiler
  code changes needed.

### Option B: sync the fork and regenerate its data (keep the pin)

1. Merge upstream `main` into `Snagnar/factorio-draftsman` (45 commits).
2. Bump the `factorio-data` submodule to the latest wube commit (2.1.x).
3. Regenerate: `draftsman update` with an empty mods dir; the wube clone must
   not contain a `.git` folder (mod discovery chokes on it).
4. Commit regenerated pickles, bump version; factompiler just reinstalls `@main`.

More moving parts than Option A; only justified if the fork must remain the
source (e.g., pending bot-specific patches).

### Option C: runtime augmentation in factompiler (stopgap only)

Inject missing prototypes into `draftsman.data.signals.raw` /
`entities.raw` at import time (`add_signal`/`add_entity` exist). Works for a
one-off missing item, but duplicates game data in factompiler and rots
silently. Not recommended as a strategy; fine as an emergency shim.

## Suggested factompiler-side hardening (small, independent of A/B/C)

- Validate `place()` prototype names against `entities.raw` in the semantic
  stage (clear error instead of silent 1×1 fallback + late emission warning).
- Add a CI smoke test compiling a sample program that uses Space Age signals
  (e.g. `space-science-pack`, `rocket-fuel`) so data regressions are caught.

## Environment notes (hit during research)

- `/tmp` is a 500 MB tmpfs; `pip install` from git (which clones submodules
  into `/tmp`) fails with a "disk space" error until you set
  `TMPDIR=/workspace/...` for the pip invocation.
- The venv is `.venv` (not `venv`) in this checkout.

## Artifacts left in `temp/` (gitignored)

- `wube-data/` — shallow clone of wube/factorio-data @ master (2.1.21)
- `sandbox/` — copy of installed draftsman used to reproduce the 3.2.0
  `draftsman update` crash on 2.1 data
- `dv4/` — scratch venv with upstream draftsman 4.0.1 + deps (feasibility proof)
- `space_age_test.facto`, `space_age_place.facto`, `space_age_v4.facto` —
  smoke-test programs
