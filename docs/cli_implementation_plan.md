# Implementation Plan — CLI Adapter (View)

## 0) Quick orientation (read this first)

* **Goal of this change:** Add a thin, testable CLI that parses user args, validates them syntactically, calls the Presenter, and renders results (table/json).
* **Out of scope:** Any business logic (parsing GPX, geometry, scoring, providers). That lives in services/presenter and is **not** implemented here.
* **Dependency direction (don’t violate this):**
  `CLI Adapter → Presenter → Services`
  Presenter must **not** import CLI code. CLI should use **dependency injection** to receive a presenter instance.
* **Where to look for context:** the design doc (already attached), plus `README.md` usage section (you’ll add/adjust it as part of this PR—not the whole README, just the CLI bits).

---

## 1) Files to create or touch

```
pizza_along_route/
├── app/
│   ├── main.py                          # Entry point (thin wrapper)
│   ├── presenter.py                     # Presenter interface/stub only (no business logic)
│   └── views/
│       ├── cli_adapter.py               # <-- Implemented in this task
│       └── renderers/
│           ├── table_renderer.py        # Minimal table renderer (tabulate optional)
│           └── json_renderer.py         # JSON renderer
├── tests/
│   ├── unit/
│   │   └── views/
│   │       ├── test_cli_adapter_args.py
│   │       ├── test_cli_adapter_presenter_integration.py
│   │       └── test_renderers.py
│   └── integration/
│       └── test_cli_end_to_end.py
└── pyproject.toml (or setup.cfg)        # Add console_scripts entry point
```

> If the `views/` or `renderers/` folders don’t exist yet, create them and **remember `__init__.py`** so tests can import.

---

## 2) Public CLI behavior & arguments

**Command name:** `pizza-along-route` (adjust if you already picked a name)
**Default behavior:** human-readable table
**Usage:**

```
pizza-along-route \
  --gpx /path/route.gpx \
  --range-start 7 \
  --range-end 14 \
  --offroute-distance 2 \
  [--use-distance] \
  [--use-price] \
  [--output table|json]
```

**Arguments (syntax-level validation in CLI):**

* `--gpx` (str, **required**): path to GPX file. Validate it exists and is a file (syntactic).
* `--range-start` (float, **required**, ≥ 0)
* `--range-end` (float, **required**, > range-start)
* `--offroute-distance` (float, default: `2.0`, ≥ 0)
* `--use-distance` (flag, default: `False`)
* `--use-price` (flag, default: `False`)
* `--output` (str, default: `table`, choices: `table`, `json`)

**Note:** Do **not** attempt to validate semantic constraints that require domain data (e.g., “range within route length”). That belongs to Presenter/Services.

---

## 3) Data contract between CLI and Presenter

* **CLI → Presenter**: a simple dataclass or dict, e.g.:

  ```python
  @dataclass(frozen=True)
  class RouteQuery:
      gpx_path: str
      range_start_km: float
      range_end_km: float
      offroute_distance_km: float
      use_distance: bool
      use_price: bool
  ```
* **Presenter → CLI**: a **presentation-ready** structure the renderers can handle without domain knowledge. Keep it dead simple:

  ```python
  @dataclass(frozen=True)
  class PlaceViewModel:
      name: str
      rating: Optional[float]
      price_level: Optional[int]  # e.g., 1–4
      distance_km: Optional[float]  # crow-flies
      nearest_km_marker: Optional[float]

  @dataclass(frozen=True)
  class ResultsViewModel:
      items: list[PlaceViewModel]
      # metadata fields are okay but optional
  ```
* **Renderer inputs:** `ResultsViewModel` + output format selection.

> ✅ YAGNI: Resist adding score, capacity, or internal fields—the design doc says **score not displayed**.

---

## 4) Implementation tasks (bite-sized, TDD-friendly)

### Task 1 — Scaffold presenter interface (no logic)

**Files:** `app/presenter.py`

