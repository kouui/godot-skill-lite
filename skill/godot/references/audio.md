# Audio: Make Sound Without A Sound Source, Check It Without Ears

Read this when the game needs sound effects or music. You have no sample library and cannot hear, so generate with pure-Python tools and verify with `inspect_audio`. Nothing needs pip, network or an audio device.

| Need | Tool | Verify with |
| --- | --- | --- |
| Sound effects | `scripts/assets/make_sfx.py` (sfxr-style synth) | `inspect_audio` + the `expect` block the tool prints |
| Music | `scripts/assets/make_music.py` (step-sequencer) | `inspect_audio` with `expect.loopable` |

Both print a JSON report whose `files[0].expect` (sfx) / `expect` (music) states what the sound should be. **Paste it into `inspect_audio`**: the generator states intent, the inspector measures the file, the exit code is the answer. Generated WAVs have no `.import` yet (`"needs_import": true`).

## 1. Sound effects

```bash
uv run /absolute/path/to/godot/scripts/assets/make_sfx.py --list-presets
uv run /absolute/path/to/godot/scripts/assets/make_sfx.py --preset jump --out /absolute/path/to/project/audio/
uv run /absolute/path/to/godot/scripts/assets/make_sfx.py --preset explosion --seed 7 --variations 3 --out /absolute/path/to/project/audio/
```

Output is a 16-bit mono WAV (normalised to -1 dBFS, click-free, deterministic per seed). `--variations N` writes `name_1.wav ... name_N.wav` with audible pitch/decay jitter that stays inside the preset's duration bounds (use it for footsteps, hits, explosions so repetition does not machine-gun). Other flags: `--seed`, `--duration S` (rescales the ADSR), `--volume 0-1`, `--peak-db`, `--sample-rate`, `--no-overwrite`, `--pretty`.

### Presets and which to use

| Event | Preset (`pitch_direction` measured) | Duration | Note |
| --- | --- | --- | --- |
| Coin / gem / score | `coin` (flat), `pickup` (rising) | 0.25-0.70 / 0.08-0.45 s | `coin` longer jingle, `pickup` quick blip |
| Jump | `jump` (rising) | 0.08-0.40 s | raise `freq_end` for a higher jump |
| Land / footstep | `land` (not gated), `footstep` (none) | 0.06-0.45 / 0.02-0.15 s | footstep x 3-4 variations |
| Shoot / laser | `shoot` (falling), `laser` (falling) | 0.05-0.30 / 0.08-0.40 s | `laser` is the long pew |
| Enemy hit / player hurt | `hit` (none), `hurt` (falling) | 0.05-0.30 / 0.10-0.50 s | |
| Explosion / death | `explosion` (none) | 0.50-1.20 s | variations per barrel |
| Power-up / level up / heal | `powerup` (rising) | 0.30-1.00 s | `--duration 0.9` for a bigger moment |
| Menu move / highlight | `blip` (flat), `select` (flat) | 0.02-0.15 s | short so holding a direction is not a drone |
| Accept / cancel / invalid | `confirm` (rising), `cancel` (falling), `error` (flat) | 0.10-0.60 s | up / down / buzz |
| Door, lever, chest | `door` (falling) | 0.30-1.10 s | |
| Dash, swoosh, wind | `dash` (none) | 0.12-0.60 s | |
| Water, splash | `splash` (none) | 0.25-0.90 s | |
| Alarm, warning | `alarm` (falling) | 0.40-1.50 s | |

### Custom sounds

`--params '{...}'` (or `--params-file`) merges over a preset or the defaults; start from `--list-presets` output. Unknown keys are rejected with the nearest real name.

```bash
uv run /absolute/path/to/godot/scripts/assets/make_sfx.py --name swoop --out /absolute/path/to/project/audio/ \
  --params '{"wave":"saw","freq_start":200,"freq_end":900,"attack":0.01,"decay":0.05,"sustain":0.1,"release":0.1}'
```

