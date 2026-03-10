---
name: code-summarizer
description: Produces natural-language summaries of what code does at the function, class, module, or subsystem level, with length and abstraction scaled to the scope. Explains purpose, side effects, and non-obvious behavior rather than restating syntax. Use when onboarding to unfamiliar code, when the user asks what something does, when generating docstrings or architecture notes, or when preparing a handoff document.
license: Apache-2.0
metadata:
  category: "code-analysis"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
  related: "legacy-code-summarizer, component-boundary-identifier"
---

# Code Summarizer

Say what the code is *for*, not what it *says*. "Iterates over the list and appends items where status is active" is a translation, not a summary. "Returns only the invoices that are eligible for payment" is a summary.

## Step 1 — Set the abstraction level from the scope

The answer to "what does this do" changes with how much code "this" covers. Decide up front:

| Scope         | Length        | Abstraction level                              | Goal                                        |
| ------------- | ------------- | ---------------------------------------------- | ------------------------------------------- |
| Single expr   | 1 phrase      | What value, in domain terms                    | Replace with a well-named variable          |
| Function      | 1–3 sentences | Contract: inputs → outputs → effects           | A reader can call it correctly without reading the body |
| Class         | 1 paragraph   | Responsibility + lifecycle + main collaborators| A reader knows when to use it and when not to |
| Module / file | 2–4 paragraphs| Subsystem role + what enters/leaves            | A reader can decide whether to open it      |
| Package / dir | Bullets       | Architectural role + public surface            | A reader can navigate to the right file     |

If the user doesn't specify scope, **infer from what they pasted**: 10 lines → function-level; 500 lines → module-level; a directory path → package-level.

## Step 2 — Read for signals, not linearly

You don't read a 300-line function top-to-bottom to summarize it. You scan for high-information locations:

1. **Name and signature.** If well-named, you're 60% done. If the name is `process` or `handle`, skip to step 2.
2. **Return statements.** What the function produces is usually what it's for. Multiple returns → multiple cases worth noting.
3. **Exceptions thrown.** Part of the contract. A reader who doesn't know about them will be surprised.
4. **Side effects.** Writes outside local scope: DB queries, file I/O, HTTP calls, logging, cache ops, global mutations, message publishes. These are often the *real* purpose.
5. **Callers.** If the purpose is unclear from the body, look at 2–3 callsites — how it's *used* tells you what it's *for*.
6. **Tests.** Test names encode intent: `test_rejects_expired_token` tells you more about the function's purpose than its body does.

## Step 3 — Name the purpose in one sentence

Before writing anything else, complete: **"This exists so that ___."** If you can't, you don't understand it yet — go back to Step 2.

Good patterns:
- Transforms *X* into *Y* — *"Turns raw webhook payloads into normalized Order events."*
- Decides *X* — *"Determines whether a user qualifies for the free tier."*
- Enforces *X* — *"Ensures no request reaches the handler without a valid session."*
- Coordinates *X* — *"Runs the nightly export: pull → transform → upload → notify."*

Bad patterns (translation, not summary):
- "Loops through items and checks conditions" — *which* conditions? *why*?
- "Helper function for X" — helps X do *what*?
- "Handles the data" — every function handles data

## Step 4 — Add what the reader needs and the code doesn't say

After the purpose sentence, add only things that are **true**, **not obvious from the signature**, and **would change how a caller uses it**:

- **Side effects.** "Writes to the `audit_log` table on every call." Always worth stating — they're invisible from the signature.
- **Error behavior.** "Throws `QuotaExceededError` rather than returning null" — if there's a choice a reasonable person might get wrong.
- **Preconditions that aren't type-enforced.** "Assumes `items` is already sorted by timestamp."
- **Cost warnings.** "Makes one HTTP call per item — do not call in a tight loop."
- **Asymmetries.** "Idempotent for inserts, not for updates."

**Leave out** what the types already say, what any function in this language does (e.g., "returns a value"), and any uncertainty you'd be guessing at.

## Function-level template

