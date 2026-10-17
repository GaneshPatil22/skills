---
name: git-commit
description: Analyze changed files, stage them, and write a structured commit message describing the actual changes. Apply when committing work, writing commit messages, or splitting a messy working tree into clean commits.
---

# Git Commit Skill

Turn a working tree full of changes into clean, well-described commits by reading what actually changed—never by guessing from file names.

## Core Principles

### 1. **Read the Diff, Not the Filenames**
- A filename tells you where, the diff tells you what
- Always run `git diff` before writing a single word
- Describe behavior change, not file movement
- Never write "updated auth.ts"—write what the update does

### 2. **Explain Why, Not How**
- The diff already shows how; the message must supply why
- Future readers debug with messages, not with code
- State the problem solved or the capability added
- Skip restating code that is visible in the patch

### 3. **One Commit, One Logical Change**
- A commit should be revertable on its own
- Unrelated fixes belong in separate commits
- Refactoring and behavior change never share a commit
- If the message needs "and", consider splitting

### 4. **Stage Deliberately**
- Review every file before it enters the index
- Secrets, debug logs, and junk files never get staged
- `git add -A` is a decision, not a reflex
- Confirm the staged set matches the message you wrote

## Commit Message Format

Every commit message follows this exact shape:

```
<type>: <title>

<description>

- <detail line 1>
- <detail line 2>
- <detail line 3>
```

### Format Rules

| Part | Rule |
|------|------|
| **Title** | One line, imperative mood, under 72 characters, no trailing period |
| **Description** | One or two sentences explaining why the change was needed |
| **Details** | **Maximum 3 bullets**, each **10–15 words**, one change each |
| **Blank lines** | Required between title, description, and details |

**The 3-line limit is a hard cap.** If the change needs more than three detail
lines to explain, the commit is doing too much—split it.

### Title Types

| Type | Use For |
|------|---------|
| `feat` | New capability visible to a user or caller |
| `fix` | Corrects broken behavior |
| `refactor` | Restructures code without changing behavior |
| `perf` | Improves speed or resource usage |
| `test` | Adds or repairs tests only |
| `docs` | Documentation only |
| `chore` | Build config, dependencies, tooling |
| `style` | Formatting with zero logic change |

### A Correctly Formatted Message

```
feat: add retry logic to payment webhook handler

Stripe webhooks were dropped during brief database outages, leaving
orders unpaid with no record of the failed delivery.

- Retries failed webhook writes three times with exponential backoff
- Persists undeliverable payloads to the dead_letter table for replay
- Emits a metric on each retry so alerts fire before data loss
```

## The Commit Workflow

### Step 1: Survey the Working Tree

Run these together before anything else:

```bash
git status                  # what is modified, staged, untracked
git diff                    # unstaged changes, line by line
git diff --staged           # already-staged changes
git log --oneline -10       # the repo's existing message style
```

**What you are looking for**:
- Which files changed and whether they form one story or several
- Untracked files that may be junk, secrets, or genuinely new source
- Whether anything is already staged from a previous session
- The conventions the repo already follows

### Step 2: Read Every Change

```bash
git diff --stat                      # scale: files and line counts
git diff -- path/to/file.ts          # one file in detail
git diff --word-diff                 # precise small edits
git diff -M                          # detect renames, not delete+add
```

**Classify each file**:

```
Source logic      → the substance of the commit message
Tests             → supporting evidence, usually one detail line
Config / deps     → may belong in a separate chore commit
Generated files   → lockfiles, builds: mention briefly or exclude
Accidental edits  → stray console.log, commented code: remove first
```

### Step 3: Check Before Staging

Scan the diff for anything that must never be committed:

```bash
git diff | grep -nE '(API_KEY|SECRET|PASSWORD|TOKEN|BEGIN.*PRIVATE KEY)'
git diff | grep -nE '(console\.log|debugger|binding\.pry|TODO: remove)'
git status --porcelain | grep '^??'   # untracked: .env, dumps, builds
```

❌ **Stop and ask the user** if you find:
- Credentials, API keys, or private keys in the diff
- `.env` files or anything matching a secrets pattern
- Large binaries or build artifacts that belong in `.gitignore`
- Debug statements that look unintentional

### Step 4: Stage the Files

```bash
# All changes, tracked and untracked
git add -A

# Only specific files, when the tree holds unrelated work
git add src/payments/webhook.ts src/payments/webhook.test.ts

# Only tracked modifications, leaving untracked files alone
git add -u

# Interactive hunk selection, to split one file across commits
git add -p
```

