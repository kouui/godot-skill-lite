# Shaders: Copy A Template, Attach It, Prove It Compiles

Read this before writing or attaching any `.gdshader`.

Do not write a shader from memory: models produce outdated syntax (`hint_color`, `SCREEN_TEXTURE`, `WORLD_MATRIX`) or plain GLSL, and **a shader that fails to compile is silently replaced by the default material at draw time**. The sprite keeps rendering, the effect is missing, the log says nothing. Copy a file from `templates/shaders/` and edit it; its header documents uniforms, ranges, renderer support, attach commands and how to drive it. Read the header first. `lint_project.py` flags outdated shader names.

## Templates (in `/absolute/path/to/godot/templates/shaders/`)

2D (`canvas_item`):

- `flash`: hit flash (`flash_amount` 0..1, `flash_color`)
- `outline_2d`: sprite outline (`outline_width`, `outline_color`, `inside`)
- `dissolve_2d`: burn away / materialise (`progress`, `noise`, `edge_color`)
- `palette_swap`: exact-colour team swap (`source_colors[]`, `target_colors[]`, `color_count`)
- `hsv_shift`: hue rotation for any art (`hue_shift`, `saturation`, `value`)
- `grayscale`: death / disabled look (`amount`, `tint`)
- `scroll`: parallax, conveyor, waterfall (`scroll_speed`, `scroll_offset`, `tiling`)
- `wave`: swaying grass/flags (`amplitude`, `speed`, `phase`, `anchor`)
- `pixelate`: block-down transition (`block_size`)
- `vignette`: darkened screen edges (`strength`, `inner_radius`, `outline_radius`)
- `crt`: old-TV post-process (`curvature`, `scanline_count`, `aberration`)
- `water_2d`: water/lava/heat-haze strip (`distortion`, `water_color`, `foam_height`)
- `shockwave`: screen-warping ring (`center`, `radius`, `strength`)

3D (`spatial`; use `material_override`, `surface_material_override/0`, or `next_pass`):

- `toon`: cel shading (`bands`, `band_softness`, `ambient`)
- `dissolve_3d`: mesh burns away (`progress`, `noise`, `triplanar`)
- `outline_3d`: black outline, goes in `next_pass` (`outline_width`, `outline_color`)
- `water_3d`: depth tint and foam (`depth_fade_distance`, `wave_strength`, `compatibility_depth`)

## Attach it

A `.gdshader` is a native text resource: copy it into the project and you are done (no `.import`). Shared material:

```bash
mkdir -p /absolute/path/to/project/shaders /absolute/path/to/project/materials
cp /absolute/path/to/godot/templates/shaders/flash.gdshader /absolute/path/to/project/shaders/flash.gdshader
godot --headless --path /absolute/path/to/project \
  --script /absolute/path/to/godot/scripts/core/dispatcher.gd \
  resource_batch '{"resource_path":"materials/flash.tres","create_if_missing":true,"resource_type":"ShaderMaterial","actions":[{"type":"set_properties","properties":{"shader":{"__resource":"res://shaders/flash.gdshader"}}},{"type":"set_indexed_properties","properties":{"shader_parameter/flash_color":{"__type":"Color","r":1,"g":1,"b":1,"a":1},"shader_parameter/flash_amount":0.0}}]}'
godot --headless --path /absolute/path/to/project \
  --script /absolute/path/to/godot/scripts/core/dispatcher.gd \
  configure_node '{"scene_path":"scenes/player.tscn","node_path":"root/Sprite2D","properties":{"material":{"__resource":"res://materials/flash.tres"}}}'
```

- Order: `shader` in a `set_properties` action first, uniforms in `set_indexed_properties` after it (a `ShaderMaterial` has no `shader_parameter/*` names until it has a shader).
- Change a saved uniform later with `resource_batch` on the `.tres`. `configure_node` with `indexed_properties` `material:shader_parameter/x` on an **external** `.tres` also saves that file, so every node sharing it changes; on an inline material it touches only that scene.
- Inline (per-node) material: put `"material":{"__resource_type":"ShaderMaterial","properties":{"shader":{"__resource":"res://shaders/x.gdshader"},"shader_parameter/amount":1.0}}` in `add_node.properties`. `shader` must be the **first** key, or you get `Property does not exist at properties.material.properties`.
- 3D: `configure_node` with `"properties":{"material_override":{"__resource":"res://materials/toon.tres"}}`. One surface: `"indexed_properties":{"surface_material_override/0":{...}}`. Outline: `next_pass` belongs to the material, so assign the base material first, then `"indexed_properties":{"material_override:next_pass":{"__resource":"res://materials/outline_3d.tres"}}` (it fails on a null `material_override`).
- Texture uniforms (`noise`, ...): `{"__resource":"res://art/x.tres"}`. A PNG just written by `draw_image` has no `.import` yet; run `import_project.py` first. A `.tres` such as `NoiseTexture2D` needs no import.

