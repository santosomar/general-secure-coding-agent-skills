---
name: requirement-coverage-checker
description: Checks whether an implementation covers a set of requirements by tracing each requirement to code, tests, or both — and flagging gaps where a requirement has no evidence of implementation. Use when auditing for compliance, when answering "is this spec implemented", or before claiming a standard is supported.
license: Apache-2.0
metadata:
  category: "requirements"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "traceability-matrix-generator, req-to-test"
---

# Requirement Coverage Checker

Every requirement should trace to at least one of: **code that implements it**, or **a test that verifies it**. Preferably both. A requirement with neither is either unimplemented or undocumented — both are problems.

## Coverage levels

| Level              | Evidence                                         | Confidence                           |
| ------------------ | ------------------------------------------------ | ------------------------------------ |
| **Tested**         | A test asserts the required behavior             | High — behavior is checked           |
| **Implemented**    | Code exists that (claims to) satisfy it          | Medium — no automated check          |
| **Documented**     | A comment/doc references the requirement         | Low — intent, not evidence           |
| **Uncovered**      | No trace anywhere                                | Zero — gap                           |

## Step 1 — Decompose requirements into checkable claims

A requirement like "The API MUST validate input and return 400 on invalid data" is two claims:
1. Input is validated.
2. Invalid input → HTTP 400.

Trace each separately. The code might validate (1) but return 500 on failure (2 uncovered).

## Step 2 — Search for evidence

For each atomic claim, search the codebase in order:

| Search                                         | Evidence type    |
| ---------------------------------------------- | ---------------- |
| Test name/docstring mentions the requirement ID | Tested (explicit trace) |
| Test asserts the specific behavior             | Tested (implicit) |
| Code comment references the requirement        | Documented       |
| Code structure matches the requirement         | Implemented      |
| Nothing                                        | Uncovered        |

Use → `code-search-assistant` tactics: grep for requirement IDs, grep for key domain terms, structural search for the code pattern.

## Step 3 — Rate the trace strength

Not all traces are equal:

| Trace                                                         | Strength |
| ------------------------------------------------------------- | -------- |
| `def test_req_4_2_rejects_oversized_payload(): ... assert resp.status == 413` | Strong — ID + behavior |
| `def test_payload_limit(): ... assert resp.status == 413`     | Good — behavior matches |
| `# Implements REQ-4.2` above a function                       | Weak — claim, not check |
| Function named `validate_payload_size`                        | Weak — name suggests, body might differ |
| A constant `MAX_PAYLOAD = 10 * 1024 * 1024`                   | Circumstantial          |

## Worked example

**Requirements (subset):**
- REQ-3.1: Passwords MUST be hashed with bcrypt, cost ≥ 12.
- REQ-3.2: Failed login attempts MUST be rate-limited to 5 per minute per account.
- REQ-3.3: Sessions MUST expire after 30 minutes of inactivity.

**Search results:**

| Req    | Evidence found                                                  | Level       |
| ------ | --------------------------------------------------------------- | ----------- |
| REQ-3.1 | `auth/hash.py`: `bcrypt.hashpw(pw, bcrypt.gensalt(rounds=12))`. `test_hash.py::test_bcrypt_cost_12` asserts `rounds >= 12`. | **Tested** ✓ |
| REQ-3.2 | `auth/login.py`: `@ratelimit(key="account", rate="5/m")` decorator. No test exercises the 6th attempt. | **Implemented** — untested |
| REQ-3.3 | `settings.py`: `SESSION_COOKIE_AGE = 1800`. That's 30 min — but it's *absolute* expiry, not *inactivity* expiry. | **PARTIAL / WRONG** — implements a different requirement |

**Report:**
- REQ-3.1: Covered.
- REQ-3.2: Implemented but not tested. **Gap: write a test** that makes 6 login attempts and asserts the 6th is rejected.
- REQ-3.3: **Miscovered.** `SESSION_COOKIE_AGE` is time-since-creation, not time-since-last-activity. The requirement says "inactivity." Need `SESSION_SAVE_EVERY_REQUEST = True` or a sliding-window middleware. Flag as a compliance bug.

## The "looks covered but isn't" traps

| Trap                                    | How to catch                                           |
| --------------------------------------- | ------------------------------------------------------ |
| Right behavior, wrong condition         | Req says "on timeout"; code does it "on any error." Test the timeout case specifically. |
| Implemented but dead code               | The validating function exists but is never called. Check the call graph. |
| Configured but overridable              | `MAX_RETRIES = 3` — but an env var overrides it to 0 in prod. |
| Test exists but is skipped              | `@pytest.mark.skip` — it's there but not running.      |
| Test asserts implementation, not requirement | Test checks `bcrypt.hashpw` was called; doesn't check cost ≥ 12. |

## Do not

- **Do not** count a comment as coverage. `# TODO: implement REQ-4.2` traces to the requirement and means the opposite of covered.
- **Do not** stop at the first trace. A requirement with 3 clauses needs 3 traces. Finding 1 and marking it covered misses 2 gaps.
- **Do not** trust naming. `validate_input()` might not validate anything. Read the body.
- **Do not** report a binary covered/not. "Partially covered — clause 1 tested, clause 2 implemented, clause 3 gap" is the useful answer.

## Output format

```
## Requirements audited
<count> requirements, <count> atomic claims

## Coverage summary
| Level        | Count | % |
| ------------ | ----- | - |
| Tested       |       |   |
| Implemented  |       |   |
| Documented   |       |   |
| Uncovered    |       |   |
| Miscovered   |       |   |

## Per-requirement trace
| Req ID | Claim | Evidence (file:line) | Level | Notes |
| ------ | ----- | -------------------- | ----- | ----- |

## Gaps (actionable)
### Uncovered
<req → what's missing → suggested test or implementation>

### Miscovered
<req → what's there → why it doesn't satisfy the req → fix>

### Untested
<req → code exists → suggested test>
```
