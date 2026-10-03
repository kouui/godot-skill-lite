# Godot 3 to 4 Migration

Read when a project or snippet uses Godot 3 API (or you are porting). The linter needs no Godot binary and prints a `Fix` per finding; the full rename table is `uv run /absolute/path/to/godot/scripts/debug/lint_project.py --list-rules`.

```bash
uv run /absolute/path/to/godot/scripts/debug/lint_project.py /absolute/path/to/project --pretty
uv run /absolute/path/to/godot/scripts/debug/lint_project.py /absolute/path/to/project --only godot3_api,godot3_shader
```

## Renames models most often get wrong

- `onready var` / `export var` / `export(Type)` / `tool` -> `@onready` / `@export` / `@export_range`, `@export_enum` / `@tool`
- `yield(obj, "sig")` -> `await obj.sig`; `setget` -> `set:` / `get:` property blocks
- `scene.instance()` -> `instantiate()`; `.empty()` -> `.is_empty()`
- `KinematicBody2D/3D` -> `CharacterBody2D/3D`; `move_and_slide(vel, up)` -> set `velocity` and `up_direction`, then `move_and_slide()`; `_with_snap` -> `floor_snap_length`
- `get_tree().change_scene(path)` -> `change_scene_to_file(path)` / `change_scene_to_packed(packed)`
- `Tween` node / `interpolate_property` -> `create_tween().tween_property(...)`
- `TileMap` -> `TileMapLayer` nodes; `ParallaxBackground`/`ParallaxLayer` -> `Parallax2D`
- Shaders: `hint_color` -> `source_color`, `hint_albedo` -> `source_color`; `uniform`/render-mode/built-in renames are listed in `--list-rules` (Shaders group).

## Unchanged: do not "fix"

Current 4.x API that only looks like Godot 3: `instance_from_id()`, `Label3D`, `Camera2D`, `Path2D`/`PathFollow2D`, `RayCast2D`, `CollisionShape2D`, `StaticBody2D`, `RigidBody2D`, `Area2D`, `AnimationPlayer`, `Timer`, `CanvasLayer`, `visible_characters`, `theme_override_constants/margin_left`. Shaders: `hint_normal`, `hint_default_transparent`, `hint_roughness_gray`, `OUTPUT_IS_SRGB`, `AT_LIGHT_PASS`, `ATTENUATION`, `SCREEN_PIXEL_SIZE`, `TEXTURE_PIXEL_SIZE`, `INSTANCE_CUSTOM`, render modes `specular_toon` / `specular_disabled`.

## Migration order

1. Run the linter; fix `error` rules first (project will not parse), then `warning` rules.
2. Convert `project.godot`/`.tscn`/`.tres` with the editor or by hand (see `references/tscn_format.md`); never leave `config_version=4`.
3. Re-annotate types while converting (`references/debugging.md`, typing rule).
4. Run `validate_project.py` and `smoke_scenes.py`, then repeat until clean.

Suppress an intentional old name with `# lint:ignore godot3_api` (`//` in shaders).
