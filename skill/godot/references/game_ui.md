# Game UI (Control Layout, Theme, Feel)

Read this for menus, HUDs, dialogs, inventories, or any `Control` work. Build with `scene_batch` / `configure_control` / `build_theme` / `project_batch` (parameters: `references/automation_api.md` or `help '{"op":"<name>"}'`).

## Layout doctrine: containers, not coordinates

**The #1 cause of broken game UI is stacked siblings under a bare `Control`.** A bare `Control`, `Node2D` or `CanvasLayer` does not lay out children. Every `Control` child defaults to `TOP_LEFT` with zero offsets, so ten siblings all sit at `(0, 0)` and paint on top of each other. Hand-assigning `position` appears to fix it, then breaks on the next resolution, font or translation. Containers are the only nodes that position children: build the spine from containers, then hang leaves off it.

```
Root Control            layout_preset FULL_RECT      <- preset lives here
└── MarginContainer     screen-edge breathing room
    └── VBoxContainer   the page rhythm
        ├── Label
        ├── CenterContainer   size_flags_vertical EXPAND_FILL
        │   └── VBoxContainer custom_minimum_size.x = 320
        │       └── Button x N
        └── Label
```

- `layout_preset` **only at a container boundary** (scene root, direct `Control` child of a `CanvasLayer`, full-screen overlay), never on a node already inside a `Container`.
- **Never set `position`, `size` or `offset_*` on a child of a `Container`** (overwritten on every sort). Use `custom_minimum_size`, size flags, `stretch_ratio`.
- Containers: `VBox`/`HBox` rows and columns, `GridContainer` (`columns: 2`) label/widget pairs, `PanelContainer` stylebox that fits its content, `MarginContainer` insets, `CenterContainer` for content of unknown size (prefer it over `PRESET_CENTER`, which freezes the authoring-time size), `ScrollContainer` for overflow.
- Size flags: `EXPAND_FILL` on the one child that absorbs free space; `SHRINK_CENTER`/`SHRINK_BEGIN` for buttons/icons that must not stretch; `custom_minimum_size` floors. Use a container's `alignment` (`0` begin, `1` center, `2` end), not spacer nodes.
- Separation and margins are theme constants: set them in the Theme, override per node via `configure_control.theme_overrides.constants`. Stay on one spacing unit (multiples of 4/8).
- **Anchored controls must grow inward.** A min-size control anchored to a right/bottom edge grows outward (`grow_*` defaults to `END`) and slides off-screen. `TOP_RIGHT`: `grow_horizontal` 0; `BOTTOM_LEFT`/`BOTTOM_WIDE`: `grow_vertical` 0; `CENTER_BOTTOM`: `grow_horizontal` 2, `grow_vertical` 0. `configure_control` has no key for these: set them in `add_node.properties` / `configure_node.properties`.
- **Overlays must not eat input.** `mouse_filter` defaults to `STOP`: set `"mouse_filter": 2` on backgrounds, HUD containers and decorative labels.
- HUDs, pause menus and dialogs go on a `CanvasLayer` (ignores camera; `layer` orders them). A pause menu also needs `"process_mode": 2` (WHEN_PAUSED) or its buttons are dead while `get_tree().paused`.

### One compact pattern (title screen; HUD/pause/dialog use the same spine)

```bash
godot --headless --path /absolute/path/to/project \
  --script /absolute/path/to/godot/scripts/core/dispatcher.gd \
  scene_batch '{
    "scene_path": "scenes/title.tscn", "create_if_missing": true,
    "root_node_type": "Control", "root_node_name": "TitleScreen",
    "actions": [
      {"type":"configure_control","node_path":"root","layout_preset":"FULL_RECT"},
      {"type":"add_node","parent_node_path":"root","node_type":"ColorRect","node_name":"Background",
       "properties":{"color":{"__type":"Color","r":0.05,"g":0.063,"b":0.09,"a":1},"mouse_filter":2}},
      {"type":"configure_control","node_path":"root/Background","layout_preset":"FULL_RECT"},
      {"type":"add_node","parent_node_path":"root","node_type":"MarginContainer","node_name":"Frame"},
      {"type":"configure_control","node_path":"root/Frame","layout_preset":"FULL_RECT",
       "theme_overrides":{"constants":{"margin_left":48,"margin_right":48,"margin_top":40,"margin_bottom":32}}},
      {"type":"add_node","parent_node_path":"root/Frame","node_type":"VBoxContainer","node_name":"Column"},
      {"type":"add_node","parent_node_path":"root/Frame/Column","node_type":"Label","node_name":"Title",
       "properties":{"text":"EMBER HOLLOW","horizontal_alignment":1,"theme_type_variation":"TitleLabel"}},
      {"type":"add_node","parent_node_path":"root/Frame/Column","node_type":"CenterContainer","node_name":"MenuSlot"},
      {"type":"configure_control","node_path":"root/Frame/Column/MenuSlot","size_flags_vertical":"EXPAND_FILL"},
      {"type":"add_node","parent_node_path":"root/Frame/Column/MenuSlot","node_type":"VBoxContainer","node_name":"Menu"},
      {"type":"configure_control","node_path":"root/Frame/Column/MenuSlot/Menu",
       "custom_minimum_size":{"__type":"Vector2","x":320,"y":0},"theme_overrides":{"constants":{"separation":12}}},
      {"type":"add_node","parent_node_path":"root/Frame/Column/MenuSlot/Menu","node_type":"Button","node_name":"NewGame","properties":{"text":"New Game"}},
      {"type":"add_node","parent_node_path":"root/Frame/Column/MenuSlot/Menu","node_type":"Button","node_name":"Quit","properties":{"text":"Quit"}}
    ]
  }'
```

