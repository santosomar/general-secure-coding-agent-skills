---
name: config-consistency-checker
description: Detects inconsistencies across configuration files, environments, and deployment manifests — missing keys, drifted values, type mismatches. Use when debugging why staging behaves differently from production, before a deploy to catch config drift, or when auditing multi-environment configs.
license: Apache-2.0
metadata:
  category: "devops"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "configuration-generator, cd-pipeline-generator"
---

# Config Consistency Checker

"Works in staging, broken in prod" is almost always a config diff. Find it before deploy, not after.

## What to compare

| Axis                              | Compare                                                  | Looking for                                       |
| --------------------------------- | -------------------------------------------------------- | ------------------------------------------------- |
| Across environments               | `staging.yaml` ↔ `production.yaml`                       | Keys present in one, missing in other             |
| Config file ↔ code                | Config keys ↔ `config.get(...)` / `os.environ[...]` calls | Dead keys (in config, never read) and missing keys (read, never defined) |
| Config file ↔ deployment          | App config ↔ K8s ConfigMap/Deployment env                | Config expects `FOO`; deployment sets `FOO_URL`   |
| Config file ↔ schema/defaults     | Loaded values ↔ type annotations / validators            | `"30"` where the code expects `30`                |
| Across replicas (runtime)         | Live config on pod A ↔ pod B                             | Partial rollout, stale ConfigMap mount            |

## The key-diff algorithm

1. Flatten each config to `dotted.key.path → value` pairs.
2. Set-diff the key sets.
3. For keys in both, value-diff.
4. Classify each difference.

```
staging_keys − production_keys   → "missing in prod" (usually a bug)
production_keys − staging_keys   → "missing in staging" (usually a bug, unless it's a prod-only integration)
staging[k] ≠ production[k]       → "differs" — intentional or drift?
```

## Classifying differences: intentional vs. drift

Not every difference is a bug. The signal is whether the difference is **explicable**:

| Difference class                    | Intentional?                                              |
| ----------------------------------- | --------------------------------------------------------- |
| Database host differs               | Intentional — they're different databases                 |
| Pool size differs                   | Maybe — prod should be ≥ staging; if prod is *smaller*, bug |
| Log level differs                   | Intentional (prod=INFO, dev=DEBUG). If prod=DEBUG, bug    |
| Feature flag differs                | Depends entirely on the flag — ask                        |
| Key exists in one, not the other    | **Almost always drift** — someone forgot to add it to both |
| Typo'd key (close string distance)  | **Always drift** — `databse_host` is reading the default  |

## Worked example

**Files:** `config/staging.yaml`, `config/production.yaml`

**Flattened:**

```
staging:
  db.host                = staging-db.internal
  db.pool_size           = 20
  cache.ttl_seconds      = 300
  feature.new_checkout   = true
  log.level              = debug

production:
  db.host                = prod-db.internal
  db.pool_size           = 10
  cache.ttl_seconds      = 300
  feature.new_checkout   = true
  log.level              = info
  sentry.dsn             = https://...
```

**Diff:**

| Key                     | Staging | Prod         | Verdict                                  |
| ----------------------- | ------- | ------------ | ---------------------------------------- |
| `db.host`               | differs | differs      | ✅ Intentional — different databases      |
| `db.pool_size`          | 20      | 10           | ⚠️  Prod pool *smaller* than staging — probably wrong |
| `cache.ttl_seconds`     | 300     | 300          | ✅ Match                                  |
| `feature.new_checkout`  | true    | true         | ✅ Match                                  |
| `log.level`             | debug   | info         | ✅ Intentional — expected split           |
| `sentry.dsn`            | —       | set          | ⚠️  Missing in staging — errors from staging go nowhere |

Two flags, both worth raising.

## Code ↔ config cross-check

Grep the codebase for config reads:

```
rg 'config\[|config\.get|os\.environ\[|getenv\(' --type py
```

Extract the key names. Compare against what's actually in the config files:
- **In code, not in config:** The app will hit a default or crash. Which one? Read the code.
- **In config, not in code:** Dead config. Delete it, or it's a typo for something the code *does* read.

## Edge cases

- **Secrets show as "differs" because they're redacted:** Compare key *presence*, not value, for anything marked secret. `password: ***` ≠ `password: ***` by string comparison but they're both set — that's what matters.
- **Array ordering differs:** `[a, b]` vs `[b, a]` — depends on semantics. A list of allowed origins is a set (order irrelevant); a list of middleware is a sequence (order critical). You have to know which.
- **One config is generated (Helm/Kustomize), the other is hand-written:** Compare the *rendered* output, not the template. `helm template` / `kustomize build` first.
- **The app has fallback defaults in code:** A missing key isn't an error if the code has `config.get("foo", sensible_default)`. But flag it — implicit defaults drift silently.

## Do not

- **Do not** report "47 keys differ" without classifying. The user wants the 2 that are bugs, not the 45 that are intentional environment splits.
- **Do not** compare secret values. Compare presence. Never echo a secret value in the diff output.
- **Do not** trust the config file as source of truth. The code's reads are the truth; the config is what's supposed to satisfy them.
- **Do not** ignore near-miss key names. `database_url` in config and `DATABASE_URI` in code — Levenshtein distance 2, probably the bug.

## Output format

```
## Missing keys
  <key>  — in <envA>, missing from <envB>  [<likely intentional | likely drift>]

## Suspicious value diffs
  <key>  <envA>=<val>  <envB>=<val>  — <why suspicious>

## Dead config (in file, never read by code)
  <key>

## Unbacked reads (in code, never in config — will default or crash)
  <file>:<line>  reads <key>  — default: <value | NONE — will crash>

## Clean
  <N> keys match across all environments
```