1. Define a lightweight interface/stub (do **not** implement real logic):

   ```python
   from dataclasses import dataclass
   from typing import Optional

   @dataclass(frozen=True)
   class RouteQuery:
       gpx_path: str
       range_start_km: float
       range_end_km: float
       offroute_distance_km: float = 2.0
       use_distance: bool = False
       use_price: bool = False

   @dataclass(frozen=True)
   class PlaceViewModel:
       name: str
       rating: Optional[float]
       price_level: Optional[int]
       distance_km: Optional[float]
       nearest_km_marker: Optional[float]

   @dataclass(frozen=True)
   class ResultsViewModel:
       items: list[PlaceViewModel]

   class RoutePlannerPresenter:
       def run(self, query: RouteQuery) -> ResultsViewModel:  # pragma: no cover (stub)
           raise NotImplementedError
   ```

**Commit:** `feat(cli): add presenter contracts (no logic)`

---

### Task 2 — Implement renderers (json + table)

**Files:** `app/views/renderers/json_renderer.py`, `app/views/renderers/table_renderer.py`
**Tests first** (`tests/unit/views/test_renderers.py`)

* **json_renderer**:

  * `render(results: ResultsViewModel) -> str` using `json.dumps` with `default=asdict` pattern.
* **table_renderer**:

  * Prefer `tabulate` if installed; else manual fixed-width columns (keep simple).
  * Columns: **Name | Rating | Price | Distance (km) | KM marker**
  * `None` values display as `-`.

**Examples (implementation hints):**

```python
# json_renderer.py
import json
from dataclasses import asdict

def render(results) -> str:
    return json.dumps(asdict(results), indent=2, sort_keys=False)

# table_renderer.py
def render(results) -> str:
    headers = ["Name", "Rating", "Price", "Distance (km)", "KM marker"]
    rows = []
    for p in results.items:
        rows.append([
            p.name,
            f"{p.rating:.1f}" if p.rating is not None else "-",
            str(p.price_level) if p.price_level is not None else "-",
            f"{p.distance_km:.2f}" if p.distance_km is not None else "-",
            f"{p.nearest_km_marker:.1f}" if p.nearest_km_marker is not None else "-",
        ])
    try:
        from tabulate import tabulate
        return tabulate(rows, headers=headers, tablefmt="github")
    except Exception:
        # simple fallback
        return _fallback_table(headers, rows)

def _fallback_table(headers, rows) -> str:
    # minimal padding; keep it tiny—YAGNI
    widths = [max(len(str(x)) for x in col) for col in zip(headers, *rows)] if rows else [len(h) for h in headers]
    def fmt(row): return " | ".join(str(cell).ljust(w) for cell, w in zip(row, widths))
    out = [fmt(headers), "-+-".join("-"*w for w in widths)]
    out += [fmt(r) for r in rows]
    return "\n".join(out)
```

**Commit:** `feat(cli): add json/table renderers with tests`

---

### Task 3 — Implement CLI parser & validation

**Files:** `app/views/cli_adapter.py`
**Tests first:** `tests/unit/views/test_cli_adapter_args.py`

* Provide a **`build_parser()`** factory; unit-test it in isolation.
* Provide **`parse_argv(argv: list[str]) -> Namespace`** so tests can pass synthetic argv.
* Provide **`to_query(args) -> RouteQuery`** which:

  * Validates syntax (file exists, numeric ≥0, start<end).
  * Converts to `RouteQuery`.
* Provide **`dispatch(presenter, query, output)`** which:

  * Calls presenter.run(query) → ResultsViewModel
  * Selects renderer based on `output` and returns final string.
* Provide **`main(argv: list[str] | None = None, presenter=None) -> int`**:

  * Build parser → parse → to_query → dispatch.
  * Print output to stdout.
  * Return `0` on success, `2` on arg errors (argparse convention), `1` on runtime errors.

**Implementation sketch:**

