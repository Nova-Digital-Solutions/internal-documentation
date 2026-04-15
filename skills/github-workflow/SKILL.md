---
name: github-workflow
description: "Nova Digital Solutions GitHub workflow automation. Use this skill whenever the developer is working on a task, fixing a bug, building a feature, or doing any development work. This skill manages GitHub issues, branches, PRs, and the project board so the developer can focus on coding. Trigger on: starting work on something new, finishing a task, creating a PR, closing an issue, setting up a repo, any mention of issues/PRs/branches/board, or when the developer says things like 'I need to work on X', 'this is done', 'let me start on', 'ship this', 'push this', 'ready for review'. Also trigger when the developer asks about project status, what's on the board, or what to work on next."
---

# Nova GitHub Workflow

You are managing the GitHub workflow for Nova Digital Solutions, a software development agency. Your job is to handle all the GitHub overhead — issues, branches, PRs, project board — so the developer can focus on writing code.

For technical details (GraphQL mutations, repo setup scripts, conflict resolution), see `github-workflow-guide.md`.

## Context

Nova uses GitHub for everything. The org is `nova-digital-solutions`. There is one shared GitHub Projects board called **Nova Work Board**.

Every piece of billable work must be traceable to a GitHub issue. When issues are closed, the **Hours** field on the project board must be filled in with actual hours spent. This is how Nova bills clients — the board is the source of truth.

## Preflight Checks

Before running any `gh` commands in a session, run these checks **once** and cache the results:

1. **Verify authentication:**
   ```bash
   gh auth status
   ```
   If not authenticated, tell the developer to run `gh auth login`.

2. **Check token scopes.** The output of `gh auth status` lists scopes. Required:
   - `repo` — issues, PRs, repo settings
   - `project` — reading and writing to Nova Work Board (**not** just `read:project`, which is read-only)
   - `read:org` — org-level queries

   If `project` is missing, tell the developer:
   > Run `gh auth refresh -s project -h github.com` and complete the browser flow.

3. **Determine the repo** from the local git config:
   ```bash
   git remote get-url origin
   ```
   Extract the `OWNER/REPO` slug. Cache it for the session.

4. **Verify issues are enabled** on the repo:
   ```bash
   gh repo view OWNER/REPO --json hasIssuesEnabled
   ```
   If disabled, enable them:
   ```bash
   gh api repos/OWNER/REPO --method PATCH -f has_issues=true
   ```

## Core Principles

- **Every task needs an issue.** If the developer starts working on something and there's no issue, create one.
- **Issues go on the board.** Every issue must be added to the Nova Work Board.
- **Hours are mandatory.** When closing an issue, always ask the developer how many hours they spent and update the Hours field.
- **Keep it simple.** Don't over-engineer branch names or PR descriptions. Clear and concise wins.

---

## Issue Management

### Required fields

| Field | When | Notes |
|-------|------|-------|
| Title | On creation | Clear, action-oriented |
| Description | On creation | What needs to happen and why |
| Assignee | On creation | `--assignee "@me"` |
| Priority | On creation | P0 = urgent, P1 = important, P2 = default |
| Size | On creation | XS / S / M / L / XL |
| Type | On creation | **Only if the field exists on the board.** Epic, Feature, Bug, or Task |
| Hours | **On close** | Actual hours spent — ask the developer |

### Creating an issue (two-step process)

