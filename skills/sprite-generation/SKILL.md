---
name: sprite-generation
description: How to author GB Studio sprites with the plugin's `generate_sprite` and `convert_image_to_sprite` tools — provider env vars, prompt patterns that survive 4-colour quantisation, and how to wire the result into a scene.
---

# Sprite generation

## When to use which tool

| Situation | Tool |
|---|---|
| Spawn a brand-new character from a description | `generate_sprite` |
| User provided a PNG / JPG (e.g. a portrait, a scanned drawing, a sheet exported from Aseprite) | `convert_image_to_sprite` |
| User wants a `multi` / `multi_movement` sprite from a single front-facing pose | NOT SUPPORTED — `convert_image_to_sprite` requires the input PNG to already contain N frames laid out horizontally. Tell the user; suggest `generate_sprite` instead, or have them assemble the sheet manually first. |

`convert_image_to_sprite` runs entirely locally — no API key needed, no network. Prefer it whenever the user can supply art.

## Required environment variables (only for `generate_sprite`)

| Provider | Env var | Notes |
|---|---|---|
| `openai` (default) | `OPENAI_API_KEY` | Uses `gpt-image-1`, returns base64 PNG. |
| `gemini` | `GEMINI_API_KEY` (or `GOOGLE_API_KEY`) | Model `gemini-2.5-flash-image-preview` (a.k.a. nano-banana). |
| `replicate` | `REPLICATE_API_TOKEN` | Default model `lucataco/sdxl-pixel-art`. Override via `REPLICATE_MODEL`. |
| `fal` | `FAL_KEY` | Uses `fal-ai/flux-lora` with a pixel-art LoRA. Override model via `FAL_MODEL`. |

Switch the default provider with `SPRITE_PROVIDER=<name>`. Per-call override: pass `provider` to the tool. Missing key → tool errors with a clear message; `convert_image_to_sprite` is unaffected.

## Animation types

| Type | PNG dimensions | Frames generated | When |
|---|---|---|---|
| `fixed` | 16×16 | 1 | Static NPCs, signposts, items, anything that doesn't face directions. |
| `multi` | 48×16 | 3 (up / right / down; left mirrors right) | NPCs that turn but don't walk. |
| `multi_movement` | 96×16 | 6 (idle + walk per direction) | The player character or a walking NPC. |

The plugin always lays frames out horizontally and writes a `.gbsres` that mirrors the structure GB Studio's GUI emits — open the project after running the tool and the sprite shows up in the picker without further edits.

## Prompt patterns that survive 4-colour quantisation

The pipeline produces only 4 distinct shades. Anything subtle is lost. Before calling `generate_sprite`, rewrite the user's description to lean on:

- **One foreground colour, one accent colour at most** — "white robe, dark belt" survives; "iridescent rainbow cape" does not.
- **High-contrast outline** — say "thick black outline" or "bold line art".
- **No gradients, no soft shading** — "flat colours, no shading" in the prompt.
- **Centred, full body, plain background** — the cropper assumes the figure is centred; cluttered scenes get quantised into mush.
- **Distinct silhouette** — round head + rectangular body reads at 16×16; stick limbs disappear.

The tool already prepends a hard preamble enforcing pixel-art style and the DMG palette, so the user's prompt only needs the *subject*.

Bad: `"a friendly wizard with a flowing iridescent cloak and detailed runes glowing on his staff"`
Good: `"chubby wizard, tall pointed hat, thick outline, plain robe"`

## End-to-end flow

```
User: "give the result scene 16 unique character sprites, one per MBTI type"

1. for each type in [INTJ, INTP, ENTJ, ...]:
     generate_sprite({
       prompt: "<one-line silhouette + accessory description>",
       name: `mbti_<type>`,
       animationType: "fixed"
     })
   → record spriteSheetId

2. read_scene(resultSceneId) → find the actor for each type's branch
3. patch_script the EVENT_SET_VARIABLE / EVENT_SHOW_ACTOR scripts to reference
   the new sprite ids, OR call create_actor + delete_actor as needed.
4. build_rom — must pass.
5. run_emulator with input that walks through to the result, screenshot,
   verify the sprite renders.
```

For `multi` / `multi_movement` you typically don't loop — generate one good walking-character sheet for the player and reuse it. For NPCs use `fixed`.

## Hardware reminders

- 10 sprites visible per scanline — per scene, keep simultaneously-active actors below ~6 to leave headroom for projectiles and UI sprites.
- Each frame is 2 hardware tiles (16 wide × 16 tall, OBJ in 8×16 mode). A `multi_movement` sheet costs 12 sprite tiles in VRAM.
- Sprite palette index 0 is treated as **transparent** by the engine — the pipeline already enforces this; do not override.
- Sprites use OBP0 by default. If the user is on CGB and wants a specific OBJ palette, edit the sprite resource via the GB Studio GUI after generation; this round of the plugin doesn't expose CGB palette assignment as an MCP tool.
