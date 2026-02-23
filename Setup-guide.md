# The complete macOS setup guide for Claude Code, Gemini CLI, and Codex CLI

**All three major AI coding CLI tools coexist on macOS without conflicts and complement each other remarkably well.** Claude Code excels at deep reasoning and autonomous multi-file refactoring, Gemini CLI offers a **1 million token** context window and a generous free tier, and Codex CLI provides tight GitHub Actions integration with a Rust-powered TUI. Installing all three gives you the flexibility to pick the right tool for each task — and they share no config files, environment variables, or port bindings. This guide covers everything from installation through advanced multi-tool workflows, drawn from the latest official documentation as of early 2026.

---

## Installation takes five minutes for all three tools

Each tool offers Homebrew and npm installation paths on macOS. The only shared prerequisite is **Node.js 20+** (22+ for Codex CLI via npm).

```bash
# Prerequisites
brew install node@22

# 1. Claude Code (native install — npm is now deprecated)
curl -fsSL https://claude.ai/install.sh | bash
# Or: brew install --cask claude-code

# 2. Gemini CLI
npm install -g @google/gemini-cli
# Or: brew install gemini-cli

# 3. Codex CLI
npm install -g @openai/codex
# Or: brew install --cask codex
```

Claude Code's native installer produces signed, Apple-notarized binaries with **automatic background updates** — run `claude doctor` to verify the installation. Gemini CLI follows a weekly stable release cadence (currently **v0.28.x**) with preview and nightly channels available via `npm install -g @google/gemini-cli@preview`. Codex CLI has been rewritten in **Rust** for performance and sits around **v0.77.x**, with pre-built Apple Silicon binaries also available from GitHub Releases.

System requirements are modest: **4 GB RAM minimum** for all three, macOS 10.15+ for Claude Code, macOS 15+ for Gemini CLI, and Xcode Command Line Tools recommended for Codex CLI's sandbox. After installation, each tool registers a distinct shell command — `claude`, `gemini`, and `codex` — so there are zero naming conflicts.

---

## Authentication and API key configuration

Each tool uses completely separate credentials and environment variables, meaning you can configure all three in your shell profile without interference.

### Claude Code

Claude Code offers three auth paths. The simplest is **OAuth via Claude Pro ($20/month) or Max ($100–$200/month) subscription** — just run `claude` and follow the browser login. Alternatively, set an API key for pay-per-use billing:

```bash
export ANTHROPIC_API_KEY="sk-ant-..."
```

**Important caveat**: if `ANTHROPIC_API_KEY` is set as an environment variable, Claude Code uses it *instead of* your subscription, resulting in API charges. Remove it to use your Pro/Max allocation. Enterprise users can connect via AWS Bedrock (`CLAUDE_CODE_USE_BEDROCK=1`) or Google Vertex AI (`CLAUDE_CODE_USE_VERTEX=1`). Credentials are stored securely in the **macOS Keychain**.

### Gemini CLI

Gemini CLI's standout is its **free tier**: log in with any Google account and get **60 requests/minute and 1,000 requests/day** with access to Gemini 2.5 Pro's full 1M-token context window. Run `gemini` and select "Login with Google" for the OAuth flow. For API-key access:

```bash
export GEMINI_API_KEY="AIza..."
```

Vertex AI users authenticate via Application Default Credentials (`gcloud auth application-default login`) with `GOOGLE_CLOUD_PROJECT` and `GOOGLE_CLOUD_LOCATION` set. Credentials are stored at `~/.gemini/oauth_creds.json`.

### Codex CLI

Codex CLI is **included with ChatGPT subscriptions** (Plus at $20/month, Pro at $200/month). Run `codex` and select "Sign in with ChatGPT" for OAuth. For API access:

```bash
export OPENAI_API_KEY="sk-..."
```

Codex also supports device-code auth (`codex login --device-code`) for headless environments. Credential storage is configurable between file-based (`~/.codex/auth.json`) and OS keychain via the `credential_store` setting in `config.toml`.

### Recommended shell profile setup

```bash
# ~/.zshrc — AI CLI Tools
# Claude Code: Uses OAuth subscription (no API key needed)
# Uncomment below only for API pay-per-use:
# export ANTHROPIC_API_KEY="sk-ant-..."

# Gemini CLI: Uses Google OAuth free tier (no key needed)
# Uncomment for API key access:
# export GEMINI_API_KEY="AIza..."

# Codex CLI: Uses ChatGPT OAuth subscription (no key needed)
# Uncomment for API access:
# export OPENAI_API_KEY="sk-..."
```

