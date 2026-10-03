# Pixel Art Without An Image Generator

Read this when a task needs 2D art (sprite, animation, tileset, UI panel, placeholder), pixel-art project settings, or when generated art must become real pixel art.

Two dispatcher ops, both on Godot's `Image` (no GPU, no import pass):

- `draw_image`: ASCII rows and shape primitives -> PNG.
- `process_image`: ordered pixel pipeline over one image, a list, or a directory (cleanup, packing, measuring).

Run `help '{"op":"draw_image"}'` / `help '{"op":"process_image"}'` for every shape and operation key.

## The loop

`draw_image` -> check `frame_reports[*].rows` (the written PNG read back one character per pixel, `legend` maps characters to hex) -> `inspect_image` with `expect` -> only then reference the file from a `.tscn`/`.tres`.

A fresh PNG has no `.import` sidecar (payload says `"needs_import": true`) and `load("res://...")` fails on it in a later process. Run `uv run /absolute/path/to/godot/scripts/import/import_project.py /absolute/path/to/project` before any scene or resource points at it. A theme texture reading back `null` means this was skipped.

## 1. Sprite as ASCII rows

One character = one pixel. `.` and space are always transparent. Every row must be the same length (ragged row is an error).

```bash
godot --headless --path /absolute/path/to/project \
  --script /absolute/path/to/godot/scripts/core/dispatcher.gd \
  draw_image '{
    "output_path": "art/hero.png",
    "palette": {"o": "#000000", "s": "#ffccaa", "h": "#ab5236", "b": "#29adff"},
    "rows": ["........", "....hhhh", "..hhhhhh", ".hhhhhhh", ".hssssss", ".hsossss",
             "..ssssss", "...sssss", "..bbbbbb", ".bbbbbbb", ".bsbbbbb", "..bbbbbb",
             "...bbbbb", "...bb...", "..ooo...", "........"],
    "mirror_x": true,
    "outline": "#000000"
  }'
```

8 authored columns, mirrored and outlined = 16x16.

- Sizes: 8x8 pickups/tiles, 16x16 characters/icons, 32x32 bosses. Silhouette first; three tones per material (base, shadow, highlight).
- `outline` never grows the canvas: leave a blank margin row/column or that side is clipped (`"outline_clipped": true`). In `process_image`, `trim` then `outline` clips; use `{"type": "trim", "padding": 1}`.
- `mirror_x`: `true`/`"append"` output `2w` (put the centre line in the last authored column); `"append_odd"` `2w-1` (last column is shared); `"fold"` keeps `w`. `mirror_y` likewise. Mirroring runs before outline/shadow.
- `"palette_name"`: `pico8`, `sweetie16`, `db16`, `db32`, `endesga32`, `c64`, `cga`, `gameboy`, `grayscale4`, `nes` bind characters `0-9` then `a-v` in order; an explicit `"palette"` is laid on top. Unknown name errors with the list.

## 2. Shapes: tiles, panels, placeholders

Without `rows`, `width` + `height` give a canvas and `shapes` draws in order (`rect`, `rect_outline`, `circle`, `ellipse`, `line`, `polygon`, `pixel`, `pixels`, `gradient_rect`, `checker`, `noise`, `text`). Colours: `"#rrggbb[aa]"`, `null` = transparent, or a palette character. `x`/`y` default 0, `width`/`height` default to the rest of the canvas. A shape fully off-canvas is an error. `noise` is seeded (reproducible). `text` is a built-in 3x5 uppercase font for placeholders only; real UI text is a `Label`.

### Tileset in one call

```bash
godot --headless --path /absolute/path/to/project \
  --script /absolute/path/to/godot/scripts/core/dispatcher.gd \
  draw_image '{
    "output_path": "art/tiles.png",
    "width": 16, "height": 16,
    "tile_check": true,
    "read_back": false,
    "frames": [
      {"name": "grass", "shapes": [
        {"type": "rect", "color": "#3e8948"},
        {"type": "noise", "colors": ["#63c74d", "#265c42"], "seed": 7, "density": 0.3}]},
      {"name": "stone", "shapes": [
        {"type": "rect", "color": "#8b9bb4"},
        {"type": "checker", "colors": ["#8b9bb4", "#5a6988"], "cell": 8}]}
    ]
  }'
```

Result is a 32x16 atlas; each frame reports `seamless: true/false`. `tile_check` is a heuristic (a left-to-right gradient fails, a checkerboard passes); the definitive check is `process_image` with `crop` then `{"type":"tile","columns":3,"rows":3}` and `"ascii_width"`.

