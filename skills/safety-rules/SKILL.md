---
name: safety-rules
description: Mandatory rules for any work touching GB Studio 4.x projects (.gbsproj, .gbsres, GBVM scripts, GBDK C plugins) or the GB hardware. Hard limits, tool priority, decision tree, and the feedback loop every change must follow. Load this whenever the conversation involves scenes, actors, triggers, variables, sprites, build_rom, or any GB Studio resource file.
---

# Safety Rules — GB Studio 4.x project work

These rules apply to any task that mutates a GB Studio project, writes GBVM, generates assets, or runs the build / emulator loop. Violating any item is an error — do not explain, do not compromise.

## Hard Rules (Never Do)

1. **Do not use Write / Edit to directly modify GB Studio project resource files.** This includes the top-level `.gbsproj` and every `.gbsres` file under `project/`. All changes go through MCP tools (`patch_script` / `set_variable` / `create_*` / `delete_*` / `set_start_scene`). The only exception is the initial write performed by an MCP tool when bootstrapping a project.
2. **Do not use Write / Edit to modify any engine-managed file under `<project>/gbsproj/`** (`.gbsproj`, `.gbsres`, `scenes/`, `backgrounds/`, `sprites/`, `scripts/`, `variables.gbsres`, `settings.gbsres`, etc.).
3. **Do not use `int32_t` / `long` / `long long` / `float` / `double` in GBDK C code.** GB is an 8-bit CPU (Sharp LR35902); these types make SDCC emit bloated and slow code. Use `uint8_t` / `int8_t` / `uint16_t` / `int16_t`.
4. **Do not assume a banked function pointer can be called like an ordinary function pointer.** Cross-bank calls must go through GBDK's `BANKREF` + `BANK()` mechanism, the `__banked` calling convention, or a trampoline.
5. **Do not exceed scene resource limits.** GB Studio 4.x: `MAX_ACTORS=20` (small-size scene `MAX_ACTORS_SMALL=10`), `MAX_TRIGGERS=30`, `MAX_PROJECTILES=5`. Hardware: `MAX_ONSCREEN=10` per scanline, 40 sprites global. Actors, the player, and UI sprites share the same hardware pool.
6. **Do not use more than 4 shades of grey in DMG mode.** In CGB mode each BG/OBJ palette holds at most 4 colours (including transparent), with up to 8 BG palettes + 8 OBJ palettes.
7. **Do not assume `.gbsproj` field order / indentation / formatting is stable.** All reads and writes must go through JSON parsing — no string replace.
8. **Do not guess at fixes after a compile failure.** Run `read_compile_log` and read the full log to pinpoint the exact error line before touching anything.
9. **Do not put text that exceeds the per-line character limit directly into dialogue.** English defaults to ~18 chars/line; Chinese requires width calculation against the custom font (see the `dialogue-and-ui` skill). Estimate width before generating dialogue.
10. **Do not declare "modification complete" without a `build_rom` verification.** Structured modifications must be followed by a passing build; visual changes must additionally be screenshot-compared.
11. **Do not change scene type without user confirmation** (`topdown` / `platform` / `adventure` / `pointnclick` / `shmup` / `logo`) — it resets a large amount of engine configuration.
12. **Do not treat GB Studio variables as 32-bit.** GBS variables are 16-bit signed integers (-32768..32767) and overflow silently. Before designing a scoring algorithm, verify intermediate values stay in range.
13. **Do not do heavy loops / complex arithmetic inside GBVM scripts.** GBVM is an event interpreter; each event costs far more than native code. Push heavy computation into a C plugin or drive it via an event table.
14. **Do not generate "placeholder `.gbsproj` content".** When bootstrapping a new project, the GB Studio GUI is responsible for the initial `.gbsproj` skeleton; the MCP layer only fills in resources after that exists.

---

## Decision Tree (When in Doubt)

