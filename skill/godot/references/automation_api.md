# Automation API

Read this when you call the dispatcher, write a batch (`scene_batch` / `resource_batch` / `project_batch`), encode typed JSON values, write a `run_scenario.py` scenario, or need the flags and result fields of `run_project.py`, `smoke_scenes.py`, `validate_project.py`, `lint_project.py`, `import_project.py`, `move_resource.py` or `scaffold_project.py`.

Owned elsewhere: unit tests `references/testing.md`; `api_lookup.py` and `run_gdscript` `references/api_lookup.md`; `spatial_report` `references/spatial_verification.md`; node config warnings and the physics layer survey `references/node_config.md`; export presets, `export_project.py`, `serve_web.py` `references/export_targets.md`; UI layout `references/game_ui.md`; audio `references/audio.md`; `draw_image` / `process_image` `references/pixel_art.md`.

## Dispatcher

```bash
godot --headless --path /absolute/path/to/project \
  --script /absolute/path/to/godot/scripts/core/dispatcher.gd \
  <op> '<one JSON object>'
```

- Paths are project-relative or `res://`; output normalizes to `res://`.
- Exit code is `1` when the op logged an error, `0` otherwise. Gate on it.
- Parameters are checked before anything runs, and a rejected call writes nothing: an unknown key (`did you mean parent_node_path?`), or a key that belongs to a different op or batch action. `--skip-param-check` after the JSON disables the check.
- Headless-saved `.tres`/`.tscn` have no `uid=` header. If the project cross-references by UID, run `resave_resources` after bulk creation and check its created/missing counts.

### Discover ops with `help`

Do not grep for op names or parameters; ask the dispatcher.

```bash
... dispatcher.gd help '{}'                                  # every op, one line each, plus the skill's python tools
... dispatcher.gd help '{"op":"add_node","format":"text"}'   # params, defaults, example, notes, ready-to-edit command
```

`params` marks each key `(required)` or `(default: X)`. Batch ops add `action_types` (one schema per `actions[*].type`). An unknown op name exits 1 and names the nearest real ones. `"verbose": true` adds `accepted_keys` (the full set the key check allows); use it only when a key you believe valid is rejected. The examples are written against one small sample project (`scenes/main.tscn` with `root/Player`, ...): copy the shapes, adapt the paths.

## Typed JSON values

Plain JSON is accepted everywhere a property is written; tag anything JSON cannot express. This is the single copy of the encoding.