### Full-screen post-process (crt, vignette, shockwave, pixelate)

`hint_screen_texture` sees what is already drawn, so the `ColorRect` goes on a `CanvasLayer` above the content (a higher layer, e.g. the HUD, stays sharp). Two things make it invisible if forgotten: `configure_control` `"anchors_preset":15` (else the rect is 0x0; `ui_report` flags `zero_size`) and `"mouse_filter":2` in the node properties (else it eats every click). Declaring `uniform sampler2D screen_texture : hint_screen_texture;` is enough; no `BackBufferCopy` node is needed.

```bash
godot --headless --path /absolute/path/to/project \
  --script /absolute/path/to/godot/scripts/core/dispatcher.gd \
  add_node '{"scene_path":"scenes/main.tscn","parent_node_path":"root","node_type":"CanvasLayer","node_name":"PostProcess","properties":{"layer":100}}'
godot --headless --path /absolute/path/to/project \
  --script /absolute/path/to/godot/scripts/core/dispatcher.gd \
  add_node '{"scene_path":"scenes/main.tscn","parent_node_path":"root/PostProcess","node_type":"ColorRect","node_name":"CRT","properties":{"mouse_filter":2,"material":{"__resource_type":"ShaderMaterial","properties":{"shader":{"__resource":"res://shaders/crt.gdshader"},"shader_parameter/curvature":0.08}}}}'
godot --headless --path /absolute/path/to/project \
  --script /absolute/path/to/godot/scripts/core/dispatcher.gd \
  configure_control '{"scene_path":"scenes/main.tscn","node_path":"root/PostProcess/CRT","anchors_preset":15}'
```

### Typed uniform values (the one JSON trap)

`set_indexed_properties` does not coerce numbers into vectors/colors.

| Uniform | JSON |
| --- | --- |
| `float` / `int` / `bool` | `0.5` / `3` / `true` |
| `vec2` / `vec3` / `vec4` | `{"__type":"Vector2","x":1,"y":0}` (`Vector3`, `Vector4`) |
| `vec4 : source_color` | `{"__type":"Color","r":1,"g":0,"b":0,"a":1}` (a `"#rrggbb[aa]"` string also works once `shader` is assigned) |
| `vec4[] : source_color` | `{"__type":"PackedColorArray","values":[{"__type":"Color",...}]}` |
| `sampler2D` | `{"__resource":"res://art/noise.png"}` |

## Drive uniforms

- GDScript: `(sprite.material as ShaderMaterial).set_shader_parameter("flash_amount", 1.0)`; for a one-shot, `create_tween().tween_method(func(v): material.set_shader_parameter("flash_amount", v), 1.0, 0.0, 0.12)`.
- `AnimationPlayer` / scenario `set_property` path: `Sprite2D:material:shader_parameter/flash_amount` (3D: `Character:material_override:shader_parameter/x`; outline: `...material_override:next_pass:shader_parameter/x`).

## One material, many nodes

A `ShaderMaterial` is a shared resource: `set_shader_parameter` on it flashes every enemy at once. Fixes, cheapest first:

1. `instance uniform float flash_amount : hint_range(0.0, 1.0, 0.01) = 0.0;` then `node.set_instance_shader_parameter("flash_amount", 1.0)` (works for `canvas_item` and `spatial`). An instance uniform cannot be an array or a sampler; templates ship plain uniforms, so switching is a one-word edit.
2. `resource_local_to_scene = true` on a material stored **inside** the scene (`configure_node` `indexed_properties` `material:resource_local_to_scene`): each scene instance gets its own copy.
3. `material = material.duplicate()` in `_ready`.

## Godot 4.7 shading traps

