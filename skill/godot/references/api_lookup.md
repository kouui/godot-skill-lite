# API Lookup And Running Code

Read this before calling an engine method you have not just read, and whenever a question is cheaper to answer by running it than by reasoning about it.

| Tool | Answers |
| --- | --- |
| `scripts/docs/api_lookup.py` | Does `CharacterBody2D.move_and_slide` exist, what does it take and return, which class declares `position`? Reads the installed engine's own class dump (full signatures, defaults, enums, no prose). |
| `run_gdscript` (dispatcher op) | What does this snippet evaluate to in this project, with its autoloads loaded? |

## api_lookup.py

```bash
uv run /absolute/path/to/godot/scripts/docs/api_lookup.py CharacterBody2D.move_and_slide Timer Area2D.body_entered
```

Positional queries, any number per call (the first call builds a cache, later calls are fast).

| Query | Resolves to |
| --- | --- |
| `Timer` | class card: ancestor chain, own members, one summary line per ancestor (`--inherited` expands them) |
| `CharacterBody2D.position` | member resolved **through the chain**, naming the declaring class (`Node2D.position`) |
| `Area2D.body_entered` | a signal, plus the ready-to-paste `connect` line and handler signature |
| `String.begins_with`, `Vector2.slide`, `lerp`, `preload` | Variant-type members and `@GlobalScope` / `@GDScript` functions |
| `KEY_SPACE`, `Control.SIZE_EXPAND_FILL`, `Control.SizeFlags` | constants (with their enum) and whole enums |
| `Button --kind theme_items` | only one member kind; `--kind` takes a comma list of `methods`, `properties`, `signals`, `constants`, `enums`, `theme_items`, `operators`, `constructors`, `annotations` |

Search: `api_lookup.py --search move_and --limit 6` (ranked exact > whole word > prefix > substring).

Wrong names fail loudly (exit 1, results that did resolve still print on stdout):

- An outdated name gets the 4.7 answer: `KinematicBody2D` -> "use `CharacterBody2D`".
- A misspelling gets the nearest real names (`CharcterBody2D`, `Vector2.lenght` -> `length`).

Project classes: `--project /absolute/path/to/project Pickup Pickup.monitoring` indexes every compiling script in the project (regenerated each call) with its exported properties, signals and methods (signatures only, no `##` comments), inheriting the engine chain. A script without `class_name` answers to `res://scripts/player.gd`, `scripts/player.gd` or `player.gd` (member: `res://scripts/player.gd.jump`). `--project DIR --search .gd` lists every script and what it extends. Scripts that do not compile are skipped with a stderr note; a freshly added `class_name` may need `godot --headless --path <project> --import` once to resolve.

Other flags: `--godot BIN` (else `$GODOT_BIN`, else `godot`), `--refresh` rebuilds the cache (keyed by engine version), `--json`/`--pretty`. Exit `0` all found, `1` something not found, `2` usage error or engine could not run. It gives exact signatures, not descriptions of behaviour.

## run_gdscript

The escape hatch: "what does the engine actually return here". Use a dedicated op first (`configure_node`, `resource_batch`, `run_scenario`, `check_project`) because those record what they did. It is **not a sandbox**: autoloads run, code can write and delete files.

Exactly one mode:

- **`code`**: statements forming the body of `func run(tree: SceneTree) -> Variant`; may `await` (the tree is live) and `return`; autoloads are global identifiers.
- **`expression`**: one `Expression` string, no statements, no `var`, no `await`. Only engine singletons (`OS`, `Input`, `InputMap`, `ProjectSettings`, `Time`, `ClassDB`, `RenderingServer`, ...), project autoloads, `tree` and `scene` are in scope; a `class_name`, node path or local needs `code` (otherwise `self can't be used`).
- **`script_path` + `method`** (+ `args`, a typed-JSON array): `instantiate` defaults to auto, so a `static func` is called on the script and anything else on `script.new()`. That makes an **orphan node**: `_ready()` has not run and `@onready` vars are `null`; pass `scene_path` and use `code` when the node must be alive.
- `scene_path` (any mode) instantiates that scene under the root first and exposes it as `scene`.

```bash
godot --headless --path /absolute/path/to/project \
  --script /absolute/path/to/godot/scripts/core/dispatcher.gd \
  run_gdscript '{"code":"var body := CharacterBody2D.new()\nbody.velocity = Vector2(120, 0)\nvar report := {\"speed\": body.velocity.length()}\nbody.free()\nreturn report"}'
godot --headless --path /absolute/path/to/project \
  --script /absolute/path/to/godot/scripts/core/dispatcher.gd \
  run_gdscript '{"expression":"ProjectSettings.get_setting(\"physics/2d/default_gravity\")"}'
godot --headless --path /absolute/path/to/project \
  --script /absolute/path/to/godot/scripts/core/dispatcher.gd \
  run_gdscript '{"scene_path":"scenes/ui.tscn","code":"return (scene.get_node(\"StatusLabel\") as Label).text"}'
```

Output is one JSON line `{ok, mode, completed, result, result_type, elapsed_ms}` plus `prints`, `warnings` and `errors`. `result` is typed-JSON encoded (`{"__type":"Vector2","x":..,"y":..}`, the same shape other ops accept); `max_depth` (default 4) caps resource nesting.

**A crashed snippet is never reported as success.** A GDScript runtime error normally just yields `null`; this op captures engine errors, so you get `ok: false`, `errors[].where` as `code:<line>` (numbered against your snippet, not the wrapper) and exit 1. Parse errors are reported the same way. GDScript catches mistakes on statically typed values at compile time and on untyped values only at runtime; both come back as `code:<line>`. `timeout_seconds` (default 10) aborts with `"timed_out": true`, but a `while true:` loop that never yields cannot be interrupted and has to be killed from outside.

Look the API up first; it removes the reason most snippets fail.
