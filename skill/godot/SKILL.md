---
name: godot
description: Build, fix and verify Godot 4.7 games headlessly with bundled tools - scaffold projects, edit scenes through a dispatcher, look up the installed engine's exact API, lint GDScript/scenes/shaders without the engine, generate placeholder art and sound, unit-test, run scripted gameplay scenarios, smoke-run every scene, and export desktop/Web builds, all verified as text with no vision needed.
---

# Godot

This file is the index. Read it fully, then open only the reference the task table points to. Every reference stands alone.

## Paths and environment

- `/absolute/path/to/godot` (also `<skill>` in `help` output) is this skill's root, the folder holding this file. Replace it with the real absolute path everywhere, in double quotes so a Windows path survives the shell.
- `/absolute/path/to/project` is the Godot project, and `/absolute/<word>` is a scratch path beside it. Never write outside the project, and never use `/tmp`.
- Python tools are PEP 723 scripts: `uv run "/absolute/path/to/godot/scripts/<area>/<tool>.py" ... --help`. They need `uv`, and nothing else.
- Godot is found as `godot`, or through `GODOT_BIN` or `--godot-bin`. On Windows `godot_console` is the console build, so use it when `godot` prints nothing.
- Dispatcher operations (scene, resource and project edits, inspection, asset drawing):

  ```bash
  godot --headless --path /absolute/path/to/project \
    --script /absolute/path/to/godot/scripts/core/dispatcher.gd <op> '<json>'
  ```

  - Run `help '{}'` to list ops and Python tools.
  - Run `help '{"op":"<op>","format":"text"}'` for one op's parameters, an example and its gotchas. Do this instead of guessing a key: unknown keys are rejected and nothing is written.
  - Typed JSON values (`Vector2`, `Color`, resources, NodePaths) are described in `references/automation_api.md`.

## Rules that prevent most failures

1. **Look before you type.** Run `uv run .../scripts/docs/api_lookup.py Class.member ...` for any engine class, method, signal or constant you are not certain of. A wrong or outdated name exits 1 with the real 4.7 one.
2. **Lint after every edit.** `uv run .../scripts/debug/lint_project.py <project> --pretty` needs no engine and finishes in under a second. It catches:
   - outdated (pre-4.7) API, class and shader names
   - `:=` on values with no static type
   - dead `$Path`/`%Name`/signal targets
   - unknown input actions, groups and `res://` paths

   Fix every `error`.
3. **Type GDScript explicitly.** Annotate variables, parameters and returns. Never use `:=` on `$Node`, `get_node()`, `instantiate()`, `Dictionary`/`Array` reads, `JSON.parse_string()` or untyped calls; Godot refuses to parse them (`references/debugging.md`).
4. **Copy, don't invent.** Start from `templates/gdscript/` for players, enemies, health and hitboxes, state machines, UI, dialogue, inventory, save and settings, and from `templates/shaders/` for shaders.
   - Each template header lists its scene tree, input actions, autoloads and `attach_script` call.
   - After copying a `class_name` script, run `godot --headless --path <project> --import`.
5. **Import new assets.** Run `uv run .../scripts/import/import_project.py <project>` after adding images, audio or `class_name` scripts, before any scene uses them.
6. **Edit scenes with dispatcher ops** (`scene_batch` and friends), not hand-written `.tscn`. If you must write `.tscn` text, read `references/tscn_format.md` first: a tree with every node at `parent="."` loads with no error and is still wrong.
7. **Move files only with `move_resource.py`.** A plain `mv` silently breaks `uid` references and `.import` sidecars.
8. **Silent failures are real.** Exit code 0 is not proof.
   - A shader that fails to compile renders the default material.
   - A body with no shape or an empty collision mask never collides.
   - Sibling Controls outside a container stack at (0,0).
   - `project_batch` rewrites `project.godot`.

   Trust the read-backs listed below.

## Task index

