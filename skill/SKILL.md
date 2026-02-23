---
name: team-code-review
description: Multi-tool code review workflow. Sends code to Gemini CLI (George) and Codex CLI (Oscar) for independent reviews, then critically analyzes their suggestions and triages them into 3 buckets for the user to pick from. Invoke with /team-code-review.
user-invocable: true
disable-model-invocation: true
---

# Team Code Review

You are orchestrating a multi-tool code review. Two external AI reviewers — **George** (Gemini CLI) and **Oscar** (Codex CLI) — will independently review the code. You will then critically analyze their suggestions and present them to the user for action.

## Step 0: Create a Unique Temp Folder

Before doing anything else, create an isolated temp folder for this review run. This prevents concurrent reviews from clobbering each other's files.

```bash
REPO_NAME=$(basename "$(git rev-parse --show-toplevel 2>/dev/null)" 2>/dev/null || echo "unknown")
SHORT_HASH=$(git rev-parse --short HEAD 2>/dev/null || echo "0000000")
TIMESTAMP=$(date +%Y%m%d-%H%M%S)
# Scope will be filled in during Step 1 (e.g., pr42, staged, feat-auth)
```

After parsing the user's input in Step 1, finalize the folder:

```bash
REVIEW_TMP="/tmp/team-review-${TIMESTAMP}-${REPO_NAME}-${SCOPE}-${SHORT_HASH}"
mkdir -p "$REVIEW_TMP"
```

All temp files for this review go inside `$REVIEW_TMP/`. Pass `$REVIEW_TMP` to both subagents.

## Step 1: Identify the Code to Review

The user's input: `$ARGUMENTS`

Parse the input to determine what to review. The user may provide any of these:

- **PR number** (e.g., `PR #42`, `pr 42`, `#42`) — Set `SCOPE="pr42"`. Run `gh pr view 42` to get the PR title, description, and metadata. Save the diff to a file with `gh pr diff 42 > "$REVIEW_TMP/review-raw-diff.txt"`
- **File paths** (e.g., `src/auth.ts src/middleware.ts`) — Set `SCOPE="files"`. Note the file paths (do NOT read them into context yet)
- **Branch name** (e.g., `feature/payments`) — Set `SCOPE` to a slug of the branch name (e.g., `feat-payments`). Save the diff to a file with `git diff main...feature/payments > "$REVIEW_TMP/review-raw-diff.txt"`
- **"staged"** or **"uncommitted"** — Set `SCOPE="staged"` or `SCOPE="uncommitted"`. Save the diff to a file with `git diff --staged > "$REVIEW_TMP/review-raw-diff.txt"` or `git diff > "$REVIEW_TMP/review-raw-diff.txt"`
- **Review notes** (e.g., `focus on error handling in the payment flow`) — Use as guidance for what to emphasize, and ask the user which files or PR to apply it to
- **Combination** (e.g., `PR #42 focus on the auth changes`) — Parse the PR number AND pass the notes as review focus

After setting `SCOPE`, create the `$REVIEW_TMP` folder as shown in Step 0.

**IMPORTANT: Do NOT read the diff into your conversation context.** Save it to `$REVIEW_TMP/review-raw-diff.txt` and let the review-collector subagent work from that folder. This avoids consuming the diff in context three times (main conversation + subagent + analysis). You will read specific files/lines later when analyzing findings — not the full diff.

For PR reviews, DO run `gh pr view` to get the title and description (small, useful context). Do NOT run `gh pr diff` into your context.

If `$ARGUMENTS` is empty, **STOP and ask the user before doing anything else.** Use AskUserQuestion with these options:

Question: "What should the team review?"
Options:
- **A Pull Request** — "I'll ask for the PR number and optional focus areas"
- **Specific files** — "I'll ask which files to send to George and Oscar"
- **Staged/uncommitted changes** — "I'll review your current git changes"

After the user responds, follow up to get the specifics (PR number, file paths, focus notes, etc.) before proceeding to Step 2.

## Step 1.5: Delegate to the Research Agent

Before sending code to reviewers, gather documentation context for the libraries and APIs being changed. Use the **team-research-agent** subagent. Pass it:
- The path to the raw diff file (`$REVIEW_TMP/review-raw-diff.txt`)
- The review scope (PR number, branch name, staged, uncommitted, file paths)
- The repository name
- Any focus notes from the user

The research agent will:
1. Analyze the raw diff to identify key libraries/frameworks/APIs being changed
2. WebSearch for latest docs, security advisories, and common mistakes
3. Write research files to a folder in `~/Downloads/Team-Research/`
4. Return the research folder path and a summary of topics researched

**Capture the research folder path** from the subagent's response. You'll pass this to the review-collector in the next step.

**If the research agent fails, times out, or returns "Research skipped":** Proceed to Step 2 without research. Research is an enhancement, not a requirement — the review workflow works fine without it.

## Step 2: Delegate to the Review Collector