Then verify the index matches your intent:

```bash
git diff --staged --stat
```

### Step 5: Write and Create the Commit

```bash
git commit -m "$(cat <<'EOF'
feat: add retry logic to payment webhook handler

Stripe webhooks were dropped during brief database outages, leaving
orders unpaid with no record of the failed delivery.

- Retries failed webhook writes three times with exponential backoff
- Persists undeliverable payloads to the dead_letter table for replay
- Emits a metric on each retry so alerts fire before data loss
EOF
)"
```

The quoted heredoc preserves newlines and prevents shell expansion of
backticks and `$` inside the message.

### Step 6: Verify

```bash
git log -1 --stat           # message plus the files it carries
git status                  # confirm the tree is clean
```

If the message is wrong and the commit is not yet pushed:

```bash
git commit --amend          # fix message or add a forgotten file
```

## Deriving the Message From the Diff

### Title: Name the Outcome

Read the diff, then answer: *what can the system do now that it could not before?*

```
Added a function that validates coupon codes against expiry dates
→ feat: validate coupon expiry before applying discount

Changed < to <= in a boundary check inside pagination
→ fix: include final page in paginated results

Extracted duplicated query building into a shared helper
→ refactor: extract shared query builder from report handlers
```

### Description: Name the Reason

```
❌ "This commit changes the UserService class and adds a method."
   Restates the diff, adds nothing.

✅ "Users with expired sessions saw a blank dashboard instead of a
   login prompt, because the session check ran after the render."
   Explains the failure the change fixes.
```

### Details: One Change Per Line, 10–15 Words

```
❌ Too short, says nothing
- Fixed bug
- Updated tests

❌ Too long, exceeds 15 words
- Added a brand new retry mechanism to the webhook handler that will attempt
  the request again up to three times using exponential backoff timing

✅ 10–15 words, one concrete change each
- Retries failed webhook writes three times with exponential backoff
- Persists undeliverable payloads to the dead_letter table for replay
```

**Choosing the three lines**: pick the three changes a reviewer most needs
to know. Mechanical edits—import reordering, formatting, renames—do not earn
a line unless they are the whole commit.

## Splitting a Mixed Working Tree

When `git status` shows unrelated work, do not force one commit.

### Recognizing a Mixed Tree

```
🚩 A bug fix and a new feature in the same set of files
🚩 Source changes plus an unrelated dependency bump
🚩 Formatting across the repo plus one real logic change
🚩 A title that needs "and" to be accurate
```

### Splitting by File

```bash
git add src/auth/session.ts src/auth/session.test.ts
git commit -m "fix: ..."

git add package.json package-lock.json
git commit -m "chore: ..."
```

### Splitting Inside One File

```bash
git add -p src/api/handler.ts
# y = stage hunk, n = skip, s = split further, q = quit
git commit -m "fix: ..."

git add src/api/handler.ts
git commit -m "feat: ..."
```

### Ordering the Commits

```
1. chore / deps     → nothing depends on them
2. refactor         → prepares the ground, no behavior change
3. fix / feat       → the substance
4. test             → unless tests ship with their code, which is better
5. docs             → describes what now exists
```

## Worked Examples

### A Bug Fix

**Diff**: `src/cart/total.ts` changes `Math.round` to `Math.floor` on tax;
`src/cart/total.test.ts` gains two cases.

```
fix: floor tax amounts to prevent overcharging by one cent

Rounding tax up meant carts ending in certain subtotals charged a cent
more than the invoice displayed, triggering reconciliation failures.

- Replaces rounding with floor so tax never exceeds the displayed value
- Adds regression cases for the two subtotals that triggered the mismatch
```

### A New Feature

**Diff**: new `src/export/csv.ts`, a route in `src/routes/reports.ts`,
a button in `src/ui/ReportToolbar.tsx`, plus tests.

```
feat: export report results to CSV from the toolbar

Analysts were copying tables into spreadsheets by hand, which broke on
large result sets and lost column formatting.

- Streams query results to CSV without buffering the full set in memory
- Adds a download route guarded by the existing report permission check
- Wires an export button into the report toolbar with progress state
```

### A Refactor

**Diff**: three handlers lose duplicated validation; a new
`src/lib/validate.ts` gains it; no test changes.