- `{"__type":"Vector2","x":160,"y":96}`: also `Vector2i, Rect2, Rect2i, Vector3, Vector3i, Vector4, Vector4i, Transform2D, Transform3D, Basis, Quaternion, Plane, AABB, Projection, Color, StringName, NodePath`. `Rect2` takes `position`+`size` or `x,y,width,height`.
- Packed arrays: `{"__type":"PackedVector2Array","values":[...]}` (same for Byte/Int32/Int64/Float32/Float64/String/Vector3/Color).
- Colors: where the target property is declared `Color` (including a `ShaderMaterial`'s `shader_parameter/<uniform>`, once the `shader` is assigned), `"#rrggbb"`, `"#rrggbbaa"` or a color name is converted; a string that is not a color fails.
- Existing resource: `{"__resource":"res://theme/main.tres"}`.
- Engine resource built inline: `{"__resource_type":"Gradient","properties":{...},"method_calls":[{"method":"add_point","args":[...]}]}`. The resource's own properties MUST sit under `"properties"`; `{"__resource_type":"RectangleShape2D","size":...}` is an error. `method_calls` is for builder-only state.
- Project resource class: `{"__script":"res://items/item_data.gd","properties":{...}}` (script must extend `Resource`; do not combine with `__resource_type`). Run `godot --headless --path PROJECT --import` once after adding a new `class_name` script, or scripts annotating `@export var x: ItemData` fail to compile.
- Sugar: `{"__curve":{"points":[{"x":0,"y":0},{"x":1,"y":1,"left_tangent":0,"right_tangent":0}]}}` builds a `Curve`; `{"__gradient":{"points":[{"offset":0,"color":"#fff"},...]}}` (or `offsets`+`colors`) builds a `Gradient` with exactly those stops.

Property writes are type-checked: an untyped JSON array becomes the property's declared `Array[T]`/`Dictionary[K,V]`; a mismatch fails naming the expected type; a misspelled property lists the real names.

Indexed paths (`properties`-style keys such as `"shadow_offset:x"`, `"material:shader_parameter/tint"`): sub-paths use `:`; `/` is only ever part of a property name (`theme_override_colors/font_color`, `content_margin_left` is a plain property, not `content_margin/left`). A bad hop is an error with a suggestion. A path that passes through another saved resource file edits and saves that file too (`[INFO] Also saved res://materials/flash.tres`); a path into an imported file (`texture:resource_name` on a `.png`) is refused.

## Batch ops

All three apply their actions in memory and save only after every action succeeds, so a failed batch writes nothing. Prefer one batch over many single calls.

- `scene_batch`: `scene_path` (+ optional `save_path`) and `actions[]`, each action `{"type": <scene op>, ...that op's keys minus scene_path/save_path}`. `help '{"op":"scene_batch"}'` prints every action type. Node paths start at `root`.
- `resource_batch`: loads `resource_path`, or creates it with `create_if_missing` plus `resource_type` (engine class) or `script` (project `Resource` class; `resource_type` is then ignored), or deep-copies `duplicate_from` (keeps the source's script). Actions: `set_properties`, `set_indexed_properties`, `set_metadata`, `remove_metadata`, `set_resource_name`, `call_method`, `bake_navmesh`.
  - Prefer property writes (inspectable with `inspect_resource`); use `call_method` (`method`, typed `args`, optional `expect_ok`) only for builder APIs with no property form: `Curve2D.add_point`, `Gradient.add_point`, `Animation.add_track`, `TileSet.add_source`, `SpriteFrames.add_animation`. Dedicated ops exist for tilesets, themes, sprite frames, animations; use them instead.
  - `bake_navmesh`: synchronous bake from procedural geometry (2D `traversable_outlines`/`obstruction_outlines`; 3D `faces`/`source_meshes`). Set agent/cell params with a preceding `set_properties`. Feed collision or procedural geometry, not visual meshes: the headless dummy renderer cannot read mesh data back.
  - A `script`-backed `.tres` saves as `script_class=...` with typed exports round-tripped.
- `project_batch`: actions `set_setting`, `clear_setting`, `add_input_action` (`action_name`, `deadzone`, `replace`), `remove_input_action`, `add_input_event` (`action_name`, typed `event`), `remove_input_event` (`event_index`), `add_autoload` (`autoload_name`, `path`, `singleton`), `remove_autoload`, `set_layer_name` (`layer_type` in `2d_physics|3d_physics|2d_render|3d_render|2d_navigation|3d_navigation`, `layer` 1-32, `layer_name`; `""` clears), `set_main_scene`, `add_translation`/`remove_translation`, `set_shader_global` (`name`, `global_type`, `value`), `clear_shader_global`.
  - **`ProjectSettings.save()` rewrites `project.godot` wholesale: comments are dropped, keys re-sorted.** Pass `"backup_path": "project.godot.bak"` to snapshot first. Values equal to the engine default are omitted from the file by design.
  - Input events: Godot's built-in `ui_*` actions bind by `keycode`; actions you create use `physical_keycode`. The wrong field produces an event that matches nothing.

  ```json
  {"type":"add_input_event","action_name":"jump","event":{"__resource_type":"InputEventKey","properties":{"physical_keycode":32}}}
  ```

## Inspecting

- `inspect_project` (`include_files`), `inspect_scene` (`scene_path`, `include_properties`, `max_resource_depth`), `inspect_resource` (`resource_path`, `include_schema`): read `SceneState`/stored values without running code. `inspect_scene` with `config_warnings: true` is the one option that instantiates the scene (root `_init()` and stored-property setters run). For a script-backed `.tres`, `resource_type` is the engine base; read `script_path`, `script_class`, `script_properties`.
- `inspect_tilemap` (`scene_path`, optional `node_path`, `legend`, `bounds`, `format:"text"`): reads a `TileMapLayer` or `GridMap` back as ASCII rows (`.` = empty). The returned `legend` + `rows` paste straight into `paint_tilemap`/`paint_gridmap`. It is the only check that catches an off-by-one `origin`, a legend character mapped to the wrong tile, or a terrain fill that matched nothing; the scene saves and reports `ok` in all three cases. Run it after every paint.
- `inspect_image` reads any image file (res://, user://, absolute; never through the import pipeline, so fresh art works before `--import`) as numbers and ASCII, with no GPU:

  ```bash
  ... dispatcher.gd inspect_image '{"image_path":"art/player.png","format":"text","ascii":true,"expect":{"not_blank":true,"has_alpha":true,"max_unique_colors":32}}'
  ```

  Fields: `blank` (one flat colour or nothing opaque; the "nothing rendered" signal), `opaque_ratio`, `has_alpha`, `content_bbox` / `content_bbox_normalized` (centred means `x+w/2` and `y+h/2` near 0.5), `quadrants` (shares of content; one at 1.0 means content stuck in a corner), `unique_colors` (tens = pixel art, thousands = resampled), `dominant_colors`, `ascii`. Background is the colour the four corners agree on; lossy files get `background_tolerance` 0.12 automatically (pass `0` for exact). `compare_to` adds `diff_ratio` (sizes must match, else exit 1). `expect` keys: `not_blank, min_opaque_ratio, max_unique_colors, has_alpha, width, height, max_diff_ratio, frames_consistent`; any violated key makes the op exit 1. `image_paths` takes files and directories (natural order); use `expect.frames_consistent` before `build_sprite_frames`.
- `inspect_audio` reads `.wav/.ogg/.mp3` directly, no audio device: `silent` (nothing synthesized), `duration_s`, `peak_db`, `clipping_ratio`, `leading_silence_ms`, `loop_seam_delta`, pitch contour (`pitch_direction`), `import_loop` (what the game will actually loop). `expect` keys: `not_silent, no_clipping, min_duration, max_duration, max_peak_db, min_rms_db, sample_rate, channels, max_leading_silence_ms, max_trailing_silence_ms, loopable, pitch_direction`. `make_sfx.py`/`make_music.py` print a ready `expect` block; see `references/audio.md`.

## Content-authoring gotchas

Parameters come from `help '{"op":"<name>"}'`; only the traps are here.

- `build_tileset`: without collision polygons a TileSet is decorative and characters fall through. Set `physics_layers` and per-tile (or `sources[*].tile_defaults`) `collision: "full_cell"` or a point list. `tiles: "all"` exposes every grid cell. Run the importer first when the texture is fresh.
- `paint_tilemap` (standalone or as a `scene_batch` action): use `TileMapLayer` nodes, never the deprecated `TileMap`. Author levels with `ascii_map` (`legend` char -> `{source_id, atlas_coords}` or `{terrain_set, terrain}`; `.`/space/`null` leave cells untouched, `erase_unlisted:true` erases them). Order per call: `tile_set`, `clear`, `erase`, `cells`, `fills`, `ascii_map`, `terrain_fills`, so `ascii_map` wins overlaps. An unknown character is an error and nothing saves; ragged rows are only `[WARN]`. Painting fails if the atlas source has not exposed those `atlas_coords` (run `build_tileset` first). A terrain fill with no matching tile paints nothing and emits `[WARN] ... left N of M cells empty`.
- `paint_gridmap`: needs a `mesh_library` (build with `export_mesh_library`). `ascii_layers[]` slabs: rows map to z, characters to x, `y` names the slab; same legend rules as `paint_tilemap`; `orient` is 0-23.
- `build_sprite_frames`: either `animation_name` + `frames_dir`/`frame_paths`, or `spritesheet` + `grid` + `animations[]` (frame specs `{row,cols}`, `{index}`, `{region}`, `{path}`).
- `build_animation`: track paths are `"NodePath:property"` relative to the AnimationPlayer's `root_node`; the default library is `""`, so `play("blink")` works by bare name.
- `build_animation_tree`: a `Start -> first state` transition is added automatically (without it the machine never enters a state). Drive with `tree.get("parameters/playback").travel("run")`. Blend trees and blend spaces go through `resource_batch` `call_method`.
- `build_theme`: item names are not validated; a typo silently falls back to the default theme. Wire it with `project_batch` `set_setting` on `gui/theme/custom` or `configure_node` `theme`. See `references/game_ui.md`.
- `set_import_options`: patches the `.import` sidecar and invalidates the artifact; follow with `import_project.py`. WAV loop uses `edit/loop_mode` with importer enum `0` Detect, `1` Disabled, `2` Forward, `3` Ping-Pong, `4` Backward (offset from the resource enum); Ogg/MP3 use `loop` + `loop_offset`.
- `setup_audio_buses`: list a bus before any bus that `send`s to it; Master has no send; `effects` replaces the bus's whole chain; route players with `configure_node` `{"bus":"Music"}`.
- `bake_collision` (3D mesh to `StaticBody3D`: `trimesh` static only, `convex`, `multi_convex`), `collision_from_sprite` (alpha silhouette to `CollisionPolygon2D`; lower `epsilon` = more vertices), `bake_csg` (freeze a CSG tree to a mesh; point `node_path` at the CSG root).
- `draw_image` / `process_image` write PNGs with no `.import` yet (`needs_import: true`): run `import_project.py` before a scene references them. See `references/pixel_art.md`.

## run_scenario.py

`--summary` prints `ok`, `errors`, failed assertions, each report's `counts` and first `findings`, screenshot paths and log problems.

```bash
uv run /absolute/path/to/godot/scripts/debug/run_scenario.py /absolute/path/to/project /absolute/scenario.json --pretty
```

Uses a rendered window when a `screenshot` step exists, headless otherwise (`--headless` / `--no-headless` to force). Other flags: `--log-file PATH`, `--timeout S`, `--no-debugger`. Exit 0 only when every assertion, log assertion, performance assertion, gated report finding and screenshot `expect` passed.

```json
{
  "scene_path": "scenes/menu.tscn",
  "viewport_size": {"width": 1280, "height": 720},
  "settle_frames": 2,
  "steps": [
    {"type": "action", "action_name": "ui_accept", "pressed": true, "release_after": true},
    {"type": "mouse_button", "button_index": 1, "position": {"x": 640, "y": 360}, "pressed": true},
    {"type": "wait_until", "node_path": "Status", "property": "text", "expected": "Ready", "timeout_seconds": 5},
    {"type": "assert", "assertion": "property", "node_path": "Status", "property": "text", "expected": "Ready"},
    {"type": "screenshot", "path": "/absolute/output/menu.png"}
  ],
  "assertions": [
    {"assertion": "node_exists", "node_path": "StartButton"},
    {"assertion": "visible", "node_path": "Status", "expected": true}
  ],
  "log_assertions": [{"contains": "Level loaded", "min_count": 1}],
  "performance_frames": 30,
  "performance_assertions": [{"monitor": "process_time", "statistic": "maximum", "operator": "less_or_equal", "value": 0.02}]
}
```

Top-level keys: `scene_path`, `viewport_size`, `settle_frames`, `steps`, `assertions`, `log_assertions` (`contains` or `regex`, `min_count`), `log_errors`, `performance_frames`, `performance_assertions`, plus free-form `name`/`description`/`comment`/`notes` and anything starting with `_`. Any other key fails the run. Steps run in order, then `assertions`, then the performance sample. A failed step (e.g. `wait_until` timeout) stops the remaining steps, but `assertions` still run. `node_path` is relative to the scene root (`"."` = root; `/root/...` absolute). Expected values are typed JSON.

Step types (an unknown type is rejected with this list):

| type | fields |
| --- | --- |
| `wait_frames` | `frames` |
| `wait_seconds` | `seconds` |
| `wait_until` | `node_path`, `property`, `expected`, `operator`, `tolerance`, `timeout_seconds` (default 5). Polls every frame; prefer it to guessed waits |
| `action` | `action_name`, `pressed` (default true), `strength`, `frames`, `release_after` |
| `key` | `keycode` / `physical_keycode` / `unicode`, `pressed`, `echo`, `frames` |
| `mouse_button` | `button_index`, `position`, `pressed`, `double_click`, `frames` |
| `mouse_motion` | `position`, `relative`, `velocity`, `frames` |
| `joypad_button` / `joypad_motion` | `device`, `button_index`+`pressed`+`pressure` / `axis`+`axis_value` |
| `assert` | an assertion (below) |
| `set_property` | `node_path`, `property` (indexed ok), typed `value`; waits one frame |
| `screenshot` | `path`, `expect`, `describe` (below) |
| `ui_report`, `dump_tree`, `spatial_report` | below; `spatial_report` in `references/spatial_verification.md` |
| `log_marker` | `message`; prints `[SCENARIO] <message>` to search the log between phases |

Assertions (in `assertions[]` or an `assert` step): `node_exists`; `visible` (`expected`, default true); `property` (`node_path`, `property`, `expected`, `operator`, `tolerance`). Operators: `equals, not_equals, greater_than, greater_or_equal, less_than, less_or_equal, contains, approx`. Performance monitors: `fps, process_time, physics_process_time, static_memory, node_count, resource_count, draw_calls, primitives, video_memory`; statistics `average, minimum, maximum`.

### Rules that bite

- **The log is part of the verdict.** Output is parsed like `run_project.py`; any error-level line (`SCRIPT ERROR`, parse error, engine `ERROR:`, shader error) fails the scenario even when every assertion passed. To expect an error, require it with a `log_assertions` entry (`min_count` >= 1) or set `"log_errors": "allow"`. Warnings never fail it.
- `mouse_*` positions are window pixels, while `ui_report` rects and node positions are in base-viewport coordinates. Under `canvas_items` stretch, click at node position x `viewport_size` / base size (or omit `viewport_size`, so the window equals the base size).
- The root viewport is sized to `viewport_size`, else the project's `display/window/size/viewport_*` (a headless window is 64x64 otherwise). With `stretch/mode = canvas_items`, `viewport_size` only resizes the window; UI still lays out against the base viewport, so "two resolutions" is one layout.
- An `action` step calls `Input.action_press/release`: it moves polled state only, so `_input`/`_unhandled_input`/`_gui_input` never run. Player controllers that poll work; pause menus, dialog advance and interact prompts need a `key` step (a real `InputEventKey` through `Input.parse_input_event`).
- `wait_frames` counts process frames, which a headless run spins far faster than the 60 Hz physics tick: `wait_frames: 60` is a fraction of a simulated second. Use `wait_seconds` or `wait_until` for anything driven by gravity, `move_and_slide` or a Tween.
- A step that reloads or changes the scene (`reload_current_scene`, a Restart button) frees the scenario's root: later node steps and `assertions[]` fail with "node not found". Prove a restart with `log_assertions` on a line the level prints at start (`min_count` 2).
- Physics overlaps lag a teleport by a physics frame: after `set_property` on `position`, wait (`wait_seconds` 0.1) before asserting on `body_entered` effects.
- In `log_assertions` regexes written through a shell heredoc, avoid backslash escapes (`\[`): use `.` or a bracket class (`[[]`), or a plain `contains`.
- A screenshot step is the only thing that forces a rendered window. A `blank: true` capture of a scene that should draw means the node is hidden, off-screen or was never added.

### screenshot

```json
{"type": "screenshot", "path": "/absolute/output/menu.png",
 "expect": {"not_blank": true, "min_opaque_ratio": 0.1, "compare_to": "res://tests/reference/menu.png", "max_diff_ratio": 0.02},
 "describe": {"ascii": true, "ascii_width": 80, "ascii_color": true}}
```

`path` is res://, user:// or absolute (parents created). Each violated `expect` key fails the scenario but the run continues; a size mismatch against `compare_to` is its own failure (capture the reference at the same `viewport_size`). Unknown keys are rejected. Results land in `screenshots[]` with a `summary` like `inspect_image`'s, and one line `[SCENARIO] screenshot <path> blank=false opaque=1 bbox=... dominant=#111122`.

### ui_report

Reports every visible Control's post-layout rect and machine-checks the layout. It is how to catch controls authored with `layout_mode = 0` and no offsets all landing at (0, 0). Rects resolve the same headless; it never forces a window. Layout guidance: `references/game_ui.md`.

```json
{"type": "ui_report", "label": "boot", "node_path": ".", "ascii": true, "fail_on": ["overlap", "zero_size", "offscreen"], "path": "/absolute/output/boot_ui.json"}
```

- Options: `node_path` (subtree; report paths stay relative to the scene root), `include_hidden`, `label`, `path` (also write JSON), `ascii` + `ascii_width` (default 80), `fail_on` (finding kinds or `["any"]`; findings alone never fail a run), `min_overlap_ratio` (0.1), `strict_overlap`, `settle_frames` (default 2: containers place children through a deferred sort).
- Result in `ui_reports[]`: `viewport`, `counts`, `controls[{path, class, rect:[x,y,w,h], text?}]`, `findings[{kind, nodes, rects, message}]`. Log line: `[SCENARIO] ui_report <label> controls=12 ... findings=0 zero_size=0 offscreen=0 overlap=0`, which `log_assertions` can regex.
- Finding kinds: `zero_size` (width or height 0, which also covers inverted offsets); `offscreen` (rect entirely outside the viewport; partial clipping is not reported); `overlap` (visible siblings sharing >= `min_overlap_ratio` of the smaller rect). Overlap is checked only under non-Container parents and under Box/Grid/Flow/Split containers; other containers (Margin, Panel, Center, Scroll, Tab, ...) give every child the same slot, so stacking there is skipped. A background-like node (`ColorRect`, `Panel`, `TextureRect`, ...) fully covering a sibling is not an overlap unless `strict_overlap`. There is no text-clipping finding: Godot grows a Control to its text minimum size.
- `ascii: true` adds `ascii` rows: each Control a box with its name in the top edge, parents drawn first so colliding children visibly collide.

### dump_tree

Discovery step: dump once after boot, read the real node paths and values, then write precise `assert`/`set_property`/`ui_report` `node_path`s against them; dump again after an interaction to see what it changed.

```json
{"type": "dump_tree", "node_path": ".", "label": "boot", "properties": ["visible", "position", "text"], "max_depth": 6, "include_internal": false}
```

`properties` defaults to `["visible","position","text"]` (a node without the property just omits it; script `@export` vars work). `max_depth` 0 = only the start node. Result in `tree_dumps[]`: `lines` (indented text) and `nodes[{path, type, props}]` with props as typed JSON. Log line: `[SCENARIO] dump_tree <label> node_path=. nodes=6 max_depth=6`.

## Running and validating

Pick the cheapest check that proves the point (`references/debugging.md` has the decision table): `lint_project.py` (static) < `validate_project.py` (load + instantiate every file) < `run_project.py` (boot the main scene) < `smoke_scenes.py` (boot every scene) < `run_scenario.py` (drive and assert).

### run_project.py

```bash
uv run /absolute/path/to/godot/scripts/debug/run_project.py /absolute/path/to/project [res://scenes/x.tscn] --quit-after 120 --timeout 60 --pretty
```

Boots the main scene (or the one given) headless with `-d --ignore-error-breaks` and prints `{ok, counts, diagnostics[]}` (`diagnostics`: `severity, category, message, file, line, suggested_fix`). `--quit-after N` frames (default 120, minimum 2), `--timeout S` wall clock (default 60), `--log-file PATH`, `--raw` (include the raw log), `--no-warnings`, `--no-headless`, `--extra-arg`, `--dry-run`. `--no-debugger` hides every GDScript warning the editor would show (warnings only travel through the debugger channel), so avoid it. Pass criteria: `ok: true` and `counts.errors == 0`.

### validate_project.py

`--summary` prints `ok`, `counts`, `failed[]`, the first problems, non-hint config warnings and physics-layer findings, and hint counts.

```bash
uv run /absolute/path/to/godot/scripts/debug/validate_project.py /absolute/path/to/project --pretty
```

Runs lint first, then loads every GDScript, scene, shader, resource, GDExtension and editor plugin, and instantiates every scene. Output: `ok`, `counts`, `diagnostics` (lint entries first, with `"source": "lint"`), `lint`, `node_config`, and `static` (the `check_project` result: `failed[]`, `config_warnings[]`, `physics_layers`). Because it runs with `-d`, it reports GDScript warnings for every script, not only the ones a boot loads. Flags: `--warnings-as-errors`, `--no-warnings`, `--no-lint`, `--no-instantiate`, `--no-config-warnings`, `--no-physics-layers` (see `references/node_config.md`), `--csharp auto|always|never`. `probe_environment.py PROJECT` reports the engine and host toolchain.

The scene instantiate pass catches hierarchy mistakes that `load()` accepts: a root with `parent="."` or a non-root node with no `parent=` fails the scene (the engine message names the node, not the scene: read the path from `static.failed[]`); a `parent=` naming a missing node only warns (the node is reparented to the root as `VBox#Label`, so use `--warnings-as-errors`). A fully flat tree (every node `parent="."`) is valid and not caught: only `inspect_scene` shows it. Instantiating runs the root script's `_init()` and stored-property setters, not `_ready()`; autoloads are available. `--no-instantiate` skips it and re-opens that blind spot.

The underlying op is `check_project` (`project_path`, `instantiate`, `config_warnings`, `physics_layers`), callable through the dispatcher.

### lint_project.py

```bash
uv run /absolute/path/to/godot/scripts/debug/lint_project.py /absolute/path/to/project --pretty [--only godot3_api,input_action] [--path scripts] [--warnings-as-errors]
```

Static, no Godot binary, well under a second. Output: `ok`, `counts`, `diagnostics[{severity, category, rule, message, file, line, suggested_fix}]`, `scan_summary`, `suppressions`. Exit 1 on any error. Error = Godot refuses to parse or load it; warning = runs but risky. Categories for `--only`:

| category | catches |
| --- | --- |
| `godot3_api` | outdated API and class names in `.gd` and `.tscn`/`.tres`, with the 4.7 replacement (`--list-rules` prints them) |
| `godot3_shader` | outdated shader names (`hint_color`, `SCREEN_TEXTURE`, ...). A shader that fails to compile renders as the default material with no runtime error, so this static pass matters |
| `inference` | `:=` from an untyped expression: error for `.get()`, untyped `Array`/`Dictionary` reads, `JSON.parse_string()`, `null`, `.call()`, same-file functions without `-> Type`; warning for `$Node`, `%Name`, `get_node()`, `.instantiate()` (typed bare `Node`) |
| `node_ref`, `unique_name` | `$Path`, `%Name`, `get_node("...")` not present in the `.tscn` that attaches the script |
| `signal_target` | `[connection]` nodes that do not exist or handlers the target script lacks (Godot drops a wrong path silently) |
| `missing_resource` | `[ext_resource]` paths, `preload()`/`load()` literals, `run/main_scene`, `[autoload]` that do not exist (a scene with a missing ext_resource still loads) |
| `input_action` (error) | action names used in code that `project.godot` `[input]`, built-in `ui_*`, or `InputMap.add_action` never define. The fix is a `project_batch` call |
| `group_ref` (warning) | groups queried but never provided by a `.tscn` `groups=`, `add_to_group()` or `[global_group]` |
| `res_path` | `res://` literals that miss on disk, case-sensitively (`Hero.png` loads on Windows and macOS, fails on Linux and in every exported `.pck`) |
| `animation_ref` (warning) | `play("name")` on an `AnimatedSprite`/`AnimationPlayer` whose resource has no such animation (it silently keeps the old animation) |

It errs toward silence: runtime-built paths are skipped, only a bare `$`/`%`/`get_node` is resolved, a name the project itself defines suppresses the matching rename rule. `--include-addons` also lints `addons/`; `.godot/`, hidden dirs and any directory with a `.gdignore` are always skipped.

Suppress one finding with `# lint:ignore <category>[,<category>]` on the line or the line above (bare `# lint:ignore` drops all; marker is `#` in `.gd`, `;` in `.tscn`/`.tres`/`project.godot`, `//` in `.gdshader`). Unused suppressions are listed under `suppressions.unused` and never fail the run.

### smoke_scenes.py

`run_project.py` boots only the main scene and `check_project` never runs `_ready`/`_process`. This boots **every** scene, one Godot process per scene, for a bounded amount of game time, so a crash or hang cannot take the others down.

```bash
uv run /absolute/path/to/godot/scripts/debug/smoke_scenes.py /absolute/path/to/project --seconds 2 --jobs 4 --pretty
uv run /absolute/path/to/godot/scripts/debug/smoke_scenes.py /absolute/path/to/project --scenes res://scenes/main.tscn --seconds 4 --fuzz --fuzz-mouse --fuzz-seed 7 --profile --pretty
```

- Default is every `.tscn` except `addons/`, hidden dirs and `.gdignore` folders. Narrow with `--scenes`, `--include GLOB`, `--exclude GLOB`. `--seconds` is game time (default 1.0; costs almost no wall time under the default `--fixed-fps 60`). Also `--jobs N`, `--timeout S`, `--warnings-as-errors`, `--no-warnings`, `--log-dir DIR`, `--dry-run`.
- Output: `{ok, counts{scenes, passed, failed, errors, warnings, findings}, scenes[]}`; each scene has `ok, returncode, timed_out, frames, seconds_simulated, counts, findings[], diagnostics[], perf?, fuzz?, destination?`. Exit 0 only when every scene passed, 1 when one failed, 2 for a usage error.
- A scene fails on an error diagnostic, crash, timeout or error-level finding; warnings only under `--warnings-as-errors`.
- Finding types. Warnings: `node_growth` (node count still climbing: bullets/particles never freed), `orphan_nodes` (nodes alive outside the tree after the scene was freed), `static_memory_growth`. Errors: `timed_out`, `crashed`, `instantiate_failed`/`load_failed`/`scene_missing`. Info (not failures): `scene_changed` (`destination` says where), `quit_called`.
- `--fuzz` feeds the project's real `InputMap` actions (and a few `ui_*`) through `Input.parse_input_event`, so `_input` handlers and polling both see them; `--fuzz-mouse` adds motion and clicks, including on visible buttons. Events are printed live as `[SMOKE_INPUT] f42 +jump` and kept in `fuzz.events`; the same `--fuzz-seed` replays identically under the default fixed fps. Fuzzing proves a path crashes, never that one is safe.
- `--profile` adds `perf.monitors` (avg/p95/max for `process_ms`, `physics_process_ms`, `object_count`, `node_count`, `orphan_node_count`, `static_memory_mb`, ...). Under fixed fps the engine's own `fps`/`process_ms` monitors read stale, so the flat `fps_avg`, `process_ms_p95`, `physics_ms_p95` are `null`; read `perf.frame_ms` (wall time per frame, pure CPU cost) instead, or re-run one scene with `--real-time --seconds 3`. Node, object, orphan and memory monitors are exact in both modes; draw-call and video-memory numbers are meaningless on the dummy renderer.
- A scene is booted in isolation: one that expects a parent (a pause menu reading `get_parent().player`) reports errors the real game would not. Read the diagnostic before "fixing" it, or `--exclude` that scene.

### import_project.py and audit_imports

`--summary` prints `ok`, the import exit code, audit scalars, `counts` and the first errors.

```bash
uv run /absolute/path/to/godot/scripts/import/import_project.py /absolute/path/to/project --pretty
```

Runs `godot --headless --import` and then audits import state (statuses `ok`, `missing`, `invalid`, `stale`, `orphaned`). `--audit-only` skips the reimport; the same audit is the dispatcher op `audit_imports` (`project_path`, `include_entries`). Run it after adding any image, audio, model or font file, before a scene references it, and after `set_import_options`.

## move_resource.py

**Never `mv` a file inside a Godot project.** A shell move rewrites no references, and the damage is often silent: an `[ext_resource]` carrying both `uid=` and `path=` resolves through the uid, so moving a texture with its `.import` leaves the project running with zero diagnostics while the recorded path rots (only the `missing_resource` lint notices). Moving it without the `.import` breaks loudly. Use the tool for both.

```bash
uv run /absolute/path/to/godot/scripts/project/move_resource.py /absolute/path/to/project scripts/main.gd actors/main.gd --dry-run --pretty
uv run /absolute/path/to/godot/scripts/project/move_resource.py /absolute/path/to/project art/enemy.png art/sprites/
uv run /absolute/path/to/godot/scripts/project/move_resource.py /absolute/path/to/project --map /absolute/moves.json
```

- `SRC` is a file or folder; `DST` follows `mv` semantics (an existing folder or trailing `/` moves into it, keeping the basename). `--map FILE.json` (`[{"from":"...","to":"..."}, ...]`) applies a whole reorganisation as one atomic plan; conflicts, nesting and cycles are rejected before anything moves. `--dry-run` prints the plan and touches nothing. `--no-import` skips the final `--import` (the uid cache then stays stale: a uid-bearing reference resolves through the cache before its corrected text path and fails with `Resource file not found: <old path>`, so run the import). `--extra-ext .json,.md`, `--include-addons`.
- Rewrites every `res://` reference in `.tscn`, `.tres`, `.gd` (string literals only; a path in a `#` comment is listed under `mentions[]` and left alone), shaders (including relative `#include`), `.import`, `project.godot`, `export_presets.cfg`, `.gdextension`, `.cs`. `.import` and `.uid` sidecars travel with the file, so `uid://` references stay valid.
- Refuses with exit 2 and touches nothing: missing `SRC`, existing `DST`, a destination outside the project, `project.godot` or `.godot/`, a folder moved into itself, an invalid `--map`. A mid-apply failure is rolled back.
- It proves the move: `missing_resource` lint runs before and after. Read `verify.new_missing`; it must be `[]` (`ok` is false only when the move introduced a broken reference). Blind spots are reported, not missed: binary `.scn`/`.res` (re-save as `.tscn`/`.tres`) and non-UTF-8 files.
- A move never changes a `class_name`. Renaming a class or node is a source edit; check the fallout with `lint_project.py --only node_ref,unique_name,signal_target`.

## scaffold_project.py

`godot --headless` cannot create a project; this does it through the dispatcher and then validates the result.

```bash
uv run /absolute/path/to/godot/scripts/project/scaffold_project.py /absolute/path/to/new_project --preset pixel2d --name "Cave Diver" --with-tests --pretty
```

- `--preset` (required): `pixel2d` (320x180, `viewport` stretch, integer scale, nearest filtering), `hd2d` (1920x1080, `canvas_items`), `3d` (camera, light, environment and ground already in the main scene), `ui` (resizable, low-processor mode, a container starter screen). `--name`, `--size WxH`, `--autoloads a,b,c` (template stems from `templates/gdscript`; default `game_manager,save_manager,scene_transition`, none for `ui`), `--no-autoloads`, `--with-tests` (runs `run_tests.py DEST --init-mini`), `--skip-validate`.
- Writes the folder layout, `project.godot`, `icon.svg`, `.gitignore`, `scenarios/boot_check.json`, `scenes/main.tscn` and the autoload scripts. The input map binds `move_left/right/up/down`, `jump`, `attack`, `interact`, `pause` to physical keys, gamepad buttons and the left stick. Physics layers 1-6 are named `world player enemies pickups hitboxes hurtboxes`.
- Then runs `--import` and `validate_project.py --warnings-as-errors`. Read `ok`, `validate`, and `next[]` (copy-paste run/boot-scenario/validate commands). Exit 2 means it refused before touching anything (the destination already holds a `project.godot`, bad `--size`, unknown autoload template, no `godot` on PATH); exit 1 means a step failed and the JSON names it.
- A fresh project of every preset validates clean and passes `scenarios/boot_check.json`.