Use the **review-collector** subagent to invoke both Gemini CLI and Codex CLI. Pass it:
- The review mode (PR number, file paths, staged, uncommitted, branch)
- **The temp folder path** (`$REVIEW_TMP`) — the diff is at `$REVIEW_TMP/review-raw-diff.txt`
- Any focus notes from the user
- Any relevant project context (tech stack, conventions from CLAUDE.md, etc.)
- **The research folder path** from Step 1.5 (if research was performed), or indicate that no research is available

The subagent will read the research files, inject documentation context into both reviewer prompts, filter the diff, send it to both reviewers, and return an assembled review package with all suggestions numbered sequentially.

## Step 3: Read Code for Specific Findings

After receiving the review package, **now** read the specific files and lines referenced in each finding. Do NOT read the entire diff — only read the files/lines that the reviewers flagged. This gives you the code context needed for analysis while keeping context usage minimal.

## Step 4: Critically Analyze the Reviews

**This is the most important step.** Apply this framing:

> Two friends reviewed your code — they could be wrong. They often over-engineer, take shortcuts, or miss business goals and project requirements. Don't assume their suggestions are accurate. But let's do high-quality work we're proud of that breezes through PR review. What do you think?

For EACH numbered suggestion, evaluate it against:
- **Project context**: Does it align with the project's conventions, goals, and constraints?
- **Practical value**: Does it fix a real bug, security issue, or meaningful quality problem?
- **Proportionality**: Is the effort proportional to the benefit?
- **Over-engineering risk**: Is it adding unnecessary complexity, abstraction, or premature optimization?
- **Best practices**: Is it genuinely best practice, or just stylistic preference?

## Step 5: Triage into 3 Buckets

Present ALL suggestions organized into these three buckets, keeping the original numbers:

### BUCKET 1: Absolutely Should Do
Suggestions that are clearly correct, fix real issues, or meaningfully improve code quality. These should be no-brainers.

### BUCKET 2: Let's Talk About This
Suggestions that have merit but involve tradeoffs, are context-dependent, or where you're not sure if they align with project goals. Include a brief note on what you'd want to discuss.

### BUCKET 3: Bad Idea / Over-Engineering
Suggestions you disagree with. Explain briefly WHY each is a bad idea for this project (over-engineering, wrong pattern, stylistic preference disguised as best practice, etc.).

## Step 6: Present for User Action

After presenting the buckets, tell the user:

> **Pick by number.** For example:
> - "Do 1, 3, 5" — I'll implement these
> - "Skip 7, 8" — acknowledged, moving on
> - "Discuss 4" — let's talk about this one

## Step 7: Implement

After the user picks:
- Implement all "Do" items
- Skip all "Skip" items
- Discuss any "Discuss" items before implementing
- After implementation, briefly summarize what was changed

## Output Format

Follow this example output EXACTLY. The executive summary comes first, then detailed findings in numerical order. Every finding uses the same 4-part structure: title line, file, code block, analysis paragraph. No exceptions, no variations.

### Example Output

---

## Team Code Review Results

**Scope reviewed:** PR #13 — feat(email): Complete SendGrid integration
**Files reviewed:** 8 files (service, webhook, templates, types, tests)
**George (Gemini):** 6 suggestions | **Oscar (Codex):** 3 suggestions

---

### Executive Summary

**Absolutely Should Do:** #3, #4, #9
- **#3** — XSS in confirm dialog
- **#4** — Gitignore generated PII file
- **#9** — DND false-positive logic bug

**Let's Talk About:** #2, #5
- **#2** — Hardcoded PII in setup script
- **#5** — NANPA validation gaps

**Bad Idea / Over-Engineering:** #1, #6, #7, #8
- **#1** — Full auth layer for local admin
- **#6** — DRY extraction for XML builder
- **#7** — HTTPS for local phone API
- **#8** — Standardize flash categories

**Pick by number:** "Do 3, 4, 9" / "Skip 1, 6, 7, 8" / "Discuss 2, 5"

---

### Detailed Findings

The detailed findings section is NOT where you pass judgement. The executive summary already has the do/skip/discuss triage. This section presents each reviewer's feedback neutrally so the user can see the details and evaluate for themselves.

**Attribution rules:**
- If only George flagged it: *(George)*
- If only Oscar flagged it: *(Oscar)*
- If both flagged it independently: *(George and Oscar)*

**Tone rules:**
- Do NOT use judgement language like "False positive", "Over-engineering", "Bikeshedding", "Bad suggestion"
- DO present what the reviewer found, show the code, and provide factual technical context
- The analysis should help the user understand the issue — not tell them it's wrong or right
- Let the executive summary handle the triage; the details just show the evidence

---

**#1** — No authentication or login gate on Flask admin app *(George)*

**File:** `admin/app.py:792`

```python
if __name__ == '__main__':
    app.run(host='127.0.0.1', port=5000, debug=False)
```