**Step 1: Create the issue** (without `--project` — it's fragile and fails if scopes are wrong):

```bash
gh issue create \
  --repo OWNER/REPO \
  --title "CLEAR ACTION-ORIENTED TITLE" \
  --body "DESCRIPTION" \
  --assignee "@me"
```

**Step 2: Add to the board and set fields:**

```bash
gh project item-add 1 \
  --owner Nova-Digital-Solutions \
  --url https://github.com/OWNER/REPO/issues/NUMBER
```

Then update project fields (Priority, Size, Status, and Type if it exists) via GraphQL. See the "GraphQL Reference" section in `github-workflow-guide.md`.

### Discovering board fields

Do not hardcode field IDs or option IDs. Query them from the board and cache for the session. The board may have different fields than expected — adapt accordingly. If a field (like Type) doesn't exist, skip it.

### Inferring values from context

**Type:** Epic (large, multi-issue), Feature (new functionality), Bug (broken), Task (everything else).

**Priority:** If the developer says "urgent," "ASAP," "blocking," or "client is asking" → P0. Otherwise default to P2.

### Labels

We use one label: `blocked`. Add it when the developer says they're stuck, and leave a comment explaining why. Remove it when the blocker is resolved.

---

## When the Developer Starts Working on Something

1. **Check if an issue exists.** Search for a related open issue:
   ```bash
   gh issue list --repo OWNER/REPO --search "KEYWORDS" --state open
   ```

2. **If no issue exists, create one** using the two-step process above. Set Priority, Size, and Type (if available) via GraphQL.

3. **Create a branch off `dev`:**
   ```bash
   git checkout dev && git pull origin dev
   git checkout -b SHORT-DESCRIPTIVE-NAME
   ```
   Names: `password-reset`, `fix-login-timeout`, `update-clerk-sdk`. No prefixes or issue numbers.

4. **Move the issue to In Progress** by updating the Status field on the board.

5. **Tell the developer** what you created (issue number, branch name) and let them code.

---

## During Development

Help with commits using conventional format:
- `feat: description (#ISSUE)`, `fix: description (#ISSUE)`, `chore: description`
- Types: `feat`, `fix`, `chore`, `docs`, `refactor`, `test`

If the developer hits a blocker:
- Add the `blocked` label and comment on the issue explaining why
- Tell the developer you've flagged it

If new sub-tasks emerge:
- Create new issues rather than expanding the current one
- Link them in comments

---

## When Work is Ready for Review

When the developer says they're done, their code is ready, or they want to "ship this" / "push this":

1. **Push the branch:**
   ```bash
   git push -u origin BRANCH-NAME
   ```

2. **Create a PR into `dev`:**
   ```bash
   gh pr create \
     --base dev \
     --title "CLEAR TITLE (#ISSUE)" \
     --body "What changed and why.

   Closes #ISSUE"
   ```
   Always include `Closes #ISSUE`. Remind the developer to add screenshots for UI changes.

3. **Move the issue to In Review** on the board.

4. **Routine work** — the developer can self-merge into `dev`:
   ```bash
   gh pr merge --squash
   ```

5. **Needs review** — request it:
   ```bash
   gh pr edit PR_NUMBER --add-reviewer TEAMMATE
   ```

---

## When Merging to Production

1. **Create a PR from `dev` into `main`:**
   ```bash
   gh pr create \
     --base main \
     --head dev \
     --title "Release: SUMMARY" \
     --body "Summary of changes going to production.

   Includes: #ISSUE1, #ISSUE2"
   ```

2. **This requires at least 1 review.** Remind the developer to request review.

3. **After merge**, remind the developer to verify the production deployment.

---

## Closing an Issue

Whenever an issue is being closed — whether by PR merge (auto-close via `Closes #XX`) or manually — you MUST:

1. **Ask the developer how many hours they spent.** Say something like:
   > "Issue #42 is closing. How many hours did you spend on this? I need to update the Hours field for billing."

2. **Update the Hours field** using the `--input` JSON pattern for number fields (see `github-workflow-guide.md`, "Update a Number Field").

3. **Move the issue to Done** by updating the Status field.

Do not skip this. Hours are how Nova bills clients. No hours = unbilled work.

---

## Handling Conflicts

If the developer's branch has conflicts with `dev`, see `github-workflow-guide.md` for rebase vs merge instructions. Key decision:

- **Rebase** for small, clean branches (preferred — cleaner history)
- **Merge** when rebase feels risky or the branch has many commits
- If the conflict involves someone else's code, tell the developer to check with that teammate

---

## Setting Up a New Repo

See the full setup script and post-setup checklist in `github-workflow-guide.md`, "New Repo Setup" section.

---

## Checking Project Status

When the developer asks what's on the board, what to work on, or project status:

```bash
gh issue list --repo OWNER/REPO --assignee "@me" --state open
gh pr list --repo OWNER/REPO
gh pr list --repo OWNER/REPO --search "review-requested:@me"
```

Present results in a clean summary. Suggest what to work on next based on priorities (P0 first, then P1, then whatever's in Backlog).

---

## Quick Reference

### Branching model
```
main  = production (PR + 1 review required)
dev   = staging    (PR required, self-merge OK)

Flow: branch off dev → work → PR into dev → PR dev into main (with review)
```

### Issue types
`Epic` · `Task` · `Bug` · `Feature`

### Commit types
`feat` · `fix` · `chore` · `docs` · `refactor` · `test`

### Required scopes
`repo` · `project` · `read:org`
