---
name: traceability-matrix-generator
description: Builds a bidirectional traceability matrix linking requirements to design elements, code, and tests — so every requirement traces forward to its implementation and every test traces back to its justification. Use for compliance audits, when answering why a piece of code exists, or when checking that nothing was built without a reason.
license: Apache-2.0
metadata:
  category: "requirements"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "req-to-test, coverage-enhancer"
---

# Traceability Matrix Generator

Traceability answers two questions: **"What implements this requirement?"** (forward) and **"Why does this code exist?"** (backward). The matrix is the answer in table form.

## The chain

```
Requirement ──► Design element ──► Code ──► Test
     ▲                                         │
     └─────────────────────────────────────────┘
                    (test verifies req)
```

Every link is a traceable edge. Gaps are rows with empty cells.

## Matrix structure

| Req ID | Requirement (brief) | Design | Code | Test | Status |
| ------ | ------------------- | ------ | ---- | ---- | ------ |
| REQ-1.1 | Rate limit: 100/min/user | RateLimiter component | `middleware/ratelimit.py` | `test_ratelimit_100_per_min` | ✓ |
| REQ-1.2 | Return 429 on limit | — | `ratelimit.py:L45` | `test_ratelimit_returns_429` | ✓ |
| REQ-2.1 | Audit all writes | AuditLogger | `audit.py` | — | ⚠ No test |
| REQ-3.4 | Support IPv6 | — | — | — | ❌ Gap |

## Building it — forward trace

1. **Enumerate requirements.** Every MUST/SHOULD with an ID. Decompose compounds — one row per atomic claim.
2. **For each requirement, find code.** → `requirement-coverage-checker` techniques: grep for IDs, grep for domain terms, structural search.
3. **For each code location, find tests.** What tests exercise this code? Coverage tools (`pytest --cov`) tell you which tests hit which lines.
4. **Fill the matrix.** One row per requirement, cells for each link in the chain.

## Building it — backward trace (orphan detection)

Forward trace finds unimplemented requirements. Backward trace finds **unrequired code**:

1. **Enumerate code units** (functions, endpoints, modules).
2. **For each: what requirement justifies this?** If none — it's either:
   - Implicitly required (infrastructure — logging, config loading). Fine.
   - Speculatively built (YAGNI violation). Consider removing.
   - Undocumented requirement. The code is right, the spec is incomplete — add the requirement.

## Trace strength

| Trace type                          | Strength | Maintenance cost              |
| ----------------------------------- | -------- | ----------------------------- |
| Explicit ID in code/test            | Strong   | Low — grep finds it           |
| `@covers("REQ-1.1")` decorator      | Strong   | Low — machine-checkable       |
| Mention in docstring                | Medium   | Medium — can drift            |
| Structural match (inferred)         | Weak     | High — re-derive every audit  |

**For auditable systems:** use explicit IDs. `# REQ-1.1` in the code, `@pytest.mark.req("1.1")` on the test. Then the matrix is a grep, not an archaeology dig.

## Worked example — generating from a tagged codebase

**Convention in this codebase:** tests carry `@pytest.mark.req("X.Y")`; code has `# REQ-X.Y` comments.

```python
# middleware/ratelimit.py
# REQ-1.1, REQ-1.2
@app.middleware("http")
async def ratelimit(request, call_next):
    ...

# tests/test_ratelimit.py
@pytest.mark.req("1.1")
def test_ratelimit_allows_100_per_minute(): ...

@pytest.mark.req("1.2")
def test_ratelimit_returns_429_on_excess(): ...
```

**Matrix generation (scripted):**

```python
import re, ast, pathlib

reqs = load_requirements("spec.md")          # {id: text}
code_traces = {}   # {req_id: [file:line, ...]}
test_traces = {}   # {req_id: [test_name, ...]}

for f in pathlib.Path("src").rglob("*.py"):
    for lineno, line in enumerate(f.read_text().splitlines(), 1):
        for rid in re.findall(r"REQ-(\d+\.\d+)", line):
            code_traces.setdefault(rid, []).append(f"{f}:{lineno}")

for f in pathlib.Path("tests").rglob("*.py"):
    tree = ast.parse(f.read_text())
    for node in ast.walk(tree):
        if isinstance(node, ast.FunctionDef):
            for dec in node.decorator_list:
                # match @pytest.mark.req("X.Y")
                if (isinstance(dec, ast.Call) and ast.unparse(dec.func) == "pytest.mark.req"):
                    rid = dec.args[0].value
                    test_traces.setdefault(rid, []).append(f"{f.name}::{node.name}")

# Emit matrix
for rid, text in reqs.items():
    code = code_traces.get(rid, [])
    tests = test_traces.get(rid, [])
    status = "✓" if code and tests else ("⚠" if code else "❌")
    print(f"| {rid} | {text[:40]} | {', '.join(code) or '—'} | {', '.join(tests) or '—'} | {status} |")
```

Mechanical, reproducible, runs in CI.

## Do not

- **Do not** build the matrix once and let it rot. If it's not regenerated on every commit, it's fiction by month two. Script it, run it in CI.
- **Do not** trace to whole files. `auth.py` implements 30 requirements — that's not a trace, that's a guess. Trace to functions or regions.
- **Do not** count inferred traces as equal to explicit ones. If you had to read the code to figure out it implements REQ-1.1, the next auditor will too. Add the tag.
- **Do not** ignore backward orphans. Unrequired code is either dead (remove it) or under-specified (add the requirement). Both are actionable.

## Output format

```
## Matrix
| Req ID | Requirement | Design | Code | Test | Status |
| ------ | ----------- | ------ | ---- | ---- | ------ |

## Gaps (forward — unimplemented)
| Req ID | Missing | Action |
| ------ | ------- | ------ |

## Orphans (backward — unjustified code)
| Code | Classification | Action |
| ---- | -------------- | ------ |
| <func> | Implicit infra | None — expected |
| <func> | Undocumented req | Add REQ-X.Y to spec |
| <func> | Speculative | Consider removal |

## Trace strength
Explicit (tagged): <N>  Inferred: <M>  — lower M by tagging

## Regeneration
<command to rebuild this matrix — goes in CI>
```