**Analysis:** George flags the lack of authentication on the admin interface. The app is currently bound to `127.0.0.1` (localhost only, line 792). Adding Flask-Login would require user registration, password management, and session handling. If the app were bound to `0.0.0.0`, auth would become critical.

---

**#2** — Hardcoded PII in setup script (DID, address, email, PIN) *(George)*

**File:** `setup_voipms.py:14-22`

```python
ACCOUNT_DID = "5551234567"
HOME_ADDRESS = "123 Main St, Springfield"
ACCOUNT_EMAIL = "user@example.com"
VOICEMAIL_PIN = "1234"
```

**Analysis:** George flags the DID, home address, and email sitting directly in source. This is a one-time setup script designed to run once with project-specific constants. Moving them to `.env` would parameterize them but makes the script harder to follow. The risk depends on whether this repo is public or private.

---

**#3** — XSS via unescaped contact name in confirm dialog *(George)*

**File:** `admin/templates/whitelist.html:87`

```html
<button onclick="confirm('Delete {{ contact.name }}?')">
```

**Analysis:** George flags that if `contact.name` contains an apostrophe (e.g., O'Brien), the inline JS string breaks. A crafted name could execute arbitrary code. The fix would be `{{ contact.name|tojson }}` which properly escapes for JS string contexts.

---

**#4** — `phonebook.xml` contains real names and numbers, should be gitignored *(George)*

**File:** `configs/phonebook.xml:1-5`

```xml
<AddressBook>
  <Contact>
    <LastName>Smith</LastName>
    <FirstName>John</FirstName>
    <Phone><phonenumber>5551234567</phonenumber></Phone>
```

**Analysis:** George flags that this generated file contains PII (real names and phone numbers of family members) and is tracked in git. Adding it to `.gitignore` is a one-liner.

---

**#5** — NANPA normalization doesn't validate length or handle 7-digit numbers *(George)*

**File:** `admin/app.py:156-164`

```python
def normalize_phone(number: str) -> str:
    digits = re.sub(r'\D', '', number)
    if len(digits) == 11 and digits.startswith('1'):
        digits = digits[1:]
    return digits
```

**Analysis:** George flags that the function strips non-digits and handles 1+10 → 10, but doesn't reject invalid lengths or handle 7-digit local numbers. VoIP.ms CallerID filtering uses 10-digit NANPA, so an invalid-length entry would silently fail to match. Whether this matters depends on whether contacts are always entered as full 10-digit numbers.

---

**#6** — XML builder logic is duplicated between two files *(George)*

**Files:** `admin/app.py:320-345`, `generate_phonebook_xml.py:48-72`

```python
# Nearly identical in both files:
entry = ET.SubElement(root, 'Contact')
ET.SubElement(entry, 'LastName').text = parts[-1]
ET.SubElement(entry, 'FirstName').text = parts[0] if len(parts) > 1 else ''
phone = ET.SubElement(entry, 'Phone')
ET.SubElement(phone, 'phonenumber').text = number
```

**Analysis:** George flags nearly identical XML builder code in both files. The XML format is dictated by Grandstream hardware. Extracting to a shared module would add a file and import path. The two components currently work independently.

---

**#7** — `GrandstreamClient` uses plain HTTP for phone API calls *(George)*

**File:** `admin/app.py:89`

```python
self.base_url = f"http://{phone_ip}"
```

**Analysis:** George flags the use of HTTP rather than HTTPS for the Grandstream phone management API. The GHP631W communicates on the local network. Whether the device's management interface supports HTTPS depends on the firmware.

---

**#8** — Inconsistent flash categories and silent failures *(George)*

**File:** `admin/app.py:410, 425, 438`

```python
flash("Filter update failed", "warning")   # line 410
flash("DND toggle failed", "danger")        # line 425
flash("Phone unreachable", "warning")       # line 438
```

**Analysis:** George flags that some failures use "warning" while others use "danger". The current usage is `warning` for non-critical issues and `danger` for hard failures. `_get_phone_status` returns defaults on failure; the troubleshoot page shows separate health flags.

---

**#9** — `_is_dnd_active` returns True when filter lookup returns empty list *(Oscar)*

**File:** `admin/app.py:245-252`

```python
def _is_dnd_active(self):
    filters = self.client.get_filters(account=self.account)
    for f in filters:
        if f['filter'] == self.family_filter_id:
            if f['routing'] == 'voicemail':
                return True
            return False
    return True  # empty list falls through here
```

**Analysis:** Oscar flags that if a filter ID in `.env` becomes stale (deleted on VoIP.ms), the function iterates over an empty list, never enters the loop, and falls through to `return True`. This reports DND as active when it isn't. A guard like `if not filters: return False` at the top would handle this case.

---

**Pick by number.** For example:
- "Do 3, 4, 9" — I'll implement these
- "Skip 1, 6, 7, 8" — acknowledged, moving on
- "Discuss 2, 5" — let's talk about these

---

### End of Example Output