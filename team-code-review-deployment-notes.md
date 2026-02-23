---
title: Team Code Review — Multi-Tool Review Workflow
description: Skill + Subagent deployment for cross-tool code review using Gemini CLI and Codex CLI alongside Claude Code.
created: 2026-02-21
updated: 2026-02-23
status: implementing
tags: [claude-code, skill, subagent, team-code-review, gemini-cli, codex-cli]

source_of_truth_for:
  - "Team Code Review Skill: /team-code-review slash command"
  - "Review Collector Subagent: Gemini + Codex CLI orchestration"
  - "Team Research Agent: Pre-review documentation researcher"
  - "Multi-Tool Review Workflow: 3-bucket triage system"

related:
  - "[[Setup-guide]]"
  - "[[code-review-cycle.mermaid]]"
---

**System:** Claude Code (global ~/.claude/ config)
**Last Updated:** February 2026

---

## 1. /team-code-review Skill

**Status:** Implementing
**Priority:** High
**Added:** February 2026
**Location:** `~/.claude/skills/team-code-review/SKILL.md`

### Description

User-invocable skill that orchestrates a multi-tool code review workflow. Delegates review collection to the review-collector subagent, then critically analyzes both reviews using a "friends who could be wrong" framing. Triages all suggestions into 3 numbered buckets:

1. **Absolutely Should Do** — clear wins, real bugs, meaningful quality improvements
2. **Let's Talk About This** — has merit but involves tradeoffs or context-dependent decisions
3. **Bad Idea / Over-Engineering** — over-engineered, wrong pattern, or stylistic preference

User picks items by number ("Do 1, 3, 5 / Skip 7, 8 / Discuss 4") and Claude implements.

### Key Design Decisions

- `disable-model-invocation: true` — only runs when user explicitly invokes `/team-code-review`
- Contains the critical thinking framing prompt that positions reviewers as "friends who could be wrong"
- All suggestions numbered sequentially across both reviewers for easy reference
- Separates collection (subagent) from analysis (main conversation) for clean responsibility split
- If invoked with no arguments, prompts user via AskUserQuestion (PR, files, or staged changes)
- Accepts inline arguments: `/team-code-review PR #42 focus on error handling`
- **Context-efficient diff handling:** Diff is saved to an isolated temp folder (`$REVIEW_TMP/review-raw-diff.txt`) via Bash, never loaded into the main conversation context. The main conversation only reads `gh pr view` for PR metadata. After reviews return, Claude reads only the specific files/lines flagged by reviewers — not the entire diff. This avoids consuming the diff in context three times.

### Output Format

Two-part output enforced by a full 9-item example in the SKILL.md:

**1. Executive Summary** — scannable bullet list grouped by bucket, with 3-5 word descriptions per item. The "Pick by number" prompt lives here so the user can act immediately without reading the details.

