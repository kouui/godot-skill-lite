# Debugging, Validation and GDScript Parse Safety

Read before writing `.gd` files (typing rule) and whenever you run the project, read errors, and fix them. Tools in `scripts/debug/`: `run_project.py`, `godot_log_parser.py`, `validate_project.py` (`check_project` + lint + C# build), `smoke_scenes.py`, `run_scenario.py`.

## Typing rule (prevents most first-run parse errors)

Annotate every variable, parameter and return type in generated code. Use `:=` only when you can name the resulting type from that one line: `Vector2.ZERO`, `0`, `"x"`, `create_tween()`, `ConfigFile.new()`, `body as Enemy`.

Never `:=` when the right side is:
- `$Node`, `%Unique`, `get_node()`, `get_parent()`, `find_child()`, `instantiate()`: typed bare `Node`, so it silently widens (and becomes a parse error under `unsafe_*_access` warnings). Write `@onready var label: Label = %Label`.
- `$Node.prop` / `$Node.method()`, `dict[key]`, `dict.get()`, `untyped_array[i]`, a call to a function with no return type: no static type, parse error. Write `var hp: int = int(data.get("hp", 0))`.
- `JSON.parse_string()` or any Variant API: `inference_on_variant` ships as an error. Write `var raw: Variant = JSON.parse_string(s)` then `var d: Dictionary = raw if raw is Dictionary else {}`.
- `null` (`var x: Node = null`), or a `class_name` not yet imported.
- Empty collections: `var e := []` is a plain `Array`. Write `var enemies: Array[Enemy] = []`; `Dictionary[String, int]` is also valid. `Array[Enemy]()` is invalid syntax.

Downcast with `as` plus a null check (`var enemy := body as Enemy`, `if enemy == null: return`); `var x: Enemy = node` is checked at runtime and errors instead of yielding null. `@export` always needs a type or initializer: `@export var speed: float = 200.0`.

## The loop

1. Reproduce (omit the scene to boot `run/main_scene`):
   ```bash
   uv run /absolute/path/to/godot/scripts/debug/run_project.py /absolute/path/to/project \
     scenes/main.tscn --quit-after 120 --timeout 60
   ```
   JSON: `ok`, `counts`, `diagnostics[]` (`severity`, `category`, `message`, `file`, `line`, `function`, `stack`, `suggested_fix`).
2. Open `file` at `line`; fix using the table below. Prefer dispatcher ops over hand-editing `.tscn` for structural fixes (NodePath, signal wiring, exported values).
3. Re-run. Pass = `"ok": true`, `counts.errors == 0`, `counts.parse_errors == 0`. Fix order: parse errors, load errors, runtime errors, warnings (one parse error cascades into `Failed to load script` and null-instance errors).
4. Widen: errors on paths a short boot never reaches need that scene, a larger `--quit-after`, or the input that triggers them.

Whole-project pass without running gameplay (loads every script, scene, shader, resource, plugin; instantiates every scene, running root `_init()` and setters but not `_ready`):

```bash
uv run /absolute/path/to/godot/scripts/debug/validate_project.py /absolute/path/to/project --pretty
```

It runs static lint, `check_project`, node-config warnings (`references/node_config.md`) and parses the log; it refuses `ok` while any error diagnostic exists. Add `--warnings-as-errors` for strict. A `failed_count` of 0 with `ERROR:` lines in a raw log is still a failure. Raw dispatcher form: `godot --headless --debug --ignore-error-breaks --path P --script .../dispatcher.gd check_project '{}' 2>&1 | uv run .../godot_log_parser.py -` (scope with `{"project_path":"scripts/enemies"}`; `{"instantiate": false}` skips instantiation).

## Smoke-run every scene

Run after each batch of edits and before calling a build done; it is fast (fixed 60 fps, game time decoupled from wall time):

```bash
uv run /absolute/path/to/godot/scripts/debug/smoke_scenes.py /absolute/path/to/project --seconds 2 --jobs 4 --pretty
uv run /absolute/path/to/godot/scripts/debug/smoke_scenes.py /absolute/path/to/project --seconds 4 --jobs 4 --fuzz --fuzz-mouse --fuzz-seed 7
```

One process per scene; exit 0 = every scene booted, ran, no error diagnostic. `--fuzz` replays the project's `InputMap` actions (`--fuzz-mouse` also clicks visible Buttons) with a replayable seed. Scoping: `--scenes`, `--include`/`--exclude` globs. Per-scene findings a log never shows: `node_growth` (spawner never frees), `orphan_nodes`, `static_memory_growth`, `timed_out` (infinite loop), `crashed`, `scene_changed`/`quit_called` (run ended early, covered less than `--seconds`).

Isolated boot can be unfair: a pause menu reading `get_parent().player` errors in isolation only, so read the diagnostic before "fixing" (or `--exclude` it). `--profile` fps monitors are `null` on short fixed-fps runs; use `perf.frame_ms` or add `--real-time`.

Which tool: files load and node trees valid = `validate_project.py`; game boots = `run_project.py`; every scene survives = `smoke_scenes.py`; this input sequence gives that state = `run_scenario.py`; logic correct = `run_tests.py` (`references/testing.md`).

## Capturing output correctly

- Always run Godot with `-d --ignore-error-breaks` as a pair. GDScript warnings (unused variable, shadowing, integer division, narrowing...) only reach stdout through the script debugger channel, so a plain `godot --headless` run prints nothing while the editor shows many. `--ignore-error-breaks` stops the debugger breaking to a `debug>` prompt that would end the run. The bundled Python tools pass both (`--no-debugger` opts out) and give Godot `/dev/null` stdin.
- Exit code 0 is necessary, not sufficient: an engine `SCRIPT ERROR` inside a dispatcher op (e.g. `attach_script` loading a script with a parse error) is printed but the dispatcher still exits 0, and the scene still saves without the script. Only errors the op itself logs (`[ERROR] ...`) give exit 1 (`run_gdscript` and `check_project` do). The Python wrappers gate on the parsed log. When calling the dispatcher directly, pipe through `godot_log_parser.py`.
- Bound every run: `--quit-after N` (use >= 2; 1 fails first-launch import) plus `--timeout`. `"timed_out": true` is a finding (infinite loop or blocking call).
- Headless catches logic errors; rendering- or audio-only problems need `--no-headless`.
- `ERROR: N resources still in use at exit` / `ObjectDB instances were leaked at exit` are `info` (`exit_leak`, usually audio playing at quit); for real leaks use `smoke_scenes.py`. Missing GPU/sound-card driver probe errors are `info` (`host_capability`); the `switching to OpenGL 3` / dummy-audio-driver warnings stay warnings because the renderer changed under you (`RenderingServer.get_current_rendering_method()` tells the truth, not the project setting).
- A shader that fails to compile is silently replaced by the default material at draw time; `check_project` reports it in `failed[]` with line and message.
- After adding a `class_name`, run `godot --headless --path <project> --import` before validating, or it reads as undeclared (the global class cache is not rebuilt by `--script`).

## Message -> cause -> fix

`category` is matched from the message text and falls back to `unknown`. A parse error has `severity: parse_error` (counted in `counts.parse_errors`) and usually `category: unknown`; `Out of bounds get index` and `Can't emit non-existing signal` are also `unknown`. Match on `message`, not `category`.

| Message contains | `category` | Cause / fix |
| --- | --- | --- |
| `null instance`, `on a null value` | `null_reference` | Node accessed before it is in the tree, or wrong/renamed path. `@onready`, access in `_ready`, `get_node_or_null()` + guard, fix path. |
| `Invalid get index` / `Invalid set index` / `Out of bounds get index` | `invalid_index` | Missing key/index or wrong base type; check names and non-null. |
| `Invalid access to property or key` | `invalid_member` | Member absent on that object type (base may be null). |
| `nonexistent function`, `not found in base` | `missing_method` | Typo, wrong node class, or outdated API: fix the name or cast; check with `api_lookup.py`. |
| `nonexistent signal`, `non-existing signal`, `Signal ... is not declared` | `signal` | Declare `signal name(...)` or fix target; prefer the `connect_signal` op. |
| `not declared in the current scope` | `undeclared_identifier` | Typo or missing `preload`/`class_name`; for a new `class_name` run `--import` (above) or `const X = preload("res://x.gd")`. |
| `Trying to assign value of type`, `Cannot convert`, `Cannot assign a value of type ... to variable` | `type_mismatch` / `parse_error` | Fix the declared type, convert, or `as` + null guard (often a `Node` from `$`/`instantiate()` into an unrelated type). |
| `Cannot infer the type of "x" variable because the value doesn't have a set type` | `parse_error` | `:=` on untyped RHS; annotate (typing rule). |
| `Cannot infer ... value is "null"` | `parse_error` | `var x: Node = null`. |
| `The variable type is being inferred from a Variant value` | `parse_error` | `:=` on a Variant API; annotate and narrow (typing rule). |
| `The property/method "x" is not present on the inferred type` | `parse_error` | Wide inferred `Node` under `unsafe_*_access` errors; annotate the node reference. |
| `Cannot use simple "@export" annotation with variable without type or initializer` | `parse_error` | `@export var speed: float = 200.0`. |
| `hides an autoload singleton`, `hides a native class` | `parse_error` | `class_name` collides with an autoload or engine class; rename (autoload scripts need no `class_name`). Check with `ClassDB.class_exists`. |
| `Used space character for indentation instead of tab` (or reverse) | `parse_error` | Mixed tabs/spaces, typical when appending generated lines; match the file (tabs). |
| `Unexpected "?" in source` | `parse_error` | No `?:`; `var m: String = "a" if c else "b"`. |
| `is a coroutine, so it must be called with "await"` | `parse_error` | `var s: int = await load_score()`. |
| `The function signature doesn't match the parent` | `parse_error` | Match exactly: `_ready() -> void`, `_process(delta: float) -> void`, `_input(event: InputEvent) -> void`. |
| `Function "x()" not found in base` (after `super`) | `parse_error` | `super()` only inside a real override. |
| `"@onready" can only be used in classes that inherit "Node"` | `parse_error` | RefCounted/Resource has no `_ready`: init in `_init()` or extend Node. |
| `Could not find base class`, `Could not resolve super class inheritance from` | `parse_error` | Two scripts `extends` each other; break with a shared base or composition. Annotation/`preload` cycles between `class_name` scripts are fine. |
| `Not all code paths return a value` | `parse_error` | Add a final `return`; do not drop the return type. |
| `Node not found` | `node_path` | Fix NodePath / `%UniqueName` or use `get_node_or_null()`. |
| `Failed to load resource`, `Resource file not found`, `Cannot open file` | `resource_load` | Restore path; repair UID sidecars with `get_uid` / `resave_resources` after moves. |
| `Failed to load script ... Parse error` | `resource_load` | Fix that script's parse error first. |
| `Condition "..." is true` | `engine_assertion` | Call in wrong state/order (often before node is in tree). |
| `Invalid scene: root node X cannot specify a parent node`, `... does not specify its parent node` | `scene_hierarchy` | Root `[node]` has no `parent=`; every other needs one naming an existing node. Message names the node; the scene is in `check_project`'s `failed[]`. |
| `Parent path '...' for node '...' has vanished` | `scene_hierarchy` | Unresolvable `parent=` (root-relative, no `./`, parent declared first); node is reparented silently. Confirm with `inspect_scene`. |
| `The local variable "x" is shadowing an already-declared property` | warning | Rename the local (`name`, `position`, `owner`, `visible` are inherited). |
| shader compile messages | `shader` | Fix the reported line; match uniforms/varyings. Surfaces in `check_project` without running. |
