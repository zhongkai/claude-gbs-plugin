# claude-gbs-plugin

Claude Code plugin for GB Studio 4.x. Gives the model:

- **MCP tools** for structured project editing (`patch_script`, `set_variable`, `create_scene`, `create_actor`, `create_trigger`, `create_variable`, `create_custom_event`, `set_start_scene`, `delete_*`, `build_rom`, `run_emulator`, `screenshot`, `generate_sprite`, `convert_image_to_sprite`, …) backed by the upstream `chrismaltby/gb-studio` 4.x resource schema. MCP server lives in the [`@gbs-toolkit/mcp-server`](https://www.npmjs.com/package/@gbs-toolkit/mcp-server) npm package and is launched on demand via `npx`.
- **A `safety-rules` skill** that auto-loads whenever the conversation touches GB Studio resources — encodes hardware limits, the 4.x per-resource on-disk layout, the mandatory build / emulator / screenshot feedback loop, and the things the model must never do (e.g. directly editing `.gbsres` with `Write`).
- **Six topic skills** the model can pull in on demand: `gbvm-scripting`, `dialogue-and-ui`, `sprite-generation`, `actor-patterns`, `scene-design`, `debugging-rom`.

## Install

The plugin is distributed through a private marketplace. From any Claude Code session:

```
/plugin marketplace add https://github.com/zhongkai/claude-gbs-marketplace
/plugin install gbs@gbs-toolkit
```

(Or `/plugin marketplace add` with a local path during development.)

## Configure

The plugin's `.mcp.json` references several env vars; set whichever you need before launching Claude Code:

| Variable | Required | Purpose |
|---|---|---|
| `GBS_PROJECT_ROOT` | yes | Directory containing `<name>.gbsproj` |
| `SPRITE_PROVIDER` | optional | `openai` (default) / `gemini` / `replicate` / `fal` |
| `OPENAI_API_KEY` / `GEMINI_API_KEY` / `REPLICATE_API_TOKEN` / `FAL_KEY` | conditional | Required for the matching provider |
| `GBS_CLI_PATH` | optional | Override probing for `gb-studio-cli.js` |
| `GBS_MGBA_RUNNER` | optional | Path to a libmgba-linked binary used by `run_emulator` |

If your client doesn't expand `${VAR}` in the plugin's `.mcp.json` at runtime, override the values in your project-level `.mcp.json` (project-level config beats plugin-level config).

## Develop

Skills are plain Markdown under `skills/<name>/SKILL.md` with YAML frontmatter — edit and reload. The MCP server source lives in a sibling repo at [`zhongkai/gbs-toolkit-mcp-server`](https://github.com/zhongkai/gbs-toolkit-mcp-server) (TODO if/when split out) and is published to npm as `@gbs-toolkit/mcp-server`.

To use a local checkout of the MCP server instead of the npm release, change `.mcp.json` `args` to point at a local path:

```json
{
  "mcpServers": {
    "gbs": {
      "command": "node",
      "args": ["/abs/path/to/mcp-server/dist/index.js"],
      "env": { "GBS_PROJECT_ROOT": "..." }
    }
  }
}
```

## License

MIT
