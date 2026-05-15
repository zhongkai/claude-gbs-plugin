---
name: gbvm-scripting
description: Use when writing or editing GB Studio scripts (ScriptEvent[]). Covers what GBVM is, the shape of events, the common command whitelist, how children/branches work, and how to discover commands you don't recognise.
---

# GBVM scripting

GB Studio 4.x compiles visual scripts to **GBVM**: a byte-code event interpreter that runs inside the compiled ROM. Each script is an ordered `ScriptEvent[]`. LLMs never "write GBVM assembly" directly — you write structured events and the compiler handles the rest.

## ScriptEvent shape (canonical)

```ts
interface ScriptEvent {
  id: string;                                      // UUID, unique per event instance
  command: string;                                 // e.g. "EVENT_TEXT"
  args?: Record<string, unknown>;                  // command-specific payload
  children?: Record<string, ScriptEvent[] | undefined>;  // named branches
}
```

- `id` must be unique within the enclosing array. Generate with a UUID when inserting.
- `command` is an open string in upstream (**no enum**). The engine dispatches on it at compile time; a typo becomes a compile error.
- `children` is a **Record**, not an array. Composite events (if, switch, group) use named branches:
  - `EVENT_IF_VALUE` → `children: { true: [...], false: [...] }`
  - `EVENT_SWITCH` → `children: { false: [...], true0: [...], true1: [...], ... }` — fallback branch is `false`, matching cases use `true<N>` keyed by `args.value<N>`
  - `EVENT_GROUP` → `children: { script: [...] }`

## Discovering a command

I do not maintain a complete list of GBVM commands. The authoritative source is:

```
chrismaltby/gb-studio src/lib/events/event*.ts    # one TS module per command
```

Each `event*.ts` file exports an `id` (the command string), `fields` (its args spec), `compile` (the emitter), and usually `autoLabel`. When you need a command you don't have here, first check that directory and **cite the upstream file path** in any non-trivial change so reviewers can verify.

## Common command whitelist

The following commands cover >90% of typical scripting. Args below are the **named fields** the event expects — exact shapes may be nested (e.g. `value: { type: "number", value: 5 }` for a constant input). Verify against upstream before committing unusual combinations.

| Command | Purpose | Key args |
|---|---|---|
| `EVENT_TEXT` | Show dialogue box | `text` (string, `\n` for line break), `avatarId?` |
| `EVENT_CHOICE` | Text + 2-option menu, writes choice to a variable | `variable`, `trueText`, `falseText`. Picking **trueText** (upper option) writes **1**; picking **falseText** (lower option) writes **0**. Source: `src/lib/events/eventTextChoice.js`, helper `scriptBuilder.ts::textChoice` uses `.UI_MENU_LAST_0`. |
| `EVENT_MENU` | Multi-option menu (2–8 items) | `variable`, `items[]`, `layout` |
| `EVENT_IF_VALUE` | Compare a variable to a number/variable. **Marked `deprecated: true` upstream but still functional**; new code may prefer `EVENT_IF` (general expression). | `variable`, `operator`, `comparator`; branches in `children.true` / `children.false` |
| `EVENT_SWITCH` | N-way branch on a variable. Field shape is `{ variable, choices: N, value0, value1, …, valueN-1 }`; matching branches live at `children.true0`, `children.true1`, …, with the fallback at `children.false` (NOT `default`). Source: `src/lib/events/eventSwitch.js`. | see right column |
| `EVENT_SET_VALUE` | Assign literal/expression to variable. Args: `variable`, `value: { type: "number", value: N }` (ScriptValue union). Note the command id is `EVENT_SET_VALUE`, NOT `EVENT_VARIABLE_SET_TO_VALUE` — the latter is the older name still seen in legacy projects. Source: `src/lib/events/eventVariableSetToValue.js`. |
| `EVENT_INC_VALUE` / `EVENT_DEC_VALUE` | ±1 on a variable. Note the ids are `EVENT_INC_VALUE` / `EVENT_DEC_VALUE`, NOT the historic `EVENT_VARIABLE_INCREMENT` / `_DECREMENT`. Source: `src/lib/events/eventVariableInc.js` / `eventVariableDec.js`. | `variable` |
| `EVENT_VARIABLE_MATH` | Arithmetic between two operands into a variable | `vectorX`, `operation`, `vectorY`, `clamp?`, `minValue?`, `maxValue?` |
| `EVENT_SWITCH_SCENE` | Jump to another scene | `sceneId`, `x`, `y`, `direction?`, `fadeSpeed?` |
| `EVENT_SCENE_RESET` / `EVENT_RESET_VARIABLES` | Re-run scene init or clear vars | various |
| `EVENT_CALL_CUSTOM_EVENT` | Invoke a reusable script with args | `customEventId`, `__params` |
| `EVENT_GROUP` | Label / group a block (no runtime effect) | `children.script` |
| `EVENT_WAIT` | Sleep | `time` (frames or seconds; verify unit) |
| `EVENT_ACTOR_MOVE_TO` | Move actor (optionally awaits completion) | `actorId`, `x`, `y`, `moveType`, `useCollisions?` |
| `EVENT_ACTOR_SET_DIRECTION` | Face direction | `actorId`, `direction` |
| `EVENT_ACTOR_HIDE` / `EVENT_ACTOR_SHOW` | Visibility | `actorId` |
| `EVENT_ACTOR_ACTIVATE` / `EVENT_ACTOR_DEACTIVATE` | Toggle update script | `actorId` |

