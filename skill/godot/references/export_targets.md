# Export Targets (Desktop And Web)

Read this when the task is to package, run, or ship a build for Windows, Linux, macOS or Web. A headless agent has no Export dialog: `add_export_preset` writes the preset, `export_project.py` exports it.

## From zero to a running build

1. Make the project exportable. With no main scene an export builds fine and then **hangs** when run; on macOS the binary inside the bundle is named after `application/config/name`. The key is `application/run/main_scene` (writing `run/main_scene` creates a `[run]` section the engine ignores: `Can't run project: no main scene defined`).

```bash
godot --headless --path /absolute/path/to/project \
  --script /absolute/path/to/godot/scripts/core/dispatcher.gd \
  project_batch '{"actions":[
    {"type":"set_setting","name":"application/run/main_scene","value":"res://scenes/main.tscn"},
    {"type":"set_setting","name":"application/config/name","value":"Game"}]}'
```

2. Write the preset (`platform`: `web`, `windows`, `linux`, `macos`; it also creates `../build/<platform>/` beside the project):

```bash
godot --headless --path /absolute/path/to/project \
  --script /absolute/path/to/godot/scripts/core/dispatcher.gd \
  add_export_preset '{"platform": "web"}'
```

3. Export (preflight runs first and fails loudly with fixes on stderr and in `errors[]`/`fixes[]`):

```bash
uv run /absolute/path/to/godot/scripts/export/export_project.py \
  /absolute/path/to/project "Web" /absolute/build/web/index.html
```

`export_project.py PROJECT PRESET OUTPUT [--mode release|debug|pack|patch] [--preflight-only] [--patches BASE.pck]`. Preset names are exact and case-sensitive (`Web`, `Windows Desktop`, `Linux`, `macOS`); a wrong name lists every preset in the file. `--preflight-only` checks preset, templates and output path without exporting; `--mode pack` needs no export templates. Prefer `--mode debug` for a first smoke test. Godot does not create the output directory (the wrapper creates its parent) and has exited 0 while producing nothing, so the wrapper checks the artifact exists and is non-empty.

`add_export_preset` params: `platform` (required), `name`, `export_path` (default `../build/<platform>/<artifact>`, beside the project, never inside), `options` (free-form, merged over verified defaults, never key-checked), `overwrite` (default false: an existing name is an error), `export_filter`/`include_filter`/`exclude_filter`, `custom_features`. Run `help '{"op":"add_export_preset"}'` for the rest. The file is edited as text, so presets authored in the editor (signing, profiles) keep their exact bytes.

`platform=` in the file is `Web`, `Windows Desktop`, `Linux`, `macOS`. `Linux/X11` is the Godot 3 name and exports nothing in 4.x.

## Per-platform facts

### Web

- `variant/thread_support` defaults to **false**, on purpose. A threaded build needs `SharedArrayBuffer`, which needs a cross-origin-isolated page (`Cross-Origin-Opener-Policy: same-origin` and `Cross-Origin-Embedder-Policy: require-corp`). itch.io, GitHub Pages and most static hosts send neither, so a threaded build is a blank page there. Set `true` only once the real host sends both headers; `serve_web.py` does, so test threaded builds locally.
- A web export writes `index.html`, `.js`, `.wasm`, `.pck`, icons and two audio worklet `.js` files. **All ship together**; the `.html` alone is a blank page.
- **A web build always runs the Compatibility renderer** (4.7 overrides `rendering_method.web` to `gl_compatibility`), so a `forward_plus` project silently loses SDFGI, volumetric fog, SSIL, SSR in the browser. If web is the target, set `rendering/renderer/rendering_method = "gl_compatibility"` for desktop too so you test what ships.
- No ETC2/ASTC setting needed.

Serve and check (`python3 -m http.server` is not enough: no COOP/COEP, and the browser rejects `.wasm` without `application/wasm`):

```bash
uv run /absolute/path/to/godot/scripts/export/serve_web.py /absolute/build/web --check --pretty
uv run /absolute/path/to/godot/scripts/export/serve_web.py /absolute/build/web
```

`--check` starts the server, requests `index.html`, `.wasm` and `.pck`, asserts status, content types, COOP/COEP and a 404 for a missing file, prints JSON (`ok`, `cross_origin_isolated`, `threaded_build`, `checks[]`, `failed[]`) and exits 0/1; a directory that is not a web export exits 2. Without `--check` it serves on `127.0.0.1:8060` (`--port 0` picks a free port, `--no-isolation` mimics a plain static host).

### Windows

Plain release export gives a `.exe` plus a sibling `.pck` with no `rcedit` or wine needed (`rcedit` only stamps the icon/version; without it the build runs with Godot's icon). `binary_format/embed_pck=true` makes one self-contained file.

### Linux

`binary_format/architecture="x86_64"`, `.pck` beside the binary (`embed_pck=false`). Other architectures (`arm64`, `arm32`, `x86_32`) are selectable there.

### macOS

Four facts that cost an hour each:

1. The official 4.7 template zip holds only `godot_macos_{release,debug}.universal`. Asking for `x86_64` or `arm64` fails with `Requested template binary ... not found`. The preset uses `binary_format/architecture="universal"`.
2. universal/arm64 needs the **project** setting `rendering/textures/vram_compression/import_etc2_astc = true` (default false) or the export aborts. `add_export_preset` sets it and lists it in `project_settings_changed`.
3. `codesign/codesign=1` is built-in ad-hoc signing (no Apple account); the `.app` runs on the machine that built it, and Gatekeeper still blocks it after download (needs a Developer ID plus notarization). The "notarization is disabled" / "ad-hoc signature" lines are warnings.
4. `application/bundle_identifier` is required (`Invalid bundle identifier: Identifier is missing.`); segments are letters, digits, hyphens, not starting with a digit.

Cross-exporting from another OS works; only `.dmg` needs a macOS host. Output `.app` writes a bundle directory. The binary is `Contents/MacOS/<application/config/name>` (not the preset or `.app` name) and takes engine flags, which is the cheapest "did the build run" check (needs a main scene or it hangs):

```bash
/absolute/build/macos/Game.app/Contents/MacOS/Game --headless --quit-after 20
```

## Pack and patch

`--mode pack` writes only a `.pck`; `--mode patch --patches /absolute/build/base.pck` writes a delta against a base from the same project and preset, and fails `Save PCK: No files or changes to export.` (exit 1) if nothing changed since the base.

## Checklist

- Reuse the preset names already in `export_presets.cfg` (`export_project.py PROJECT ANYNAME OUT --preflight-only` prints them).
- Export templates must match the engine build exactly (`godot --version`).
- Keep keystores, certificates and passwords out of git. The script encryption key lives in `res://.godot/export_credentials.cfg`, not in `export_presets.cfg`, so the preset file is safe to commit.
- Platform branches use `OS.has_feature("web" | "windows" | "macos" | "linux" | "pc" | "debug" | "release" | "editor")`; the macOS tag is `macos`, not `osx`. Search for `OS.has_feature`, per-feature overrides in `project.godot` and `custom_features` before changing platform logic.
