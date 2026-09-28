# PostlyBee for AI agents

Let your AI agent schedule, draft, and publish social media posts through
[PostlyBee](https://postlybee.com): Instagram, Facebook, LinkedIn, X, TikTok,
YouTube, Pinterest, Threads, Bluesky, Discord, Telegram, WordPress, Dev.to,
and more.

This repository packages two ways for agents to use PostlyBee:

- **The `postlybee` skill**: instructions that teach an agent to use the
  [PostlyBee CLI](https://www.npmjs.com/package/postlybee-cli) safely,
  including uploading images and videos from disk.
- **The hosted PostlyBee MCP server** at `https://api.postlybee.com/mcp`, which
  signs in with OAuth and needs nothing installed.

You need a PostlyBee account with the social channels you want to post to
already connected. MCP and API access are included in every paid plan, with a
7-day free trial.

## Install

### Any agent that supports skills

Works with Claude Code, Codex, Cursor, Gemini CLI, Kimi Code CLI, OpenClaw,
Hermes Agent, GitHub Copilot, and
[many others](https://github.com/vercel-labs/skills#supported-agents):

```bash
npx skills add JimBob-indie/postlybee-agent
```

Add `-a <agent>` to pick an agent, for example `-a codex`, and `-g` to install
for all your projects.

### Claude Code plugin

Installs the skill and the PostlyBee MCP server together:

```text
/plugin marketplace add JimBob-indie/postlybee-agent
/plugin install postlybee@postlybee
```

Run `/mcp` and authenticate `postlybee` to sign in.

### Gemini CLI extension

Installs the skill and the PostlyBee MCP server together:

```bash
gemini extensions install https://github.com/JimBob-indie/postlybee-agent
```

Then run `/mcp auth postlybee` inside Gemini CLI to sign in.

### MCP only

Add `https://api.postlybee.com/mcp` to any client that supports Streamable HTTP
MCP servers. [`mcp.json`](mcp.json) has a ready-made entry. Setup guides:

- [Claude](https://postlybee.com/agents/claude)
- [ChatGPT](https://postlybee.com/agents/chatgpt)
- [Claude Code](https://postlybee.com/agents/claude-code)
- [Codex](https://postlybee.com/agents/codex)
- [Cursor](https://postlybee.com/agents/cursor)
- [Gemini CLI](https://postlybee.com/agents/gemini)

## Sign in for the CLI

The skill uses the PostlyBee CLI. Create a `pbk_` API token in PostlyBee under
**Settings → Public API**, then run this in your own terminal:

```bash
npm install -g postlybee-cli
postlybee auth:login
```

In CI, set `POSTLYBEE_API_TOKEN` from a secret and `POSTLYBEE_WORKSPACE_ID`
instead. Never paste the token into a chat with your agent.

Give the token only the scopes the agent needs. Read-only scopes are enough for
research and reporting; add `posts:write` and `media:write` where the agent
should create posts. See the [CLI guide](https://postlybee.com/agents/cli).

## What the agent will do

The skill tells the agent to:

- show you each post and wait for your approval before creating it,
- save drafts when no time was given or nobody is watching,
- check each platform's rules and settings before writing,
- never delete posts or accounts unless you ask for exactly that.

## Repository layout

| Path | Purpose |
| --- | --- |
| `skills/postlybee/SKILL.md` | The skill |
| `skills/postlybee/references/post-payload.md` | Post JSON reference the skill links to |
| `.claude-plugin/` | Claude Code plugin and marketplace manifests |
| `gemini-extension.json` | Gemini CLI extension manifest |
| `mcp.json` | MCP server entry for other clients |

## License

[MIT](LICENSE)