```
<One-sentence purpose, active voice.>

<Side effects, if any. Be specific.>
<Error behavior, if non-obvious.>
<Preconditions or cost warnings, if any.>
```

**Example.** For:

```python
def reconcile(ledger, bank_txns):
    matched, unmatched = [], []
    for txn in bank_txns:
        entry = ledger.find(amount=txn.amount, date_within=timedelta(days=3))
        if entry and not entry.reconciled:
            entry.reconciled = True
            matched.append((entry, txn))
        else:
            unmatched.append(txn)
    ledger.save()
    return matched, unmatched
```

**Bad** (translates): *"Iterates over bank transactions, calls `ledger.find` for each, checks if reconciled, appends to matched or unmatched lists, then saves the ledger."*

**Good** (summarizes): *"Matches bank transactions against ledger entries by amount within a 3-day window, marking each matched entry as reconciled. Persists the ledger. Returns (matched pairs, unmatched transactions). A transaction with no same-amount entry in the window, or whose entry is already reconciled, lands in unmatched."*

The good version lets a caller predict behavior without reading the body. The bad version just makes them read the body slower.

## Class-level: responsibility, not inventory

Don't list every method. Answer: **what is this class responsible for, and what does it deliberately leave to others?**

```
RateLimiter

Tracks request counts per client and decides whether to admit or reject
the next request. Uses a sliding window — a client at the limit is
rejected until its oldest counted request ages out, not until a fixed
interval elapses.

State is in-memory only. Does not persist across restarts; does not
coordinate across instances. For distributed limiting, wrap a shared
store.

Thread-safe (internal lock). Not async-safe — do not `await` while
holding a reference to a window.
```

Note: "what it doesn't do" (persist, coordinate) is as valuable as what it does.

## Module-level: the boundary, not the contents

At module scope, the reader is deciding whether to open this file. Tell them what crosses the boundary:

```
payments/stripe_adapter.py

Isolates all Stripe SDK usage behind four operations: charge, refund,
subscribe, cancel. The rest of the codebase speaks in our `Payment` and
`Subscription` types and never imports `stripe` directly.

Incoming: our domain types + tenant config (API key per tenant).
Outgoing: Stripe HTTP; audit log; `payment.completed` event on the bus.

Stripe errors are caught here and re-raised as our `PaymentError`
hierarchy — callers never see raw Stripe exceptions. Retries transient
failures (5xx, rate limits) internally with backoff.
```

## Edge cases

- **Code you can't figure out:** Say so. *"Purpose unclear — no callers found, no tests, name is generic. Appears to transform X→Y but the 'why' isn't evident from the code."* This is more useful than a confident wrong answer.
- **Code that does two unrelated things:** Don't force a single purpose sentence. *"Does two things: (a) validates the schema, (b) logs metrics. Consider splitting."* The summary can surface the design problem.
- **Generated code** (protobuf stubs, ORM models, parser output): Don't summarize the generated body. Summarize the *schema* or *grammar* it came from — one sentence, then point at the source.
- **Dead code:** Summarize normally, then flag: *"No callers found — possibly dead. Verify before relying on this summary."*
- **Function whose behavior depends on a global / config flag:** Describe each mode: *"In strict mode, rejects unknown fields; in lenient mode, ignores them. Mode is read from `config.VALIDATION_MODE` at call time."*

## Do not

- **Don't narrate control flow.** "First it checks X, then if Y it does Z" is reading aloud, not summarizing. The reader could do that themselves.
- **Don't lie about what you can't see.** If the summary depends on a function whose body you haven't read, say *"delegates to `foo()` — behavior depends on that."*
- **Don't pad to hit a length.** A one-line function gets a one-line summary. Longer isn't more thorough, it's noisier.
- **Don't drop the domain vocabulary.** If the code says `invoice`, say `invoice` — don't genericize to "the object" or "the record."
- **Don't editorialize in a summary.** "This should really use a factory pattern" belongs in a review, not a summary. The summary is *what it does*; judgment is a separate product.