---

## Configuration files and directory structures

The three tools use entirely separate configuration hierarchies. Understanding where each stores its files is critical for effective project setup.

### Claude Code configuration

Claude Code uses a **Markdown-first** approach centered on `CLAUDE.md` files plus JSON settings:

```
~/.claude/                      # User-level
├── CLAUDE.md                   # Global instructions (all projects)
├── settings.json               # Global settings + permissions
├── settings.local.json         # Machine-specific settings
├── commands/                   # Custom slash commands
└── agents/                     # Custom subagent definitions

~/.claude.json                  # User-level MCP server config

<project>/
├── CLAUDE.md                   # Project instructions (commit to git)
├── CLAUDE.local.md             # Personal overrides (gitignored)
├── .mcp.json                   # Project MCP servers (commit to git)
└── .claude/
    ├── settings.json           # Project settings (shared)
    ├── settings.local.json     # Personal project settings
    ├── commands/               # Project slash commands
    └── skills/                 # Reusable skill guides
```

Settings precedence flows from **enterprise policies → CLI flags → local project → shared project → user global**. The `settings.json` supports a `permissions` block for allow/deny rules on tool execution, plus hooks for event-driven automation.

### Gemini CLI configuration

Gemini CLI mirrors a similar pattern but uses `GEMINI.md` and a `.gemini/` directory with **TOML-based** custom commands:

```
~/.gemini/                      # User-level
├── GEMINI.md                   # Global instructions
├── settings.json               # Global settings + MCP servers
├── .env                        # Environment variables
├── commands/                   # Custom commands (.toml files)
└── extensions/                 # Installed extensions

<project>/
├── GEMINI.md                   # Project instructions
├── .geminiignore               # Exclude files from context
└── .gemini/
    ├── settings.json           # Project settings (overrides global)
    ├── .env                    # Project environment variables
    └── commands/               # Project-specific commands
```

A distinctive feature is the `.gemini/.env` file — Gemini CLI automatically loads environment variables from the first `.env` found searching upward from the current directory, then falls back to `~/.gemini/.env`.

### Codex CLI configuration

Codex CLI uses **TOML** for its main config and `AGENTS.md` for instructions:

```
~/.codex/                       # User-level
├── config.toml                 # Main configuration (shared with IDE)
├── AGENTS.md                   # Global instructions
├── AGENTS.override.md          # Global override instructions
└── skills/                     # User-installed skills

<project>/
├── AGENTS.md                   # Project instructions
├── AGENTS.override.md          # Override instructions (subdirectories too)
└── .codex/
    ├── config.toml             # Project config (trusted projects only)
    └── skills/                 # Project-specific skills
```

Codex CLI introduces a **trust model**: project-scoped `.codex/config.toml` files only load when you explicitly trust the project. This prevents malicious repos from altering your tool's behavior. The `config.toml` supports **profiles** — named configuration presets you can switch between with `--profile`:

```toml
[profiles.deep-review]
model = "gpt-5-pro"
model_reasoning_effort = "high"
approval_policy = "never"

[profiles.lightweight]
model = "gpt-4.1"
approval_policy = "untrusted"
```

---

## Project instruction files are the most important thing to get right

The instruction files — `CLAUDE.md`, `GEMINI.md`, and `AGENTS.md` — serve as persistent, project-specific memory for each tool. They are loaded automatically at session start and directly shape how each agent understands and interacts with your codebase.

### Writing effective instruction files

All three tools benefit from the same core content. A well-structured instruction file should include your **tech stack**, **key commands** (build, test, lint), **code conventions**, **project structure**, and **critical warnings** about things the agent should never modify. Keep them concise — every line consumes context tokens.

```markdown
# Project: MyApp

## Tech Stack
Next.js 14 with App Router, TypeScript strict mode, Prisma ORM, PostgreSQL

## Commands
- `npm run dev` — Start dev server (port 3000)
- `npm run test -- path/to/test` — Run single test
- `npm run lint` — ESLint + Prettier check

## Code Conventions
- Functional components with hooks, no class components
- 2-space indentation, semicolons required
- All API calls through /src/lib/api-client.ts

## Critical Rules
- NEVER modify files in /prisma/migrations/
- Auth uses HttpOnly JWT cookies — do not change the token flow
- The retry logic in /src/lib/retry.ts is intentionally complex
```

