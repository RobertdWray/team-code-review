# Team Code Review

A Claude Code skill and subagent that orchestrates multi-tool code reviews using **Gemini CLI** ("George") and **Codex CLI** ("Oscar") as independent reviewers, with Claude Code as the critical analyst.

## Why

AI code review tools are individually useful but have blind spots. Gemini over-engineers. Codex takes shortcuts. Both miss project context. Instead of trusting any single reviewer, this workflow:

1. Sends your code to **two independent AI reviewers** in parallel
2. Has Claude **critically analyze both reviews** using a "friends who could be wrong" framing
3. **Triages every suggestion** into 3 buckets: Absolutely Should Do, Let's Talk About This, Bad Idea
4. Lets you **pick by number** — "Do 1, 3, 9. Skip 7. Discuss 4." — and Claude implements

The result is higher-quality code reviews that catch real issues while filtering out noise, over-engineering, and stylistic bikeshedding.

## How It Works

```
/team-code-review PR #42 focus on error handling
        |
        v
  Claude Code (main conversation)
  ├── Saves diff to temp file (NOT into conversation context)
  ├── Gets PR title/description only
  └── Delegates to review-collector subagent
        |
        v
  review-collector subagent
  ├── Filters out auto-generated files from diff
  ├── Sends filtered diff to George and Oscar in parallel
  ├── 15-min timeout per reviewer, 1 retry on failure
  ├── Numbers all suggestions sequentially
  └── Returns assembled review package
        |
        v
  Claude Code (main conversation)
  ├── Reads ONLY the specific files/lines flagged by reviewers
  ├── Applies critical thinking framing
  ├── Triages into 3 buckets
  └── Presents executive summary + detailed findings
        |
        v
  User picks by number → Claude implements
```

See [code-review-cycle.mermaid](code-review-cycle.mermaid) for the full visual workflow diagram.

## Installation

### Prerequisites

All three CLI tools installed and authenticated:
- **Claude Code** — `claude` ([install guide](https://docs.anthropic.com/en/docs/claude-code))
- **Gemini CLI** — `gemini` (`npm install -g @google/gemini-cli`)
- **Codex CLI** — `codex` (`npm install -g @openai/codex`)

See [Setup-guide.md](Setup-guide.md) for detailed installation, authentication, and configuration for all three tools.

### Deploy the Skill and Subagent

Copy the two files to your global Claude Code config:

```bash
# Skill (creates the /team-code-review slash command)
mkdir -p ~/.claude/skills/team-code-review
cp skill/SKILL.md ~/.claude/skills/team-code-review/SKILL.md

# Subagent (handles Gemini + Codex invocation)
cp agents/review-collector.md ~/.claude/agents/review-collector.md
```

That's it. The skill is now available as `/team-code-review` in every Claude Code session.

## Usage

```bash
# Review a PR
/team-code-review PR #42

# Review with focus notes
/team-code-review PR #42 focus on error handling

# Review specific files
/team-code-review src/auth.ts src/middleware.ts

# Review staged changes
/team-code-review staged

# No arguments — Claude will ask what to review
/team-code-review
```

## Output Format

### Executive Summary (scan and act)

```
Absolutely Should Do: #3, #4, #9
- #3 — XSS in confirm dialog
- #4 — Gitignore generated PII file
- #9 — DND false-positive logic bug

Let's Talk About: #2, #5
- #2 — Hardcoded PII in setup script
- #5 — NANPA validation gaps

Bad Idea / Over-Engineering: #1, #6, #7, #8
- #1 — Full auth layer for local admin
- #6 — DRY extraction for XML builder

Pick by number: "Do 3, 4, 9" / "Skip 1, 6, 7, 8" / "Discuss 2, 5"
```

### Detailed Findings (every item, same structure)

Each finding follows a strict 4-part format. Findings are presented **neutrally** — no judgement labels like "False positive" or "Over-engineering." The executive summary handles the triage; the details show the evidence. Attribution says *(George)*, *(Oscar)*, or *(George and Oscar)*.

```
**#3** — XSS via unescaped contact name in confirm dialog *(George)*

**File:** `admin/templates/whitelist.html:87`

    <button onclick="confirm('Delete {{ contact.name }}?')">

**Analysis:** George flags that if contact.name contains an apostrophe
(e.g., O'Brien), the inline JS string breaks. A crafted name could
execute arbitrary code. Using {{ contact.name|tojson }} would properly
escape for JS string contexts.
```

## Review Logging

Every run saves a full log to `~/Downloads/team-review-YYYY-MM-DD-HHMMSS.md` containing:
- Review scope, files, focus notes
- Raw vs. filtered diff size
- Complete unedited output from both George and Oscar
- Exit codes and retry status

## Key Design Decisions

- **Context-efficient** — Diff is saved to a temp file, never loaded into the main conversation. Claude reads only the specific files/lines referenced in findings, not the entire diff
- **Branch-agnostic PR reviews** — Uses `gh pr diff` piped to both reviewers, not `codex review --base main` which depends on checked-out branch
- **Diff pre-filtering** — Strips lockfiles, build output, and auto-generated code before sending to reviewers
- **Anti-hallucination prompt** — George's prompt explicitly instructs: only flag issues in authored code, not reference docs or generated files
- **True parallel execution** — Both reviewers launched in a single Bash call so they run concurrently
- **Single retry** — Non-timeout failures get one retry before marking reviewer as unavailable
- **Neutral detailed findings** — Detailed section presents reviewer feedback without judgement; triage lives only in the executive summary
- **Separation of concerns** — The skill handles analysis/triage, the subagent handles collection. Swap the subagent for an MCP server later without changing the skill

## Repository Contents

```
.
├── README.md                              # This file
├── code-review-cycle.mermaid              # Visual workflow diagram
├── Setup-guide.md                         # Full setup guide for all three CLI tools
├── team-code-review-deployment-notes.md   # Detailed deployment and architecture notes
├── skill/
│   └── SKILL.md                           # The skill file (copy to ~/.claude/skills/team-code-review/)
└── agents/
    └── review-collector.md                # The subagent file (copy to ~/.claude/agents/)
```

## Future

The collection mechanism could be upgraded to a Python MCP server (~200-300 lines) for even more robust parallelization, retry logic, and extensibility. The skill instructions transfer directly — only the collection layer changes.