# Unit Testing Game Logic

Read when writing, running or fixing unit tests for the game's own code (damage formula, inventory, state machine, signals, a body landing on the floor). The bundled mini framework needs no addon and no download.

## Install and run

```bash
uv run /absolute/path/to/godot/scripts/test/run_tests.py /absolute/path/to/project --init-mini
uv run /absolute/path/to/godot/scripts/test/run_tests.py /absolute/path/to/project --pretty
```

`--init-mini` writes `res://tests/test_case.gd` (base class, no `class_name`) and `res://tests/test_example.gd`; it refuses to overwrite (exit 1; use `--tests-dir other` to install alongside). Delete `test_example.gd` once real suites exist.

Detection order is gut, gdunit4, mini; force with `--framework mini`. Exit 0 only when `ok` is true. An empty tests dir fails (`--allow-empty` opts out).

Flags: `--select TEXT` (substring of test name or script path), `--test-timeout N` (per test, default 10 s), `--timeout N` (whole run, default 600 s), `--strict` (`risky` fails), `--tests-dir`, `--log-file PATH`, `--dry-run`.

## Write a suite

File `res://tests/test_<thing>.gd`, `extends "res://tests/test_case.gd"`, methods named `test_*`. Write the test, run with `--select`, fix, repeat.

```gdscript
extends "res://tests/test_case.gd"
const Damage = preload("res://scripts/damage.gd")   # preload needs no class cache

func test_armor_never_heals() -> void:
	assert_eq(Damage.resolve(3, 9, false), 1, "a hit does at least 1")
```

- Assertions (optional trailing `message`, return bool): `assert_true/false`, `assert_eq/ne` (deep for Array/Dictionary, mismatched types are unequal), `assert_almost_eq(a, b, tol)` (use for all floats, also Vector/Color), `assert_null/not_null`, `assert_gt/lt/between`, `assert_has/does_not_have`, `assert_string_contains`, `assert_is`, `fail`, `skip("why")`/`pending("why")` (mark only; `return` next line), `allow_errors()`.
- Lifecycle: `before_all`, `before_each`, `after_each`, `after_all` (may be coroutines). `add_child_autofree(node)`, `add_scene(path)` (instantiate + add + autofree), `autofree(obj)`.
- Frames: `await wait_frames(n)`, `wait_physics_frames(n)`, `wait_seconds(s)`, `wait_until(callable, timeout)`. Headless physics runs normally, so "player lands on floor" is a unit test: `add_scene`, `await wait_physics_frames(45)`, `assert_true(player.is_on_floor())`.
- Input: `press_action(&"jump")` / `release_action` / `send_key(KEY_SPACE)` deliver real events immediately (polled checks and `_input` both see them). Released automatically after the test.
- Signals: call `watch_signals(obj)` BEFORE the emitting code, then `assert_signal_emitted`, `assert_signal_not_emitted`, `assert_signal_emit_count`, `assert_signal_emitted_with(obj, name, [args])`, `get_signal_args(obj, name, idx=-1)`. Unwatched objects fail loudly, not as "never emitted". Await a frame for deferred emissions.
- Autoloads are registered before tests load but state persists across tests in the one process: reset in `before_each`.
- A project `class_name` needs one `godot --headless --path <project> --import` first (the cache is not rebuilt by `--script`); otherwise the suite fails to load ("Identifier not declared"; `hints[]` says so).
- Headless viewport is 64x64: physics and logic fine, UI layout is not. Check layout with `run_scenario.py` `ui_report`.

## Statuses

| Status | Meaning | Fails run |
| --- | --- | --- |
| `passed` | Ran, asserted at least once, none failed | no |
| `failed` | Assertion failed (`line` = assert line) | yes |
| `error` | Engine printed an error during the test (runtime error, `push_error`, failed load) | yes |
| `timeout` | Exceeded `--test-timeout` | yes |
| `skipped` / `pending` | `skip()` / `pending()` called | no |
| `risky` | No assertion at all | only with `--strict` |

`error` is the key one: a GDScript runtime error aborts the function silently, so a test that dereferences null looks like a pass. The runner attributes `SCRIPT ERROR:` lines to the running test and reports `error` with the engine message. If the code under test should print an error (error path, deliberate `push_error`), call `allow_errors()` in that test. Mark unwritten tests `pending`, not empty (`risky`).

## Result JSON

Check `ok`, `counts` (`passed failed errors skipped pending risky timed_out non_test_failures`), and `failures[]` (`script`, `test`, `line`, `message`, `status`). `non_test_failures` = a `before_all`/`after_all` hook or a script that failed to load. `notes[]` reports leaks and `test_*.gd` files not extending the base class.

## Limits

- No mocks or parameterised tests: inject collaborators via exports/constructor args.
- A timed-out coroutine is abandoned, not killed; keep `await`s bounded.
- Warnings are not failures here; use `validate_project.py` (see `references/debugging.md`).
- Scenes, screenshots, whole-game flows: use `run_scenario.py` (`references/automation_api.md`) instead.