- **Compile failure is silent at draw time.** `check_project` assigns every `.gdshader` to a `ShaderMaterial`, forcing the compile, and reports `SHADER ERROR` with a line number. Run it after every shader edit (below).
- **`COLOR` in `fragment()` already equals `texture(TEXTURE, UV) * modulate * self_modulate`.** `COLOR = texture(TEXTURE, UV)` throws the node's modulate away (fades, tints and modulate animation stop working). Read and write `COLOR` when you can.
- **`MODULATE` does not exist** (documented, but `Unknown identifier` in every stage), and because of the silent fallback the shader just renders unshaded. When you must sample `TEXTURE` at other UVs, capture the tint in `vertex()`:
  ```glsl
  varying vec4 tint;
  void vertex() { tint = COLOR; }
  void fragment() { COLOR = texture(TEXTURE, other_uv) * tint; }
  ```
- Stage built-ins (`TEXTURE`, `UV`, `SCREEN_UV`) are not visible in user functions; pass them as parameters. Passing the `TEXTURE` built-in itself to a function logs `Condition "!actions.custom_samplers.has(...)"`; keep those taps inline.
- No shader warnings outside the editor: a headless run reports errors only, so "zero warnings" means none were looked for.
- Depth reconstruction from `hint_depth_texture` differs by renderer: `ndc.z = raw` on Forward+/Mobile, `raw * 2.0 - 1.0` on Compatibility. The wrong one does not error, it saturates. `water_3d` exposes a `compatibility_depth` uniform; set it from the renderer actually running: `RenderingServer.get_current_rendering_method() == "gl_compatibility"` (not the project setting; GPU-less machines silently fall back to OpenGL).
- `hint_normal_roughness_texture` is Forward+ only. `hint_screen_texture` works on all three renderers.
- Uniform arrays have no default (unset slots are zero); always set the companion count uniform (`color_count` for `palette_swap`); 0 means the shader does nothing.

## Verify

1. **Compiles?** Capture stdout and stderr into one stream (the log parser attributes pathless `SHADER ERROR` lines to the last `Compiling shader:` line, so separate streams blame the wrong file):

```bash
godot --headless --debug --ignore-error-breaks --path /absolute/path/to/project \
  --script /absolute/path/to/godot/scripts/core/dispatcher.gd check_project '{"project_path":"res://shaders"}' \
  2>&1 | uv run /absolute/path/to/godot/scripts/debug/godot_log_parser.py - --pretty
```

2. **Project still sane?** `uv run /absolute/path/to/godot/scripts/debug/validate_project.py /absolute/path/to/project --pretty`
3. **Changes the picture?** Render off and on with a scenario (the sprite needs a texture) and gate on numbers:

```json
{
  "scene_path": "scenes/player.tscn",
  "viewport_size": {"width": 256, "height": 144},
  "steps": [
    {"type": "wait_frames", "frames": 4},
    {"type": "screenshot", "path": "before.png", "expect": {"not_blank": true}},
    {"type": "set_property", "node_path": "Sprite2D", "property": "material:shader_parameter/flash_amount", "value": 1.0},
    {"type": "wait_frames", "frames": 2},
    {"type": "screenshot", "path": "after.png", "expect": {"not_blank": true, "max_diff_ratio": 0.999, "compare_to": "before.png"}}
  ],
  "assertions": []
}
```

Run: `uv run /absolute/path/to/godot/scripts/debug/run_scenario.py /absolute/path/to/project /absolute/path/to/project/scenario.json --pretty`. Read `dominant_colors`, `unique_colors`, `content_bbox`, `opaque_ratio` from the screenshot summary or `inspect_image`:

| Effect | Number that proves it |
| --- | --- |
| `flash` at 1.0 | the flash colour replaces the sprite colours in `dominant_colors` (the background stays first when the sprite is small), `content_bbox` unchanged |
| `grayscale` at 1.0 | every dominant colour has R == G == B |
| `outline_2d` | `content_bbox` grows by `outline_width` per side (only if the art has that much transparent margin: `process_image` `pad` with `width`/`height`) |
| `dissolve_2d` at 1.0 | `blank: true` |
| `palette_swap` | source colour gone, target present |
| `pixelate` | `unique_colors` drops sharply |
| `toon` | `dominant_colors` shows about `bands` shades + background (`unique_colors` stays in the hundreds from edge, specular and rim blending) |
| screen reader (`crt`, `vignette`) | frame differs from the same scene without the rect |
