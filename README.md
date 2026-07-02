# Sillage Skills

[![skills.sh](https://skills.sh/b/espaidon/sillage-skills)](https://skills.sh/espaidon/sillage-skills)

> Install Sillage Skills and run your entire go-to-market motion — targeting, agents, signals, content — from a natural-language conversation with Claude, Cursor, or any MCP-capable assistant.

Without skills, an assistant connected to the [Sillage MCP](https://api.getsillage.com/api/mcp/v2) knows _which_ tools exist but has to guess _how_ to use them well: how to turn a vague "we sell to sales teams" into a precise persona, which keywords actually catch buying intent, what to track. With Sillage Skills installed, it already knows. You just describe what you want.

```
"Set up my Sillage workspace."
  → grill-me interviews you about what you sell, who buys, and the signals that matter
  → expands your answers into a precise persona + high-signal agents
  → writes them through the Sillage MCP — no app, no forms
```

---

## 1. Connect the Sillage MCP

The skills drive the [Sillage MCP server](https://api.getsillage.com/api/mcp/v2). Connect it once; log in with your Sillage account when prompted.

```bash
claude mcp add --transport http sillage https://api.getsillage.com/api/mcp/v2
```

Any MCP-compatible client works — point it at the URL above and complete the Sillage login. No API key required. Each connection is scoped to one workspace.

---

## 2. Install the skills

```bash
npx skills add espaidon/sillage-skills
```

This drops a `.skills/` directory into your project that your assistant reads automatically. Commit it so every teammate and CI environment shares the same setup with no separate install step.

---

## 3. Verify

Ask your assistant:

```
What Sillage skills do you have available?
```

It will list the skills below and what each one does.

---

## Skills

| Skill                                             | What it does                                                                                                                                                                                                                                                                                             |
| ------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [`grill-me`](./skills/grill-me)                   | Interviews you to extract and **expand** your go-to-market context — what you sell, who buys, the buying signals that matter — then writes a precise persona and proposes high-signal agents through the MCP. The piece the tools can't do for you: turning sparse human answers into sharp targeting.   |
| [`manage-workspace`](./skills/manage-workspace)   | The write and edit engine. Stands up a new workspace and **safely edits a live one** — the setup loop (persona → accounts → coverage → agents → runs) and the edit loop (read → diff → patch), respecting the write rules (persona replaces whole, the target list only appends) that keep changes safe. |
| [`how-sillage-works`](./skills/how-sillage-works) | The HELP skill. Explains the **whole logic** of a workspace and the **glossary** behind it — how persona, target accounts, coverage, watchlists, agents, signal runs, detections and content connect, and what every term means. The mental model the other skills assume you already have.              |

_More skills land here over time. Each one teaches the assistant a workflow the raw tools assume you already know._

---

## Supported assistants & editors

Any tool that follows the [skills.sh](https://skills.sh) standard works with the same install command: **Claude Code**, **Cursor**, **Claude Desktop** (via MCP), **GitHub Copilot** (agent mode), **Windsurf**, **Codex CLI**, and more. The `.skills/` directory is the universal integration point — no per-editor configuration.

---

## Contributing

These skills are open. Found a sharper question to ask, a better keyword-expansion pattern, a missing signal type? Open an issue or a PR. The skills are plain Markdown — read [`skills/grill-me/SKILL.md`](./skills/grill-me/SKILL.md) to see the shape.

## License

[MIT](./LICENSE)