The `EXPAND_FILL` `CenterContainer` pins the menu to the centre while the title stays on top. Variants:

- **HUD**: `CanvasLayer` root with one `MarginContainer` per corner/edge cluster (`layout_preset` `TOP_LEFT`/`TOP_RIGHT`/`CENTER_BOTTOM`, `mouse_filter` 2, grow direction inward, margins via theme constants), a `Box` inside each. Drive numbers from a script on the root; never rebuild the scene at runtime. Add it to levels with `instantiate_scene`.
- **Pause / settings**: `CanvasLayer` (`layer` 10, `process_mode` 2) > `ColorRect` dim (`FULL_RECT`) + `CenterContainer` (`FULL_RECT`) > `PanelContainer` (`custom_minimum_size.x` floor) > `VBoxContainer` > heading, `HSeparator`, `GridContainer` of label + `HSlider` (`EXPAND_FILL`, `SHRINK_CENTER`), action `HBoxContainer`. Hide with root `visible` alongside `get_tree().paused`.
- **Dialog box**: `CanvasLayer` > `MarginContainer` `BOTTOM_WIDE` (`grow_vertical` 0) > `PanelContainer` > `VBoxContainer` > speaker `Label`, `RichTextLabel` (`bbcode_enabled`, `fit_content`, `scroll_active` false, `autowrap_mode` 3, `custom_minimum_size.y` so the panel does not jump between lines), footer `HBoxContainer` `alignment` 2. Type-on effect: tween `visible_ratio`, never rewrite `text`.

## Theme: never ship the default gray

An unthemed project renders the engine's editor-gray fallback, which is what makes agent UI read as a web form. Build the theme before wiring behavior, with the `build_theme` op (types, styleboxes, colors, constants, font_sizes, variations; it merges into an existing theme; `help '{"op":"build_theme"}'` prints the schema and an example). Assign every color a role (`background`, `panel`, `border`, `accent`, `text`, `muted`, `danger`) and use only those.

```bash
godot --headless --path /absolute/path/to/project \
  --script /absolute/path/to/godot/scripts/core/dispatcher.gd \
  build_theme '{
    "resource_path": "theme/game.tres", "default_font_size": 18,
    "types": {
      "Button": {
        "styleboxes": {
          "normal":  {"bg_color":"#222d42","border_width":2,"border_color":"#3a4a68","corner_radius":4,"content_margin_left":22,"content_margin_right":22,"content_margin_top":10,"content_margin_bottom":10},
          "hover":   {"bg_color":"#2e3c58","border_width":2,"border_color":"#5c76a3","corner_radius":4,"content_margin_left":22,"content_margin_right":22,"content_margin_top":10,"content_margin_bottom":10},
          "pressed": {"bg_color":"#161d2b","border_width":2,"border_color":"#ffb02e","corner_radius":4,"content_margin_left":22,"content_margin_right":22,"content_margin_top":12,"content_margin_bottom":8},
          "disabled":{"bg_color":"#161b26","border_width":2,"border_color":"#262f40","corner_radius":4,"content_margin_left":22,"content_margin_right":22,"content_margin_top":10,"content_margin_bottom":10},
          "focus":   {"draw_center":false,"border_width":2,"border_color":"#ffd479","corner_radius":4,"expand_margin":3}
        },
        "colors": {"font_color":"#e9eff8","font_hover_color":"#ffffff","font_pressed_color":"#ffb02e","font_disabled_color":"#5b6478"}
      },
      "PanelContainer": {"styleboxes": {"panel": {"bg_color":"#182031ee","border_width":2,"border_color":"#3a4a68","corner_radius":6,"content_margin_left":20,"content_margin_right":20,"content_margin_top":16,"content_margin_bottom":16}}}
    },
    "variations": {
      "TitleLabel": {"base":"Label","colors":{"font_color":"#ffb02e"},"font_sizes":{"font_size":56}},
      "CaptionLabel": {"base":"Label","colors":{"font_color":"#8a9bb5"},"font_sizes":{"font_size":13}}
    }
  }'
godot --headless --path /absolute/path/to/project \
  --script /absolute/path/to/godot/scripts/core/dispatcher.gd \
  project_batch '{"backup_path":"project.godot.bak","actions":[{"type":"set_setting","name":"gui/theme/custom","value":"res://theme/game.tres"}]}'
```