| Task | Read / use | Verify |
| --- | --- | --- |
| New project | `scripts/project/scaffold_project.py <dest> --preset pixel2d\|hd2d\|3d\|ui` | its `validate.counts` errors and warnings both 0 |
| Shared setup: templates, autoloads, buses, layers | `references/playbooks.md` Standard Setup | `validate_project.py` ok |
| Player, enemy, tile level, UI, dialogue, save, audio wiring, inventory, settings, 3D starter | `references/playbooks.md` §1-§11 | that playbook's Verify block |
| A whole small game | `references/playbooks.md` §12 | its core-loop scenario wins |
| Menus, HUD, any `Control` layout and theme | `references/game_ui.md` | `run_scenario.py` `ui_report` findings 0 |
| Sprites, tiles, icons and 9-patch panels with no image generator; cleaning generated art | `references/pixel_art.md`, `references/asset_pipeline.md` | `inspect_image` `expect` |
| Sound effects and music | `references/audio.md` | `inspect_audio` `expect` exit 0 |
| Shader effects | `references/shaders.md` | `check_project` `failed_count == 0` |
| Engine API and running quick GDScript (`run_gdscript`) | `references/api_lookup.md` | exit 0 |
| Dispatcher, typed JSON, batch ops, scenarios, runners, linter | `references/automation_api.md` | as documented there |
| Runtime errors, a project that will not boot, parse errors | `references/debugging.md` | `run_project.py` `"ok": true` |
| Something silently does nothing (falls through floor, no collision, invisible) | `references/node_config.md` | `validate_project.py` 0 config warnings, `physics_layers.findings` empty |
| Is it on screen, in the floor, lit, in front of the camera | `references/spatial_verification.md` | `spatial_report` findings 0 |
| Unit tests for game logic | `references/testing.md` | `run_tests.py` `"ok": true`, `counts.tests > 0` |
| Hand-writing `.tscn`/`.tres` | `references/tscn_format.md` | `inspect_scene` nesting matches intent |
| Desktop or Web build | `references/export_targets.md` | artifact exists and boots |

## Tools at a glance

All Python tools take `--help`. Paths are relative to the skill root.

| Tool | Use |
| --- | --- |
| `scripts/docs/api_lookup.py` | exact signatures from the installed engine; `--search TEXT` for half-remembered names |
| `scripts/debug/lint_project.py` | static checks without the engine; `--list-rules` |
| `scripts/debug/validate_project.py` | engine loads every script, scene, shader and resource; `--warnings-as-errors` |
| `scripts/debug/run_project.py` | boot and play for N frames, structured errors |
| `scripts/debug/run_scenario.py` | scripted input plus `ui_report` / `spatial_report` / `screenshot` / assertions |
| `scripts/debug/smoke_scenes.py` | boot every scene, `--fuzz` input; catches errors, hangs, leaks |
| `scripts/test/run_tests.py` | unit tests; `--init-mini` installs the zero-dependency framework |
| `scripts/assets/make_sfx.py`, `make_music.py` | synthesized WAV effects and looping music |
| `scripts/project/scaffold_project.py`, `move_resource.py` | new project; safe move or rename |
| `scripts/import/import_project.py` | (re)import assets and rebuild the class cache |
| `scripts/export/export_project.py`, `serve_web.py` | export a preset; serve a Web build with the right headers |
| dispatcher ops | `help '{}'`: scene/resource/project batches, `build_tileset`, `paint_tilemap`, `build_theme`, `draw_image`, `process_image`, `inspect_*`, `check_project`, `run_gdscript`, `add_export_preset` and more |

## Small-game architecture

- A scene's root script owns its children. Children signal up, parents call down.
- Use autoloads only for truly global services (save, settings, audio, scene change). Avoid a catch-all GameManager holding mutable state.
- Put pure rules (damage, inventory, scoring) in `RefCounted` or `Resource` scripts so `run_tests.py` can test them without a scene. Put data in `Resource` files, not code.
- Use a flat layout: `scenes/`, `scripts/`, `art/`, `audio/`, `tests/`. Add feature folders only when the game outgrows it.
- Without a stated art direction, keep the project's existing style, or default to pixel art.

## Not possible headlessly

Do not attempt these; tell the user they need the editor:

- `LightmapGI` and occluder bakes
- `VisualShader` graphs (write a text `.gdshader` instead)
- `EditorScript._run()`
- POT translation template generation
- scaffolding a first C# solution

`ReflectionProbe` and `CompositorEffect` cannot be verified under the headless driver.

## Before you finish

Run `references/playbooks.md` §13. In short, all of these must hold:

1. `lint_project.py` reports `counts.errors == 0`.
2. `validate_project.py --warnings-as-errors` reports `"ok": true`, with zero config warnings and no `physics_layers.findings`.
3. `run_tests.py` reports `"ok": true` with `counts.tests > 0` for logic you wrote.
4. A `run_scenario.py` scenario drives the core loop and passes. Include `ui_report` (`fail_on: ["any"]`) for UI, `spatial_report` for placement, and one `screenshot` with `expect.not_blank`.
5. `smoke_scenes.py --seconds 2 --jobs 4 --fuzz` exits 0.
6. Generated art passed `inspect_image` `expect`; generated audio passed `inspect_audio` `expect`, and loops carry the loop import option.
7. Every moved file went through `move_resource.py`; every write stayed inside the project.
8. For builds, the artifact came from the intended preset and was booted once.
