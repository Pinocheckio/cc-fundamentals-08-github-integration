# Claude Code Extensibility: Commands, Hooks, Plugins, Skills, and MCPs

A reference guide explaining the five main ways to extend and customize Claude Code behavior.

---

## Summary Table

| Concept | Where defined | When it runs | Who defines it | Primary use |
|---------|--------------|--------------|----------------|-------------|
| **/commands** | `.claude/commands/*.md` | When user types `/name` | Developer/user | Reusable prompt workflows |
| **Hooks** | `settings.json` (hooks key) | Automatically on lifecycle events | User/admin | Side effects, validation, logging |
| **Plugins** | N/A (not a native concept) | — | — | See note below |
| **Skills** | `.claude/commands/*.md` + Claude Code Action config | When invoked in GitHub/chat context | Repo author | Structured, user-invocable workflows |
| **MCPs** | `settings.json` (mcpServers key) | On demand, via tool calls | Developer | External tools, data, APIs |

---

## /Commands (Slash Commands)

### What they are
Slash commands are **user-defined reusable prompts** stored as Markdown files. When you type `/command-name` in Claude Code, Claude reads the corresponding `.md` file and executes it as a set of instructions.

### Where they live
```
Project-level:  .claude/commands/<name>.md   (shared with team)
User-level:     ~/.claude/commands/<name>.md  (personal)
```

### How they work
1. User types `/EA-prime` in the chat
2. Claude Code reads `.claude/commands/EA-prime.md`
3. The file's content becomes the active prompt/instructions
4. Claude follows those instructions in the current session

### Anatomy of a command file
```markdown
---
description: Short description shown in the /help list
allowed-tools: Bash, Read, Glob
model: sonnet
---

# Command Title

Instructions for Claude go here. You can include:
- Step-by-step workflows
- Tool usage guidance
- Output format templates
- Any prompt content
```

### Front matter fields
| Field | Purpose |
|-------|---------|
| `description` | Shown in `/help` command listing |
| `allowed-tools` | Restricts which tools Claude may use |
| `model` | Override the default model for this command |

### Example in this repo
The `EA-prime` command (`.claude/commands/EA-prime.md`) instructs Claude to run `git ls-files`, read `README.md`, identify the tech stack, and produce a structured project summary.

### Key characteristics
- Stored as plain Markdown — easy to read, edit, version-control
- Can be parameterized using `$ARGUMENTS` in the file body
- Scoped: project-level commands override user-level commands of the same name
- No code execution by themselves — they are prompt templates

---

## Hooks

### What they are
Hooks are **shell commands that run automatically** in response to lifecycle events inside a Claude Code session. They let you inject behavior around Claude's actions without modifying prompts.