```python
# app/views/cli_adapter.py
import argparse, os, sys
from typing import Optional
from ..presenter import RouteQuery, RoutePlannerPresenter

def build_parser() -> argparse.ArgumentParser:
    p = argparse.ArgumentParser(prog="pizza-along-route", description="Find pizza along your route.")
    p.add_argument("--gpx", required=True, help="Path to GPX file.")
    p.add_argument("--range-start", type=float, required=True, help="Start kilometer (>= 0).")
    p.add_argument("--range-end", type=float, required=True, help="End kilometer (> start).")
    p.add_argument("--offroute-distance", type=float, default=2.0, help="Max crow-flies distance off route in km (>= 0).")
    p.add_argument("--use-distance", action="store_true", help="Include distance in scoring.")
    p.add_argument("--use-price", action="store_true", help="Include price in scoring.")
    p.add_argument("--output", choices=["table","json"], default="table", help="Output format.")
    return p

def parse_argv(argv: Optional[list[str]] = None) -> argparse.Namespace:
    return build_parser().parse_args(argv)

def _validate_file(path: str) -> None:
    if not os.path.isfile(path):
        raise ValueError(f"GPX file not found: {path}")

def _validate_ranges(start: float, end: float) -> None:
    if start < 0:
        raise ValueError("--range-start must be >= 0")
    if end <= start:
        raise ValueError("--range-end must be > --range-start")

def _validate_offroute(d: float) -> None:
    if d < 0:
        raise ValueError("--offroute-distance must be >= 0")

def to_query(args: argparse.Namespace) -> RouteQuery:
    _validate_file(args.gpx)
    _validate_ranges(args["range_start"] if isinstance(args, dict) else args.range_start,
                     args["range_end"] if isinstance(args, dict) else args.range_end)
    _validate_offroute(args.offroute_distance)
    return RouteQuery(
        gpx_path=args.gpx,
        range_start_km=args.range_start,
        range_end_km=args.range_end,
        offroute_distance_km=args.offroute_distance,
        use_distance=bool(args.use_distance),
        use_price=bool(args.use_price),
    )

def dispatch(presenter: RoutePlannerPresenter, query: RouteQuery, output: str) -> str:
    from .renderers import json_renderer, table_renderer
    results = presenter.run(query)
    return json_renderer.render(results) if output == "json" else table_renderer.render(results)

def main(argv: Optional[list[str]] = None, presenter: Optional[RoutePlannerPresenter] = None) -> int:
    try:
        ns = parse_argv(argv)
        if presenter is None:
            # presenter must be provided by composition root (app/main.py), not created here
            raise RuntimeError("Presenter not provided.")
        query = to_query(ns)
        rendered = dispatch(presenter, query, ns.output)
        print(rendered)
        return 0
    except SystemExit:
        # argparse already printed errors/help; let exit code pass through
        raise
    except ValueError as ve:
        print(f"error: {ve}", file=sys.stderr)
        return 2
    except Exception as e:
        print(f"unexpected error: {e}", file=sys.stderr)
        return 1
```

**Unit tests to write (examples):**

* `--help` shows expected flags and exits `0`.
* Missing `--gpx` → exit `2` (argparse).
* Invalid path → exit `2` with `error:` message.
* `--range-start -1` or `--range-end <= start` → exit `2`.
* Default `--output` is `table`; explicit `--output json` works.
* `dispatch` calls `presenter.run` exactly once with expected `RouteQuery` (use `unittest.mock.Mock`).

**Commit:** `feat(cli): argparse-based CLI adapter with validation and DI`

---

### Task 4 — Wire the composition root (entry point)

**Files:** `app/main.py`, `pyproject.toml`

