---
name: team-code-review
description: Multi-tool code review workflow. Sends code to Gemini CLI (George) and Codex CLI (Oscar) for independent reviews, then critically analyzes their suggestions and triages them into 3 buckets for the user to pick from. Invoke with /team-code-review.
user-invocable: true
disable-model-invocation: true
---

# Team Code Review

You are orchestrating a multi-tool code review. Two external AI reviewers — **George** (Gemini CLI) and **Oscar** (Codex CLI) — will independently review the code. You will then critically analyze their suggestions and present them to the user for action.

## Step 1: Identify the Code to Review

The user's input: `$ARGUMENTS`

Parse the input to determine what to review. The user may provide any of these:

- **PR number** (e.g., `PR #42`, `pr 42`, `#42`) — Run `gh pr diff 42` to get the changes and `gh pr view 42` for context
- **File paths** (e.g., `src/auth.ts src/middleware.ts`) — Read those files directly
- **Branch name** (e.g., `feature/payments`) — Run `git diff main...feature/payments` to get the changes
- **"staged"** or **"uncommitted"** — Run `git diff --staged` or `git diff` respectively
- **Review notes** (e.g., `focus on error handling in the payment flow`) — Use as guidance for what to emphasize, and ask the user which files or PR to apply it to
- **Combination** (e.g., `PR #42 focus on the auth changes`) — Parse the PR number AND pass the notes as review focus

If `$ARGUMENTS` is empty, **STOP and ask the user before doing anything else.** Use AskUserQuestion with these options:

Question: "What should the team review?"
Options:
- **A Pull Request** — "I'll ask for the PR number and optional focus areas"
- **Specific files** — "I'll ask which files to send to George and Oscar"
- **Staged/uncommitted changes** — "I'll review your current git changes"

After the user responds, follow up to get the specifics (PR number, file paths, focus notes, etc.) before proceeding to Step 2.

Once you have the code, read the relevant files/diff to understand it yourself before sending for review.

## Step 2: Delegate to the Review Collector

Use the **review-collector** subagent to invoke both Gemini CLI and Codex CLI. Pass it:
- The file paths or diff to review
- Any relevant project context (tech stack, conventions from CLAUDE.md, etc.)

The subagent will return an assembled review package with all suggestions numbered sequentially.

## Step 3: Critically Analyze the Reviews

**This is the most important step.** Apply this framing:

> Two friends reviewed your code — they could be wrong. They often over-engineer, take shortcuts, or miss business goals and project requirements. Don't assume their suggestions are accurate. But let's do high-quality work we're proud of that breezes through PR review. What do you think?

For EACH numbered suggestion, evaluate it against:
- **Project context**: Does it align with the project's conventions, goals, and constraints?
- **Practical value**: Does it fix a real bug, security issue, or meaningful quality problem?
- **Proportionality**: Is the effort proportional to the benefit?
- **Over-engineering risk**: Is it adding unnecessary complexity, abstraction, or premature optimization?
- **Best practices**: Is it genuinely best practice, or just stylistic preference?

## Step 4: Triage into 3 Buckets

Present ALL suggestions organized into these three buckets, keeping the original numbers:

### BUCKET 1: Absolutely Should Do
Suggestions that are clearly correct, fix real issues, or meaningfully improve code quality. These should be no-brainers.

### BUCKET 2: Let's Talk About This
Suggestions that have merit but involve tradeoffs, are context-dependent, or where you're not sure if they align with project goals. Include a brief note on what you'd want to discuss.

### BUCKET 3: Bad Idea / Over-Engineering
Suggestions you disagree with. Explain briefly WHY each is a bad idea for this project (over-engineering, wrong pattern, stylistic preference disguised as best practice, etc.).

## Step 5: Present for User Action

After presenting the buckets, tell the user:

> **Pick by number.** For example:
> - "Do 1, 3, 5" — I'll implement these
> - "Skip 7, 8" — acknowledged, moving on
> - "Discuss 4" — let's talk about this one

## Step 6: Implement

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

---

**#1** — Flask app has no authentication or login gate *(George)*