### Where they're configured
In your Claude Code settings file (`~/.claude/settings.json` or project-level `.claude/settings.json`):

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "echo 'About to run a shell command'"
          }
        ]
      }
    ],
    "PostToolUse": [...],
    "Notification": [...],
    "Stop": [...]
  }
}
```

### Lifecycle events
| Event | When it fires |
|-------|--------------|
| `PreToolUse` | Before Claude calls any tool (Bash, Read, Edit, etc.) |
| `PostToolUse` | After a tool call completes |
| `Notification` | When Claude sends a notification |
| `Stop` | When Claude finishes responding |

### How hooks communicate back
- **Exit code 0**: success, Claude continues normally
- **Exit code 2**: blocks the action and feeds stdout back to Claude as context
- **Other non-zero**: error, logged but Claude continues

### Common hook use cases
- **Linting**: Run `eslint` or `ruff` after every file edit
- **Secret scanning**: Block writes that contain API keys
- **Logging**: Record every shell command Claude runs
- **Testing**: Auto-run tests after code changes
- **Formatting**: Run `prettier` or `black` automatically

### Key characteristics
- Hooks run in your shell, with full system access
- They are event-driven, not user-triggered
- They can block actions (exit code 2) or just observe
- Configured separately from prompts — no Markdown files involved

---

## Plugins

### What they are (and aren't)
**"Plugins" is not a first-class concept in Claude Code.** There is no official plugin system distinct from MCPs, commands, or hooks.

In practice, "plugin" is used informally to mean:

| Informal usage | Actual Claude Code concept |
|----------------|---------------------------|
| "Install a plugin for X tool" | Add an MCP server |
| "A plugin that runs on every response" | A hook |
| "A plugin command I can invoke" | A slash command |
| "IDE plugin" | The Claude Code VS Code/JetBrains extension |

If someone says "install a plugin," they almost always mean **add an MCP server** (see below). If you encounter the term in documentation, check whether it refers to an IDE extension or an MCP integration.

---

## Skills

### What they are
Skills are **user-invocable command workflows** surfaced specifically within Claude Code Action (the GitHub App integration) and the Claude agent SDK. They are functionally the same as slash commands but are exposed and invoked differently depending on context.

### Two contexts where "skills" appear

**1. Claude Code Action (GitHub)**
In the GitHub integration, skills are pre-defined workflows that appear in the system prompt and can be triggered by users mentioning `/skill-name` in issue or PR comments. The Claude Code Action configuration maps skill names to their `.md` command files.

**2. Claude Agent SDK**
When building multi-agent systems with the Claude SDK, "skills" refer to specialized sub-agents with specific capabilities and tool access. Each skill/agent type has a defined purpose (e.g., `Bash`, `Explore`, `Plan`).

### Skills vs. /commands: key differences

| | /Commands | Skills |
|--|-----------|--------|
| Invocation | `/name` in local Claude Code chat | Mentioned in GitHub comments; or spawned programmatically |
| Context | Local development session | GitHub Action CI; or agent orchestration |
| Defined in | `.claude/commands/*.md` | `.md` files + Action/SDK configuration |
| Metadata | Front matter in `.md` | Action `skills` config; SDK agent type |
| User-facing | Listed in `/help` | Listed in GitHub comment system prompts |

### Skills in this repo
The `EA-*` commands (EA-prime, EA-install, EA-handoff, EA-pickup) are exposed both as local slash commands and as skills in the GitHub Action configuration, which is why they appear in both places.

### Key characteristics
- Skills are the "published" face of commands for external/automated contexts
- They bridge local development workflows and CI/CD environments
- In agent SDK contexts, skills represent distinct agent personas with different tool access

---

## MCPs (Model Context Protocol)

### What they are
MCPs are **external servers that give Claude access to tools, data, and APIs** beyond what's built in. The Model Context Protocol is an open standard that defines how Claude communicates with these servers.

An MCP server can expose:
- **Tools** — callable functions (search a database, send an email, run a query)
- **Resources** — readable data sources (files, database records, API responses)
- **Prompts** — reusable prompt templates the server provides

### Where they're configured
```json
// ~/.claude/settings.json or .claude/settings.json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "your-token"
      }
    },
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres", "postgresql://localhost/mydb"]
    }
  }
}
```

### How MCPs work
1. Claude Code starts the MCP server process on launch
2. The server registers its available tools/resources with Claude
3. During conversation, Claude can call MCP tools like any built-in tool
4. The MCP server executes the action and returns results to Claude

### Common MCP servers
| Server | What it provides |
|--------|-----------------|
| `@modelcontextprotocol/server-github` | GitHub API (repos, issues, PRs) |
| `@modelcontextprotocol/server-postgres` | PostgreSQL query tool |
| `@modelcontextprotocol/server-filesystem` | Extended file system access |
| `@modelcontextprotocol/server-brave-search` | Web search via Brave |
| Custom servers | Any API or service you wrap |

### MCPs vs. other extension points

| | MCPs | Hooks | /Commands |
|--|------|-------|-----------|
| Runs as | Separate long-lived process | One-shot shell command | Inline prompt |
| Exposes | Tools + resources to Claude | Side effects only | Instructions only |
| Requires | Server implementation | Shell script | Markdown file |
| Complexity | Higher (server process) | Low (shell command) | Lowest (text file) |
| Use for | New capabilities (APIs, DBs) | Validation, logging | Prompt workflows |

### Key characteristics
- MCPs extend *what Claude can do*, not just how it behaves
- They communicate over stdio or HTTP using the MCP protocol
- Claude treats MCP tools identically to built-in tools
- MCP servers can be community-built, company-built, or custom
- The protocol is open — anyone can build an MCP server

---

## Choosing the Right Extension Point

```
Need to add a new API or data source?
  → MCP server

Need to run something automatically when Claude acts?
  → Hook

Need a reusable workflow you invoke manually?
  → /Command (or Skill if exposing to GitHub/agents)

Someone said "install a plugin"?
  → They probably mean an MCP server
```

---

## How They Work Together

A complete workflow might use all of them:

1. You type `/review-pr` — a **slash command** loads review instructions
2. Claude calls a **MCP** GitHub tool to fetch the PR diff
3. Claude edits a file — a **hook** fires and runs `eslint` automatically
4. The linting result feeds back to Claude, which fixes the issue
5. In CI, a **skill** configured in the GitHub Action triggers the same review automatically on new PRs