```
refactor: extract duplicated request validation into shared helper

The same field checks were copied across three handlers, so each fix had
to be applied three times and one copy had already drifted.

- Moves identical validation logic into a single reusable helper module
- Updates three handlers to call the helper instead of inline checks
- Keeps behavior identical, verified by the existing handler test suite
```

### A Dependency Bump

**Diff**: `package.json` and lockfile only.

```
chore: upgrade axios to 1.7.4 for CVE-2024-28849 fix

The pinned version carried a follow-redirects vulnerability flagged by
the dependency audit in CI.

- Bumps axios from 1.6.2 to 1.7.4 and refreshes the lockfile
```

## Handling Common Situations

### Nothing Is Staged

Survey first, then stage what belongs together. Do not assume the user
wants everything—if the tree holds unrelated work, say so and propose a split.

### Something Is Already Staged

```bash
git diff --staged
```

The user likely staged it deliberately. Commit exactly that set, and mention
any unstaged changes you are leaving behind.

### The Commit Fails a Pre-Commit Hook

```bash
git commit ...           # hook rejects, files may be auto-fixed
git status               # check what the hook modified
git add -A && git commit --amend --no-edit   # include the fixes
```

Never pass `--no-verify` to bypass a hook unless the user explicitly asks.

### The Last Commit Needs Fixing

```bash
git commit --amend                    # not pushed: rewrite freely
git commit --amend --no-edit          # add a file, keep the message
git revert <sha>                      # already pushed: revert, never amend
```

❌ Never amend, rebase, or force-push a commit that exists on a shared branch.

### A Merge Conflict Is in the Tree

```bash
git status | grep 'both modified'
```

Resolve every conflict and remove all markers before staging. Never commit
a file containing `<<<<<<<`.

## Commit Checklist

**Before staging**
- [ ] `git status` and `git diff` have both been read
- [ ] Every changed file is accounted for and intentional
- [ ] No secrets, keys, or `.env` files in the diff
- [ ] No stray debug statements or commented-out code
- [ ] Build artifacts and large binaries excluded

**The message**
- [ ] Title is imperative, under 72 characters, no trailing period
- [ ] Title carries a correct type prefix
- [ ] Description explains why, not what the diff already shows
- [ ] Detail lines number three or fewer
- [ ] Each detail line runs 10–15 words
- [ ] Blank lines separate title, description, and details

**The commit**
- [ ] Staged files match the message exactly
- [ ] The change is one logical unit, revertable on its own
- [ ] Tests ship alongside the code they cover
- [ ] `git log -1 --stat` confirms the result

## Anti-Patterns

❌ **Vague Titles**
```
"update", "fix bug", "wip", "changes", "misc"
```
- Fix: name the specific behavior that changed

❌ **Narrating the Diff**
```
"Modified UserService.ts and added three methods to it"
```
- Fix: explain the capability or fix, not the file edits

❌ **Past Tense Titles**
```
"Added retry logic" / "Fixed the pagination bug"
```
- Fix: imperative mood—"add retry logic", "fix pagination"

❌ **The Everything Commit**
```
"feat: add export, fix cart tax, bump deps, reformat"
```
- Fix: four commits, staged separately

❌ **Blind `git add -A`**
```
Stages .env, node_modules, and a 200MB dump along with the fix
```
- Fix: read `git status` first, stage by path

❌ **Detail Lines That Sprawl**
```
Eight bullets restating every line of the diff
```
- Fix: three lines maximum, the three that matter

❌ **Committing a Broken Tree**
```
Commit lands with failing tests or unresolved conflict markers
```
- Fix: run the test suite and search for markers before committing

❌ **Rewriting Shared History**
```
git commit --amend on a pushed branch, then force-push
```
- Fix: `git revert` for anything already shared

## Quick Reference

```bash
# Survey
git status
git diff
git diff --staged
git diff --stat
git log --oneline -10

# Stage
git add -A                  # everything
git add <paths>             # specific files
git add -u                  # tracked modifications only
git add -p                  # hunk by hunk
git restore --staged <path> # unstage

# Commit
git commit -m "$(cat <<'EOF'
type: title

Description.

- Detail line
EOF
)"

# Fix
git commit --amend          # unpushed: rewrite
git commit --amend --no-edit
git revert <sha>            # pushed: new commit undoing it

# Verify
git log -1 --stat
git show --stat HEAD
```

---

**Remember**: A commit message is written once and read for years. Read the diff before you write, explain why rather than what, and keep it to a title, a reason, and at most three lines that matter.
