# Spatial Verification (2D and 3D, without eyes)

Read when you cannot see WHERE something is in the world (player, floor, level, camera, light), not where a Control sits. `spatial_report` is a `run_scenario.py` step that sees the live scene after `_ready`, spawn code and driven input. Parameters and result shape: `references/automation_api.md` (spatial_report). UI layout: `ui_report`; painted cells: `inspect_tilemap`.

| Bug (silent: no error) | Detector |
| --- | --- |
| Player spawned outside the camera view | `expect_on_screen` -> `not_on_screen` |
| Player spawned inside the floor (falls through, jitters) | `embedded_in_static` |
| Level painted at the wrong origin | TileMapLayer world `rect` + ASCII map |
| 3D scene with no camera / no light (black frame) | `no_camera_3d` / `no_light_3d` |
| 3D camera pointing the wrong way | `not_in_frustum` / `behind_camera` |
| Node scaled to 0 or 100000 units away | `zero_scale`, `far_from_origin` |

Findings alone never fail a run: `expect_*` produces the finding, `fail_on` (list of ids, or `["any"]`) gates the scenario exit code. An unknown id is rejected with the full list.

## 2D gate: on screen and standing on the floor

Put it right after boot and after anything that moves the player.

```json
{"scene_path": "scenes/level.tscn", "viewport_size": {"width": 320, "height": 180},
 "steps": [{"type": "spatial_report", "label": "boot", "ascii": true,
            "expect_on_screen": ["Hero"],
            "fail_on": ["not_on_screen", "embedded_in_static", "no_camera_2d"]}],
 "assertions": [],
 "log_assertions": [{"regex": "spatial_report boot .* findings=0"}]}
```

```bash
uv run /absolute/path/to/godot/scripts/debug/run_scenario.py /absolute/path/to/project \
  /absolute/path/to/project/scenario.json --log-file /absolute/path/to/project/run.log --pretty
```

Read `spatial_reports[]` in the JSON or the `[SCENARIO] spatial_report ...` log lines. Green means `findings=0` with `fail_on` set. The ASCII map is top-down (x right, y down); the legend assigns letters per run; a mover's block overlapping the floor's block means `embedded_in_static`. `embedded_in_static` shrinks the query shape by `embed_margin` (1 px in 2D, 1 cm in 3D) so a body resting exactly on the floor is not flagged; TileMapLayer/GridMap collision counts as static.

## 3D gate: camera, light, mesh in view

```json
{"type": "spatial_report", "label": "render-check", "ascii": true, "expect_on_screen": ["Crate"],
 "fail_on": ["no_camera_3d", "no_light_3d", "not_in_frustum", "behind_camera", "embedded_in_static"]}
```

- ASCII map is XZ from above, +Z down the page: `@` camera, `^` its facing, `:` frustum footprint.
- `no_light_3d` is satisfied by a visible `Light3D` with energy > 0 or a `WorldEnvironment` with ambient/sky light. It is a report, not a verdict: unshaded materials render fine unlit, so drop it from `fail_on` for unshaded / 2.5D projects.
- `behind_camera` names the camera position and facing; `not_in_frustum` gives `fov` and `far` (aimed wrong vs too far). In-frustum entries carry `screen_pos`.

## Level placement

`inspect_tilemap` proves the cells; `spatial_report` proves where the layer is: `{"type":"spatial_report","classes":["TileMapLayer","Camera2D","CharacterBody2D"],"ascii":true,"ascii_bounds":"camera"}`. A 10x6 map of 16 px tiles painted from the origin has `rect` `[0,0,160,96]`; shifted `rect` means a wrong `origin` in `paint_tilemap`, doubled means the layer is scaled, `on_screen:false` means the camera is not on the level. `ascii_bounds`: `"content"` (default, finds wanderers) or `"camera"` (what the player sees).

## Reading the numbers

- `rect` is world space (matches the `.tscn`); `screen_rect` goes through the canvas transform (camera zoom/offset/rotation). Partly visible counts as on screen.
- Nodes under a `CanvasLayer` get `space: "screen"` and are excluded from world checks and the ASCII map; use `ui_report` for HUD.
- Nodes with no measurable extent (plain `Node2D`, mesh-less `MeshInstance3D`) are not listed; naming one in `expect_on_screen` errors instead of passing.
- Rotated cameras/nodes give axis-aligned, slightly generous bounds.

## Limits

- Geometric only: `modulate.a = 0`, discarding materials, `visibility_layer`/`cull_mask` and occluders are invisible to it. Confirm pixels with a `screenshot` step with `expect.not_blank` where a framebuffer exists.
- `embedded_in_static` only sees colliders present at that moment.
- `dimension: "auto"` picks 3d if the subtree has any `Node3D`; read a 2D HUD in such a scene with `ui_report`.
- The ASCII map samples one point per cell; raise `ascii_size` or narrow with `node_path`/`classes` for detail.
