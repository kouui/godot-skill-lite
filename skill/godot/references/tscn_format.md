# Hand-Writing And Hand-Editing `.tscn` Files

Read this only when a `.tscn`/`.tres` must be written or patched as text (no shell to run the dispatcher, or a whole new scene authored from nothing). With a shell, use `scene_batch`, `add_node`, `configure_control`, `attach_script`, `connect_signal`, `reparent_node`: they cannot produce a malformed tree. The mistakes below load **without any error**, so run the verification loop afterwards no matter how the file was produced.

## Header rules

- First section must be `[gd_scene format=3]`. `load_steps` is a pure hint: omit it. Scene-level `uid` is optional (headless saves emit none).
- Attribute values are double-quoted (`type=Button` or single quotes are hard parse errors). `;` starts a comment; `#` does not (`Invalid color code: #`).
- `[ext_resource]` and `[sub_resource]` must be declared before the first `[node]` that uses them. An undeclared `ExtResource("x")` or a late `[sub_resource]` fails the whole load. An `[ext_resource]` with a missing file only prints `referenced non-existent resource` and the property stays null: read the log, not the exit code.
- Property lines follow their node: `key = value`, Godot literals (`Vector2(1, 2)`, `Color(1, 0.5, 0, 1)`, `ExtResource("1_a")`, `&"name"`, `NodePath("A/B")`). Packed arrays take a flat list (`PackedVector2Array(0, 0, 10, 10)` is two points). Enums are plain ints. `layout_mode`/`anchors_preset` are editor bookkeeping and not needed for runtime rects.
- `.tres`: header `[gd_resource type="..." format=3]`, same declarations, properties under one final `[resource]`. Prefer `resource_batch` / `build_theme`.

## UIDs: write `path`, omit `uid`

| `[ext_resource]` form | Result |
| --- | --- |
| `path` only | Loads, no diagnostic. **Use this.** |
| `uid` + `path`, uid unregistered/stale | Warning `invalid UID ... using text path instead`, loads via path |
| `uid` + `path`, uid registered to a **different** file | **The uid wins, silently.** Editing only `path` on a pasted line loads the old resource |
| `uid` only | Hard parse error |

Moving a file never updates its uid, so a shell `mv` leaves scenes loading the file with a wrong recorded `path` and no diagnostic. Move with `uv run /absolute/path/to/godot/scripts/project/move_resource.py PROJECT SRC DST` (moves sidecars, rewrites `path=`, `preload()`, `project.godot`, re-imports, re-lints). See `references/automation_api.md`.

## Node hierarchy (breaks silently)

The tree comes only from `parent=`; block order and indentation mean nothing.

1. The first `[node]` is the root and has **no** `parent`.
2. Direct children of the root: `parent="."`.
3. Deeper nodes: path from the root **excluding the root's name**, `/` the only separator.

```
[node name="Menu" type="Control"]

[node name="Panel" type="PanelContainer" parent="."]

[node name="VBox" type="VBoxContainer" parent="Panel"]

[node name="StartButton" type="Button" parent="Panel/VBox"]
text = "Start"
```

A child block must come after its parent's block. Other node attributes: `groups=[...]`, `index="0"`, `instance=ExtResource("...")` (no `type=`), `unique_id` (omit). `unique_name_in_owner = true` is a property line, which is what makes `%Name` resolve.

Override a property inside an instanced scene with a `[node name="Title" parent="Card/VBox" index="0"]` block (no `type`) plus property lines.

### Connections

`[connection signal="pressed" from="Panel/VBox/StartButton" to="." method="_on_start_pressed"]` goes after all `[node]` blocks; paths are root-relative like `parent=`. A wrong `from`/`to` (root name included, unknown node) is **silently dropped**: zero live connections, button does nothing. A nonexistent `method` only fails when the signal fires.

### Node names

Keep names to `[A-Za-z0-9_]`, matched case-sensitively in `parent=`, `[connection]` and `NodePath`. The loader does not sanitize, but `/` and `:` make a node unreachable, and `. / : @ % "` are what the engine's own validator rewrites to `_` (the node renames later and every path to it goes stale).

### Common mistakes

| Mistake | Symptom | Fix |
| --- | --- | --- |
| Every node `parent="."` | **Silent.** Loads clean, all Controls stack at (0,0), containers stay empty | Full root-relative path: `parent="Panel/VBox"` |
| Duplicate sibling names | **Silent.** `get_node()` finds only the first | Unique names |
| Wrong separator or path (`Panel#VBox`, `Panel.VBox`, `Menu/Panel` with root name, `root/Panel` dispatcher style, unknown node, child before parent) | `WARNING: Parent path ... has vanished`; node reparented to the root as `Parent#Name` (`#` in a runtime name means a broken parent path) | `/` only, root name dropped, `.` alone = root; `root/Panel` is dispatcher style, not `.tscn` |
| Non-root node without `parent=`, or root with `parent="."` | `ERROR: Invalid scene ...`; `instantiate()` returns null | Fix `parent=` |
| `uid=` copied from another resource | **Silent wrong resource** | `path` only |
| `[connection]` from/to with root name or wrong path | **Silent**, never connected | Root-relative, `to="."` |
| `parent="Panel/"` trailing slash | Tolerated, correct | |

## Verify (all three; none alone is enough)

Write the file, then:

**1. `check_project`** (parse and resource failures; it also instantiates every scene, which exposes parent-path problems):

```bash
godot --headless --debug --ignore-error-breaks --path /absolute/path/to/project \
  --script /absolute/path/to/godot/scripts/core/dispatcher.gd check_project '{}' 2>&1 \
  | uv run /absolute/path/to/godot/scripts/debug/godot_log_parser.py -
```

Expect `"counts": {"total": 0, ...}`. Invalid-scene errors land in `failed[]` (the error names the node, read the scene from the entry). A vanished-parent **warning** leaves `failed_count` 0 and `validate_project.py` ok unless `--warnings-as-errors`: do not skim past it. **A fully flat tree still passes.**

**2. `inspect_scene`** (the only check that reveals a wrong tree):

```bash
godot --headless --path /absolute/path/to/project \
  --script /absolute/path/to/godot/scripts/core/dispatcher.gd \
  inspect_scene '{"scene_path":"scenes/menu.tscn","include_properties":false}'
```

Read every node `path`. Intended `./Panel/VBox/StartButton`; flat bug `./StartButton`; bad separator `./Panel#VBox/StartButton`. Every `connections` `source`/`target` must match a real node path.

**3. Run it:** `uv run /absolute/path/to/godot/scripts/debug/run_project.py /absolute/path/to/project res://scenes/menu.tscn --quit-after 120 --timeout 60` (exercises `_ready`, `@onready`, container layout, signals; still silent about a flat tree).

Repair structure with `reparent_node` / `reorder_node` / `remove_node`, not by re-editing text. `save_scene` round-trips a hand-written file into canonical form. A correct tree can still overlap if siblings are absolutely positioned: see `references/game_ui.md`.