* **`app/main.py`** (do not implement business logic here):

  ```python
  from .views.cli_adapter import main as cli_main
  from .presenter import RoutePlannerPresenter  # will be concrete later

  class _FakePresenter(RoutePlannerPresenter):  # temporary until real presenter lands
      def run(self, query):
          from .presenter import ResultsViewModel, PlaceViewModel
          return ResultsViewModel(items=[
              PlaceViewModel("Demo Pizza", 4.5, 2, 0.3, 8.0)
          ])

  def run_cli():
    presenter = _FakePresenter()  # replace with real injection in future
    raise SystemExit(cli_main(presenter=presenter))
  ```

* **`pyproject.toml`**:

  ```toml
  [project.scripts]
  pizza-along-route = "pizza_along_route.app.main:run_cli"
  ```

> The fake presenter is only to keep the CLI runnable during development. Your tests should **mock** the presenter; the fake is not used in unit tests.

**Commit:** `chore(cli): add entrypoint and temporary fake presenter`

---

### Task 5 — Presenter-call tests (unit, not full pipeline)

**Files:** `tests/unit/views/test_cli_adapter_presenter_integration.py`

* Test that:

  * With **valid argv**, CLI:

    * builds `RouteQuery` correctly (assert values),
    * calls `presenter.run(query)` once,
    * prints table by default,
    * returns exit code `0`.

* Use `capsys` to capture stdout and `Mock` for presenter:

  ```python
  def test_cli_happy_path_calls_presenter_and_prints_table(capsys):
      from pizza_along_route.app.views.cli_adapter import main as cli_main
      from pizza_along_route.app.presenter import ResultsViewModel, PlaceViewModel
      presenter = Mock()
      presenter.run.return_value = ResultsViewModel(items=[
          PlaceViewModel("A", 4.7, 2, 0.25, 9.0)
      ])
      argv = ["--gpx", "tests/fixtures/mini.gpx", "--range-start","1","--range-end","3"]
      # Make sure fixture file exists; create a tiny empty file in test setup
      code = cli_main(argv=argv, presenter=presenter)
      assert code == 0
      out = capsys.readouterr().out
      assert "A" in out
      presenter.run.assert_called_once()
  ```

**Commit:** `test(cli): verify CLI→Presenter call and default table output`

---

### Task 6 — End-to-end CLI (thin) integration test

**Files:** `tests/integration/test_cli_end_to_end.py`

* Call the installed console script entry (or `python -m pizza_along_route.app.main`) with `subprocess.run` or invoke `run_cli()` directly.
* Inject a **temporary presenter** via environment variable or monkeypatch `app.main._FakePresenter` to return a deterministic small result.
* Assert:

  * Exit code `0`
  * Output contains expected columns and values for both `table` and `json`.

**Commit:** `test(integ): e2e CLI prints expected table and json`

---

### Task 7 — Polish: Errors & UX

* Ensure error messages are **helpful** and **actionable** (e.g., “GPX file not found: …”).
* Help text includes one short example usage block.
* **No secrets** handled here (don’t accept API keys on CLI; services resolve from env/config).

**Commit:** `feat(cli): improve help and error messages`

---

## 5) TDD workflow & commit cadence

* **Red → Green → Refactor** on every micro-feature (parser arg, validation rule, renderer path).

* Keep commits small (5–50 lines). Example cadence:

  1. Add presenter contracts (no logic).
  2. Add json renderer + tests.
  3. Add table renderer + tests.
  4. Add parser + flags (happy path) + tests.
  5. Add validation errors + tests.
  6. Wire CLI main with DI + tests.
  7. Add end-to-end test.
  8. Polish messages.

* Run `pytest -q` locally and in CI.

* Keep **coverage high** for CLI path (>90% of cli_adapter).

---

## 6) Testing details (what to test vs. not)

### Unit tests (CLI-focused)

* **Argument parsing:** required flags, defaults, choices, `--help`.
* **Validation:** file exists (use temporary file or `tmp_path`), numeric constraints (`>=0`, `end>start`).
* **Dispatch:** correct **output selection** (`table` default, `json` when requested).
* **Presenter call:** once, with the right `RouteQuery` values.

