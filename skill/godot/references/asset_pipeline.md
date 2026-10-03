# Asset Pipeline

Read when the task involves art: generating it, cutting out a background, cleaning it up, or turning frames into animation. Pixel-art drawing and `process_image` pixel cleanup (pixelate, quantize, alpha threshold): `references/pixel_art.md`.

## Which path

| Situation | Do |
| --- | --- |
| Pixel art (sprites, tiles, UI, icons, placeholders) | `draw_image` (text-verifiable): `references/pixel_art.md` |
| Generated art looks soft, off-grid, or has a baked background | `process_image` (`remove_background`, `trim`, `pixelate`, `quantize`): `references/pixel_art.md` |
| Painterly / high-res art from an image generator | `imagegen`, then the cutout and frame rules below |
| Pack frames to a sheet / slice a sheet | `process_image` `pack_frames` (returns the `grid` that `build_sprite_frames` takes) / `split_sheet` |

`draw_image` / `process_image` write raw files with NO `.import` sidecar (`"needs_import": true`). Run `uv run /absolute/path/to/godot/scripts/import/import_project.py /absolute/path/to/project` before any `.tscn`/`.tres` references them.

## Generated art: transparent first

1. Request PNG with `background=transparent` for anything that becomes a sprite, prop, UI element or frame.
2. If transparency is unavailable or the edges are poor, request a FLAT background and strip it with `process_image` `remove_background`.
   - Background: pure green `#00FF00`; pure magenta `#FF00FF` if the subject has strong green. Never white, black, gradients, scenes, or shadows.
   - Identical flat colour across every frame of a sequence; pass the explicit hex.
3. Prompt: subject centered and fully in frame, clean hard silhouette, no text/watermark/motion blur/bloom/depth of field, no cast shadow unless wanted.

```bash
godot --headless --path /absolute/path/to/project \
  --script /absolute/path/to/godot/scripts/core/dispatcher.gd \
  process_image '{"input_paths":["source/hero_idle_raw"],"output_dir":"textures/hero_idle_cutout",
    "operations":[{"type":"remove_background","color":"#00ff00","tolerance":0.08},
                  {"type":"trim","padding":1},{"type":"pad","multiple":16}]}'
```

`remove_background` floods inward from the corners, so the same colour inside the subject survives. `replace_color` with `"to": null` clears every matching pixel including enclosed ones (blunter).

## Frame animation

- Deliver independent frame files, not a sheet (sheet only on request, or when `pack_frames` is cheaper than many files).
- Same canvas size, camera distance, view angle and anchor across frames; change only pose/timing; regenerate or repair frames individually.
- Unequal sizes cannot be packed: `{"type":"pad","multiple":16}` (or explicit `width`/`height`) first.
- Verify before building: `inspect_image '{"image_paths":["textures/hero_idle_cutout"],"expect":{"frames_consistent":true,"not_blank":true,"has_alpha":true}}'` fails on an empty frame, lost transparency, or changed canvas size.
- Import, then build `SpriteFrames` (`frames_dir` is naturally sorted; use explicit `frame_paths` when order matters or inputs are resources):

```bash
godot --headless --path /absolute/path/to/project \
  --script /absolute/path/to/godot/scripts/core/dispatcher.gd \
  build_sprite_frames '{"scene_path":"scenes/player.tscn","node_path":"root/AnimatedSprite2D",
    "frames_dir":"textures/hero_idle_cutout","animation_name":"idle","fps":12,"loop":true,
    "resource_save_path":"animations/hero_idle_frames.tres"}'
```