Each tool offers an **auto-generation command**: Claude Code's `/init`, Gemini CLI's `/init`, and Codex's skill-based generation. Use these as starting points, then refine manually.

### Keeping instructions in sync across tools

The **ai-rules CLI** solves the multi-tool instruction problem by maintaining a single source of truth:

```bash
curl -fsSL https://raw.githubusercontent.com/block/ai-rules/main/scripts/install.sh | bash
ai-rules init
ai-rules generate   # Produces CLAUDE.md, GEMINI.md, and AGENTS.md
```

Alternatively, maintain a canonical `AI-INSTRUCTIONS.md` and use symlinks or a simple script to copy it into each tool's expected filename.

---

## MCP server integration works across all three tools

All three CLI tools support the **Model Context Protocol (MCP)** for extending their capabilities with external tools. They share a compatible MCP server ecosystem, meaning a single MCP server (like the GitHub MCP server) can be configured in all three tools simultaneously.

### Claude Code MCP setup

```bash
# Add via CLI (user-level)
claude mcp add --transport http -s user github https://api.githubcopilot.com/mcp

# Add via CLI (project-level, saved in .mcp.json)
claude mcp add --transport stdio -s project postgres \
  -- npx -y @modelcontextprotocol/server-postgres postgres://localhost/mydb

# Or edit ~/.claude.json directly for user-level servers
# Or edit .mcp.json in project root for project-level servers
```

### Gemini CLI MCP setup

```bash
# Add via CLI
gemini mcp add github --scope user -- docker run -i --rm \
  -e GITHUB_PERSONAL_ACCESS_TOKEN ghcr.io/github/github-mcp-server

# Or configure in ~/.gemini/settings.json under "mcpServers" key
```

Gemini CLI adds useful MCP management features: `gemini mcp enable/disable <name>` toggles servers without removing them, and the `trustTools` array in config auto-approves specific tools.

### Codex CLI MCP setup

```bash
# Add via CLI
codex mcp add context7 -- npx -y @upstash/context7-mcp

# Or configure in ~/.codex/config.toml
```

```toml
[mcp_servers.github]
command = "npx"
args = ["-y", "@modelcontextprotocol/server-github"]
env = { GITHUB_PERSONAL_ACCESS_TOKEN = "ghp_..." }
startup_timeout_sec = 10
tool_timeout_sec = 60
```

A unique Codex CLI capability: it can **run itself as an MCP server** (`codex mcp`), allowing other agents or the OpenAI Agents SDK to invoke Codex as a tool.

### Recommended MCP servers for all three tools

Configure these high-value MCP servers across all three tools for maximum productivity: **GitHub** (PR and issue management), **Context7** (up-to-date library documentation), **Brave Search or Google Search** (web research), and **PostgreSQL or your database** (direct DB queries). Each tool's MCP servers run in separate processes, so having the same server configured in multiple tools causes no conflicts — though running them simultaneously does consume more system resources.

---

## Terminal workflows and shell integration

Each tool supports both interactive REPL and headless scripting modes, making them suitable for both exploration and CI/CD automation.

### Interactive usage patterns

```bash
# Claude Code
claude                           # Start REPL
claude "explain this codebase"   # REPL with initial prompt
claude -c                        # Resume most recent session

# Gemini CLI
gemini                           # Start REPL
gemini -i "analyze this code"    # Interactive with initial prompt

# Codex CLI
codex                            # Start TUI
codex "fix the failing tests"    # TUI with initial prompt
codex resume --last              # Resume most recent session
```

### Headless mode for scripting and piping

All three tools support non-interactive execution — essential for CI/CD and shell scripts:

```bash
# Claude Code headless
cat error.log | claude -p "explain these errors"
claude -p --output-format json "list all TODO comments" > todos.json
claude -p --max-turns 5 "fix all linting errors in src/"

# Gemini CLI headless
cat file.py | gemini -p "review this code"
gemini -p "explain the architecture" --output-format json

# Codex CLI headless
echo "add error handling" | codex exec -
codex exec --full-auto --json "run and fix failing tests"
```

### Approval and safety modes compared

Each tool handles autonomous execution differently:

- **Claude Code** uses granular permission prompts (configurable via `settings.json` allow/deny lists) and an optional **sandbox mode** that isolates filesystem and network access
- **Gemini CLI** offers a `-y` / `--yolo` flag to auto-approve all tool calls, plus Docker/Podman sandboxing via `--sandbox docker`
- **Codex CLI** provides the most structured approach with three explicit modes: **suggest** (read-only), **auto-edit** (edits files, asks before commands), and **full-auto** (everything auto-approved, network disabled, directory-sandboxed)

For production use, Codex CLI's `full-auto` mode with its automatic sandboxing is the most safety-conscious approach to autonomous execution. Claude Code's permission allow-lists offer the most granular control for habitual workflows.

---

## IDE integration with Cursor and VS Code

All three tools work in Cursor, though with varying levels of native integration.

**Claude Code** offers an official VS Code extension that works in Cursor. Install it manually since Cursor may not auto-detect it:

```bash
cursor --install-extension ~/.claude/local/node_modules/@anthropic-ai/claude-code/vendor/claude-code.vsix
```

The extension provides a graphical chat panel, checkpoint-based undo, `@`-mention files with line ranges, and parallel conversation support. You can also simply run `claude` in Cursor's integrated terminal — Claude auto-detects the IDE environment and enables diff viewing.

**Gemini CLI** integrates via the "Gemini CLI Companion" extension, available on the Open VSX Registry (which Cursor can access). Run `/ide install` inside Gemini CLI to set it up. The extension shares workspace context — the 10 most recently accessed files, cursor position, and selected text (up to 16 KB) — and opens code changes in VS Code's native diff viewer.

**Codex CLI** has a dedicated IDE extension for VS Code, Cursor, and Windsurf that shares the same `~/.codex/config.toml` as the CLI. Search "Codex" in Cursor's extension marketplace. Cursor tip: the horizontal activity bar can hide the Codex icon — pin it and consider switching to vertical activity bar orientation in settings.

The most flexible approach is to **run all three tools in Cursor's integrated terminal** alongside your IDE's native completions. This avoids extension conflicts and lets you switch tools with a simple command.

---

## Pricing comparison reveals different value propositions

| Aspect | Claude Code | Gemini CLI | Codex CLI |
|--------|------------|------------|-----------|
| **Free tier** | None | **1,000 requests/day** free | Limited trial |
| **Entry subscription** | $20/mo (Pro) | Free / Google AI Pro | $20/mo (ChatGPT Plus) |
| **Power subscription** | $100–$200/mo (Max) | Pay-as-you-go | $200/mo (ChatGPT Pro) |
| **API pay-per-use** | $3–$15/M tokens (Sonnet) | Varies by model | $1.50–$6/M tokens (codex-mini) |
| **Best value** | Max subscription for heavy use | Free tier for exploration | ChatGPT Plus if already subscribed |

**Gemini CLI's free tier is unmatched** — it provides production-grade Gemini 2.5 Pro access with a 1M-token context window at zero cost, making it ideal for budget-conscious development and large-codebase analysis. Claude Code's **Pro subscription offers roughly $150 worth of API usage for $20/month**, making it the best per-dollar value for daily coding work. Codex CLI's inclusion with existing ChatGPT subscriptions means many developers already have access.

---

## How the three tools compare for real-world tasks

Community benchmarks and practitioner reports paint a consistent picture of each tool's strengths. **Claude Code consistently ranks highest** for code quality, autonomous multi-step operations, and complex debugging — developers describe it as "the escalation path when other tools fail." Its auto-compaction and context management produce the most reliable results across long sessions.

**Gemini CLI's 1M-token context window is its killer feature** — it can ingest an entire mid-size codebase in a single session, making it unmatched for large-scale analysis, codebase understanding, and cross-file refactoring. The built-in Google Search grounding also makes it the best choice for research tasks requiring current documentation. However, community reports note **higher error rates** compared to Claude Code, and some users have experienced destructive file operations in autonomous mode — always use version control and sandboxing.

**Codex CLI differentiates through its CI/CD integration and cloud execution model.** It can run parallel cloud tasks, integrates natively with GitHub Actions via `openai/codex-action@v1`, and its three-tier approval system (suggest → auto-edit → full-auto) provides the clearest safety model. The Rust-based TUI is notably snappy. Its skills system and the ability to run as an MCP server make it particularly composable in larger automation pipelines.

### Recommended task-based tool selection