Atlas coordinates = frame order: grass `(0,0)`, stone `(1,0)`. Wire to a level:

```bash
uv run /absolute/path/to/godot/scripts/import/import_project.py /absolute/path/to/project
godot --headless --path /absolute/path/to/project \
  --script /absolute/path/to/godot/scripts/core/dispatcher.gd \
  build_tileset '{"resource_path": "tilesets/world.tres", "tile_size": {"x": 16, "y": 16},
    "physics_layers": [{"collision_layer": 1, "collision_mask": 1}],
    "sources": [{"source_id": 0, "texture": "art/tiles.png", "tiles": "all",
                 "tile_defaults": {"collision": "full_cell"}}]}'
godot --headless --path /absolute/path/to/project \
  --script /absolute/path/to/godot/scripts/core/dispatcher.gd \
  paint_tilemap '{"scene_path": "scenes/level.tscn", "node_path": "root/Ground", "tile_set": "tilesets/world.tres",
    "ascii_map": {"legend": {"g": {"source_id": 0, "atlas_coords": {"x": 0, "y": 0}},
                             "s": {"source_id": 0, "atlas_coords": {"x": 1, "y": 0}}},
                  "rows": ["ssssss", "sggggs", "ssssss"]}}'
```

(`root/Ground` is a `TileMapLayer` added beforehand.) Verify with `inspect_tilemap '{"scene_path":"scenes/level.tscn","node_path":"root/Ground"}'`, which returns the painted level as ASCII rows plus a legend.

### 9-patch panel and pixel-art UI (the one place for this)

```bash
godot --headless --path /absolute/path/to/project \
  --script /absolute/path/to/godot/scripts/core/dispatcher.gd \
  draw_image '{"output_path": "art/panel.png", "width": 24, "height": 24, "read_back": false,
    "shapes": [{"type": "rect", "color": "#262b44"},
               {"type": "rect_outline", "x": 0, "y": 0, "width": 24, "height": 24, "color": "#5a6988", "thickness": 2},
               {"type": "rect_outline", "x": 2, "y": 2, "width": 20, "height": 20, "color": "#3a4466", "thickness": 1}]}'
godot --headless --path /absolute/path/to/project \
  --script /absolute/path/to/godot/scripts/core/dispatcher.gd \
  process_image '{"input_path": "art/panel.png", "output_path": "art/panel.png",
                  "operations": [{"type": "nine_patch_margins"}], "describe": false}'
```

`nine_patch_margins` returns `margins`, plus ready `stylebox_texture` / `nine_patch_rect` field sets (heuristic: longest run of identical columns/rows; a `note` appears when the centre is under half the image). Import, then theme it. The inline resource form needs its fields under `properties`; anything else in that object is ignored:

```bash
uv run /absolute/path/to/godot/scripts/import/import_project.py /absolute/path/to/project
godot --headless --path /absolute/path/to/project \
  --script /absolute/path/to/godot/scripts/core/dispatcher.gd \
  build_theme '{"resource_path": "theme/main.tres",
    "variations": {"OrnateFrame": {"base": "PanelContainer", "styleboxes": {"panel": {
      "__resource_type": "StyleBoxTexture",
      "properties": {"texture": {"__resource": "res://art/panel.png"},
        "texture_margin_left": 3, "texture_margin_top": 3, "texture_margin_right": 3, "texture_margin_bottom": 3,
        "content_margin_left": 12, "content_margin_right": 12, "content_margin_top": 10, "content_margin_bottom": 10}}}}}}'
```

Apply with `"theme_type_variation": "OrnateFrame"`. `texture_margin_*` is the non-stretching corner band, `content_margin_*` the inner padding (independent; keep content larger so text clears the ornament). `axis_stretch_*`: `0` stretch, `1` tile, `2` tile-fit. Check with `inspect_resource`: texture must read back as `{"__resource": "res://art/panel.png"}`. For a `NinePatchRect` node put the numbers in `patch_margin_*` via `configure_node`.

Pixel-art project settings (via `project_batch` `set_setting`), otherwise UI looks blurry next to crisp sprites:

- `rendering/textures/canvas_textures/default_texture_filter` = `0` (nearest)
- `display/window/stretch/mode` = `viewport`, `display/window/stretch/scale_mode` = `integer`
- `gui/theme/default_font_antialiasing`, `gui/theme/default_font_subpixel_positioning`, `gui/theme/default_font_hinting` = `0`

