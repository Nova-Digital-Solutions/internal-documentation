---
name: github-workflow
description: "Nova Digital Solutions GitHub workflow automation. Use this skill whenever the developer is working on a task, fixing a bug, building a feature, or doing any development work. This skill manages GitHub issues, branches, PRs, and the project board so the developer can focus on coding. Trigger on: starting work on something new, finishing a task, creating a PR, closing an issue, setting up a repo, any mention of issues/PRs/branches/board, or when the developer says things like 'I need to work on X', 'this is done', 'let me start on', 'ship this', 'push this', 'ready for review'. Also trigger when the developer asks about project status, what's on the board, or what to work on next."
---

# Nova GitHub Workflow

You are managing the GitHub workflow for Nova Digital Solutions, a software development agency. Your job is to handle all the GitHub overhead — issues, branches, PRs, project board — so the developer can focus on writing code.

## Context

Nova uses GitHub for everything. The org is `nova-digital-solutions`. There is one shared GitHub Projects board called **Nova Work Board** with columns: Backlog → Ready → In Progress → In Review → Done.

Every piece of billable work must be traceable to a GitHub issue. When issues are closed, the **Hours** field on the project board must be filled in with actual hours spent. This is how Nova bills clients — there is no separate spreadsheet, the board is the source of truth.

## Determining the Current Repo

Before running any `gh` commands, determine the repo from the local git config:

```bash
git remote get-url origin
# Returns something like: git@github.com:nova-digital-solutions/acme-web-app.git
# Extract: nova-digital-solutions/acme-web-app
```

Cache this for the session so you don't re-query it on every command.

## Core Principles

- **Every task needs an issue.** If the developer starts working on something and there's no issue, create one.
- **Issues go on the board.** Every issue must be added to the Nova Work Board.
- **Hours are mandatory.** When closing an issue, always ask the developer how many hours they spent and update the Hours field.
- **Keep it simple.** Don't over-engineer branch names or PR descriptions. Clear and concise wins.

---

## Issue Management

### Required fields

Every issue must have these fields populated:

| Field | When | How |
|-------|------|-----|
| Title | On creation | Set via `gh issue create --title` |
| Description | On creation | Set via `gh issue create --body` |
| Assignee | On creation | Set via `--assignee "@me"` |
| Type | On creation | Set the Type field on the project: `Epic`, `Task`, `Bug`, or `Feature` |
| Priority | On creation | Set the Priority field on the project |
| Size | On creation | Set the Size field on the project |
| Project | On creation | Set via `--project "Nova Work Board"` |
| Hours | **On close** | Set the Hours field on the project (see "Closing an Issue" section) |

### Creating an issue

```bash
gh issue create \
  --repo REPO \
  --title "CLEAR ACTION-ORIENTED TITLE" \
  --body "DESCRIPTION: what needs to happen and why" \
  --assignee "@me" \
  --project "Nova Work Board"
```

After creating the issue, update the project fields (Type, Priority, Size) using the GraphQL API. See "Updating Project Fields" below.

Infer the Type from context:
- **Epic** — large body of work spanning multiple tasks
- **Feature** — new functionality
- **Bug** — something broken
- **Task** — maintenance, config, research, or anything that isn't a feature or bug

Infer Priority from the developer's language. If they say "urgent," "ASAP," "blocking," or "client is asking" → high priority. Default to medium if unclear.

### Labels

We only use one label: `blocked`. Add it when the developer says they're stuck, and leave a comment on the issue explaining why. Remove it when the blocker is resolved.

```bash
gh issue edit ISSUE_NUMBER --repo REPO --add-label "blocked"
gh issue comment ISSUE_NUMBER --repo REPO --body "Blocked: waiting on API credentials from client"
```

---

## Updating Project Fields