- **Complex debugging and architectural refactoring** → Claude Code
- **Large codebase analysis and understanding** → Gemini CLI (1M context)
- **Quick research with current documentation** → Gemini CLI (Google Search grounding)
- **CI/CD automation and parallel execution** → Codex CLI
- **Budget-constrained exploration** → Gemini CLI (free tier)
- **Daily primary coding assistant** → Claude Code (highest reliability)

---

## Running multiple tools together on the same project

The three tools have **zero conflicts** when installed on the same macOS machine. They use separate config directories (`~/.claude/`, `~/.gemini/`, `~/.codex/`), separate environment variables (`ANTHROPIC_API_KEY`, `GEMINI_API_KEY`, `OPENAI_API_KEY`), and separate shell commands. The only real risk is **file edit race conditions** when two tools try to modify the same files simultaneously.

### Parallel workflow with git worktrees

The safest approach for running multiple agents on the same repository:

```bash
git worktree add ../project-claude feature/claude-work
git worktree add ../project-gemini feature/gemini-work
cd ../project-claude && claude "implement the auth module"
cd ../project-gemini && gemini "write comprehensive tests for auth"
```

### Cross-tool orchestration

Several open-source projects enable sophisticated multi-agent workflows. The **Roundtable MCP Server** (`pip install roundtable-ai`) lets you orchestrate all three CLIs via MCP, running them in parallel and aggregating results. **CCB** provides tmux-based pane management with cross-calling between tools. For simpler setups, you can invoke one tool from within another using headless mode — for example, adding to your `CLAUDE.md`:

```markdown
When you need to analyze files exceeding 200K tokens, use:
gemini -p "analyze the following files: ..." for large-context work.
```

### The recommended multi-tool project setup

```
my-project/
├── CLAUDE.md              # Claude Code instructions
├── GEMINI.md              # Gemini CLI instructions
├── AGENTS.md              # Codex CLI instructions
├── .mcp.json              # Claude Code MCP servers
├── .claude/
│   └── settings.json      # Claude Code project settings
├── .gemini/
│   ├── settings.json      # Gemini CLI project settings (+ MCP servers)
│   └── .env               # Gemini-specific env vars
├── .codex/
│   └── config.toml        # Codex CLI project settings (+ MCP servers)
├── .gitignore             # Include: CLAUDE.local.md, .claude/settings.local.json
└── src/
```

Commit `CLAUDE.md`, `GEMINI.md`, `AGENTS.md`, `.mcp.json`, `.claude/settings.json`, `.gemini/settings.json`, and `.codex/config.toml` to version control so your entire team benefits from the AI tool configurations.

## Team Code Review Skill Deployment

The `/team-code-review` skill requires all three CLI tools above plus two subagent files deployed to Claude Code's global config:

```bash
# Skill (creates the /team-code-review slash command)
mkdir -p ~/.claude/skills/team-code-review
cp skill/SKILL.md ~/.claude/skills/team-code-review/SKILL.md

# Subagents
cp agents/review-collector.md ~/.claude/agents/review-collector.md
cp agents/team-research-agent.md ~/.claude/agents/team-research-agent.md
```

### Research Agent Prerequisites

The **team-research-agent** uses Claude Code's built-in WebSearch and WebFetch tools to gather documentation. These are available by default in Claude Code — no additional MCP servers or API keys are required beyond your existing Claude Code authentication.

The research agent writes temporary files to `~/Downloads/Team-Research/`. This folder is created automatically during reviews and cleaned up by the review-collector after each run. If cleanup fails (e.g., the review is interrupted), the folder persists with small markdown files that can be safely deleted manually.

## Conclusion

The AI coding CLI landscape has matured rapidly through 2025 into a genuinely multi-tool ecosystem. The three tools occupy distinct niches: Claude Code as the **most capable autonomous coding agent**, Gemini CLI as the **highest-context, most accessible** option, and Codex CLI as the **most automation-friendly** with the tightest CI/CD integration. Rather than picking one, the optimal setup installs all three and selects tools based on task requirements. The tools' completely separate configuration systems make this frictionless — the hardest part is keeping instruction files in sync, which tools like `ai-rules` solve elegantly. With MCP server support universal across all three, your tool integrations (GitHub, databases, search) work everywhere. The key best practice: always use git worktrees or separate branches when running multiple agents autonomously, and lean on each tool's sandboxing capabilities to prevent unintended destructive operations.