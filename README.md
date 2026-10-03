# godot-skill-lite

A Claude Code plugin that packages the **Godot** Agent Skill: Godot 4.x game development (verified on 4.7). It scaffolds, inspects and edits projects, scenes, UI and resources headlessly; authors tilesets, levels, shaders, audio and export presets; lints GDScript, scenes and shaders; and unit-tests, smoke-runs, debugs and exports projects. Every result can be checked as text, so no vision is needed.

## Install

In Claude Code:

```
/plugin marketplace add kouui/godot-skill-lite
/plugin install godot@godot-skill-lite
```

To try a local checkout:

```
/plugin marketplace add ./path/to/godot-skill-lite
```

## Layout

```
.claude-plugin/
  plugin.json        plugin manifest
  marketplace.json   single-plugin marketplace (source: ./)
skills/
  godot/
    SKILL.md         skill entry point
    references/      on-demand reference docs
    scripts/         bundled Python / GDScript tooling
    templates/       GDScript, shader and test templates
```

## Requirements

- Python 3
- Godot 4.x on `PATH`, or set `GODOT_BIN` / pass `--godot-bin`, for anything that runs the engine

## Credits

Based on [haxqer/godot-skill](https://github.com/haxqer/godot-skill) by Qian Xiao, MIT License. Changes in this repo restructure it as a Claude Code plugin and drop the Codex-specific `agents/openai.yaml` metadata.