Project fields (Type, Priority, Size, Hours, Status) are managed via the GitHub GraphQL API. You need three IDs: the project ID, the item ID (issue's entry on the board), and the field ID.

### Step 1: Get project ID and field IDs (cache these — they don't change)

```bash
# Get project ID
gh api graphql -f query='
  query {
    organization(login: "nova-digital-solutions") {
      projectsV2(first: 10) {
        nodes { id title }
      }
    }
  }'

# Get all field IDs for the project
gh api graphql -f query='
  query {
    node(id: "PROJECT_ID") {
      ... on ProjectV2 {
        fields(first: 30) {
          nodes {
            ... on ProjectV2Field { id name }
            ... on ProjectV2SingleSelectField { id name options { id name } }
            ... on ProjectV2IterationField { id name }
          }
        }
      }
    }
  }'
```

This returns field IDs and, for single-select fields (Type, Priority, Status), the option IDs for each value (e.g., the ID for "Bug", the ID for "In Progress", etc.). Cache all of these.

### Step 2: Get the item ID for a specific issue

```bash
# Find the item ID for an issue on the project board
gh api graphql -f query='
  query {
    node(id: "PROJECT_ID") {
      ... on ProjectV2 {
        items(first: 100) {
          nodes {
            id
            content {
              ... on Issue { number }
            }
          }
        }
      }
    }
  }'
```

Match by issue number to get the item ID.

### Step 3: Update a field value

**For number fields (Hours, Size):**
```bash
gh api graphql -f query='
  mutation {
    updateProjectV2ItemFieldValue(
      input: {
        projectId: "PROJECT_ID"
        itemId: "ITEM_ID"
        fieldId: "FIELD_ID"
        value: { number: VALUE }
      }
    ) { projectV2Item { id } }
  }'
```

**For single-select fields (Type, Priority, Status):**
```bash
gh api graphql -f query='
  mutation {
    updateProjectV2ItemFieldValue(
      input: {
        projectId: "PROJECT_ID"
        itemId: "ITEM_ID"
        fieldId: "FIELD_ID"
        value: { singleSelectOptionId: "OPTION_ID" }
      }
    ) { projectV2Item { id } }
  }'
```

Use the option IDs from Step 1 (e.g., the ID for "Bug" in the Type field, or "In Progress" in the Status field).

---

## When the Developer Starts Working on Something

1. **Check if an issue exists.** Search for a related open issue:
   ```bash
   gh issue list --repo REPO --search "KEYWORDS" --state open
   ```

2. **If no issue exists, create one** with all required fields (see "Creating an Issue" above). Set the Type, Priority, and Size fields on the project.

3. **Create a branch off `dev`:**
   ```bash
   git checkout dev
   git pull origin dev
   git checkout -b SHORT-DESCRIPTIVE-NAME
   ```
   Names should be short and descriptive: `password-reset`, `fix-login-timeout`, `update-clerk-sdk`. No prefixes or issue numbers needed.

4. **Move the issue to In Progress** by updating the Status field on the project (see "Updating Project Fields").

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

3. **Move the issue to In Review** on the board (update Status field).

4. **Routine work** — the developer can self-merge into `dev`:
   ```bash
   gh pr merge --squash
   ```

5. **Needs review** — request it:
   ```bash
   gh pr edit --add-reviewer TEAMMATE
   ```

---

## When Merging to Production

When the developer wants to push `dev` to production:

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
   > "Issue #42 is closing. How many hours did you spend on this? I need to update the Hours field on the board."

2. **Update the Hours field** on the project board using the GraphQL mutation for number fields (see "Updating Project Fields" Step 3).

3. **Move the issue to Done** by updating the Status field on the board.

Do not skip this. Hours are how Nova bills clients. No hours = unbilled work.

---

## Handling Conflicts

If the developer's branch has conflicts with `dev`:

1. Fetch the latest `dev`:
   ```bash
   git fetch origin
   ```

2. Rebase onto `dev` (preferred — cleaner history):
   ```bash
   git rebase origin/dev
   # resolve conflicts
   git rebase --continue
   git push --force-with-lease
   ```

3. Or merge `dev` into the branch (simpler if rebase feels risky):
   ```bash
   git merge origin/dev
   # resolve conflicts
   git push
   ```

If the conflict involves code written by someone else, tell the developer to check with that teammate before resolving.

---

## Setting Up a New Repo

When the team needs a new repo:

```bash
# Create the repo
gh repo create nova-digital-solutions/CLIENT-PROJECT \
  --private \
  --clone \
  --gitignore Node \
  --description "DESCRIPTION"

cd CLIENT-PROJECT

# Create dev branch
git checkout -b dev
git push -u origin dev

# Share with the Nova team (everyone gets write access)
gh api orgs/nova-digital-solutions/teams/nova/repos/nova-digital-solutions/CLIENT-PROJECT \
  --method PUT \
  --field permission=push

# Set up labels — remove defaults, add only what we need
for label in "bug" "documentation" "duplicate" "enhancement" "good first issue" "help wanted" "invalid" "question" "wontfix"; do
  gh label delete "$label" --repo nova-digital-solutions/CLIENT-PROJECT --yes 2>/dev/null
done
gh label create "blocked" --repo nova-digital-solutions/CLIENT-PROJECT --color "F9D0C4" --description "Cannot proceed — see comments" --force

# Configure repo settings
gh api repos/nova-digital-solutions/CLIENT-PROJECT --method PATCH \
  --field has_wiki=false \
  --field has_issues=true \
  --field has_projects=true \
  --field has_discussions=false \
  --field delete_branch_on_merge=true \
  --field allow_squash_merge=true \
  --field allow_merge_commit=false \
  --field allow_rebase_merge=true \
  --silent

# Branch protection: main (requires PR + 1 review)
gh api repos/nova-digital-solutions/CLIENT-PROJECT/branches/main/protection --method PUT --input - <<'EOF'
{
  "required_status_checks": { "strict": true, "contexts": [] },
  "enforce_admins": false,
  "required_pull_request_reviews": {
    "required_approving_review_count": 1,
    "dismiss_stale_reviews": true
  },
  "restrictions": null,
  "required_conversation_resolution": true
}
EOF

# Branch protection: dev (requires PR, no review needed, self-merge OK)
gh api repos/nova-digital-solutions/CLIENT-PROJECT/branches/dev/protection --method PUT --input - <<'EOF'
{
  "required_status_checks": null,
  "enforce_admins": false,
  "required_pull_request_reviews": {
    "required_approving_review_count": 0,
    "dismiss_stale_reviews": false
  },
  "restrictions": null
}
EOF
```

Then remind the developer to:
- Add the repo to the Nova Work Board (Projects → Auto-add)
- Link to Vercel (production from `main`, staging from `dev`)
- Enable CodeRabbit
- Subscribe the Slack channel: `/github subscribe nova-digital-solutions/CLIENT-PROJECT`

---

## Checking Project Status

When the developer asks what's on the board, what to work on, or project status:

```bash
# Issues assigned to me
gh issue list --repo REPO --assignee "@me" --state open

# All open PRs
gh pr list --repo REPO

# PRs waiting for my review
gh pr list --repo REPO --search "review-requested:@me"
```

Present results in a clean summary. Suggest what to work on next based on priorities (high priority first, then whatever's in Ready).

---

## Reference

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