Use a real pixel font at native size or an exact integer multiple; 1-2px borders; `corner_radius` 0-2; spacing on the art's integer grid.

## 3. Frame sheet that `build_sprite_frames` reads

```bash
godot --headless --path /absolute/path/to/project \
  --script /absolute/path/to/godot/scripts/core/dispatcher.gd \
  draw_image '{
    "output_path": "art/hero_walk.png",
    "palette": {"o": "#000000", "s": "#ffccaa", "b": "#29adff"},
    "layout": "horizontal",
    "frames": [
      {"name": "walk_0", "rows": ["..ssss..", ".ssssss.", ".s.ss.s.", "..ssss..", "..bbbb..", ".b.bb.b.", "...bb...", "..o..o.."]},
      {"name": "walk_1", "rows": ["..ssss..", ".ssssss.", ".s.ss.s.", "..ssss..", "..bbbb..", ".b.bb.b.", "..b..b..", ".o....o."]}
    ]
  }'
```

Payload carries `"grid": {"cell_width": 8, "cell_height": 8, ...}` to pass straight on. All frames must be the same size. `layout`: `horizontal` (default), `vertical`, `grid` (with `columns`); `separation` adds gutters. After `import_project.py`, with an `AnimatedSprite2D` node already in the scene:

```bash
godot --headless --path /absolute/path/to/project \
  --script /absolute/path/to/godot/scripts/core/dispatcher.gd \
  build_sprite_frames '{"scene_path": "scenes/hero.tscn", "node_path": "root/Sprite",
    "spritesheet": "art/hero_walk.png", "grid": {"cell_width": 8, "cell_height": 8},
    "animations": [{"name": "walk", "fps": 8, "loop": true, "frames": [{"row": 0, "cols": [0, 1]}]}],
    "resource_save_path": "art/hero_walk_frames.tres"}'
```

Separate frame files (generator output): `process_image` with `{"type": "pack_frames"}` over a directory returns the same `grid`; `{"type": "split_sheet"}` goes back.

## 4. Cleaning art that came from an image generator

Generated "pixel art" has soft edges, off-grid pixels, thousands of colours and a baked background. `process_image` fixes all four, in this order (this is also the replacement for chroma-key cutout):

```bash
godot --headless --path /absolute/path/to/project \
  --script /absolute/path/to/godot/scripts/core/dispatcher.gd \
  process_image '{
    "input_path": "art/raw_hero.png", "output_path": "art/hero.png",
    "operations": [
      {"type": "remove_background", "tolerance": 0.1},
      {"type": "trim", "padding": 1},
      {"type": "pixelate", "target_width": 32, "mode": "average", "palette_name": "pico8", "alpha_threshold": 0.5}
    ]
  }'
```

- `remove_background` floods inward from the four corners, so a background colour also present inside the subject survives. For a flat chroma key name it: `{"type":"remove_background","color":"#00ff00","tolerance":0.05}` (clears every matching pixel, enclosed or not); `replace_color` with `"to": null` also works. Fails loudly if it changes nothing.
- `trim` crops to content (`"color"` trims a flat colour instead of transparency).
- `pixelate` makes it real art: `target_width` or `cell_size` sets the grid; `mode` `average` (soft art), `mode` (already blocky), `nearest`. `palette_name`/`palette`/`max_colors` fix the palette; `alpha_threshold` makes pixels fully opaque or transparent, which kills the soft halo.
- Optional: `{"type":"pad","multiple":16}` to land on a tile grid, `{"type":"outline"}`.

Gate it: `inspect_image '{"image_path":"art/hero.png","expect":{"not_blank":true,"has_alpha":true,"max_unique_colors":16,"width":32}}'`. `max_unique_colors` catches fake pixel art (a soft blob reports hundreds).

## 5. Verifying without eyes

| question | how |
| --- | --- |
| Did the file get what I typed? | `frame_reports[*].rows` + `legend` |
| Size / palette / not empty? | `inspect_image` `expect` (`width`, `height`, `max_unique_colors`, `not_blank`, `has_alpha`) |
| Changed since last time? | `inspect_image` `compare_to` + `expect.max_diff_ratio` |
| Do sheet frames line up? | `inspect_image` `image_paths` + `expect.frames_consistent` |
| Looks right in game? | scenario runner `screenshot` step (`references/debugging.md`) |

The ASCII in `summary`/`ascii` is a luminance ramp for shape only; use the character read-back for exact pixels.
