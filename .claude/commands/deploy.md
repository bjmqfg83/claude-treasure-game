Deploy the project to GitHub Pages.

Follow these phases in order. Use the Bash tool for shell commands. Use AskUserQuestion for decisions. Run independent commands in parallel where possible.

---

## Phase 0 — Gather Context (silent — no output to user)

Run these commands **in parallel** and store results as internal variables:

**Batch 1 (parallel):**
- `git remote get-url origin 2>/dev/null` → parse `REPO_OWNER`/`REPO_NAME` from URL (strip `.git`). If fails → `HAS_REMOTE=false`
- `git branch --show-current` → `CURRENT_BRANCH`
- `uname -s` → `OS_TYPE`
- `gh --version 2>/dev/null` → `GH_INSTALLED` true/false
- `test -f .github/workflows/deploy.yml && echo yes || echo no` → `WORKFLOW_EXISTS`
- `test -d node_modules && echo yes || echo no` → `DEPS_INSTALLED`
- `test -f package-lock.json && echo yes || echo no` → `HAS_LOCKFILE`
- `git status --porcelain` → `GIT_STATUS`

**Batch 2 (depends on batch 1):**
- If `GH_INSTALLED=true`: `gh auth status 2>/dev/null` → `GH_AUTHED` true/false
- Read `vite.config.ts` → extract `base:` value as `VITE_BASE`

---

## Phase 1 — Prerequisites Gate

Check each prerequisite in order. **Stop at the first failure.**

### 1a. GitHub CLI installed?
If `GH_INSTALLED=false`: show install command by `OS_TYPE`:
- Darwin → `brew install gh`
- Linux → `sudo apt install gh` or `sudo dnf install gh`
- Other → https://cli.github.com/

Tell user: "Run `! brew install gh` (or the appropriate command) in this prompt, then re-run `/deploy`." **Stop.**

### 1b. GitHub CLI authenticated?
If `GH_AUTHED=false`: tell user "Run `! gh auth login` to sign in, then re-run `/deploy`." **Stop.**

### 1c. On main branch?
If `CURRENT_BRANCH` ≠ `main`: use AskUserQuestion "You're on branch `{CURRENT_BRANCH}`, not `main`. Deploy from this branch or switch to main?" with:
- Option 1: Switch to `main` (run `git checkout main`)
- Option 2: Deploy from `{CURRENT_BRANCH}` anyway

If switched, update `CURRENT_BRANCH=main`.

### 1d. Dependencies installed?
If `DEPS_INSTALLED=false`: run `npm install`. If it fails, show the error and **stop**.

### 1e. Lock file exists?
If `HAS_LOCKFILE=false`: warn "No `package-lock.json` found — CI uses `npm ci` which requires it. Running `npm install` to generate one." Run `npm install`. This creates `package-lock.json` so CI won't break.

---

## Phase 2 — Repository Setup

**Skip entirely if `HAS_REMOTE=true`.**

1. Read `package.json` `name` → slugify (lowercase, spaces→hyphens) as `SUGGESTED_NAME`.

2. Use AskUserQuestion with **two questions in one call**:
   - Q1: "No GitHub remote found. Repository name?" — Options: "Use `{SUGGESTED_NAME}`" / "Enter custom name"
   - Q2: "Visibility?" — Options: "Public" / "Private"

   If custom name chosen, use AskUserQuestion for the name. Store `REPO_NAME` and `REPO_VISIBILITY`.

3. Create the repo **without `--push`** (avoids HTTP 400 on large repos):
   ```
   gh repo create {REPO_NAME} --{REPO_VISIBILITY} --source=. --remote=origin
   ```
   If fails → show error and **stop**.

4. Get owner: `gh repo view --json owner -q .owner.login` → `REPO_OWNER`.

5. Push initial code separately:
   ```
   git push -u origin {CURRENT_BRANCH}
   ```
   **On push failure** (HTTP 400 / "unexpected disconnect" — common with large binary assets):
   - Retry with `git push -u origin {CURRENT_BRANCH} --no-thin`
   - If still fails: tell user "Run `! git push -u origin {CURRENT_BRANCH}` directly, then re-run `/deploy`." **Stop.**

6. Set `HAS_REMOTE=true`.

---

## Phase 3 — Infrastructure Validation

Run all checks silently, collect issues, then resolve.

**Checks (run in parallel where possible):**