| Situation | First step |
|---|---|
| Unsure about GBVM event semantics | Read the `gbvm-scripting` skill |
| Unsure whether a `.gbsproj` field can be changed | Do not edit the file directly; use the matching MCP tool. If the tool refuses, ask the user |
| Compile failure | `read_compile_log`, do not guess |
| Visual anomaly / unsure about UI effect | `run_emulator(durationMs to reach target scene)` → `screenshot` → read the image, then decide |
| Chinese / CJK text / font / text overflow | Read the `dialogue-and-ui` skill |
| Scene-level structural change (add / delete / change type / resize) | Ask the user to confirm the plan first — do not decide unilaterally |
| Actor / Trigger-level change | `read_scene` to inspect current state first, then design the patch |
| Need a new scene / actor / trigger / variable / custom event | Use `create_scene` / `create_actor` / `create_trigger` / `create_variable` / `create_custom_event`. Switching the start scene → `set_start_scene`. Do NOT edit `.gbsres` files directly. |
| Need to remove a scene / actor / trigger / custom event / variable | Use the matching `delete_*` tool. They scan the whole project for references first; if any are found the call refuses unless you pass `force: true`. `delete_scene` additionally refuses (with no force-bypass) when the target is the current `startSceneId` — switch the start scene first. |
| Fuzzy memory of upstream GB Studio / GBDK API | Do not fabricate. Check `https://github.com/chrismaltby/gb-studio` or `https://gbdk-2020.github.io/gbdk-2020/docs/api/`; if not found, mark TODO |
| Same operation fails twice in a row | Stop, report to the user, do not try a third time |

---

## Tool Priority

Choose in this order:

1. **Read GB Studio project state**: `list_scenes` / `read_scene` / `list_actors` / `read_script` / `read_compile_log`
2. **Write GB Studio project**: `patch_script` / `set_variable` / `create_scene` / `create_actor` / `create_trigger` / `create_variable` / `create_custom_event` / `set_start_scene` / `delete_scene` / `delete_actor` / `delete_trigger` / `delete_custom_event` / `delete_variable`
3. **Generate art**: `generate_sprite` / `convert_image_to_sprite` (sprite-only; backgrounds and tilesets stay GUI-side)
4. **Verify changes**: `build_rom` → `run_emulator` → `screenshot`
5. **Business data / non-engine files**: `Read` / `Edit` / `Write` (only for files outside the `<project>/gbsproj/` tree)
6. **Clarify with the user**: `AskUserQuestion` — only for structural decisions (scene type, algorithm design, art style). Decide small things yourself.
7. **Code search**: `Grep` / `Glob`; spawn the Explore sub-agent for cross-project search.

---

## GB Hardware Quick Reference

| Item | Value |
|---|---|
| CPU | Sharp LR35902 @ 4.194304 MHz, 8-bit |
| WRAM | 8 KB (DMG) / 32 KB banked (CGB) |
| VRAM | 8 KB (DMG) / 16 KB banked (CGB) |
| Screen resolution | 160×144 px (20×18 tiles visible) |
| Tile size | 8×8 px |
| BG tile capacity | DMG: 192 (`MAX_BACKGROUND_TILES`), CGB: 384 (spanning 2 VRAM banks) |
| Sprites (hardware) | Up to 40 total, ≤10 per scanline (`MAX_ONSCREEN`) |
| Sprite size | 8×8 or 8×16 (global mode) |
| Max scene size | 2040 px per side (`SCENE_MAX_SIZE_PX`) = 255 tiles |
| Scene actor cap | 20 (`MAX_ACTORS`); 10 for small scenes (`MAX_ACTORS_SMALL`) |
| Scene trigger cap | 30 (`MAX_TRIGGERS`) |
| Script nesting depth | 5 (`MAX_NESTED_SCRIPT_DEPTH`) |
| BG map | 32×32 tiles, scrollable |
| Palettes (DMG) | BGP / OBP0 / OBP1, 4 greys each |
| Palettes (CGB) | 8 BG palettes + 8 OBJ palettes, 4 colours each |
| Sprite transparent colour | Palette index 0 is transparent for sprites |
| Cart ROM | GB Studio default MBC5, up to 8 MB |
| Cart SRAM | Typically 32 KB (MBC5) |
| GBS variable width | 16-bit signed (-32768..32767) |

Constants sourced from `chrismaltby/gb-studio` `src/consts.ts` (develop), mirrored in the MCP server's `gbsproj/schema.ts`.

---

## GB Studio 4.x Concept Map

**On-disk layout** (one JSON file per resource, all using the `.gbsres` extension; the top-level `.gbsproj` only stores project metadata — name / author / notes / `_version` — and contains no resource index):