**2. Detailed Findings** — all items in numerical order (#1, #2, #3...) regardless of bucket. No bucket labels repeated (that's in the summary). Each finding follows a strict 4-part structure:

| Section | Content |
|---------|---------|
| **Title line** | `#N` — one-line description *(George/Oscar)* |
| **File(s)** | Specific path with line range: `admin/app.py:245-252` |
| **Code block** | Verbatim snippet from the actual file in a fenced code block |
| **Analysis** | Plain English assessment — what the issue is, why it matters, what to do |

Each finding separated by a `---` horizontal rule. The consistent structure means every item reads the same way — description, where, code, why.

**Tone rule:** Detailed findings present each reviewer's feedback neutrally. No judgement language ("False positive", "Over-engineering", "Bikeshedding"). The analysis provides factual technical context — the executive summary is the only place where do/skip/discuss triage lives. Attribution says *(George)*, *(Oscar)*, or *(George and Oscar)* depending on who flagged it.

---

## 2. review-collector Subagent

**Status:** Implementing
**Priority:** High
**Added:** February 2026
**Location:** `~/.claude/agents/review-collector.md`

### Description

Subagent that invokes Gemini CLI ("George") and Codex CLI ("Oscar") via Bash in headless mode, collects both review outputs, numbers all suggestions sequentially, and returns an assembled review package for Claude's critical analysis.

### Technical Details

#### Diff Capture Strategy (branch-agnostic)

For PR and branch reviews, the diff is captured via `gh pr diff` or `git diff` into a temp file. George receives the diff piped via stdin. Oscar receives the diff **embedded in a prompt file** (`$REVIEW_TMP/oscar-prompt.md`) fed via `codex exec --sandbox read-only - < $REVIEW_TMP/oscar-prompt.md` — because `codex exec "string"` ignores piped stdin (see Stdin Gotcha below). This ensures **neither reviewer depends on the checked-out branch**.

| Review mode | Diff capture | Oscar invocation | George invocation |
|-------------|-------------|-----------------|-------------------|
| PR review | `gh pr diff <N>` | `codex exec --sandbox read-only -` with diff embedded in prompt file via stdin | `gemini -p` with diff via stdin |
| Branch review | `git diff main...<branch>` | `codex exec --sandbox read-only -` with diff embedded in prompt file via stdin | `gemini -p` with diff via stdin |
| Uncommitted/staged | `git diff` / `git diff --staged` | `codex review --uncommitted` (native) | `gemini -p` with diff via stdin |
| Specific files | Read files directly | `codex exec --sandbox read-only -` with files embedded in prompt file via stdin | `gemini -p` with `@file` references |

#### Diff Pre-Filtering

Before sending to reviewers, auto-generated and noise files are stripped from the diff using an awk filter. This prevents reviewers from wasting context on lockfiles, build output, and generated code — which caused a 141KB diff in the first live run.

**Always excluded:** `package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`, `Cargo.lock`, `poetry.lock`, `baml_client/`, `*.generated.*`, `.cursor/rules/`, `node_modules/`, `.next/`, `dist/`, `build/`, `*.min.js`, `*.min.css`

The log records raw vs. filtered diff size for transparency.

#### Reviewer-Specific Details

**Oscar (Codex CLI):**
- For PR/branch reviews: Diff is embedded in a prompt file (`$REVIEW_TMP/oscar-prompt.md`) and fed via `codex exec --sandbox read-only - < $REVIEW_TMP/oscar-prompt.md`. The `-` placeholder tells Codex to read the prompt from stdin. `--sandbox read-only` prevents filesystem writes and reduces autonomous exploration.
- **Stdin gotcha:** `codex exec "prompt string"` ignores piped stdin entirely — the diff goes nowhere. Confirmed by OpenAI Cookbook, CLI reference, and [GitHub Issue #1123](https://github.com/openai/codex/issues/1123). The fix is to embed the diff in the prompt text and pipe the whole file via `codex exec -`.
- For uncommitted/staged: `codex review --uncommitted` (native subcommand — works correctly when user is on the right branch)
- Accepts custom focus notes as the prompt argument
- Reads `AGENTS.md` for project-specific review context automatically

**George (Gemini CLI):**
- No built-in review command (feature request #16947 is open)
- Uses `gemini -p "review prompt" --output-format text` with filtered diff piped via stdin
- Can use `@file.ts` references to include specific files in its 1M-token context
- Reads `GEMINI.md` for project-specific review context automatically
- Prompt includes explicit instruction: **only flag issues in authored code, not in reference docs, auto-generated files, or boilerplate** — prevents hallucinations from context bleed

#### Reviewer Exploration Behavior

Both reviewers are free to explore the repository using their own techniques. Oscar may run git commands and inspect files beyond the piped diff; George may use its large context window to analyze related code. This is expected and encouraged — each tool's exploration techniques can surface bugs that a narrowly scoped diff review would miss. The output cleaning step (Step 3b) separates their findings from their exploration noise.

#### Output Noise Patterns

Both tools emit significant CLI noise alongside their review findings. The review-collector separates stderr from stdout and cleans stdout before packaging findings.

**George (Gemini CLI) noise:**
- *stderr:* Node.js deprecation warnings (`(node:...) [DEP...]`), GaxiosError stack traces on rate limits (429s are retried internally but the full error + HTTP config/response objects are logged), auth library traces
- *stdout:* Startup messages (`Loaded cached credentials.`, `Loading extension: conductor`, `Server '...' supports tool updates`, `Hook registry initialized with N hook entries`)

**Oscar (Codex CLI) noise:**
- *stderr:* Rust-level ERROR logs (`ERROR codex_core::...`, `ERROR rmcp::transport::...`), MCP server startup failures
- *stdout:* Session header (version, model, provider, sandbox mode, session ID, `--------` separators), `thinking`/`exec`/`codex` phase markers with full command output between them, bold thinking summaries (`**Planning...`, `**Reviewing...`), prompt echo (Codex repeats the entire prompt it received), `tokens used` summary, and sometimes duplicate findings (numbered list appears once in a `codex` block then repeats as final output)

In the first live run, George's actual findings were ~30 lines buried in ~120 lines of noise. Oscar's findings were ~50 lines buried in ~1000 lines of exploration output and phase markers.

#### Execution and Reliability

- **True parallel execution:** Both reviewers launched in a single Bash tool call with `&` + `wait` so they run concurrently (separate Bash calls would be sequential)
- 15-minute timeout per reviewer via `timeout 900`
- **Single retry** on non-timeout failures (exit != 0 and exit != 124) before marking as unavailable
- Graceful degradation: if one reviewer fails even after retry, the other's output is still returned
- Both attempts logged in the review log for troubleshooting

### Review Logging

Every review run saves a full log to `~/Downloads/` for troubleshooting:

- **File:** `~/Downloads/team-review-YYYYMMDD-HHMMSS-reponame-scope-shorthash.md` (e.g., `team-review-20260223-165406-TrayVerify-pr42-abc1234.md`)
- **Contains:** Date, review scope, files reviewed (file count + additions/deletions), focus notes, diff size (raw → filtered with excluded file count), exit codes, retry status
- **Three sections per reviewer:**
  1. **Cleaned Review Output** — just the findings, with CLI noise stripped (what gets sent for analysis)
  2. **Raw Output (stdout)** — complete unedited stdout showing the reviewer's full exploration and thought process
  3. **Diagnostic Output (stderr)** — errors, warnings, and stack traces captured separately
- Lets you inspect each reviewer's full process for a deeper dive while keeping the findings scannable
- Persists after the review — not cleaned up automatically

### Technical Considerations

- Codex `review` is branch-dependent — only used for uncommitted/staged where branch is correct
- Codex `exec` is branch-agnostic — used for PR/branch reviews with diff embedded in prompt file, `--sandbox read-only` to prevent filesystem writes
- Gemini has 1M token context (can review entire medium codebases); Codex context varies by model
- User focus notes (e.g., "focus on auth changes") are passed through to both reviewers
- **Concurrent-safe temp folder:** Each review run creates an isolated folder at `/tmp/team-review-YYYYMMDD-HHMMSS-reponame-scope-shorthash/`. All working files (diffs, prompts, reviewer output, stderr) live inside. The folder name matches the log file name for easy correlation. The entire folder is cleaned up with `rm -rf` after the log is saved
- Error output (stderr) captured to separate files from review output (stdout) — prevents rate limit stack traces, deprecation warnings, and MCP errors from burying review findings

---

## 3. team-research-agent Subagent

**Status:** Implementing
**Priority:** Medium
**Added:** February 2026
**Location:** `~/.claude/agents/team-research-agent.md`

### Description

Pre-review documentation researcher that analyzes diffs to identify the key libraries/frameworks/APIs being changed, then researches their latest documentation, security advisories, and common mistakes. Writes research files that the review-collector injects into reviewer prompts. Does NOT review code — only researches the tools the code uses.

### Key Design Decisions

- **Separate agent, not embedded in review-collector:** The review-collector is explicitly a "mechanical collection only" agent. Adding web research would violate its separation of concerns. The research agent has a distinct responsibility — it gathers knowledge, not reviews.
- **Reads raw diff, not filtered:** The research agent needs to see `package.json`, `requirements.txt`, `Cargo.toml`, etc. to identify version changes and new dependencies. These files get stripped by the review-collector's diff filter, so the research agent reads `$REVIEW_TMP/review-raw-diff.txt` directly.
- **Prompt injection, not file references:** Research context is compiled into a compact text block and injected directly into George and Oscar's prompts. This is necessary because Codex `exec` processes piped stdin and can't read external files during execution. Injecting into the prompt text ensures uniform behavior across all review modes.
- **Enhancement, not a requirement:** If the research agent fails, times out, or finds nothing to research, the review workflow proceeds normally without research context. Partial research (2 out of 4 topics) is also valid.

### Research Strategy

1. **Diff analysis** — Scan `+` lines for imports, API calls, framework patterns, dependency file changes
2. **Topic prioritization** — Rank by: security-relevant > breaking changes > framework conventions > new deps > complex integrations. Max 5 topics, typically 2-3.
3. **Web search** — For each topic: official docs, security advisories (`"{library}" CVE`), common mistakes, migration guides for version bumps
4. **File writing** — One markdown file per topic with structured sections: Key Documentation Points, Common Mistakes, Security Considerations (if applicable), Sources

### Output Folder Structure

```
~/Downloads/Team-Research/{yyyymmddhhmmss}-{reponame}-{scope}-{short-hash}/
├── research-flask-security.md
├── research-sqlalchemy-queries.md
└── research-jwt-authentication.md
```

The folder name is dynamically constructed:
- **Timestamp:** `yyyymmddhhmmss` for sorting and uniqueness
- **Repo name:** from the current git repository
- **Scope:** a concise slug the agent determines from context (e.g., `pr42`, `feature-auth`, `staged`, `uncommitted`)
- **Short hash:** `git rev-parse --short HEAD` for searchability

### Integration Points

- **SKILL.md Step 1.5** invokes the research agent after diff capture and before review collection
- **review-collector Step 1** reads research files and compiles `RESEARCH_CONTEXT`
- **review-collector Step 3** injects `RESEARCH_CONTEXT` into both George and Oscar's prompts
- **review-collector Step 4** logs which research files were consumed
- **review-collector cleanup** deletes the research folder after the review log is saved

### Time Budget and Limits

- Max 5 research topics per review
- Target 2-3 minutes total research time
- 30-45 seconds per topic
- WebSearch failures → skip that topic, don't retry

### Error Handling

| Scenario | Behavior |
|----------|----------|
| Research agent timeout/crash | SKILL.md proceeds to Step 2 without research |
| WebSearch fails for a topic | Agent skips that topic, continues with others |
| Partial success (2 of 4 topics) | Writes whatever was found — partial research is still valuable |
| Trivial diff (README-only) | Returns "Research skipped" with no folder created |
| Research folder missing at review time | review-collector sets RESEARCH_CONTEXT to empty, proceeds normally |
| Cleanup failure | Research folder persists in ~/Downloads (small markdown files, user can clean manually) |

---

## 4. Future: MCP Server Upgrade

**Status:** Planned
**Priority:** Low
**Added:** February 2026

### Description

If sequential CLI calls prove too slow, upgrade the collection mechanism to a Python MCP server (~200-300 lines) for true async parallelization. The skill instructions transfer directly — only the collection layer changes. The MCP server would expose a `run_parallel_reviews(files, context)` tool that invokes both CLIs concurrently via Python's asyncio.

### Why Upgrade

- True parallel execution (currently pseudo-parallel via Bash `&`)
- Better error handling and retry logic
- Testable independently of Claude Code
- Could add more reviewers without changing the skill
- Same MCP server could serve Gemini CLI and Codex CLI sessions too