### Unit tests (Renderers)

* **json_renderer:** is valid JSON, contains keys/values; handles `None` as `null`.
* **table_renderer:** has headers and rows; handles `None` as `-`; does not crash when list empty.

### Integration (thin)

* Running the CLI with a **fake presenter** produces expected **stdout** and **exit codes**.

### Do NOT test here

* Business logic (geometry, scoring, providers).
* Semantic validation requiring GPX parsing. That’s Presenter/Services domain.

---

## 7) Docs the engineer should touch

* **README.md (CLI section only)**

  * Installation (Poetry or pip)
  * Example invocations (table vs json)
  * Exit codes (0 success, 2 arg error, 1 unexpected)
  * Note about environment variables (keys are handled by services; none needed for CLI)
  * Attribution/legal notes may appear elsewhere; not needed here.

* **Docstrings** in `cli_adapter.py` for `build_parser`, `to_query`, and `main`.

* Keep comments minimal; prefer clear function names and tests.

**Commit:** `docs(cli): add usage examples and exit code notes`

---

## 8) Style, lint, and CI

* Follow your repo’s defaults (e.g., `ruff`, `black`, `pytest`).
* Add `tests/fixtures/mini.gpx` (empty-but-valid file for path tests; or a 1-line dummy since we don’t parse).
* Verify `pyproject.toml` console script works: `pizza-along-route --help`.

---

## 9) Review checklist (for PR author & reviewer)

* [ ] CLI adds **no business logic**.
* [ ] Dependency direction is correct (no presenter import of CLI).
* [ ] `main()` takes a presenter (DI); composition root wires it.
* [ ] Unit tests cover parser, validation, rendering, presenter call.
* [ ] Integration test covers end-to-end prints.
* [ ] Helpful errors; defaults match spec (table).
* [ ] No secrets handled by CLI.
* [ ] Small, frequent commits with passing tests.

---

## 10) Example “good” tests (copy/paste starters)

```python
# tests/unit/views/test_cli_adapter_args.py
import os
from pathlib import Path
import builtins
import pytest
from pizza_along_route.app.views.cli_adapter import main as cli_main

def _touch(tmp_path: Path, name="mini.gpx") -> str:
    p = tmp_path / name
    p.write_text("<gpx></gpx>")
    return str(p)

def test_help_exits_zero(capsys):
    with pytest.raises(SystemExit) as e:
        from pizza_along_route.app.views.cli_adapter import build_parser
        build_parser().parse_args(["--help"])
    assert e.value.code == 0

def test_missing_required_args_returns_2():
    code = cli_main(argv=[], presenter=object())  # presenter won’t be used
    assert code == 2

def test_invalid_file_returns_2(tmp_path, capsys):
    code = cli_main(argv=["--gpx","/nope.gpx","--range-start","0","--range-end","1"], presenter=object())
    assert code == 2
    err = capsys.readouterr().err
    assert "GPX file not found" in err

def test_valid_args_call_presenter(tmp_path, capsys, mocker):
    gpx = _touch(tmp_path)
    presenter = mocker.Mock()
    from pizza_along_route.app.presenter import ResultsViewModel, PlaceViewModel
    presenter.run.return_value = ResultsViewModel(items=[PlaceViewModel("X", 4.0, 2, 0.5, 7.0)])
    code = cli_main(argv=["--gpx", gpx, "--range-start","0","--range-end","3"], presenter=presenter)
    assert code == 0
    out = capsys.readouterr().out
    assert "X" in out
    presenter.run.assert_called_once()
```

---

## 11) What success looks like (manual checks)

* `poetry run pizza-along-route --help` prints helpful usage.
* `poetry run pizza-along-route --gpx tests/fixtures/mini.gpx --range-start 2 --range-end 5`
  prints a **table** with demo data (from the temporary fake presenter).
* Same command with `--output json` prints valid JSON.