Keys: oscillator `wave` (`square`/`saw`/`triangle`/`sine`/`noise`), `duty`, `duty_sweep`; pitch `freq_start`, `freq_end`, `freq_curve`, `vibrato_depth`, `vibrato_rate`, `arpeggio` (semitone list `[0,4,7,12]` or `[{"at":0.12,"semitones":5}]`); envelope `attack`, `decay`, `sustain`, `sustain_level`, `release`, `punch`; body `noise_level`, `sub_level`, `sub_freq_start`, `sub_freq_end`, `sub_decay` (sine layer that makes an impact land); filters `lowpass`, `lowpass_sweep`, `highpass`; grit `bitcrush_bits`, `bitcrush_rate`; `repeat`.

## 2. Music

```bash
uv run /absolute/path/to/godot/scripts/assets/make_music.py --list-presets
uv run /absolute/path/to/godot/scripts/assets/make_music.py --preset overworld --out /absolute/path/to/project/audio/
uv run /absolute/path/to/godot/scripts/assets/make_music.py --preset battle --bars 16 --tempo 168 --stereo --out /absolute/path/to/project/audio/
```

Presets: `menu` (88 BPM, calm pad), `overworld` (124), `battle` (168), `victory` (140, 4 bars, a jingle). Overrides: `--bars`, `--tempo`, `--sample-rate`, `--name`, `--stereo` (`pan` only works with it), `--seed`. The output is exactly one loop long and seamless by construction; the report's `loop_seam_delta` and `expect.loopable` prove it.

Custom: `--list-presets` prints each preset's full song object; save one, edit the note rows, render with `--song FILE` (or `--song-json '...'`). `--out` ending in `.wav` names the file, `--out DIR/` uses the song name.

```json
{"name": "theme", "tempo": 124, "steps_per_beat": 4, "beats_per_bar": 4, "bars": 8,
 "tracks": [
   {"name": "lead", "wave": "square", "duty": 0.5, "volume": 0.75, "pan": 0.2,
    "envelope": {"attack": 0.004, "decay": 0.035, "sustain_level": 0.68, "release": 0.055},
    "notes": "C5 - E5 - G5 - E5 - F5 - E5 - D5 - . -"},
   {"name": "bass", "wave": "triangle", "volume": 0.85, "notes": [{"note": "C3", "len": 4}, {"note": "G2", "len": 4}]}],
 "drums": {"volume": 0.8, "kick": "x.......x.......", "snare": "....x.......x...", "hat": "x.x.x.x.x.x.x.x."}}
```

- Notes: whitespace tokens, one per step: `<A-G>[#|b]<octave>` (A4 = 440), `.` rest, `-` hold previous, `+` chord (`C4+E4+G4`).
- Drum rows: one char per step, `x` hit, `X` accent, `.` rest. Voices `kick`, `snare`, `hat`, `tom`, `clap`.
- Patterns tile to fill `bars x beats_per_bar x steps_per_beat`, so a one-bar drum row under a four-bar melody is normal.
- Bad notes, keys or voices fail with exit 1 and the nearest valid name; nothing is written.

## 3. `inspect_audio`

```bash
godot --headless --path /absolute/path/to/project \
  --script /absolute/path/to/godot/scripts/core/dispatcher.gd \
  inspect_audio '{"audio_path":"audio/jump.wav","format":"text","expect":{"not_silent":true,"no_clipping":true,"max_duration":0.4,"pitch_direction":"rising"}}'
```

Takes `audio_path` or `audio_paths` (files or directories). Run `help '{"op":"inspect_audio"}'` for all params. Fields that matter when you cannot hear:

- `silent`: nothing synthesized (an envelope summing to zero writes a valid, inaudible WAV). `expect.not_silent`.
- `peak_db`, `clipping_ratio` (> 0 means clamped). `expect.no_clipping`.
- `envelope`: one ASCII line of per-slice peak dB (`" .:-=+*#%@"`); a decay reads `@@@%%%###***++==--::.`.
- `dominant_frequency_hz`, `pitch_start_hz`, `pitch_end_hz`, `pitch_direction` (`rising`/`falling`/`flat`/`none`). `null` pitch means noise (tonality < 0.6). Sounds under ~25 ms report `none`; on a music mix the value is usually just the bass.
- `loop_seam_delta`: jump between last and first sample; `expect.loopable` gates it (default 0.02).
- `import_loop`: loop settings from the `.import` sidecar, `null` before import.
- Reads `.wav` (8/16/24/32-bit PCM, float, `smpl` loops), `.ogg`, `.mp3`. Compressed WAV (ADPCM, A-law) is an error: re-export as PCM. Pass source files, not `.godot/imported/*.sample`.

