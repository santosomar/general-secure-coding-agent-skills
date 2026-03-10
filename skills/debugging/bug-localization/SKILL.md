---
name: bug-localization
description: Pinpoints the exact file, function, or line in a codebase responsible for a reported bug using static and dynamic analysis signals. Use when a bug is reported but the fault location is unknown, when narrowing down a failure to a specific code region, when triaging an issue tracker ticket, or when the user asks to locate where a bug originates.
license: Apache-2.0
metadata:
  category: "debugging"
  suite: "general-secure-coding-agent-skills"
  version: "0.2.0"
  related: "bug-reproduction-test-generator, bug-to-patch-generator, regression-root-cause-analyzer"
---

# Bug Localization

Pinpoint the fault location — the code that must change to fix the bug — not merely where the symptom surfaces. A `NullPointerException` at line 400 is a symptom; the fault is wherever the null was introduced or wherever the null-check was forgotten.

## Step 1 — Classify the input signal

Different signals demand different first moves. Pick the row that matches:

| Signal available                   | First move                                         | Then                                    |
| ---------------------------------- | -------------------------------------------------- | --------------------------------------- |
| Stack trace / crash dump           | Extract top frame that belongs to project code     | Backward slice from that variable       |
| Failing test, no trace             | Read the failing assertion                         | Find producer of the asserted value     |
| Wrong output, no test              | Diff actual vs expected — identify which field(s)  | Trace producer of divergent field       |
| "Broke after upgrading X"          | Search for callsites into X                        | → `regression-root-cause-analyzer`      |
| "Broke in commit range A..B"       | Don't read code first — bisect                     | → `regression-root-cause-analyzer`      |
| Log shows error                    | grep for the error string literal                  | Find the condition that triggers it     |
| Intermittent / "sometimes"         | Do **not** stare at code; reproduce first          | → `bug-reproduction-test-generator`     |

If more than one signal is available, prefer stack traces > failing tests > logs > prose — in that order of precision.

## Step 2 — Establish the anchor

The **anchor** is one concrete code location you are certain is involved. You cannot localize without one.

- **From a stack trace:** Discard frames in vendor/stdlib/framework code. The topmost remaining frame is the anchor. If *every* frame is in vendor code, the anchor is the project callsite that invoked the vendor API — find it by grepping for the last vendor-frame function name.
- **From a failing assertion:** The anchor is the expression inside the assert, not the assert itself. In `assertEqual(result.total, 100)`, the anchor is whatever produces `result.total`.
- **From a log line:** `grep -rn "exact log message"` → the logger callsite is the anchor.
- **From prose only:** You have no anchor yet. Grep for domain terms in the report ("checkout", "invoice", "refresh token"). If the user-facing string appears in i18n files, follow the key back to a callsite. If nothing matches, ask the user for one of: exact error text, a screenshot, the HTTP endpoint, or the UI button label.

## Step 3 — Trace backward from the anchor

The anchor is where the bad value *arrives*. The fault is where the bad value *is born*.

1. Identify the faulty variable/field at the anchor.
2. List every assignment or mutation of that variable reachable backward: constructor params, setters, direct writes, return values flowing into it.
3. For each producer, ask: **could this produce the observed bad value?** Not "is this code suspicious" — specifically, can it yield *this exact* null / wrong number / malformed string. If yes, it's a candidate. If no, prune it and do not expand further.
4. Recurse on each surviving candidate.

Stop when you reach one of:
- An external input (HTTP param, env var, config, DB row) → fault is missing validation here
- A literal / constant → fault is a wrong constant
- A computation → fault is the computation itself
- A function with a clear contract it violates → fault is inside that function

Cap backward tracing at **~5 hops**. If still ambiguous at 5 hops, the static picture alone is insufficient — jump to Step 4.

## Step 4 — Disambiguate with dynamic evidence

You have N candidates and can't distinguish them by reading. Make the code tell you.

