---
name: scenario-generator
description: Generates concrete scenarios from a requirement — happy paths, edge cases, and error conditions — expressed as Given/When/Then or equivalent structured narratives. Use when turning a requirement into acceptance tests, when exploring what could go wrong, or when the requirement is abstract and needs grounding.
license: Apache-2.0
metadata:
  category: "requirements"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "req-to-test"
---

# Scenario Generator

A requirement says "users can reset their password." Scenarios say: *this* user, in *this* state, does *this*, and *this* happens. Scenarios are requirements made concrete enough to execute.

## Scenario structure — Given/When/Then

```
Given  <initial state / preconditions>
When   <action the actor takes>
Then   <observable outcome>
```

Every scenario is one path through the system. A requirement usually generates 3–10 scenarios: one happy path, several edges, several errors.

## Systematic scenario enumeration

For each requirement, walk the dimensions:

| Dimension                 | Questions                                                   |
| ------------------------- | ----------------------------------------------------------- |
| **Actor variants**        | Logged in? Anonymous? Admin? Suspended?                     |
| **Input boundaries**      | Empty? Max length? Just over max? Unicode? Null?            |
| **State variants**        | First time? Repeat? Concurrent? After a failure?            |
| **Timing**                | Immediately? After timeout? During maintenance?             |
| **Failure injection**     | DB down? Third-party 500? Network partition mid-flow?       |
| **Sequence**              | Out of order? Duplicate? Replayed?                          |

Cross the dimensions that matter. Not all N×M — most combinations are uninteresting. Pick the ones where the outcome *differs*.

## Worked example

**Requirement:** "A user can reset their password via email link. The link expires after 1 hour."

**Happy path:**

```
Scenario: Successful password reset
  Given a user with email alice@example.com and a known password
  When  Alice requests a password reset for alice@example.com
  Then  a reset email is sent to alice@example.com
  And   the email contains a link valid for 1 hour
  When  Alice clicks the link within 1 hour
  And   submits a new password meeting the policy
  Then  Alice can log in with the new password
  And   cannot log in with the old password
```

**Edge — timing boundary:**

```
Scenario: Link used at exactly T+60min
  Given a reset link issued at time T
  When  the link is used at exactly T + 60 minutes
  Then  <ASK: is the boundary inclusive or exclusive? — stakeholder decision>
```

**Error — expired:**

```
Scenario: Expired link rejected
  Given a reset link issued at time T
  When  the link is used at T + 61 minutes
  Then  an "expired link" error is shown
  And   no password change occurs
  And   the user can request a new link
```

**Error — wrong actor:**

```
Scenario: Reset requested for nonexistent email
  Given no user with email eve@nowhere.com exists
  When  a reset is requested for eve@nowhere.com
  Then  the response is identical to the success case (no enumeration leak)
  And   no email is sent
```

**Sequence — reuse:**

```
Scenario: Link is single-use
  Given a reset link that has already been used successfully
  When  the same link is used again
  Then  an "invalid link" error is shown
  And   no password change occurs
```

**Concurrency:**

```
Scenario: Two reset requests in flight
  Given a user requests reset, receiving link L1
  And   requests reset again, receiving link L2
  When  L1 is used
  Then  <ASK: does L2 stay valid? does issuing L2 invalidate L1? — stakeholder decision>
```

**Found by this exercise:** two `<ASK>` placeholders — boundary inclusivity and concurrent-link policy. Neither was in the original requirement. Scenarios surface gaps.

## Prioritizing scenarios

Not all scenarios are worth automating:

| Priority | Scenarios                                              | Automate?          |
| -------- | ------------------------------------------------------ | ------------------ |
| P0       | Happy path, security-relevant errors (enumeration, expiry) | Yes — blocking   |
| P1       | Boundaries (exact expiry moment, max password length)  | Yes                |
| P2       | Unlikely-but-bad (concurrent links, partial failures)  | Yes if cheap; manual test otherwise |
| P3       | Cosmetic (error message wording)                       | Spot-check         |

## Do not

- **Do not** generate only happy paths. The happy path is one scenario. The interesting scenarios are where things vary.
- **Do not** leave `<ASK>` items unresolved. They're requirements gaps — escalate to the stakeholder before the scenario becomes a test with a guessed answer.
- **Do not** conflate scenarios with test implementation. `Given a user with email alice@example.com` is a scenario. `INSERT INTO users VALUES ...` is a test. Keep scenarios implementation-agnostic.
- **Do not** generate every combination. 5 dimensions × 4 values each = 1024 scenarios. Pick the ones where the outcome differs. The rest are redundant.

## Output format

```
## Requirement
<verbatim>

## Dimensions explored
<actor variants, input boundaries, state variants, timing, failures, sequence>

## Scenarios

### Happy path
<Given/When/Then>

### Edge: <name>
<Given/When/Then>
...

### Error: <name>
<Given/When/Then>
...

## Gaps surfaced
<ASK items — requirement decisions the scenarios need that the requirement doesn't provide>

## Coverage note
<which dimension combinations were skipped and why they're uninteresting>
```