```
<projectRoot>/
  <name>.gbsproj                                 # ProjectMetadataResource only
  project/
    scenes/<sceneSlug>/
      scene.gbsres                               # scene fields (no actors/triggers)
      actors/<actorSlug>.gbsres                  # one file per actor
      triggers/<triggerSlug>.gbsres              # one file per trigger
    notes/<noteSlug>/note.gbsres
    scripts/<scriptSlug>.gbsres                  # customEvents
    palettes/<paletteSlug>.gbsres
    prefabs/actors/<slug>.gbsres
    prefabs/triggers/<slug>.gbsres
    variables.gbsres                             # singleton
    settings.gbsres                              # singleton (project-level)
    user_settings.gbsres                         # singleton (editor-level, separate from settings)
    engine_field_values.gbsres                   # singleton, snake_case
  assets/
    backgrounds/<file>.png + <file>.png.gbsres   # binary + sidecar with matching name
    sprites/<file>.png + <file>.png.gbsres
    tilesets/<file>.png + <file>.png.gbsres
    emotes/<file>.png + <file>.png.gbsres
    avatars/<file>.png + <file>.png.gbsres
    fonts/<file>.png + <file>.png.gbsres
    music/<file>.uge + <file>.uge.gbsres
    sounds/<file>.wav + <file>.wav.gbsres
  plugins/<pluginName>/<type>/...                # plugin resources, same pattern as assets
```

**Key gotchas**:
- Filenames are a slug of `entity.name` (lowercased, spaces → `_`, illegal characters stripped) — not `<id>.gbsres`. Name collisions get `_2` / `_3` suffixes.
- Every resource file is wrapped as `{ "_resourceType": "...", "id": "...", ...rest }`.
- Actors / triggers are standalone files on disk, not embedded in the scene file. A patch tool modifying an actor touches only that actor's file.
- The `.gbsres` for binary resources (images / music) is a sidecar living under `assets/`, not `project/`.
- Resource discovery uses the glob `{project,assets,plugins}/**/*.gbsres`.

**Resource types**:
- **Scene** (`_resourceType: "scene"`): `id`, `type`, `name`, `symbol`, `x`/`y`/`width`/`height`, `backgroundId`, `tilesetId`, `paletteIds[]`, `spritePaletteIds[]`, `actors[]`, `triggers[]`, `collisions`, plus script fields `script`, `playerHit1Script`, `playerHit2Script`, `playerHit3Script`.
- **Actor** (`_resourceType: "actor"`): script fields `script` (onInteract), `startScript` (onInit), `updateScript`, `hit1Script`, `hit2Script`, `hit3Script`. `prefabId` / `prefabScriptOverrides` for prefab reuse.
- **Trigger** (`_resourceType: "trigger"`): script fields `script` (enter), `leaveScript` (leave).
- **ScriptEvent** (recursive): `{ id, command, args?, children? }`. `children` is `Record<string, ScriptEvent[] | undefined>` — if/switch-style events use named branches (e.g. `"true"` / `"false"`), not a single array.
- **Variable**: `{ id, name, symbol, flags? }`. 16-bit signed enforced by the engine; the project file stores no numeric bound. Initial values are set via `VARIABLE_SET_TO_VALUE` events in an init script — they are not a resource field.
- **Script** (= CustomEvent, `_resourceType: "script"`): `id`, `name`, `description`, `variables` (formal params), `actors` (formal params), `script` (event body).

**Addressing conventions**:
- Scene addressed by `id` (UUID); on disk maps to `project/scenes/<slug>/scene.gbsres`. Slug ≠ id — the IO layer maintains the `id → path` map.
- Actor / trigger addressed by `id`; each has its own file at `project/scenes/<sceneSlug>/{actors,triggers}/<slug>.gbsres`.
- Script (= CustomEvent) addressed by `id`; file at `project/scripts/<slug>.gbsres`.
- A script location is the tuple `(ownerType, ownerId..., scriptKey)`, where `ownerType ∈ {actor, trigger, scene, customEvent}`.
- Never address by array index — indices drift under patches, and on disk it is not an array anyway.

---

## Feedback Loop (Mandatory)

```
1. read_*           # read current state
2. patch_*          # structured modification
3. build_rom        # compile-verify
4. If it fails:
     read_compile_log → fix based on the log → back to 2
5. If visual verification is needed:
     run_emulator(durationMs) → screenshot → read image, compare against expectation
6. If the screenshot does not match:
     back to 2, adjust
7. After two consecutive failures:
     Stop, report to the user, do not try a third time
```

**Disallowed behaviours**:
- Skipping `build_rom` after a patch and just saying "done"
- Retrying after a build failure without reading the log
- Looking at a screenshot only to confirm "an image exists" without verifying contents
- Infinite-loop retry

---

## Response Style

- Be concise. If one sentence will do, do not use three.
- Report staged progress: what was done, what decisions were made, what is blocking.
- Stop after a stage and wait for review — do not plough through the whole delivery in one go.
- Reference files in `path:line` format.
- Skill / doc text uses imperatives: "must…", "do not…", "first step…". Avoid mushy phrasing like "should consider" or "might possibly".
- When unsure, write a TODO and state what information is needed. Do not fabricate.
