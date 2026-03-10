---
name: multi-version-behavior-comparator
description: Compares the runtime behavior of two or more versions of the same code by running them on identical inputs and diffing outputs, side effects, and errors. Use when validating a refactor, port, or optimization; when the user asks if two implementations behave the same; or when investigating a suspected regression across versions.
license: Apache-2.0
metadata:
  category: "code-analysis"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "behavior-preservation-checker, semantic-equivalence-verifier, regression-root-cause-analyzer"
---

# Multi-Version Behavior Comparator

Differential testing, generalized: run N versions on the same inputs, and any disagreement is a bug *somewhere*. You don't need an oracle — the versions are each other's oracle.

Compared to → `behavior-preservation-checker` (yes/no answer, two versions): this handles N versions, and focuses on *characterizing* divergence, not just detecting it.

## What counts as "behavior"

| Observable               | How to capture                                              | When it matters                        |
| ------------------------ | ----------------------------------------------------------- | -------------------------------------- |
| Return value             | Direct comparison (after normalization)                     | Always                                 |
| Exceptions / errors      | Type + message (message is often unstable — compare type)   | Always                                 |
| Side effects (I/O, DB)   | Mock + record, or capture at the boundary                   | If the code does I/O                   |
| stdout/stderr            | Capture streams                                             | If output is the interface             |
| Mutation of inputs       | Deep-copy inputs before call, compare after                 | If inputs are mutable                  |
| Timing                   | Measure, but wide tolerance — don't flag unless 10×+ different | Performance regressions only       |
| Resource usage           | Memory high-water, FD count                                 | Leak hunting                           |

**Pick which observables matter before running.** Diffing everything produces noise.

## Input generation — where the coverage comes from

Differential testing is only as good as the inputs. Sources, in order of effort:

1. **Existing test inputs.** If there's a test suite, extract the inputs. They're already interesting.
2. **Boundary sweep.** Empty, single element, max size, zero, negative, None/null, unicode, very long strings. The usual suspects.
3. **Random/property-based.** Hypothesis, QuickCheck. Great at finding the 17-element list where version A and B disagree.
4. **Coverage-guided.** Use coverage of version A (or B) to guide input generation toward unexercised branches. AFL-style but for diffing.
5. **Mined from production.** Logs, traces, recorded requests. Real-world distribution.

## Normalization — avoiding false positives

Two outputs can be "the same" without being byte-identical:

| Spurious diff                          | Normalize by                                             |
| -------------------------------------- | -------------------------------------------------------- |
| Dict key order (`{a:1, b:2}` vs `{b:2, a:1}`) | Compare as dicts, not as strings/JSON             |
| Float rounding (`0.1 + 0.2`)           | `abs(a - b) < ε`, or round to N digits                   |
| Timestamps, UUIDs in output            | Regex-replace with placeholder before diff               |
| Whitespace / formatting                | Parse then compare ASTs, or normalize whitespace         |
| Order of unordered collections         | Sort before comparing (or use set equality)              |
| Error message wording                  | Compare exception type, not `.args`                      |

## Worked example — 3-way comparison

**Setup:** Three implementations of `urlparse` — stdlib, a hand-rolled one from legacy code, and a port to Go (via subprocess).

**Harness:**

```python
import subprocess, json, urllib.parse
from hypothesis import given, strategies as st
from legacy import parse_url as legacy_parse

def normalize(result: dict) -> dict:
    # Port returns "" for missing, stdlib returns None. Unify.
    return {k: (v if v else None) for k, v in result.items()}

def run_go(url: str) -> dict:
    out = subprocess.run(["./go_urlparse"], input=url, capture_output=True, text=True)
    return json.loads(out.stdout)

@given(st.text(alphabet=st.characters(blacklist_categories=["Cs"]), max_size=200))
def test_three_way(url):
    try:
        std = urllib.parse.urlparse(url)._asdict()
    except Exception as e:
        std = {"__error__": type(e).__name__}
    try:
        leg = legacy_parse(url)
    except Exception as e:
        leg = {"__error__": type(e).__name__}
    go = run_go(url)

    std_n, leg_n, go_n = normalize(std), normalize(leg), normalize(go)

    if not (std_n == leg_n == go_n):
        # Majority vote: 2-of-3 agree → the odd one out is probably wrong
        if std_n == leg_n: suspect = "go"
        elif std_n == go_n: suspect = "legacy"
        elif leg_n == go_n: suspect = "stdlib"  # unlikely but possible
        else: suspect = "all disagree"
        print(f"DIVERGENCE on {url!r}: suspect={suspect}")
        print(f"  stdlib: {std_n}\n  legacy: {leg_n}\n  go:     {go_n}")
```

**Findings after 10k inputs:**

| Input                     | stdlib           | legacy           | go               | Suspect  |
| ------------------------- | ---------------- | ---------------- | ---------------- | -------- |
| `"http://[::1]:80/x"`     | host=`::1`       | host=`[::1]`     | host=`::1`       | legacy   |
| `"a://b?c=%"`             | query=`c=%`      | query=`c=%`      | **ValueError**   | go       |
| `"//foo"`                 | netloc=`foo`     | path=`//foo`     | netloc=`foo`     | legacy   |

Legacy has IPv6 bracket handling wrong. Go is too strict on percent-decoding. Stdlib agrees with 2-of-3 every time — it's the reference.

## Majority voting

With 3+ versions, disagreement is localizable: if 2 agree and 1 differs, the 1 is *probably* wrong. Not always — maybe the 2 share a bug. But it's a strong prior. Rank bugs by (minority size, input simplicity).

## Do not

- **Do not** diff string representations of structured data. `{"a":1,"b":2}` ≠ `{"b":2,"a":1}` as strings, = as dicts. Parse then compare.
- **Do not** treat every timing difference as a regression. 2× slower on one input is noise. 100× slower on all inputs is a regression.
- **Do not** stop at the first divergence. Collect *all* divergences, then cluster by root cause. Ten failing inputs might be one bug.
- **Do not** forget to test the error paths. Both versions handle valid inputs fine and diverge on `None`, empty, or malformed input — that's where bugs live.

## Output format

```
## Versions compared
A: <description/path/commit>
B: ...
C: ...

## Observables
<what was diffed — return value, exceptions, side effects>

## Normalization
<what was canonicalized before comparison>

## Input space
<how inputs were generated — boundary, random, mined>
<N inputs tested>

## Divergences
| Input | A | B | C | Suspect | Cluster |
| ----- | - | - | - | ------- | ------- |

## Clustered root causes
1. <cause>: <N> divergences, suspect=<version>
   Example input: <minimal reproducer>
```