**File:** `admin/app.py:792`

```python
if __name__ == '__main__':
    app.run(host='127.0.0.1', port=5000, debug=False)
```

**Analysis:** This sounds alarming in the abstract, but this is a local admin interface bound to `127.0.0.1`. Adding Flask-Login, user registration, password management, and session handling would be significant complexity for a tool that's essentially a local dashboard. If this ever binds to `0.0.0.0`, auth becomes critical — but right now this is over-engineering for the deployment model.

---

**#2** — Hardcoded PII in setup script (DID, address, email, PIN) *(George)*

**File:** `setup_voipms.py:14-22`

```python
ACCOUNT_DID = "5551234567"
HOME_ADDRESS = "123 Main St, Springfield"
ACCOUNT_EMAIL = "user@example.com"
VOICEMAIL_PIN = "1234"
```

**Analysis:** This is a real concern if the repo is public or shared. The DID, home address, and email are in the source. That said, this is a one-time setup script designed to be run once, and moving everything to `.env` makes the script harder to understand and use. Worth discussing: is this repo public? If so, these should absolutely be parameterized. If it stays private, the risk is lower.

---

**#3** — XSS via unescaped contact name in confirm dialog *(George)*

**File:** `admin/templates/whitelist.html:87`

```html
<button onclick="confirm('Delete {{ contact.name }}?')">
```

**Analysis:** If `contact.name` contains an apostrophe (e.g., O'Brien), this breaks the JS string. Worse, a crafted name could execute arbitrary code. Fix: use `{{ contact.name|tojson }}` which properly escapes for JS string contexts.

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

**Analysis:** This is a generated file containing PII (real names and phone numbers of family members). It should be in `.gitignore`, not committed. Simple one-liner fix.

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

**Analysis:** The function strips non-digits and handles the 1+10 → 10 case, but doesn't reject invalid lengths or handle 7-digit local numbers. VoIP.ms CallerID filtering uses 10-digit NANPA, so an invalid-length entry would silently fail to match. The question is whether this is a real scenario — if contacts are always entered as full 10-digit numbers, the current code is fine. If the form might receive 7-digit or international numbers, validation would prevent confusing silent failures.

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

**Analysis:** Yes, the code is nearly identical in both places. But this is a small, stable piece of logic — the XML format is dictated by Grandstream hardware and won't change. Extracting it into a shared module adds a new file, a new import path, and coupling between two components that currently work independently. The pragmatic choice is to leave them separate unless you're actively iterating on the format.

---

**#7** — `GrandstreamClient` uses plain HTTP for phone API calls *(George)*

**File:** `admin/app.py:89`

```python
self.base_url = f"http://{phone_ip}"
```

**Analysis:** The Grandstream GHP631W hotel phone almost certainly doesn't support HTTPS on its management interface. This is communicating on a local network with a consumer VoIP phone. HTTPS is technically correct but practically impossible given the hardware.

---

**#8** — Inconsistent flash categories and silent failures *(George)*

**File:** `admin/app.py:410, 425, 438`

```python
flash("Filter update failed", "warning")   # line 410
flash("DND toggle failed", "danger")        # line 425
flash("Phone unreachable", "warning")       # line 438
```

**Analysis:** The flash categories (warning vs danger) are used appropriately in context — `warning` for non-critical issues, `danger` for failures. The `_get_phone_status` returning defaults on failure is intentional — the troubleshoot page already shows health flags. This is bikeshedding.

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
    return True  # BUG: empty list falls through here
```

**Analysis:** If a filter ID in `.env` becomes stale (deleted on VoIP.ms), the function iterates over an empty list, never enters the loop, and falls through to `return True`. This incorrectly reports DND as active, which means toggling DND would try to disable it when it was never actually enabled. Fix: add `if not filters: return False` at the top.

---

**Pick by number.** For example:
- "Do 3, 4, 9" — I'll implement these
- "Skip 1, 6, 7, 8" — acknowledged, moving on
- "Discuss 2, 5" — let's talk about these

---

### End of Example Output