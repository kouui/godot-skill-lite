# Playbooks — Recipes For Common Game Features

Read this when the task matches a common shape (player, enemy, tile level, UI, dialogue, save/load, audio, inventory, settings, 3D starter, a whole small game). Each playbook says what to copy, which dispatcher ops build it, the wiring traps, and how to verify. Adapt names, sizes and paths to the user's game; do not replay earlier playbooks unless the game lacks what they set up.

Every `templates/gdscript/*.gd` header lists its exact scene tree, autoloads, input actions and a runnable `attach_script` call. Copy a template instead of writing a controller from memory, then read its header. `ls templates/gdscript/` lists them.

## Route By Task

| Task | Playbook | Verify with |
| --- | --- | --- |
| New project | `scripts/project/scaffold_project.py` (see Standard Setup) | `validate.counts` errors and warnings 0 |
| Player that runs/jumps or walks 8-way | [1](#1-2d-player-controllers) | `run_scenario.py` `"ok": true` |
| Enemy, damage, pooled bullets | [2](#2-enemy-damage-and-pooled-bullets) | `run_scenario.py` `"ok": true` |
| Tile level from ASCII | [3](#3-tile-level-from-ascii) | `inspect_tilemap` rows match the map |
| Menu, HUD, pause | [4](#4-ui-theme-menu-hud-pause) | `ui_report` `findings=0` |
| NPC dialogue, branching choices | [5](#5-dialogue-box-and-branching-dialogue) | scenario reads the typed text |
| Save and load | [6](#6-save-and-load) | log shows save/load round trip |
| SFX, music, fades | [7](#7-audio-buses-audiomanager-scene-transitions) | `inspect_audio` `expect` exit 0 |
| Art with no image generator | [8](#8-art-from-text) | `inspect_image` `expect` |
| Items, inventory, loot | [9](#9-inventory-items-and-loot) | `run_tests.py` ok + scenario |
| Volume, fullscreen, key rebinding | [10](#10-settings-menu) | settings survive a second process |
| First-person or third-person 3D | [11](#11-3d-starter) | `spatial_report` clean + scenario |
| A whole small game | [12](#12-whole-game-assembly-checklists) | the core-loop scenario wins |
| Is it finished | [13](#13-before-you-call-it-done) | five commands exit 0 |
| Forgot an op's parameters | — | `dispatcher.gd help '{"op":"add_node","format":"text"}'` |

Dispatcher call used throughout (JSON below is the argument):

```bash
godot --headless --path /absolute/path/to/project \
  --script /absolute/path/to/godot/scripts/core/dispatcher.gd <op> '<json>'
```

An unknown key is rejected with the nearest valid name and nothing is written, so a typo fails loudly. Scenario input rules (`action` vs `key` steps, `physical_keycode` vs `keycode`, `wait_seconds` instead of `wait_frames` for physics) live in `references/automation_api.md`. Two extra rules from here: menus, pause, dialog advance and `interact` live in `_unhandled_input`, so they need `key` steps; and `key` has no `release_after`, send an explicit `"pressed": false`.

---

## Standard Setup (referenced by every playbook)

1. **Project.** `uv run /absolute/path/to/godot/scripts/project/scaffold_project.py /absolute/path/to/project --preset pixel2d|hd2d|3d|ui [--with-tests]` creates folders, window/stretch settings, the input map (`move_*`, `jump`, `interact`), physics layer names and `GameManager`, `SaveManager`, `SceneTransition` autoloads. For an existing project skip it. Never hand-edit `project.godot` after the dispatcher has touched it: `ProjectSettings.save()` rewrites the whole file. Use `project_batch` (`set_setting`, `add_input_action` + `add_input_event`, `add_autoload`, `set_main_scene`, `set_layer_name`).
2. **Copy templates** into `res://scripts/` with `cp`.
3. **Rebuild the class cache** after copying any template with a `class_name` (`health`, `hitbox`, `hurtbox`, `object_pool`, `state`, `state_machine`, `interactable`, `projectile`, `damage_number`, `grid_movement`, `item_data`, `inventory`, `loot_table`, `dialogue_runner`, `choice_list`): `godot --headless --path /absolute/path/to/project --import`. A `--script` dispatcher run never rebuilds `.godot/global_script_class_cache.cfg`; skipping it gives `Identifier "Health" not declared in the current scope`.
4. **Autoloads before scripts that name them.** A script that names an unregistered autoload fails to parse (`attach_script` aborts with `Property does not exist at attach_script.script_properties`). Register with `add_autoload` (`{"type":"add_autoload","autoload_name":"AudioManager","path":"scripts/audio_manager.gd"}`; `singleton` defaults to true); the scaffold already registers `GameManager`, `SaveManager`, `SceneTransition`, so `AudioManager` and `Settings` are yours to add. Never give an autoload script a `class_name` equal to its autoload name.
5. **Audio buses before `AudioManager` or `Settings`.** Players assigned to a missing bus silently route to Master and the validation run carries a warning:

```json
setup_audio_buses {"buses":[{"name":"Master","volume_db":0.0},{"name":"Music","send":"Master","volume_db":-6.0},{"name":"SFX","send":"Master","volume_db":-3.0},{"name":"UI","send":"SFX","volume_db":-4.0}],"save_path":"audio/default_bus_layout.tres","set_project_setting":true}
```

6. **Physics layers convention** used by the templates: 1 `world`, 2 `player`, 3 `enemy`, 4 `hitbox/damage` (bitmask value 8); `collision_layer`/`collision_mask` take bitmask values (1, 2, 4, 8, 16). The scaffold names layers 3-6 `enemies`, `pickups`, `hitboxes`, `hurtboxes`; rename with `project_batch` `set_layer_name` (`layer_type` `2d_physics`) so reports match the convention. A pickup sits on its own layer and scans the player layer; a mask bit nothing occupies is a query that can never hit (`mask_targets_empty_layer` in `validate_project.py`).

Wiring facts that apply everywhere:

- Node names are part of the contract: a template reading `$Sprite2D` finds nothing if the node is called anything else.
- `unique_name_in_owner` (`%Name`) resolves only within the scene that owns the node; it does not cross `instantiate_scene`. A level script reaches `$Player/Camera2D`, not `%Camera2D`.
- Inline resources put their fields under `properties`: `{"__resource_type":"CapsuleShape2D","properties":{...}}`. External: `{"__resource":"res://x.png"}`. Typed values: `{"__type":"Vector2","x":0,"y":1}`, `{"__type":"NodePath","value":"../Health"}`. A node reference cannot be passed headless (`expected Health but got NodePath`); pass a NodePath.
- `connect_signal` verifies the target method exists, so attach the target's script before connecting.
- Never rewrite a template's `@onready var x: Type = $Path` as `var x := $Path`.
- Never set `position`/`size`/`offset_*` on a child of a `Container`; set `layout_preset` only on the root and full-rect backgrounds. Violating this stacks every control at (0, 0).

---

## 1. 2D Player Controllers

Templates: `player_platformer_2d.gd` (gravity from project settings, coyote time, jump buffer, short hop), `player_topdown_2d.gd` (8-way, remembered facing; uses `Input.get_vector`, so the diagonal is normalised, do not sum axes yourself), optional `camera_shake_2d.gd` (`add_trauma()`).

Build (platformer; for top-down use `CircleShape2D`, no gravity, `move_up`/`move_down` actions):

```json
scene_batch {"scene_path":"scenes/player.tscn","create_if_missing":true,"root_node_type":"CharacterBody2D","root_node_name":"Player","actions":[
 {"type":"configure_node","node_path":"root","properties":{"collision_layer":2,"collision_mask":1},"groups_add":["player"]},
 {"type":"add_node","parent_node_path":"root","node_type":"CollisionShape2D","node_name":"CollisionShape2D","properties":{"shape":{"__resource_type":"CapsuleShape2D","properties":{"radius":6.0,"height":20.0}}}},
 {"type":"add_node","parent_node_path":"root","node_type":"Sprite2D","node_name":"Sprite2D"},
 {"type":"add_node","parent_node_path":"root","node_type":"Camera2D","node_name":"Camera2D","properties":{"position_smoothing_enabled":true}},
 {"type":"attach_script","node_path":"root","script_path":"scripts/player_platformer_2d.gd","script_properties":{"speed":220.0,"jump_velocity":-380.0}},
 {"type":"attach_script","node_path":"root/Camera2D","script_path":"scripts/camera_shake_2d.gd"}]}
```

Then a level scene with a `StaticBody2D` ground (`collision_layer` 1, a `RectangleShape2D`) and `instantiate_scene` of `scenes/player.tscn`; `set_main_scene` to it. Put the `player` group on the body: `interactable.gd`, `kill_zone.gd` and enemy chasers look players up by group.

Traps:
- A level with no collider under the player: the player falls forever with no error.
- Camera inside the player scene means the player scene alone is runnable; keep it there.

Verify: scenario with `wait_seconds` 1.0 then assert `Player` `position:y` is at rest; `action` `move_right` held 0.4 s then assert `position:x` increased; `action` `jump` with `release_after` then assert `velocity:y` < 0. `assertions`: `node_exists` `Player/Camera2D`. Run `validate_project.py` (0 errors, 0 warnings) and `run_scenario.py` `"ok": true`.

---

## 2. Enemy, Damage And Pooled Bullets

Templates: `health.gd` (`damaged`/`died`, invulnerability window), `hitbox.gd` (Area2D that deals damage, team-filtered), `hurtbox.gd` (Area2D feeding a `Health`), `enemy_patrol_2d.gd` (waypoint/raycast patrol with edge and wall checks), `object_pool.gd`, `projectile.gd`, `shooter.gd`, `damage_number.gd`, `enemy_chase_nav_2d.gd` (NavigationAgent2D chaser with straight-line fallback), `state.gd` + `state_machine.gd`. Run the class-cache `--import` after copying.

Enemy tree (root `CharacterBody2D` `collision_layer` 4 = layer 3 `enemy`, `collision_mask` 1, group `enemy`): `CollisionShape2D`, `Sprite2D`, `RayCast2D EdgeCheck` (target down), `RayCast2D WallCheck` (target forward), `Node Health` (health.gd), `Area2D Hurtbox` (hurtbox.gd, `team:"enemy"`, `health_path` = `{"__type":"NodePath","value":"../Health"}`). The bullet is an `Area2D` with `hitbox.gd` (`team:"player"`, `one_shot:true`).

Traps:
- **Hitbox/hurtbox layers.** `hitbox.gd` finds a Hurtbox through `area_entered`, so the bullet hitbox and enemy hurtbox must sit on the same bit and both scan it (damage layer 4, value 8). "Bullets never hit anything" is this, check it first.
- Pool bullets with `object_pool.gd` (`scene` = `{"__resource":"res://scenes/bullet.tscn"}`, `initial_size`, `grow`). `projectile.gd` rearms its hitbox on every `launch()`; that is what makes a pooled `one_shot` bullet work more than once. Retiring a bullet inside the `area_entered` callback needs deferred `monitoring`/free calls (the template does it); a hand-written bullet gives `Function blocked during in/out signal` or `Removing a CollisionObject node during a physics callback`, found only by `smoke_scenes.py --fuzz`.
- `StateMachine` keys states by child **node name**: `request_transition(&"Chase")` needs a child literally named `Chase` whose script `extends State`. Write and attach the state scripts first, or it pushes `has no State children` on frame 1.
- Chaser navigation: bake the polygon with `resource_batch` (`resource_type:"NavigationPolygon"`, action `bake_navmesh` with `traversable_outlines` / `obstruction_outlines`, saved to e.g. `nav/arena.tres`) and set it on a `NavigationRegion2D`; the bake is synchronous and takes plain outlines because the headless dummy renderer cannot read mesh geometry. `agent_radius` at least the enemy radius, or chasers clip corners. The chaser prints nothing: it emits `navigation_mode_changed(bool)` (true = navmesh live); connect it to a handler that prints a line. With no navmesh it walks straight.
- `shooter.gd` `aim_mode` 1 asks its parent for `facing()` (provided by `player_topdown_2d.gd`), 0 aims at the mouse. If its `../../` paths do not resolve it instantiates bullets into the current scene (fine for a lone player scene).
- Damage numbers: `add_child()` before `popup()`; `create_tween()` fails on a node outside the tree.

Verify: scenario asserting `Enemy/Health` `current_health` equals max, waiting 0.5 s, asserting `Enemy` `velocity:x` not equal 0 (patrolling), `node_exists` for `Enemy/Hurtbox` and the pool. Then `validate_project.py`, whose `static.physics_layers.findings` must hold no `mask_targets_empty_layer`.

---

## 3. Tile Level From ASCII

Flow: art, import, `build_tileset`, `TileMapLayer`, `paint_tilemap`, `inspect_tilemap`.

```json
build_tileset {"resource_path":"tilesets/world.tres","tile_size":{"x":16,"y":16},"physics_layers":[{"collision_layer":1,"collision_mask":0}],"sources":[{"source_id":0,"texture":"art/tiles.png","tiles":"all","tile_defaults":{"collision":"full_cell"}}]}
paint_tilemap {"scene_path":"scenes/level.tscn","node_path":"root/Ground","tile_set":"tilesets/world.tres","clear":true,"ascii_map":{"origin":{"x":0,"y":0},"legend":{"#":{"source_id":0,"atlas_coords":{"x":0,"y":0}},".":null},"rows":["....","####"]}}
```

Traps:
- **Import the PNG first** (`uv run .../scripts/import/import_project.py /absolute/path/to/project`). Without it `build_tileset` still saves, but the `.tres` then fails to load (`No loader found for resource`) and `paint_tilemap` says `must resolve to a TileSet resource`.
- Without `"collision": "full_cell"` the tiles are decoration and the player falls through. Tile physics layer `collision_mask` 0: a floor occupies a layer, it does not scan.
- Use `TileMapLayer`, never `TileMap`. A scene cannot hold a `StaticBody2D Ground` and a `TileMapLayer Ground` (`Node is not a TileMapLayer`): rename or delete.
- Atlas coordinates are frame order of a `draw_image` sheet (grass `(0,0)`, dirt `(1,0)`, ...). Legend `null` = empty cell; one character per cell, equal-length rows, no tabs. A painted decorative tile in the walking lane is a wall.

Verify: `inspect_tilemap {"scene_path":...,"node_path":"root/Ground"}` (`"format":"text"` prints only the rows; the default JSON also has `bounds`, `counts`, `cell_count`). It reports what is **painted**: `bounds` covers only painted cells (all-empty top rows are dropped, so add their count to the printed row index); glyphs are re-derived, so compare shapes and the `counts` map, not your legend characters; `cell_count` is the total. All-dot rows mean atlas coordinates were not exposed (`"tiles":"all"`). Then `validate_project.py` ok.

---

## 4. UI: Theme, Menu, HUD, Pause

Templates: `main_menu.gd`, `hud.gd`, `pause_menu.gd` (plus `game_manager.gd`, `scene_transition.gd`, `save_manager.gd` autoloads: Standard Setup). Theme and layout rules: `references/game_ui.md` (use its Theme Doctrine through `build_theme`, then `set_setting gui/theme/custom`).

Build each screen with `scene_batch` containers first (`MarginContainer` > `VBoxContainer`/`CenterContainer` > controls), `layout_preset` on the root only, and `unique_name_in_owner` on every node the script reads with `%`.

- **Main menu**: Control root, `FULL_RECT` background `ColorRect` with `mouse_filter` 2, buttons named `NewGameButton`, `ContinueButton`, `SettingsButton`, `QuitButton`, all unique. Script props `new_game_scene`, `save_slot`. Needs `GameManager`, `SaveManager`, `SceneTransition` registered or it will not parse.
- **HUD**: `CanvasLayer` layer 1 with `TopLeft` (`HealthBar` ProgressBar, `LivesLabel`) and `TopRight` (`ScoreLabel`), all `%` nodes unique, `mouse_filter` 2. A cluster anchored to the right edge must grow inward: `configure_node` `properties` `{"grow_horizontal":0}` on the `TOP_RIGHT` node, or it grows off-screen (`ui_report` `offscreen`). Bind health from the level script (`%HUD.bind_health($Player/Health)`); `hud.gd` deliberately does not search for a Health node. Labels update from `GameManager` signals, never `_process`.
- **Pause menu**: `CanvasLayer` layer 10, **`process_mode` 3 (ALWAYS) on the root in the scene** as well as in the script, otherwise it freezes with the tree it just paused. `PauseRoot` (unique, hidden by default) holds `Dim`, `Center/Dialog/Body` with `ResumeButton`, `QuitButton`. Script prop `title_scene`. Instantiate it in the level.

Traps:
- Size type to the project's **base viewport**, not the window (a 56 px title plus 18 px buttons overflows a 360 px height; shows as an `offscreen` finding). With `stretch/mode canvas_items` a second `viewport_size` in a scenario is not a second layout.
- Never make the Button `focus` stylebox empty: gamepad/keyboard navigation becomes invisible.
- Never add `class_name GameManager`: `Class "GameManager" hides an autoload singleton`.

Verify: scenario at the base viewport: `{"type":"ui_report","label":"menu","ascii":true,"fail_on":["overlap","zero_size","offscreen"]}` (use `"any"` for HUD/pause); for the menu, assert a button's `size:x`, send `ui_down` as a `key` step. For pause: assert `PauseMenu/PauseRoot` `visible` false, `key` `keycode` 4194305 (Esc) press+release, `wait_until` visible true (`timeout_seconds` 2), `ui_report`, Esc again, `wait_until` visible false. Both `wait_until` resolving proves the menu works while paused. Pass: `run_scenario.py` ok, `findings=0`.

---

## 5. Dialogue Box And Branching Dialogue

Templates: `dialog_box.gd` (typewriter over `Array[String]`, `ui_accept` advances; `show_lines(lines, speaker)`, signal `finished`), `interactable.gd` (Area2D prompt, `interacted` signal, only reacts to the group `player`), and for branching `dialogue_runner.gd` (graph + flags), `choice_list.gd`. The runner's header documents the JSON graph (`start`, `nodes` with `speaker/text/next/end/choices/branch/set_flag/require_flag`). Run the class-cache `--import`.

Box scene: `CanvasLayer` layer 5, `DialogRoot` Control (unique, `FULL_RECT`), `Anchor` MarginContainer (`BOTTOM_WIDE`, `grow_vertical` 0, `mouse_filter` 2) > `Box` PanelContainer > `Body` VBox with `SpeakerLabel` and `BodyLabel` RichText (`fit_content`, `autowrap_mode` 3, unique). Reserve text height with `custom_minimum_size.y`, not by measuring the English string. Attach `dialog_box.gd` with `characters_per_second`.

Plain NPC: `Area2D` with `interactable.gd` (`collision_layer` 1, `collision_mask` 2, a circle shape, and a `Label` child named `Prompt`: without it the script errors on `_ready`). Write the level script with `_on_npc_interacted(_by)` first, attach it, then `connect_signal` (`signal_name:"interacted"`). Pass lines as a typed `Array[String]` variable; a bare `[...]` literal raises a runtime type error at `show_lines`.

Branching: one scene owns the runner and the box so unique names resolve: build the box nodes directly in `talk.tscn` with `scene_batch`; do not `instantiate_scene` `dialog_box.tscn` and add `Choices` under it (children added under an instance are silently dropped on save) (`Talk` Node2D > `Dialogue` Node + `DialogueBox` CanvasLayer containing a `Choices` VBox with `choice_list.gd`). A glue script connects `runner.line_shown(text, speaker, node_id)` through a small handler that builds an `Array[String]` and calls `box.show_lines(lines, speaker)` (the signatures differ, a direct connect fails), `runner.choices_shown` to `choices.show_choices`, `choices.choice_selected` to `runner.choose`, and `box.finished` to `runner.advance`, then `runner.start()` (`graph_path` prop, `autostart:false`). `dialog_box.gd` has no `class_name`, so reach it with `call()`/`connect(&"finished", ...)`.

Traps:
- A focused `Button` does **not** consume the `ui_accept` press; it reaches `_unhandled_input` and `dialog_box.gd` would skip the line behind the question. `choice_list.gd` takes `ui_accept` in `_input` and activates the focused choice. Do not delete that handler.
- Unit-test the runner and `DialogueRunner.validate_graph(graph)` (names dangling `next` targets); see `references/testing.md`.

Verify (scenario): after `wait_seconds` 1.0 (the player must fall into the NPC area; physics time) send `interact` as a `key` step with `physical_keycode` 69, `wait_until` `DialogBox/DialogRoot` `visible` true, `ui_report` `findings=0`, `dump_tree` `visible_characters` not 0 (-1 = whole line shown). Advance with `key` `keycode` 4194309 (`ui_accept`). Branching: assert `BodyLabel` `text` per line, `ui_down` then `ui_accept` picks the second choice, a `log_assertions` entry on the glue script's print proves the end state; a choice behind an unset `require_flag` must be absent from the `Choices` dump.

---

## 6. Save And Load

Templates: `save_manager.gd` (autoload `SaveManager`: versioned JSON slots under `user://saves/`), `game_manager.gd` (`to_save_data()` / `from_save_data()`). Add game state to the payload dictionary (`payload["inventory"] = bag.to_dict()`).

Rules:
- `SaveManager.load_game(slot)` returns an **empty Dictionary** for every failure (missing, truncated, garbage). Check `is_empty()` before touching game state, or a corrupt slot silently loads as a fresh game.
- `get_tree().current_scene` is null when something other than the engine loaded the scene (scenario runner, unit test, mid change). Guard it before `.scene_file_path`.
- Annotate `var data: Dictionary = ...`; `:=` from a `Variant` is a boot-blocking parse error.
- Test the failure path, not just the happy path: write garbage into the slot with `run_gdscript` (`FileAccess` on `user://saves/slot_0.json`; it hits the same file the game reads, on every platform, do not hand-build the OS path) and assert with scenario `log_assertions` that the boot reported the bad slot, started fresh, saved over it, and the next load returned the data. A scenario fails on any error-level log line unless a `log_assertions` entry expects it.

Verify: a boot that saves then loads prints one line per step; `grep` the `run_project.py --log-file` output. The value increasing on each fresh boot (120, 240, ...) is the persistence. A second `no save` line means the write never happened; read the `SaveManager:` error above it. Pass: `validate_project.py` ok, 0 warnings.

---

## 7. Audio: Buses, AudioManager, Scene Transitions

Make and check sound with `references/audio.md` (`make_sfx.py`, `make_music.py`, `inspect_audio`). Gate every file before wiring it: `inspect_audio {"audio_paths":["audio"],"format":"text","expect":{"not_silent":true,"no_clipping":true,"max_leading_silence_ms":25}}` (a silent WAV is a valid file; `not_silent` is the only check that catches it).

Order that works:
1. Generate WAVs into `res://audio/` (`--out` a directory; `--variations 3` writes `hit_1..3.wav`).
2. `import_project.py`, then set looping for music: `set_import_options {"file_path":"audio/overworld.wav","options":{"edit/loop_mode":2}}` and `--import` again. `edit/loop_mode` 2 is **Forward** (the importer enum is offset by one from `AudioStreamWAV.LoopMode`). `inspect_audio` must then show `import_loop` with `loop_mode_name: forward` (`expect.loopable` gates the seam); no `.import` sidecar reports `none` and will not loop in game.
3. Buses (Standard Setup 5), then copy and register `audio_manager.gd` and `scene_transition.gd`.
4. `preload("res://audio/x.wav")` needs the file and its `.import` at parse time, so import before attaching scripts that preload.

Use: `AudioManager.play_sfx(stream, volume_db, pitch)`, `play_music(stream, fade)`, `stop_music(0.0)`; `await SceneTransition.change_scene("res://scenes/level_1.tscn")` (a coroutine; calling `change_scene_to_file` directly skips the fade and leaves the overlay half-drawn). A music stream still playing at exit prints `1 resources still in use at exit`; tools file it as `info` / `exit_leak` (see `audio.md`).

Verify: scenario asserting `node_exists` for `/root/SceneTransition` and `/root/AudioManager`, `dump_tree` of `/root/AudioManager` (eight `Sfx*` players on `SFX` plus `MusicA`/`MusicB` on `Music`), and `log_assertions` `{"regex":"no .SFX. audio bus","min_count":0,"max_count":0}`. `validate_project.py` 0 warnings (a `no 'SFX' audio bus` warning means buses were skipped or ran after the autoload).

---

## 8. Art From Text

Rules, palettes and the full loop are in `references/pixel_art.md`; this is only the pipeline wiring.

- `draw_image` with `palette` + `rows` (one character per pixel) or `shapes`; with `frames` it writes an atlas and answers with the read-back rows, `grid` (paste it into `build_sprite_frames`) and `seamless` per tile when `tile_check` is on. `outline` pads one pixel: leave the artwork's last row/column empty or the payload says `"outline_clipped": true`. `line` takes `from`/`to` points, not `x/y/x2/y2`. Do not `mirror_x` an asymmetric walk cycle.
- `process_image {"input_path":"art/panel.png","output_path":"art/panel_out.png","operations":[{"type":"nine_patch_margins"}]}` measures a panel's margins (`steps[0].detail.margins`) instead of guessing; it requires an output and also does background cutout of generated images.
- Run `import_project.py` before anything loads the PNG (`build_tileset`, `build_theme` with a `StyleBoxTexture`, `build_sprite_frames`). A `null` texture in `inspect_resource` output means it was not imported first.
- `build_sprite_frames` on an `AnimatedSprite2D` node: `spritesheet`, `grid`, `animations` with `{"row":0,"cols":[0,1]}` frame specs, `resource_save_path`.

Verify: `inspect_image {"image_path":"art/hero.png","expect":{"not_blank":true,"width":64,"height":16,"max_unique_colors":8,"has_alpha":true}}` exits 0; tilemaps via Playbook 3; `validate_project.py` ok.

---

## 9. Inventory, Items And Loot

Templates: `item_data.gd` (`ItemData` Resource: id, icon, stack size, value, tags), `inventory.gd` (`Inventory` node: slots, stacking, `to_dict()`/`from_dict()`), `loot_table.gd` (`LootTable` Resource: weighted drops, seedable), `inventory_ui.gd` (grid of focusable slot Buttons; scene tree in its header). Class-cache `--import` after copying.

- One `.tres` per item with `resource_batch`; the `"script":"res://scripts/item_data.gd"` key (not `resource_type`) makes it an instance of the project's own class:

```json
resource_batch {"resource_path":"items/coin.tres","create_if_missing":true,"script":"res://scripts/item_data.gd","actions":[{"type":"set_properties","properties":{"id":"coin","display_name":"Coin","max_stack":99,"value":1,"icon":{"__resource":"res://art/icon_coin.png"},"tags":["currency"]}}]}
resource_batch {"resource_path":"loot/chest.tres","create_if_missing":true,"script":"res://scripts/loot_table.gd","actions":[{"type":"set_properties","properties":{"rng_seed":1234,"rolls":3,"entries":[{"item":{"__resource":"res://items/coin.tres"},"weight":6.0,"min":1,"max":5},{"item":null,"weight":1.0}]}}]}
```

- `"item": null` is the miss slot. `rng_seed` fixes the sequence, the only way a test can assert loot; use `-1` to ship random.
- The bag is a child `Node` named `Inventory` on the player (group `player`). A pickup `Area2D` finds the player by group and the bag by that node name via `get_node_or_null("Inventory") as Inventory`, so nothing holds references. A full bag must leave the pickup on the ground (`add()` returns what fit).
- Items match by `id`, not by instance: the same `.tres` loaded twice is one item.
- The UI has no `class_name`: the level calls `ui.call(&"bind", bag)` and `call(&"toggle")`. Add an `inventory` input action and toggle from `_unhandled_input`.
- Persist with `payload["inventory"] = bag.to_dict()` (Playbook 6), restore with `from_dict` after checking the value `is Dictionary`.

Verify: unit tests for the rules (stack top-up before a second slot, `add` returns only what fit, `remove` reports what was taken, save round trip, seeded loot rolls identical, `LootTable.validate()` catches an all-zero table) per `references/testing.md` (`run_tests.py --init-mini` first). Scenario: walk onto pickups, `key` the inventory action with `physical_keycode`, `wait_until` the UI root visible, `ui_report` `findings=0`, `log_assertions` on the pickup prints. A one-shot `run_gdscript` with `"scene_path"` can add to the bag, `SaveManager.save_game({"inventory": bag.to_dict()}, 1)`, `clear()`, `load_game(1)` + `from_dict`, and return `bag.count(item)` to prove the round trip (`delete_save(1)` after).

---

## 10. Settings Menu

Templates: `settings.gd` (autoload `Settings`: three volumes mapped to buses, fullscreen, vsync, rebinds in `user://settings.cfg`; reads and applies in `_ready`), `settings_menu.gd` (sliders/switches bound to `Settings`; props `return_scene`, `save_on_change`), `input_remap.gd` (press-a-key rebinding, `actions` and `steal_on_conflict` props; rows `Row_<action>/Bind_<action>` built at runtime).

Order: buses (Standard Setup 5) before registering `Settings`, or sliders move and nothing gets quieter plus a `Settings: no 'Music' audio bus` warning. Build the screens with containers (`GridContainer` of Label + `HSlider`/`CheckButton`, `unique_name_in_owner` on the controls the template header names); give each `CheckButton` text (`ON`/`OFF`), an empty label is invisible to `ui_report` and `label_without_text` hints. `return_scene` empty means Back hides the panel and emits `closed` (pause-menu shape); set it to the main menu scene (and `settings_scene` on `main_menu.gd`) for a standalone screen.

Traps:
- `DisplayServer` is a no-op under `--headless`: verify fullscreen through `Settings.fullscreen` and the `.cfg`, not `window_get_mode()`.
- Volume 0 mutes the bus rather than writing `-inf` dB.
- Drive sliders and focus with `key` steps using `keycode` (`ui_left` 4194319, `ui_down` 4194322, `ui_accept` 4194309); the remap screen takes a plain `keycode` such as 75 for K.

Verify across processes: process 1 `run_gdscript` `Settings.reset_to_defaults(); Settings.set_music_volume(0.25); return Settings.save()`. Process 2 a scenario: the Music slider boots at 25 (proves `Settings` read the file); nudge Master and toggle Fullscreen with `key` steps and assert values; `ui_report` `findings=0`. Process 3 `run_gdscript` returns `Settings.music_volume` and `AudioServer.get_bus_volume_db(...)` (`linear_to_db(0.25)` is -12.04 dB: the bus really moved) and `InputMap.action_get_events("interact")` for the rebind (stealing a key leaves the other action unbound).

---

## 11. 3D Starter

Templates: `player_fps_3d.gd` (mouse look, sprint, jump, Esc releases the mouse) and `player_third_person_3d.gd` (SpringArm3D orbit camera, camera-relative movement; optional gamepad `look_*` actions are skipped if not defined). Headers hold the scene trees.

Set up: input actions `move_forward`, `move_back`, `sprint` (FPS) plus the 2D move set; `set_setting physics/3d/default_gravity` 9.8. Player root `CharacterBody3D` (layer 2, mask 1, group `player`), capsule `CollisionShape3D` at y 0.9, `CameraPivot` `Node3D`, `Camera3D` under it (third-person: `Body` Node3D for the visual and `SpringArm3D` between pivot and camera).

Level: `MeshInstance3D` floor, `DirectionalLight3D`, `WorldEnvironment`, instantiate the player above the floor.

Traps:
- **The pivot carries pitch** (and yaw in third person). Rotating the `CharacterBody3D` on X tips the capsule and the player falls through the floor.
- A `MeshInstance3D` has no collider: `bake_collision {"scene_path":"scenes/level_3d.tscn","node_path":"root/Floor","mode":"trimesh"}`, or add a `StaticBody3D` + `BoxShape3D` (cheaper and exact for boxes). The baked body is named `Floor_col` (`<mesh>_col`); read the real name from `inspect_scene`. Imported `.glb`: `import_project.py`, `instantiate_scene`, then `bake_collision` on its `MeshInstance3D`.
- `SpringArm3D` needs `add_excluded_object(get_rid())` (the template does it) or the camera snaps into the head. `capture_mouse_on_ready` false only for headless verification; leave it true in a real build.
- A 3D scene that renders black produces no error: use `spatial_report`.

Verify: scenario `wait_seconds` 2.0, assert `Player` `position:y` above the floor (the capsule stands on it) and `velocity:y` approx 0; hold `move_forward` 0.6 s and assert `position:z` decreased (-Z is forward at yaw 0); `{"type":"spatial_report","ascii":true,"fail_on":["no_camera_3d","no_light_3d","not_on_screen"]}` clean. Delete the `Sun` to see `no_light_3d` fail with a message naming what to add. Assert `node_exists` `Floor/Floor_col`.

---

## 12. Whole-Game Assembly Checklists

Build a small game by composing the playbooks above, not by replaying them. For each: scaffold, set layers, draw/generate assets, copy templates, assemble, then verify the core loop with one scenario.

**Collectathon platformer.** `scaffold_project.py --preset pixel2d`. Layers `world`, `player`, `pickup`. Assets: hero, tiles, a flag, coin (Playbook 8), `jump`/`coin` SFX + music loop (Playbook 7). Templates: `player_platformer_2d`, `camera_shake_2d`, `collectible`, `checkpoint`, `kill_zone`, `goal`, `moving_platform`, `health`, `hud`, `game_manager`, `audio_manager`. Tile level via Playbook 3. Level script: count coins with a group, print one line per decision (`[LEVEL] score=...`) so `log_assertions` can gate; give the HUD its health with `bind_health`.
- Pickups sit on their own layer and **scan** the player layer; the reverse pair means `body_entered` never fires, with no error.
- `moving_platform.gd` is an `AnimatableBody2D`; `sync_to_physics` carries the rider. Size gaps against the jump: at the template defaults the jump rises about 77 px and carries about 176 px (measured), so a 4-cell (64 px) ledge is climbable and a 5-cell (80 px) one is not; a pit under about 10 cells is jumpable, so a wider one needs the platform.
- `goal.gd` stays locked until its group is empty. `kill_zone.gd` teleports to the active `checkpoint`.
- `expect_on_screen` needs a node that draws (a `Sprite2D` **with a texture**, `CollisionShape2D`, a body), not a bare `Node2D` parent or an untextured sprite.
- Scenario: walk, take a coin (`log_marker`), ride the platform with no input and assert `position:x` still advances, take the rest, fall in the pit on purpose to prove respawn at the checkpoint, reach the goal. Gate with `log_assertions` and two `spatial_report`s at `findings=0`. Expected pass: `validate_project.py --warnings-as-errors` ok; one `layer_never_scanned` hint for `pickup` is information, not a defect.

**Top-down wave shooter.** Preset `pixel2d`; layers `world`, `player`, `enemy`, `damage`, every one both occupied and scanned. Templates: `player_topdown_2d`, `shooter`, `projectile`, `object_pool`, `health`, `hitbox`, `hurtbox`, `enemy_chase_nav_2d`, `wave_spawner`, `damage_number`, `camera_shake_2d`, `audio_manager`; follow Playbook 2 for the chaser/hitbox/pool traps. Arena: walls and a pillar as `StaticBody2D`, a `NavigationRegion2D` baked from outlines, `WaveSpawner` with `Marker2D` spawn points, `%WaveSpawner` unique for the HUD. A `Sprite2D` with `region_enabled`, a `region_rect` and `texture_repeat` 2 tiles a small texture over a large block without a second PNG. The arena script connects `wave_started`/`wave_cleared`/`all_waves_cleared` and enemy `died` to score/damage numbers/shake, printing a line each.
- Scenario: press the move action (facing right), fire, wait for `[ARENA] enemy-died`, `wave 1 cleared`, `wave 2 started`; gate with `log_assertions` and `spatial_report`. A printed `chaser navigation=true` line (from your handler on `navigation_mode_changed`) proves the navmesh; deleting the nav resource prints `false` and the chaser still moves straight.
- Pass: `validate_project.py` `static.physics_layers.findings` has no `mask_targets_empty_layer`.

**Grid puzzle (Sokoban).** Preset `pixel2d` (small base size such as 320x180) with `undo` and `restart` actions. Templates `grid_movement.gd` + `sokoban_level.gd` (class-cache `--import` between copying and attaching). Four tiles via `draw_image` shapes (`line` takes `from`/`to`; `"color": null` is transparent and punches holes). Map characters: `#` wall, `@` player, `$` crate, `o` target, `*` crate on target, `+` player on target, `.` floor. `ascii_level` goes on the `Puzzle` root; `cell_size`, `origin`, `move_while_held:false` on the `Player` (`grid_movement.gd`); the level prints nothing, so connect `solved(moves, pushes)` / `move_undone(moves)` to a glue node that prints the log lines. The rules are static functions over plain Dictionaries, so unit-test them with no scene (`references/testing.md`).
- Scenario: `key` steps with `physical_keycode` (actions are bound by physical key; D = 68, Z = 90), and `wait_until` `Player` `moving` false between moves; a key pressed during the slide tween is silently dropped, and `cell` changes the instant a step is accepted. Assert the solved log line and that undo restores the move count (`moves` on the root).

---

## 13. Before You Call It Done

The five commands that decide whether a project is finished, cheapest first. Run them against your own project.

```bash
uv run /absolute/path/to/godot/scripts/debug/lint_project.py /absolute/path/to/project --pretty
uv run /absolute/path/to/godot/scripts/debug/validate_project.py /absolute/path/to/project --warnings-as-errors --pretty
uv run /absolute/path/to/godot/scripts/test/run_tests.py /absolute/path/to/project --pretty
uv run /absolute/path/to/godot/scripts/debug/run_scenario.py /absolute/path/to/project /absolute/path/to/project/scenario_final.json --log-file /absolute/final.log --pretty
uv run /absolute/path/to/godot/scripts/debug/smoke_scenes.py /absolute/path/to/project --all --fuzz --pretty
```

1. **Lint** (no Godot needed, run after every edit): `"ok": true`, `counts.errors == 0`. Catches outdated API names, un-inferable `:=`, dead `$Path`/`%Name`/signal targets, input actions missing from the map, `res://` literals to missing files.
2. **Validate**: `"ok": true`, `counts.errors == 0`, `counts.warnings == 0`, `static.failed_count == 0`. `--warnings-as-errors` makes GDScript and node-configuration warnings (`node_config:body_without_shape`) fail. Read `static.physics_layers.findings` and fix any `mask_targets_empty_layer`. Keep the debugger attached (default); `--no-debugger` hides every warning.
3. **Tests**: `"ok": true`, `counts.failed == 0`. "No test framework found" is the finding: `run_tests.py ... --init-mini`, write the first suite (`references/testing.md`). Add `--strict` so a test asserting nothing fails.
4. **One scenario that looks at the running game.** lint and validate never start it. Include the three text read-backs:

```json
{"scene_path":"res://scenes/main_level.tscn","viewport_size":{"width":640,"height":360},"settle_frames":4,"steps":[
 {"type":"ui_report","label":"hud","node_path":"HUD","ascii":true,"fail_on":["any"]},
 {"type":"spatial_report","label":"world","ascii":true,"expect_on_screen":["Player/Sprite2D"],"fail_on":["not_on_screen","zero_scale"]},
 {"type":"screenshot","path":"user://final.png","expect":{"not_blank":true,"min_opaque_ratio":0.05},"describe":{"ascii":true,"ascii_width":48}}],
 "assertions":[{"assertion":"node_exists","node_path":"Player"}]}
```

   `ui_report` `fail_on: ["any"]` is the only check that catches controls stacked at (0, 0); `spatial_report` the only one that catches a player off camera or sunk into the floor (name a node that **draws**); `screenshot` `not_blank` fails on a single flat colour (write to `user://` so no PNG lands in the project).
5. **Smoke every scene with fuzzed input**: `counts.failed == 0`. Runs each `.tscn` in its own process and fires the real InputMap actions pseudo-randomly, finding errors no scripted scenario triggered (such as physics-callback errors from pooled bullets). Reports `perf` per scene (`node_count_start`/`node_count_end`, `orphan_nodes`) so an idle leak shows as growth. `exit_leak` and `host_capability` diagnostics are `info`, never failures (`references/audio.md`, `references/automation_api.md`). A GPU-less host falls back to Compatibility, so a shader with a renderer-dependent branch must not trust the `rendering_method` setting.

Every command exits non-zero on failure, so in CI they chain with `&&`. While writing: `uv run /absolute/path/to/godot/scripts/docs/api_lookup.py AnimatableBody2D.sync_to_physics` checks names against the installed engine, and `run_gdscript '{"expression":"Vector2(3, 4).length()"}'` runs one expression with project autoloads live.

---

## When A Playbook Fails

| Symptom | Cause | Fix |
| --- | --- | --- |
| `Unknown parameter for <op>: <key> (did you mean ...)` | Invented or misremembered field | `help '{"op":"<op>","format":"text"}'` and use the printed parameters |
| `Parse Error: Identifier "Health" not declared in the current scope` | `class_name` script added since the last import | `godot --headless --path /absolute/path/to/project --import`, re-validate |
| `Property does not exist at attach_script.script_properties` | Script names an unregistered autoload, so it failed to parse | `add_autoload` first |
| `ui_report` finds `overlap`, or everything at `[0, 0, ...]` | Bare `Control` parent, or `layout_preset`/`position` set on a container's child, or a hand-written `.tscn` | Insert the container; rebuild through `scene_batch`; see `references/tscn_format.md` |
| `%Name` is null at runtime | Missing `unique_name_in_owner`, or used across a scene instance boundary | `configure_node` with `"unique_name_in_owner": true` in the owning scene |
| Player falls through the level | Tiles without collision, or an unbaked 3D mesh | `"collision": "full_cell"` in `build_tileset`, or `bake_collision` |
| `body_entered`/`area_entered` never fires | Mask/layer pair backwards, or the other side does not scan the bit | Check both layers and masks; `physics_layers.findings` |
| Scenario "does nothing" | `action` step against `_unhandled_input` code, or wrong `keycode`/`physical_keycode` | See `references/automation_api.md` |
| Zero warnings on a project the editor complains about | Debugger not attached | Keep `--debug --ignore-error-breaks`; do not pass `--no-debugger` |