## 4. Import, loop, buses, players

```bash
godot --headless --path /absolute/path/to/project --import
```

Do this before `load("res://audio/...")` works in a fresh process. **Looping a WAV**: the importer's `edit/loop_mode` is offset by one from `AudioStreamWAV.LoopMode`: `0` Detect From WAV, `1` Disabled, `2` Forward, `3` Ping-Pong, `4` Backward. Set `2` for music, re-import, re-check:

```bash
godot --headless --path /absolute/path/to/project \
  --script /absolute/path/to/godot/scripts/core/dispatcher.gd set_import_options \
  '{"file_path":"audio/overworld.wav","options":{"edit/loop_mode":2}}'
godot --headless --path /absolute/path/to/project --import
godot --headless --path /absolute/path/to/project \
  --script /absolute/path/to/godot/scripts/core/dispatcher.gd inspect_audio \
  '{"audio_path":"audio/overworld.wav","envelope":false,"format":"text","expect":{"loopable":true}}'
```

`import_loop` must read `"loop_mode_name":"forward"`. Ogg and MP3 use `{"loop": true, "loop_offset": 0.0}` instead.

Buses (music and effects with independent volume; a UI bus sends to SFX):

```bash
godot --headless --path /absolute/path/to/project \
  --script /absolute/path/to/godot/scripts/core/dispatcher.gd setup_audio_buses \
  '{"buses":[{"name":"Master","volume_db":0.0},{"name":"Music","send":"Master","volume_db":-6.0},
             {"name":"SFX","send":"Master","volume_db":-3.0},{"name":"UI","send":"SFX","volume_db":-4.0}],
    "save_path":"audio/default_bus_layout.tres","set_project_setting":true}'
```

Players: `templates/gdscript/audio_manager.gd` is an autoload with an SFX player pool and a two-player music crossfade. Register it after the buses exist:

```bash
cp /absolute/path/to/godot/templates/gdscript/audio_manager.gd /absolute/path/to/project/scripts/audio_manager.gd
godot --headless --path /absolute/path/to/project \
  --script /absolute/path/to/godot/scripts/core/dispatcher.gd project_batch \
  '{"actions":[{"type":"add_autoload","autoload_name":"AudioManager","path":"res://scripts/audio_manager.gd"}]}'
```

Then `AudioManager.play_sfx(preload("res://audio/jump.wav"))` / `AudioManager.play_music(...)`. For a plain player, `add_node` an `AudioStreamPlayer` with `"properties":{"stream":{"__resource":"res://audio/x.wav"},"bus":"Music"}` (an `attach_script` to a missing file rolls back the whole `scene_batch`, so write scripts first).

Wire UI sliders with `AudioServer.set_bus_volume_db(AudioServer.get_bus_index("UI"), db)`. Keep UI sounds under ~120 ms and distinct per action (hover, press, confirm, cancel, error).

Exit leak: `1 resources still in use at exit` / `ObjectDB instances were leaked` appears whenever a bounded run ends mid-playback (always with looping music). It is harmless: the tools report it as severity `info` and it never flips `ok`. To silence it, raise `run_project.py --quit-after` past a short effect, or `stop()` and set `stream = null` a few frames before the run ends.

## 5. No-hearing checklist

1. Generate; keep the printed `expect`.
2. `inspect_audio` with that `expect`: exit 0, or the message names the wrong number.
3. `--import`, `set_import_options` for loops, `inspect_audio` again to confirm `import_loop`.
4. `run_project.py` (or a scenario) so a missing `.import`, wrong bus name or null stream surfaces as a diagnostic instead of silence.
