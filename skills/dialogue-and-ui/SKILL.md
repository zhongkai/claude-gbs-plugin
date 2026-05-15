---
name: dialogue-and-ui
description: Use when writing dialogue text, menus, or any on-screen UI. Covers the 18-char line limit, 3-line dialogue box, menu-vs-scene-jump tradeoffs, pre-flight text sizing, and the Chinese-font stretch goal.
---

# Dialogue and UI

GB Studio's default font is an 8×8 variable-width fixed bitmap (stock Latin-1 set). This skill assumes the stock font. Custom fonts are addressed at the bottom.

## The only rule you must never break

**One line of dialogue in the default font fits about 18 characters.**

The exact limit depends on kerning of the specific glyphs (a line of `i`s fits more than a line of `W`s), but 18 is the safe working ceiling. Go over and the text either wraps unpredictably or clips.

**Before inserting any text event, count characters per line.** If unsure, write it shorter and test.

## Dialogue box

| Property | Value |
|---|---|
| Lines visible at once | 3 |
| Chars per line | ~18 |
| Max chars per `EVENT_TEXT` | ~54 (3×18) — longer text spans multiple boxes |
| Line break | `\n` |
| Box break (new page) | `\n\n` or a second `EVENT_TEXT` |

A single `EVENT_TEXT` can hold several screens separated by `\n\n`; the player presses A to advance between them.

## Menu patterns

There are two ways to offer the player a choice. Pick based on how many options you need.

### A. `EVENT_CHOICE` — 2-way

The common case: two options shown on top of the dialogue text. Returns into a variable (0 or 1, or true/false — verify upstream).

```
EVENT_CHOICE
  text: "At a party, you prefer to:"
  trueText: "Meet new people"
  falseText: "Talk deeply with few"
  variable: lastChoice
```

Keep each option to ≤ 14 chars to leave room for the cursor marker.

### B. `EVENT_MENU` — 3 to 8 options

Opens a dedicated menu UI; returns selection index to a variable.

Use when > 2 options. Cost: one extra dialogue box transition.

### C. Scene-jump "menu"

Place actors / triggers in a scene and let the player walk to one. Use for:
- Save file slots ("walk up to one of three pedestals")
- World-level choices ("choose your starting region")

Avoid for per-question quiz answers — too much input per decision.

For the MBTI demo, use `EVENT_CHOICE` for every question.

## Pre-flight text sizing

A quick procedure before any `patch_script` that inserts text:

1. Split the intended text at `\n` into lines.
2. For each line, count chars. If > 18, reflow.
3. If total lines > 3, split into multiple `EVENT_TEXT` events (or `\n\n` page breaks within one).
4. For `EVENT_CHOICE`, keep the prompt to 2 lines so the 2 options fit in the last line pair.

### Example — sizing check

Intended:
```
You walk into a room full of strangers. What do you do?
```
Count: 56 chars, single paragraph.

Reflow to 3 lines:
```
You walk into a room      (23)  X — too long
```

Correct reflow:
```
You walk into a room      (20)  X — still too long
```

18-safe:
```
You enter a room of   (19)  X
```

Actually safe:
```
A room of strangers.    (19)  X
```

Better:
```
Room full of strangers.\n       (22 on line 1)   X
```

Truly 18-safe options:
```
A room of people.    (17)  OK
Strangers. What      (15)  OK
do you do?           (10)  OK
```

Combined:
```
A room of people.\nStrangers. What\ndo you do?
```

Point: **plan the reflow in the tool call, not live**. If it doesn't fit, shorten the wording.

## Typing style for Game Boy

- **Imperatives > questions where possible**: "Pick one." vs "Which one do you pick?"
- **Contractions liberally**: "you're" vs "you are" saves 2 chars.
- **Break on natural phrase boundaries**, not mid-word, not mid-article ("a\npet" — avoid).
- **No em-dashes / curly quotes** — some fonts don't have them. Stick to ASCII punctuation.

## Diagnosing text overflow

Symptom → cause:
- Text wraps to an unexpected spot → you exceeded 18 chars on that line.
- Last word "cut off" → probably a glyph with wide kerning (W/M); shorten by one char.
- Menu option labels truncated → option > ~14 chars in `EVENT_CHOICE`.
- Entire dialogue box renders blank → empty string in `text` arg, or non-ASCII char the stock font lacks.

Use `screenshot` to verify actual rendering after build.

## Chinese / CJK stretch goal

The MBTI demo v1 is **English only** (decision: prioritize ship-ability). CJK support is a stretch goal. Three implementation paths, summarised:

**Option A — Full custom font tile bank**
- Pre-generate all ~200–400 unique Han characters used in the demo as 8×8 or 8×16 tiles.
- Load them into VRAM as a replacement font.
- Cost: font tiles compete with the 192 (DMG) / 384 (CGB) tile budget. Background art must be trimmed to fit.
- Tooling: `gbdk-2020` font generation or a custom script that reads design JSONs, collects unique chars, emits a tileset.

**Option B — Font paging**
- Keep a small core set (~64 high-frequency chars) always resident.
- Stream per-screen chars from a ROM bank into VRAM between text events.
- Cost: requires a C plugin for the paging logic (GBVM alone can't manage VRAM). Extra complexity.
- Benefit: backgrounds retain art budget.

**Option C — English first (current choice)**
- Default font ships with the engine. No art budget impact.
- Trade-off: limited audience reach for a MBTI-in-Chinese demo.

Pick at start of Chinese-localisation round. Until then, keep all copy ASCII, and don't sprinkle Chinese in design JSONs.
