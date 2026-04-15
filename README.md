# Internal Documentation

Nova Digital Solutions internal knowledge base — company docs, AI skills, templates, and automation scripts.

## Structure

| Folder | What goes here |
|---|---|
| `docs/` | All written knowledge: processes, guides, policies, SOPs. See [`docs/INDEX.md`](docs/INDEX.md) for the full list. |
| `skills/` | AI agent skills — reusable SKILL.md files for Claude, Cursor, and other AI coding assistants |
| `scripts/` | Automation scripts and tooling (e.g., Ralph autonomous agent, build-plan-generator) |
| `templates/` | Reusable templates for documents, checklists, and forms |

## Conventions

- **File naming:** Use kebab-case (e.g., `client-onboarding.md`)
- **Format:** Markdown (`.md`) for all documentation
- **Images/assets:** Place in an `_assets/` subfolder within the relevant section if needed
- **Ownership:** Add a frontmatter block at the top of each document with owner and last-reviewed date

### Document frontmatter

```yaml
---
title: Document Title
area: engineering | operations | company
type: guide | process | policy
onboarding: false
owner: Name or Team
created: YYYY-MM-DD
last-reviewed: YYYY-MM-DD
status: draft | active | deprecated
---
```

## Contributing

1. Create a branch for your changes
2. Add or update documentation following the conventions above
3. Open a PR for review
