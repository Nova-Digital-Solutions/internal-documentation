# Documentation Index

All company documentation lives here as flat Markdown files. Use frontmatter to categorize; use this index to navigate.

## By Topic

### Engineering
- [GitHub Workflow](github-workflow.md) — Git branching, PRs, project board, and repo setup

### Operations
_No docs yet._

### Company
_No docs yet._

## New Here?

If you just joined, read these docs in order:

1. [GitHub Workflow](github-workflow.md) — how we use Git, GitHub issues, and the project board

More onboarding docs will appear here as we write them. Any doc with `onboarding: true` in its frontmatter is part of this path.

## Adding a New Doc

1. Create a `.md` file in this folder with a kebab-case name (e.g. `dev-environment-setup.md`).
2. Add frontmatter at the top:
   ```yaml
   ---
   title: Dev Environment Setup
   area: engineering | operations | company
   type: guide | process | policy
   onboarding: false
   owner: Your Name
   created: YYYY-MM-DD
   last-reviewed: YYYY-MM-DD
   status: draft | active | deprecated
   ---
   ```
3. Add an entry to this INDEX under the right topic heading.
4. If the doc is part of the new-hire onboarding path, set `onboarding: true` and `onboarding-order: N`, then add it to the **New Here?** list above.