| # | Check | Condition |
|---|-------|-----------|
| A | Workflow file | `.github/workflows/deploy.yml` missing |
| B | Vite base path | `VITE_BASE` ≠ `/{REPO_NAME}/` |
| C | GitHub Pages | `gh api repos/{REPO_OWNER}/{REPO_NAME}/pages` returns non-zero |
| D | Lock file in repo | `package-lock.json` not tracked (`git ls-files --error-unmatch package-lock.json` fails) |

Skip Check B if Phase 2 was just executed (already validated).

**If no issues:** "All infrastructure checks passed." → proceed to Phase 4.

**If issues found:** Show numbered list, then use AskUserQuestion:
"Found {N} issue(s) that need fixing. How to proceed?"
- Option 1: Claude fixes all automatically
- Option 2: Let me choose per issue
- Option 3: I'll fix everything manually (show me what's needed)

**If Option 2:** For each issue, AskUserQuestion "Issue: {desc}" with "Claude fix" / "I'll fix manually".
**If Option 3:** Show all required changes in one summary, then AskUserQuestion with "Continue" when user is done.

### Fix recipes:

**A — Missing workflow:** Create `.github/workflows/deploy.yml`:
```yaml
name: Deploy to GitHub Pages

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: write

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - name: Install dependencies
        run: npm ci
      - name: Build
        run: npm run build
      - name: Deploy to gh-pages
        uses: peaceiris/actions-gh-pages@v4
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./build
```

**B — Vite base mismatch:** Edit `vite.config.ts`, set `base: '/{REPO_NAME}/'`.

**C — Pages not enabled:** Run `gh api --method POST repos/{REPO_OWNER}/{REPO_NAME}/pages -f source[branch]=gh-pages -f source[path]=/`.
If it fails (gh-pages branch doesn't exist yet): this is expected for new repos. Pages activates automatically after the first deployment creates the branch. Note this and continue.

**D — Lock file not tracked:** Run `git add package-lock.json`. It will be committed in Phase 5.

---

## Phase 4 — Build

Run `npm run build`.

- **Fail:** show error. "Build failed — fix the errors and re-run `/deploy`." **Stop.**
- **Pass:** "Build succeeded."

---

## Phase 5 — Commit & Push

### 5.1 Check for changes
Run `git status --porcelain`.

- **Empty + no unpushed commits** (`git log origin/{CURRENT_BRANCH}..HEAD --oneline` also empty): "Nothing to commit, branch is up to date." → skip to Phase 6.
- **Empty + unpushed commits exist:** "No new changes, but unpushed commits found. Proceeding to push." → skip to 5.5.

### 5.2 Stage
Run `git add .`

### 5.3 Generate commit message
Run `git diff --cached --stat` and `git diff --cached`. Generate a concise commit message: imperative mood, ≤50 chars (e.g., "Add chest animation and update scoring logic").

### 5.4 Confirm message
AskUserQuestion: "Commit message:" with:
- Option 1: Use `{generated_message}`
- Option 2: Enter custom message

If Option 2: AskUserQuestion for custom text.

Commit using heredoc format (always append Co-Authored-By):
```
git commit -m "$(cat <<'EOF'
{chosen_message}

Co-Authored-By: Claude <noreply@anthropic.com>
EOF
)"
```

### 5.5 Push decision
AskUserQuestion: "Push to `{CURRENT_BRANCH}`?" with:
- Option 1: Claude pushes now
- Option 2: I'll push manually

**Option 2:** Show `git push origin {CURRENT_BRANCH}` and the Actions URL. **Stop.**

### 5.6 Push
Run `git push origin {CURRENT_BRANCH}`.

**On failure:**
- Non-fast-forward → "Run `! git pull --rebase origin {CURRENT_BRANCH}` first, then re-run `/deploy`." **Stop.**
- HTTP 400 / disconnect → retry with `--no-thin`. If still fails → tell user to push manually with `!` prefix. **Stop.**

---

## Phase 6 — Post-Deploy

1. Run `gh run list --repo {REPO_OWNER}/{REPO_NAME} --limit 1 --json status,conclusion,headBranch,createdAt,url`. Show run status.

2. Compute live URL: `https://{REPO_OWNER}.github.io/{REPO_NAME}/`

3. Display summary:
```
Deployment triggered.

Live URL: https://{REPO_OWNER}.github.io/{REPO_NAME}/
Monitor:  gh run watch --repo {REPO_OWNER}/{REPO_NAME}
Actions:  https://github.com/{REPO_OWNER}/{REPO_NAME}/actions

Typical time: 2–4 minutes.
```

If first deployment (repo just created or Pages just enabled): add "First deployment — Pages may need 1–2 extra minutes to activate."
