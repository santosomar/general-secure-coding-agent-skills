---
name: release-notes-writer
description: Produces structured, human-readable release notes from diffs, commit messages, and issue tracker data. Use when preparing a release, when the user asks for release notes, or when summarizing changes for stakeholders.
license: Apache-2.0
metadata:
  category: "devops"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
---

# Release Notes Writer

## Purpose

Generate release notes from VCS diffs, commit messages, and issue tracker data.

## Workflow

1. **Gather inputs**: Git log since last tag, diff stats, linked issues/PRs.
2. **Categorize**: Group by feature, fix, chore, breaking change.
3. **Summarize**: Write concise, user-facing descriptions per change.
4. **Format**: Output in standard format (e.g., Keep a Changelog, conventional).

## Output

- Structured release notes (Markdown)
- Sections: Added, Changed, Fixed, Removed, Security
- Links to commits/PRs where applicable