**Again — these are common names, not guaranteed. Verify against `src/lib/events/` before using anything you're unsure about.**

## Example 1 — Dialogue with conditional branch

A sign that reacts differently based on whether the player has a key.

```json
[
  {
    "id": "uuid-1",
    "command": "EVENT_IF_VALUE",
    "args": {
      "variable": "hasKey",
      "operator": "==",
      "comparator": 1
    },
    "children": {
      "true": [
        {
          "id": "uuid-2",
          "command": "EVENT_TEXT",
          "args": { "text": "The door is locked.\nUse the key?" }
        }
      ],
      "false": [
        {
          "id": "uuid-3",
          "command": "EVENT_TEXT",
          "args": { "text": "The door is locked.\nYou need a key." }
        }
      ]
    }
  }
]
```

## Example 2 — Counter increment

On a trigger enter, bump a visit counter and gate a reward at 3 visits.

```json
[
  {
    "id": "u1",
    "command": "EVENT_INC_VALUE",
    "args": { "variable": "visitCount" }
  },
  {
    "id": "u2",
    "command": "EVENT_IF_VALUE",
    "args": {
      "variable": "visitCount",
      "operator": ">=",
      "comparator": 3
    },
    "children": {
      "true": [
        {
          "id": "u3",
          "command": "EVENT_TEXT",
          "args": { "text": "You found a hidden gem!" }
        },
        {
          "id": "u4",
          "command": "EVENT_SET_VALUE",
          "args": {
            "variable": "hasGem",
            "value": { "type": "number", "value": 1 }
          }
        }
      ],
      "false": []
    }
  }
]
```

## Example 3 — Scene transition

End of a quiz → jump to the results scene.

```json
[
  {
    "id": "s1",
    "command": "EVENT_TEXT",
    "args": { "text": "Calculating result..." }
  },
  {
    "id": "s2",
    "command": "EVENT_WAIT",
    "args": { "time": 0.8 }
  },
  {
    "id": "s3",
    "command": "EVENT_SWITCH_SCENE",
    "args": {
      "sceneId": "<results-scene-uuid>",
      "x": 10,
      "y": 8,
      "direction": "down",
      "fadeSpeed": 2
    }
  }
]
```

## Anti-patterns

- **Loop in GBVM**. GBVM has no cheap `for` — it's interpreted. For anything > a handful of iterations, drive it from the outside (per-frame update scripts) or factor into an engine plugin. Never loop over "all N questions" inside one init script; use scene reuse + a cursor variable.
- **Deeply nested conditionals**. Upstream caps at `MAX_NESTED_SCRIPT_DEPTH = 5`. If an if/else tree gets deeper than 3, refactor into a custom event or `EVENT_SWITCH` with a numeric state.
- **Duplicating text across branches**. Factor into a custom event with a variable arg — changes in one place.
- **Forgetting to `EVENT_WAIT`** after cross-scene state changes. Some hardware actions aren't instantaneous; when you see flicker or missed input, try a small wait.
- **Using index-based lookup to identify scripts**. Scripts are referenced by the MCP `ScriptOwnerLocator`, not by array index. Don't hand-count indices.

## When to ask for help

If a command's arg shape isn't obvious from upstream or the engine produces a cryptic error, stop and consult `read_compile_log` plus `src/lib/events/<event>.ts`. Don't guess field names — a wrong arg silently compiles to a no-op in some cases.