Must-haves:

- **`gui/theme/custom` must be set** (second call) or nothing inherits the theme. Assign a per-scene `theme` only for a genuinely separate visual context.
- **Give every button state its own stylebox**; `normal` alone makes a button that never reacts.
- **`focus` must be visible**: `draw_center: false` + accent border + positive `expand_margin`. Gamepad/keyboard players cannot navigate an invisible focus; never `"focus": "empty"`.
- Name variations by intent (`TitleLabel`, `DangerButton`) and apply with the `theme_type_variation` property.
- **Theme item names are not validated**: a typo silently falls back to gray. Check names against the control's theme items (`api_lookup.py Button --kind theme_items`) or load the saved theme and assert `Theme.has_stylebox(name, type)`.
- Fonts: the default font is the loudest "engine default" signal. Set `"default_font": {"__resource":"res://fonts/x.ttf"}` (or a `FontVariation` via `"fonts"` in a variation). A font *file* must be imported first (`godot --headless --import --path /absolute/path/to/project`). With no font file, a `SystemFont` made by `resource_batch` (`"resource_type":"SystemFont"`, `set_properties` `{"font_names":["Sans-Serif"]}`) needs no import. Use at most two families.
- Ornate 9-patch frames, panel margins, and pixel-art UI settings (nearest filter, integer scaling, font AA/hinting off): `references/pixel_art.md`.

## Game feel

- Animating `scale` on a control **inside a Container** fights the container (it resets the transform on every sort). Tween `offset_transform_scale` instead (visual only, pivots from the centre; set `offset_transform_enabled = true`). Hook both `mouse_entered` and `focus_entered` so menus feel alive on a gamepad. Keep transitions 0.10-0.25 s.
- UI sounds: route to a `UI` bus (`setup_audio_buses`, see `references/audio.md`), short and distinct per action (hover, press, confirm, cancel, error).

## Text robustness

- A `Label` with `autowrap_mode: 0` in a narrow container reports a huge minimum width and stretches its parent off-screen. Wrap body text (`autowrap_mode` 3 word-smart); `text_overrun_behavior` 3 for ellipsis.
- Do not size panels from measured English: set a `custom_minimum_size` floor from the longest expected text (translations run 30-40% longer; CJK needs more line height) and let the container grow.

## Verify the layout

`scene_batch` exiting 0 means the scene serialized, not that anything is visible or placed. Run the scene and read the layout as text: the `ui_report` scenario step walks the live tree after the containers sort, reports each control's post-layout rect, and fails the run on `zero_size`, `offscreen` and `overlap`. It catches the stacked-at-(0,0) failure and needs no window (fields: `references/automation_api.md`).

```json
{
  "scene_path": "scenes/title.tscn",
  "viewport_size": {"width": 1280, "height": 720},
  "settle_frames": 4,
  "steps": [
    {"type": "ui_report", "label": "title", "fail_on": ["any"]},
    {"type": "assert", "assertion": "property", "node_path": "Frame/Column/MenuSlot/Menu/NewGame",
     "property": "size:x", "expected": 320.0, "operator": "approx", "tolerance": 1.0}
  ],
  "assertions": [{"assertion": "node_exists", "node_path": "Frame/Column/MenuSlot/Menu/Quit"}]
}
```

```bash
uv run /absolute/path/to/godot/scripts/debug/run_scenario.py \
  /absolute/path/to/project /absolute/path/to/project/ui_scenario.json --headless --pretty
```

- Assert the numbers that matter (a column's width, a footer's position) so regressions fail loudly.
- Re-run at a second size (e.g. 1920x1080 and 1280x720), and read `ui_reports[0].viewport` to confirm it took: `viewport_size` only applies with `stretch/mode` `disabled`; under `canvas_items` / `viewport` (every scaffold preset) layout uses the base `display/window/size/viewport_*`, so change those via `project_batch` instead. Add a `{"type":"screenshot","path":"/absolute/output/title.png"}` step and look at it when the host can view images (the runner switches to a rendered window when a screenshot step is present, unless `--headless`).
- Finish with `scripts/debug/validate_project.py`.

Done means `findings: 0` at two resolutions, a theme applied, and a visible focus state.
