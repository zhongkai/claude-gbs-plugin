---
name: scene-design
description: Use when adding / splitting / sizing scenes, choosing a scene type, or laying out actors and triggers. Covers the hardware budget, upstream-enforced scene limits, and reuse patterns that avoid scene explosion.
---

# Scene design

A scene is the unit of level state in GB Studio: one background, one collision map, up to `MAX_ACTORS` actors, up to `MAX_TRIGGERS` triggers, plus scene-level scripts.

## Hard limits (from upstream `src/consts.ts`)

| Limit | Value | Notes |
|---|---|---|
| `MAX_ACTORS` | 20 | Per scene |
| `MAX_ACTORS_SMALL` | 10 | Enforced instead when scene dimension < threshold |
| `MAX_TRIGGERS` | 30 | Per scene |
| `MAX_ONSCREEN` | 10 | Hardware sprites per scanline — exceeding makes sprites disappear |
| `MAX_PROJECTILES` | 5 | Bullets / thrown objects |
| `MAX_BACKGROUND_TILES_DMG` | 192 | Distinct BG tiles in this scene |
| `MAX_BACKGROUND_TILES_CGB` | 384 | CGB: 2 VRAM banks |
| `SCENE_MAX_SIZE_PX` | 2040 | 255 tiles per side |
| Viewport | 20×18 tiles = 160×144 px | What the player sees |
| `MAX_NESTED_SCRIPT_DEPTH` | 5 | Any scope (scene / actor / trigger scripts) |

## Scene types

| `type` | Use when | Notes |
|---|---|---|
| `topdown` | Adventure / exploration | Default choice for MBTI demo |
| `adventure` | Classic adventure (interact-heavy) | Similar to topdown |
| `platform` | Side-scroller | Entirely different physics — don't switch casually |
| `shmup` | Shoot'em up | Scroll-locked |
| `pointnclick` | Cursor interaction | Uses `cursor` sprite animation |
| `logo` | Title / splash | Minimal gameplay, usually first scene |

**Do not change an existing scene's `type` without explicit user approval** — it resets collisions, player config, and much else (CLAUDE.md hard rule #11).

## When to split a scene vs reuse one

### Split when
- Backgrounds differ visually
- Actors > 20 (or > 10 in small scene)
- Triggers > 30
- Distinct game-state / world-region boundary

### Reuse when
- Same layout, different textual content (quiz questions, NPC dialogue variants, results screens)
- Pattern: one scene + a cursor variable, dispatch content via `EVENT_SWITCH` on the cursor

**The MBTI demo uses reuse heavily**: one `quiz` scene renders 20 questions sequentially by cycling a `currentQuestion` variable and switching the scene to itself (re-running onInit) between questions. This avoids a 20-scene explosion.

## Reuse pattern: cursor-driven scene

```
Scene.script (onInit):
  SWITCH on currentQuestion:
    case 0: <show Q1 text, collect choice, update score>
    case 1: <show Q2 text, collect choice, update score>
    ...
    case N-1: <show QN text, collect choice, update score>
  INCREMENT currentQuestion
  IF currentQuestion >= N:
    SWITCH_SCENE → results
  ELSE:
    SWITCH_SCENE → this scene   (re-runs init with new cursor)
```

Alternative: factor the per-question logic into a **CustomEvent** with the question number as an arg, and call it with the current cursor. Cleaner for > ~5 cases.

## Budgeting tiles

The 192-tile DMG budget is shared across background and, if you use custom fonts, the font itself. Rough planning:

- Static background art: 100–150 tiles
- UI chrome (borders, dialogue boxes — GB Studio supplies defaults): ~20
- Remaining for on-scene decorations / variants: 20–70

If you need more distinct tiles, either target CGB (384) or split the scene at a natural boundary.

## Common layout pitfalls

- **Placing an actor where the player spawns** → overlap glitch. Always leave 1 tile clear around the player start.
- **Trigger that covers the entire scene** → fires on every step. Use `leaveScript` for one-shot effects.
- **Static NPCs as actors when triggers would do**. An actor that only displays text on interact is still fine — actors are cheap until you hit 20. But if you don't need an animated sprite, a trigger + invisible tile is lighter.
- **Using `EVENT_SCENE_RESET`** when you mean `EVENT_SWITCH_SCENE` to the same scene. `SCENE_RESET` also resets variables by default depending on args; `SWITCH_SCENE` preserves globals.

## Scene id naming

Scene `symbol` fields become C identifiers in compiled output. Stick to `lower_snake_case` and keep them short (≤24 chars). `symbol` is distinct from `name` — the user-facing label — and from `id` — the UUID used for all tool calls.
