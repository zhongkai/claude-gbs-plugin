---
name: debugging-rom
description: Use when a build fails, a ROM boots but shows wrong output, or a screenshot reveals a visual bug. Covers the diagnosis order, common compile errors and what they usually mean, and when to reach for the emulator.
---

# Debugging ROMs

## First rule

**When `build_rom` fails, call `read_compile_log` before doing anything else.** Do not modify code, do not retry the build, do not guess. The log names the file, line, and often the cause.

## Diagnosis order

```
build_rom fails?
  └─ read_compile_log → fix → build_rom → loop
build_rom succeeds, but…

ROM won't boot / black screen?
  └─ run_emulator with durationMs=3000 → screenshot
      ├─ truly black → check start scene assignment in settings
      ├─ corrupted tiles → background / tileset mismatch
      └─ garbled palette → palette id vs scene colorMode

ROM boots, wrong behaviour at runtime?
  └─ run_emulator with scripted inputs to reach the buggy state → screenshot
      ├─ variable value off → read_script on the relevant scope and trace VARIABLE_* events
      ├─ scene stuck → check SWITCH_SCENE args (wrong sceneId?)
      └─ menu wrong option → check EVENT_CHOICE / EVENT_MENU variable assignments

Visual / layout only?
  └─ screenshot → compare against expected design
      ├─ text clipped → 18-char rule (see dialogue-and-ui)
      ├─ sprites missing → per-scanline limit (10) or hit MAX_ACTORS
      └─ wrong colors → DMG vs CGB palette selection
```

**If you're two iterations deep on the same issue, stop and report to the user.** (CLAUDE.md decision tree.)

## Common compile errors

Exact wording depends on the build toolchain (`gb-studio-cli` / SDCC / GBDK make). These patterns cover most cases:

### 1. "Unknown event command"

Cause: `command` string in a `ScriptEvent` doesn't map to any `src/lib/events/event*.ts` file.

Fix:
- Did you typo it? `EVENT_SWITCH_SCENE`, not `EVENT_SWITCH_SCENES`.
- Is the event from a newer GB Studio version than the project targets? Check upstream git log for when the event was introduced.
- Is it a custom-plugin event that the project doesn't have installed?

### 2. "Scene not found: `<uuid>`"

Cause: an `EVENT_SWITCH_SCENE` (or similar) references a scene id that doesn't exist.

Fix:
- `list_scenes` to get current ids.
- Usually the target scene was deleted or its id was copy-pasted stale.

### 3. "Variable `<id>` not declared"

Cause: a script references a variable by id, but the variables.gbsres entry is missing.

Fix:
- Reading a global variable in GB Studio 4.x uses the variable `id` (UUID). The variable must exist in `variables.gbsres`.
- If the tool chain supports implicit creation, this is a warning — but don't rely on it.

### 4. "ROM too large" / bank overflow

Cause: the compiled output exceeds the configured MBC/ROM size.

Fix:
- Reduce background art — largest single contributor.
- Reduce the number of unique scripts (reuse via custom events).
- Bump `cartType` to a larger MBC (default MBC5 already supports up to 8 MB).
- If music is huge, consider trimming.

### 5. "Tile budget exceeded for scene `<id>`"

Cause: the scene's background has more distinct 8×8 tiles than the engine can fit.

Fix:
- Ensure the background image uses ≤ `MAX_BACKGROUND_TILES_DMG` (192) or the CGB limit (384).
- Look for subtle gradients / anti-aliasing in the PNG — each unique pattern becomes a tile.
- Re-export the background with tighter quantisation.

### 6. SDCC-emitted errors (plugin C code)

If the project has a GBDK C plugin and it fails compile:
- `undefined reference` → missing `BANKREF_EXTERN` / `BANKREF` pair.
- `ERROR - type mismatch` → you used `int` when the engine expected `UINT8`; review types. `int` default in SDCC is 16-bit, usually fine, but struct members and pointer arithmetic can surprise you.
- Follow GBDK API reference: https://gbdk-2020.github.io/gbdk-2020/docs/api/

## Visual debugging procedure

When you have a ROM and something looks wrong:

1. **Reproduce with a minimal input sequence**. Write it as the `inputs` arg to `run_emulator`. This makes the bug cheap to re-hit after each fix.
2. **Capture before and after**. Screenshot before the suspect event and again after. Diff in your head: is the state change correct?
3. **Isolate to the smallest scene that fails**. Duplicate the broken scene into a scratch one and strip actors/triggers until the bug disappears — the last thing removed is the culprit.
4. **Check the compile log even on "success"**. Warnings about undefined references or silent truncations flag issues a green build hides.

## When to use `screenshot` vs just reading code

- Reading code is enough for: logic errors, branch problems, variable math.
- Screenshot is **required** for: anything involving UI layout, tile placement, palette choice, sprite count. You cannot verify these without seeing the frame.

## Emulator backend

`run_emulator` drives a bundled libmgba-linked C runner (`gbs-mgba-runner`) — frame-timed button presses plus PNG screenshots. `screenshot` returns the captured PNG as a multimodal image block.

**Why a custom runner instead of `mgba --script`**: stock mGBA (SDL/Qt/perf/rom-test) does NOT accept a startup `--script` flag — Lua is reachable only from the Qt GUI or the CLI debugger. So we link libmgba directly.

**Setup**:
- Build once with the `native/build.sh` script shipped inside the `@gbs-toolkit/mcp-server` package (or check it out from GitHub). It expects a local mGBA source checkout + a cmake build dir containing `libmgba.dylib` (override via `MGBA_SRC` / `MGBA_BUILD`).
- Set `GBS_MGBA_RUNNER` to the absolute path of the resulting binary.
- Input bitmask (GB order): A=1, B=2, Select=4, Start=8, Right=16, Left=32, Up=64, Down=128 — same as the TS tool schema.
