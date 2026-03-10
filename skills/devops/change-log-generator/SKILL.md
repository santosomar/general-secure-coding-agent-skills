---
name: change-log-generator
description: Generates a structured CHANGELOG.md from VCS history and PR/issue references, categorized by change type. Use when cutting a release, when the user asks to update CHANGELOG.md, or when backfilling a changelog from git history.
license: Apache-2.0
metadata:
  category: "devops"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "release-notes-writer"
---

# Change Log Generator

A changelog is for **developers who use your project**. It answers one question: *what do I need to know before I upgrade?*

## Audience split — this skill vs. release notes

| This skill (CHANGELOG.md)            | → `release-notes-writer`                    |
| ------------------------------------ | ------------------------------------------- |
| Developers, maintainers              | End users, stakeholders                     |
| Terse, categorized, complete         | Narrative, highlights only                  |
| Every user-visible change            | The 3 things anyone actually cares about    |
| Lives in the repo                    | Goes in the release announcement            |

## Step 1 — Delimit the range

```
git log <prev-tag>..HEAD --oneline --no-merges
```

If there's no previous tag, the range is "since the last `## [x.y.z]` heading in CHANGELOG.md" — grep it out.

## Step 2 — Classify each commit

Bucket every commit. **One bucket per commit** — if a commit is both a feature and a fix, it should have been two commits, but pick the more user-impacting bucket.

| Bucket       | Commit message signals                               | What qualifies                                       |
| ------------ | ---------------------------------------------------- | ---------------------------------------------------- |
| **Breaking** | `BREAKING CHANGE:`, `!` after type, removed API      | Anything that breaks existing user code — always first, always loud |
| **Added**    | `feat:`, `add`, `new`, `implement`                   | New capabilities the user can now use                |
| **Changed**  | `refactor:` (if user-visible), behavior changes      | Existing features that work differently now          |
| **Fixed**    | `fix:`, `bug`, `patch`, `Fixes #N`                   | Things that were broken and now aren't               |
| **Security** | `security:`, CVE refs, `vuln`                        | Separate section — people grep for this              |
| **Removed**  | `remove`, `drop`, `delete` (of user-facing things)   | Deprecation follow-throughs                          |
| _(drop)_     | `chore:`, `ci:`, `docs:`, `test:`, `style:`          | Internal. Changelog readers don't care.              |

## Step 3 — Rewrite each entry

The commit message is **input**, not **output**. Transform:

| Commit says                          | Changelog says                                           |
| ------------------------------------ | -------------------------------------------------------- |
| `fix: handle null in getUserEmail`   | `getUserEmail()` no longer throws when user has no email |
| `feat: add retry param`              | `Client()` now accepts `retries=N` to retry failed calls |
| `remove v1 endpoints`                | **Breaking:** `/api/v1/*` endpoints removed; use `/api/v2/*` |

Rules for rewriting:
- Past tense or present — be consistent with the existing changelog.
- User-facing names, not internal names. `the billing module` not `BillingServiceImpl`.
- State the *effect*, not the *mechanism*. "No longer throws" not "added null check."
- Link the PR/issue: `(#123)` at the end.

## Worked example

**Input commits:**

```
a1b2c3d  feat(client): add timeout parameter
b2c3d4e  fix: race in connection pool cleanup (fixes #891)
c3d4e5f  chore: bump pytest to 8.0
d4e5f6a  feat!: drop Python 3.8 support
e5f6a7b  docs: fix typo in README
```

**Output:**

```markdown
## [2.4.0] — 2026-03-09

### Breaking
- Dropped Python 3.8 support; minimum is now 3.9 (#94)

### Added
- `Client()` now accepts a `timeout` parameter (seconds, default 30) (#92)

### Fixed
- Connection pool no longer leaks connections under concurrent close (#891)
```

Two commits dropped (`chore:`, `docs:`). Three kept, rewritten, linked.

## Edge cases

- **No conventional commits — messages are freeform:** Read the diffs. A commit touching `docs/` is probably `docs:`. A commit that adds a test and changes one line is probably `fix:`. Classify by content.
- **Merge commits with 50 sub-commits:** If merges are the unit (squash workflow), one entry per merge. If not, skip merge commits and use the underlying commits.
- **A revert of something from the same range:** Both the original and the revert should be dropped — net zero change.
- **Backfilling a changelog that was never kept:** Work tag-by-tag, oldest first. The git log between `v1.0` and `v1.1` is the `## [1.1.0]` section.

## Do not

- **Do not** include internal changes (CI, tests, refactors with no user-visible effect). The changelog is not a git log.
- **Do not** use commit messages verbatim. `fix: thing` is not an entry.
- **Do not** combine Breaking with anything else in a section. It gets its own heading even if there's one item.
- **Do not** generate an `## [Unreleased]` section and call it done. Either pin a version or be explicit that it's unreleased.
- **Do not** reorder existing changelog entries. Append only.

## Output format

```markdown
## [<version>] — <YYYY-MM-DD>

### Breaking
- <entry> (#<pr>)

### Added
- ...

### Changed
- ...

### Fixed
- ...

### Security
- ...
```

Omit empty sections. Breaking first, always.
