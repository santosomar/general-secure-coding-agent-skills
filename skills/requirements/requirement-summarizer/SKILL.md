---
name: requirement-summarizer
description: Produces a structured summary of a requirements document — the key obligations, grouped by actor and concern, with the MUST/SHOULD/MAY breakdown. Use when onboarding to a large spec, when deciding what to implement first, or when the user asks what a 200-page standard actually requires.
license: Apache-2.0
metadata:
  category: "requirements"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "requirement-comparison-reporter, requirement-enhancer"
---

# Requirement Summarizer

A spec is hundreds of pages. The engineering-relevant content is the **normative statements** — the MUSTs, SHOULDs, and MUST NOTs. Everything else is context. Summarizing means extracting and organizing the normative core.

## Normative vs informative

| Text type                    | Normative? | Include in summary?              |
| ---------------------------- | ---------- | -------------------------------- |
| "The server MUST ..."        | Yes        | Yes — this is a requirement      |
| "The server SHOULD ..."      | Yes        | Yes — recommended, note as SHOULD |
| "The server MAY ..."         | Yes        | Brief mention — optional feature |
| "For example, ..."           | No         | Skip (unless the spec says examples are normative) |
| "Historically, ..."          | No         | Skip — background                |
| "Implementations typically ..." | No      | Skip — observation, not obligation |
| Definitions                  | Semi       | Include the ones referenced by MUSTs |

**Signal words:** RFC 2119 keywords (MUST, SHALL, REQUIRED, SHOULD, MAY, OPTIONAL) are the gold standard. Specs that don't use them — look for imperative mood ("do X"), "is required", "is mandatory".

## Organizing the summary

Don't preserve the spec's section order — reorganize by **who does what**:

| Axis                 | Grouping                                                   |
| -------------------- | ---------------------------------------------------------- |
| By actor             | Client requirements / server requirements / intermediary requirements |
| By concern           | Auth / data format / error handling / performance          |
| By obligation level  | MUST (blocking) / SHOULD (review) / MAY (optional)         |
| By lifecycle phase   | Connection setup / steady state / teardown / error         |

Pick the axis that makes the spec easiest to act on. For a protocol spec: by actor then by lifecycle. For a compliance standard: by obligation level (MUSTs first — those are the gates).

## Worked example — summarizing an OAuth 2.0 subset

**Source:** RFC 6749 §4.1 (Authorization Code Grant), ~15 pages.

**Summary:**

```
## Scope
Authorization Code Grant flow only (§4.1). Other grant types not covered.

## Actors
- Client (the app requesting access)
- Authorization Server (issues codes and tokens)
- Resource Owner (the user)

## Client MUST
| Ref     | Requirement                                                    |
| ------- | -------------------------------------------------------------- |
| §4.1.1  | Direct the user to the authorization endpoint with `response_type=code`, `client_id`, `redirect_uri` |
| §4.1.1  | Include `state` parameter (CSRF protection)                    |
| §4.1.3  | Exchange the code at the token endpoint via POST with `grant_type=authorization_code`, `code`, `redirect_uri`, client credentials |
| §4.1.3  | Send `redirect_uri` identical to the one used in §4.1.1        |
| §10.12  | Verify the `state` parameter on the callback matches what was sent |

## Client MUST NOT
| §10.6   | Include credentials in the redirect URI                        |

## Authorization Server MUST
| §4.1.2  | Redirect to `redirect_uri` with `code` and `state` on success  |
| §4.1.2  | Redirect with `error` parameter on denial                      |
| §4.1.2.1| `code` MUST be short-lived (RECOMMENDED: ≤ 10 min) and single-use |
| §4.1.3  | Verify `redirect_uri` matches the one from §4.1.1              |
| §4.1.3  | Authenticate the client (if confidential)                      |

## SHOULD (not summarized in detail — 6 items, see §10)

## Key definitions
- Authorization code: short-lived credential exchanged for a token (§1.3.1)
- Confidential client: can keep `client_secret` secret — server-side apps (§2.1)
```

**What got dropped:** 12 pages of motivation, history, and examples. What remains: 10 MUSTs across 2 actors, with section references for drill-down.

## Density targets

| Source length | Summary length | Compression |
| ------------- | -------------- | ----------- |
| 20 pages      | 1 page         | 20:1        |
| 200 pages     | 5–10 pages     | 20–40:1     |
| 2000 pages    | You're not summarizing, you're indexing | — |

If the summary is > 10% of the source, you're including informative material. Cut harder.

## Do not

- **Do not** paraphrase normative statements loosely. "MUST authenticate the client" stays verbatim or near-verbatim — loose paraphrase reintroduces ambiguity.
- **Do not** drop the section references. The summary is a map; the references let readers drill into the territory.
- **Do not** flatten MUST and SHOULD into one list. MUST blocks shipping; SHOULD is a judgment call. Different audiences.
- **Do not** summarize a spec you haven't read end-to-end. Late sections modify early ones ("except as noted in §9"). Read it all, then summarize.

## Output format

```
## Source
<document, version, date, sections covered>

## Scope
<what this summary covers / excludes>

## Actors
<who the requirements apply to>

## MUST — by actor
### <Actor 1>
| Ref | Requirement |
| --- | ----------- |

### <Actor 2>
...

## MUST NOT
<same structure>

## SHOULD — by actor
<brief — count + highlights>

## MAY
<one line — "N optional features: X, Y, Z">

## Key definitions
<only terms referenced by MUSTs above>

## Compression
<source pages → summary pages — sanity check on density>
```
