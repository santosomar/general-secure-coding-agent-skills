---
name: release-notes-writer
description: Transforms a changelog or commit range into user-friendly release notes with highlights, upgrade guidance, and narrative framing. Use when publishing a release announcement, when the changelog is too dense for users to read, or when the user needs a blog-post-shaped summary of a version.
license: Apache-2.0
metadata:
  category: "devops"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "change-log-generator"
---

# Release Notes Writer

Release notes are **marketing for engineering.** The changelog lists everything; release notes tell a story about the 2–3 things that matter.

## Input

Preferably the output of → `change-log-generator`. If not available, the raw commit range — but you'll need to do the classification yourself first.

## The shape

```
<One-sentence headline — what is this release about?>

<One paragraph of context — why these changes, what problem do they solve>

## Highlights
  <2–4 items, each with a before/after>

## Upgrade notes
  <Only if there's something to do. "No breaking changes" if clean.>

## Full changelog
  <link to CHANGELOG.md — don't duplicate it>
```

## Step 1 — Find the headline

What is this release *about*? Usually one of:

| Release type        | Headline pattern                                            |
| ------------------- | ----------------------------------------------------------- |
| One big feature     | "`<product> <ver>` introduces `<feature>`"                  |
| Many small features | "`<product> <ver>`: `<theme connecting the features>`"      |
| Mostly fixes        | "`<product> <ver>`: stability and fixes" — short notes, no highlights section |
| Breaking/major      | "`<product> <ver>`: `<what changed>` — upgrade guide inside" |
| Security patch      | "`<product> <ver>`: security update" — lead with this, nothing else matters |

If you can't find a headline, the release doesn't have a story. That's fine — write two sentences and link the changelog.

## Step 2 — Pick highlights (ruthlessly)

From a 20-item changelog, pick **2–4**. Criteria:

- Will the average user notice this? (Most fixes: no.)
- Does it unblock something users have been asking for? (Check issue tracker reactions.)
- Does it change how they should write their code?

Everything else is "…and many other improvements — see the full changelog."

## Step 3 — Write before/after for each highlight

A highlight with no before/after is an assertion. A highlight with before/after is a demonstration.

**Bad:** "Added timeout support to the Client."

**Good:**

> **Request timeouts.** Previously, a hung server would hang your client forever. Now:
>
> ```python
> client = Client(timeout=5.0)   # raises TimeoutError after 5s
> ```

## Upgrade notes

Only write this section if there's something to *do*:

- Breaking changes → exact migration steps, old→new code snippets
- New minimum dependency version → say which and why
- Config key renamed → old key, new key, whether the old one still works
- Nothing → write "No breaking changes. Drop-in upgrade from `<prev>`." so readers know the absence is intentional.

## Worked example

**Changelog input (from → `change-log-generator`):**

```markdown
## [2.4.0]
### Breaking
- Dropped Python 3.8 support (#94)
### Added
- Client() accepts timeout parameter (#92)
- New retry_on parameter for selective retry (#97)
### Fixed
- Connection pool leak under concurrent close (#891)
- 7 other fixes
```

**Release notes output:**

```markdown
# httpx-thing 2.4.0 — Timeouts and selective retry

2.4.0 closes out the two most-requested reliability features: per-call
timeouts and the ability to retry only on the errors you care about.

## Highlights

**Request timeouts.** Previously, a hung server hung your client. Now:

    client = Client(timeout=5.0)

**Selective retry.** Retrying everything wastes time on unrecoverable
errors. `retry_on` lets you pick:

    client = Client(retries=3, retry_on=[ConnectionError, Timeout])

## Upgrade notes

**Python 3.8 is no longer supported.** Minimum is 3.9. If you're on
3.8, stay on httpx-thing 2.3.x until you can upgrade.

Otherwise: drop-in upgrade from 2.3.

## Full changelog

See [CHANGELOG.md](./CHANGELOG.md#240).
```

The pool-leak fix and 7 other fixes are omitted from highlights — they're in the changelog link.

## Do not

- **Do not** list every change. That's what the changelog is for.
- **Do not** lead with internal wins ("we refactored the core"). Users don't care about your architecture.
- **Do not** bury breaking changes below highlights. If it breaks, it's the first thing after the intro.
- **Do not** write "various bug fixes and improvements" as a highlight. That's a link to the changelog, not a sentence.
- **Do not** write upgrade notes that say "see the changelog for breaking changes." Inline the migration steps — that's the entire point of the section.

## Output format

Markdown, structured as above. Length: a patch release is 3 sentences; a major release with breaking changes might be 50 lines.