- **Failing test exists:** Run it with a debugger / print statements on each candidate line. The candidate that actually executes on the failing path and shows the bad value wins.
- **Coverage available:** Any candidate in code not covered by the failing test is eliminated.
- **Spectrum-based (many tests, some failing):** Score each candidate by how much more often it runs in failing tests than in passing tests. High failing-run-rate + low passing-run-rate = suspicious. This is the Ochiai/Tarantula intuition; you can do it by hand for a small candidate set.
- **No test at all:** Write a minimal reproducer before guessing further (→ `bug-reproduction-test-generator`). Localization without reproduction is speculation.

## Step 5 — Distinguish fault from propagation

Before reporting, apply this sanity check to your top candidate:

> "If I correct the value *at this exact line*, does the symptom disappear **and** every caller still receives a correct value?"

- **Yes to both** → this is the fault. Report it.
- **Symptom disappears but upstream callers would still get a bad value** → you found a *propagation point*, not the fault. Keep tracing backward.
- **No, the symptom persists** → wrong candidate. Back to Step 3 with this candidate pruned.

If two candidates both pass this check (both are valid fix points), prefer the upstream one — it fixes more paths.

## Output format

```
PRIMARY:  src/billing/invoice.py:142 — compute_tax()
  Rationale: rate field read from config as string, multiplied without
  float conversion → "0.08" * qty produces string repetition, not
  arithmetic. Fault is missing float() cast at config load.
  Confidence: High — reproduced with qty=3, rate="0.08" yielding "0.080.080.08"

ALTERNATIVE: src/config/loader.py:31 — load_tax_config()
  Rationale: upstream fix point. Config loader returns string; coercing
  to float here fixes all consumers, not just compute_tax.
  Confidence: Medium — would need to verify no consumer expects string

NEXT: → bug-to-patch-generator with PRIMARY location
```

State confidence as **High** (reproduced and confirmed), **Medium** (strong static trace but not executed), or **Low** (best guess, list what would confirm it).

## Worked example

**Input:** "POST /orders returns 500. Stack trace shows `KeyError: 'discount'` at `order_service.py:88`."

1. *Classify* → stack trace → top project frame = `order_service.py:88`.
2. *Anchor* → line 88: `total -= payload["discount"]`. Faulty value is `payload`, missing key `"discount"`.
3. *Trace* → `payload` is the parsed HTTP body. Origin is external input.
4. *External input reached* → fault is **missing validation / missing default**. Is `discount` required or optional per the API spec?
   - If optional → fault is at line 88: should be `payload.get("discount", 0)`.
   - If required → fault is in the request validation layer that let the body through.
5. *Sanity check* → if discount is optional, correcting line 88 fixes this symptom and introduces no upstream breakage. Done.

**Output:** `order_service.py:88` — missing default for optional field. Confidence: High. Alternative fix point: request schema validator (if discount should be required).

## Edge cases

- **Trace points at generated code** (ORM, gRPC stubs, template output): treat the generator callsite / template / schema as the real anchor. The fault is almost never in generated code.
- **Off-by-one and boundary bugs:** the fault line executes *correctly* for most inputs. The anchor points to the right function but the wrong iteration. Focus on loop bounds (`<` vs `<=`, `len-1` vs `len`), not the loop body.
- **Race conditions:** static tracing finds the *participants* but not the interleaving. If you see shared mutable state and no lock, report both access sites as *joint* fault locations and flag that the fix is synchronization, not a line edit.
- **Configuration/data faults:** if the trace dead-ends at a config read and the code is correct, the fault is in the config *value*. Report the config key and the code location that reads it.
- **Heisenbug** (disappears under debugger): usually timing- or optimization-sensitive. Switch to log-based tracing with high-resolution timestamps instead of breakpoints.

## Do not

- **Don't report the crash site as the fault** without tracing backward — symptom ≠ fault.
- **Don't list every file you touched.** Max 3 candidates. If you genuinely can't narrow below 3, say what dynamic evidence would disambiguate.
- **Don't trace forward.** Forward tracing finds *consequences*, which are irrelevant to localization. Always trace backward from the symptom.
- **Don't skip prose bug reports.** They often contain the only known reproduction steps ("only on Tuesdays", "only for EU users") — those constraints dramatically shrink the candidate set.
- **Don't fix while localizing.** Report the location and hand off to → `bug-to-patch-generator`. Fixing-while-finding biases you toward the first plausible candidate.
