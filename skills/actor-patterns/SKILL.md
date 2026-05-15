---
name: actor-patterns
description: Use when adding an actor with specific behaviour (static NPC, direction-gated, moving, conditionally visible). Each pattern ships as a concrete patch_script operations sequence.
---

# Actor patterns

Every actor has six script slots (upstream `ActorResource`):
- `script` → on interact (player presses A facing this actor)
- `startScript` → once when scene starts
- `updateScript` → every frame while active — **keep tiny**
- `hit1Script` / `hit2Script` / `hit3Script` → on collision with matching `collisionGroup`

All examples below are ready-to-use `patch_script` operations, assuming you've resolved the actor's `id` via `list_actors`. Generate fresh UUIDs for each event `id` on insert.

## Pattern 1 — Static NPC dialogue

NPC stands still, says one line on interact. Lightest possible actor.

**Locator**:
```json
{ "ownerType": "actor", "sceneId": "<sceneId>", "actorId": "<actorId>", "scriptKey": "script" }
```

**Operations** (assumes actor's interact script is currently empty):
```json
[
  {
    "op": "insert",
    "index": 0,
    "event": {
      "id": "<uuid-1>",
      "command": "EVENT_ACTOR_SET_DIRECTION",
      "args": { "actorId": "$self$", "direction": "down" }
    }
  },
  {
    "op": "insert",
    "index": 1,
    "event": {
      "id": "<uuid-2>",
      "command": "EVENT_TEXT",
      "args": { "text": "Welcome, traveller." }
    }
  }
]
```

`"$self$"` is GB Studio's convention for "this actor". `"$player$"` targets the player. Verify in upstream if unsure.

## Pattern 2 — Direction-gated interact

NPC only speaks when the player faces them from below (so the player has to approach deliberately).

```json
[
  {
    "op": "insert",
    "index": 0,
    "event": {
      "id": "<uuid-1>",
      "command": "EVENT_IF_ACTOR_DIRECTION",
      "args": { "actorId": "$player$", "direction": "up" },
      "children": {
        "true": [
          {
            "id": "<uuid-2>",
            "command": "EVENT_TEXT",
            "args": { "text": "You finally came." }
          }
        ],
        "false": []
      }
    }
  }
]
```

`EVENT_IF_ACTOR_DIRECTION` is the conventional name; verify against `src/lib/events/eventIfActorDirection.ts`.

## Pattern 3 — Moving NPC (patrol)

Actor walks back and forth along a line. Goes in `updateScript`.

**Locator**:
```json
{ "ownerType": "actor", "sceneId": "<sceneId>", "actorId": "<actorId>", "scriptKey": "updateScript" }
```

**Operations** (full replacement of update script body):
```json
[
  {
    "op": "insert",
    "index": 0,
    "event": {
      "id": "<uuid-1>",
      "command": "EVENT_ACTOR_MOVE_TO",
      "args": {
        "actorId": "$self$",
        "x": { "type": "number", "value": 8 },
        "y": { "type": "number", "value": 5 },
        "moveType": "horizontal",
        "useCollisions": true
      }
    }
  },
  {
    "op": "insert",
    "index": 1,
    "event": {
      "id": "<uuid-2>",
      "command": "EVENT_ACTOR_MOVE_TO",
      "args": {
        "actorId": "$self$",
        "x": { "type": "number", "value": 12 },
        "y": { "type": "number", "value": 5 },
        "moveType": "horizontal",
        "useCollisions": true
      }
    }
  }
]
```

GB Studio `MOVE_TO` by default **awaits** completion, so the update script runs one leg per full traversal. Without a `wait: false` style flag, this naturally loops each frame.

**Caveat**: move events inside `updateScript` are a common source of performance issues. Prefer a trigger-driven or event-driven walker for complex patrols.

## Pattern 4 — Conditional visibility

Actor hides until a quest variable is set, then appears.

**Locator**: actor's `startScript`.

```json
[
  {
    "op": "insert",
    "index": 0,
    "event": {
      "id": "<uuid-1>",
      "command": "EVENT_IF_VALUE",
      "args": {
        "variable": "questComplete",
        "operator": "==",
        "comparator": 0
      },
      "children": {
        "true": [
          {
            "id": "<uuid-2>",
            "command": "EVENT_ACTOR_HIDE",
            "args": { "actorId": "$self$" }
          },
          {
            "id": "<uuid-3>",
            "command": "EVENT_ACTOR_DEACTIVATE",
            "args": { "actorId": "$self$" }
          }
        ],
        "false": []
      }
    }
  }
]
```

`DEACTIVATE` stops the update script from running — important if the hidden actor has one, otherwise it still consumes frame time.

## Validation checklist before patch_script

- [ ] Scene has room: `list_actors(sceneId).length < MAX_ACTORS` (20, or 10 in small scenes)
- [ ] Actor's `id` came from `list_actors`, not guessed
- [ ] Fresh UUID for every new event `id`
- [ ] Script nesting ≤ 5 levels
- [ ] Text stays within 18-char-per-line limit (see `dialogue-and-ui` skill)
- [ ] Updated actor doesn't collide with player spawn
- [ ] Build + emulator verification queued after patch
