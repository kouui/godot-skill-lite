# Node Configuration Warnings and Physics Layers

Read when a scene "looks right" but does nothing at runtime (player falls through the floor, `AnimatedSprite2D` shows nothing, particles idle, ScrollContainer will not scroll, 3D scene black). The editor's yellow triangles cannot be read from a headless run, so `check_project` re-implements the high-signal subset over the instantiated scene tree.

## Use

Runs by default inside the standard validator:

```bash
uv run /absolute/path/to/godot/scripts/debug/validate_project.py /absolute/path/to/project --pretty
```

Read `config_warnings[]` entries `{scene, node_path, node_type, rule, severity, message, fix}` (also logged as `WARNING: [node_config:<rule>] ...`). `fix` is a runnable dispatcher call (paste it, re-validate); a `<Placeholder>` in it must be substituted first (`joint_without_nodes`, `remote_transform_bad_path`, `animation_tree_without_player`). `node_path` starts at `root`.

- `severity: warning` = editor parity; fails the run only with `--warnings-as-errors`. `severity: hint` = heuristic, never fails anything (shown under `node_config.hints`).
- Opt out: `--no-config-warnings`, `--no-physics-layers`; `--no-instantiate` disables both.
- Blind spots: anything a script assigns in `_ready()` (shape, texture, mesh, stream, SpriteFrames) is invisible to the static pass, hence the hints; only the subset below is checked. Use `run_scenario.py` for the live tree.
- A scene that cannot be instantiated is skipped (`check_project` already fails it).

## Warnings (editor parity)

| Rule | Fires when |
| --- | --- |
| `body_without_shape` | CollisionObject2D/3D has no direct `CollisionShape*`/`CollisionPolygon*` child |
| `shape_without_resource` | `CollisionShape2D/3D.shape == null` |
| `shape_wrong_parent` | shape/polygon node whose parent is not a CollisionObject (scene roots skipped) |
| `collision_polygon_empty` | polygon empty or too few points |
| `shapecast_without_shape` | `ShapeCast2D/3D.shape == null` |
| `animated_sprite_without_frames` | no `SpriteFrames`, or `animation` names a missing animation (draws nothing) |
| `particles_without_material` / `particles_without_draw_pass` | GPUParticles without `process_material`; 3D without draw pass / CPUParticles3D without mesh |
| `path_follow_wrong_parent`, `parallax_layer_wrong_parent`, `navigation_agent_wrong_parent`, `vehicle_wheel_wrong_parent` | PathFollow not under Path, ParallaxLayer not under ParallaxBackground, NavigationAgent not under Node2D/3D, VehicleWheel3D not under VehicleBody3D |
| `navigation_region_without_mesh` | region with no NavigationPolygon / NavigationMesh |
| `animation_tree_without_root` | `AnimationTree.tree_root == null` (fix: `build_animation_tree`) |
| `world_environment_without_environment`, `world_environment_duplicate`, `canvas_modulate_duplicate` | empty WorldEnvironment; more than one WorldEnvironment; more than one visible CanvasModulate |
| `remote_transform_bad_path` | empty/unresolvable `remote_path` |
| `light_occluder_without_polygon`, `point_light_without_texture` | LightOccluder2D with no polygon; PointLight2D with no texture |
| `scroll_container_child_count` | ScrollContainer does not have exactly ONE sortable child control (fix wraps children in a VBoxContainer) |
| `subviewport_container_without_viewport` | no SubViewport child |
| `physics_body_scaled` | RigidBody scaled; any 3D collision object/shape with non-uniform scale (resize the shape, keep scale 1) |
| `joint_without_nodes` | `node_a`/`node_b` empty, not bodies, or identical |
| `multiplayer_synchronizer_without_config`, `multiplayer_spawner_bad_path` | empty/unresolvable `root_path` / `spawn_path` |

## Hints

`sprite_without_texture`, `tilemap_layer_without_tileset`, `gridmap_without_mesh_library`, `audio_player_without_stream`, `mesh_instance_without_mesh`, `animation_tree_without_player`, `label_without_text` (also empty Button), `scene_3d_without_camera`, `scene_3d_without_light` (the "3D renders black" case; both suppressed for scenes another scene instantiates). Expected during WIP or when a script fills the value at runtime. Layout questions: `ui_report` step, not this pass.

## Physics layer/mask survey

`config_warnings` is accompanied by `physics_layers` in the same result: per bit, who occupies it (collision_layer of bodies, TileSet physics layers, GridMap, CSG with collision) and who scans it (masks of bodies, monitoring Areas, enabled RayCast/ShapeCast; StaticBody never scans). Names come from `layer_names/2d_physics/layer_N` (set with `project_batch` `set_layer_name`; named layers make the report readable). Findings (all hints, with a `fix`):

| Finding | Meaning |
| --- | --- |
| `mask_targets_empty_layer` | mask has a bit nothing occupies: that query never hits (fix drops dead bits) |
| `layer_never_scanned` | occupants nothing detects; normal early on |
| `body_mask_zero` | body/monitoring area/cast with mask 0; a body with mask 0 falls through every floor |
| `body_layer_zero` | PhysicsBody with layer 0: nothing can collide with or detect it (Areas exempt: detector-only pattern) |

Related: `references/debugging.md` (run -> diagnose -> fix), `references/tscn_format.md` (hierarchy errors).
