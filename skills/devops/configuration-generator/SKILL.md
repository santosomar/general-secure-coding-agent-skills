---
name: configuration-generator
description: Generates configuration files for services and tools (app config, logging config, linter config, database config) from a brief description of desired behavior, matching the target format's idioms. Use when bootstrapping a new service, when the user asks for a config file for a specific tool, or when translating config intent between formats.
license: Apache-2.0
metadata:
  category: "devops"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "config-consistency-checker, ci-pipeline-synthesizer"
---

# Configuration Generator

A config file is an **interface** between a human operator and a running system. It should read like a document, not a database dump.

## Step 1 — Identify the config consumer

The consumer dictates the format and the idioms. Don't guess.

| Consumer                   | Format              | Look for existing example at                    |
| -------------------------- | ------------------- | ----------------------------------------------- |
| Your own app               | Whatever the loader expects — check the code | `config.py`, `settings.py`, `Config.java` |
| nginx / Apache             | Custom block syntax | `/etc/nginx/sites-available/`                   |
| Postgres / MySQL           | INI-like `key = value` | `postgresql.conf` / `my.cnf` — copy the header |
| Prometheus / Grafana       | YAML                | Their docs have a canonical minimal example — start there |
| Linters (eslint, ruff, golangci) | Tool-specific (JSON/TOML/YAML) | Run `<tool> --init` first; edit its output |
| Logging (log4j, logback, Python `logging`) | XML / YAML / `dictConfig` | The framework ships a default — diff against it |

## Step 2 — Separate secrets from config

The first question for every key: **does this belong in version control?**

| Goes in config file (committed)        | Goes in env / secret store (NOT committed)  |
| -------------------------------------- | ------------------------------------------- |
| Feature flags, log levels, ports       | API keys, passwords, tokens                 |
| Service URLs (non-secret)              | Database connection strings with credentials |
| Timeouts, pool sizes, retry counts     | Signing keys, certificates' private halves  |
| Anything that differs by design intent | Anything that differs by who's allowed to know |

For every secret, emit a **placeholder** with a loud comment:

```yaml
database:
  host: db.internal
  port: 5432
  # DATABASE_PASSWORD is read from env — NEVER commit a real value here
  password: ${DATABASE_PASSWORD}
```

## Step 3 — Layer by environment

One base config, per-environment overlays that only contain **differences**:

```
config/
  base.yaml          ← everything
  staging.yaml       ← only what differs from base (3 keys, not 300)
  production.yaml    ← only what differs from base
```

If the app doesn't support layered loading, generate full per-env files — but also generate a diff so drift is visible.

## Defaults — what to emit when the user doesn't say

| Setting domain    | Safe default                                  | Rationale                          |
| ----------------- | --------------------------------------------- | ---------------------------------- |
| Log level         | `INFO` (prod), `DEBUG` (dev)                  | `DEBUG` in prod leaks + floods     |
| Timeouts          | Always set one — 30s if no better info        | Infinite timeout = hung thread     |
| Connection pools  | Small (10) — tune up with evidence            | Oversized pools thrash the DB      |
| TLS               | On, TLS 1.2 minimum                           | Plaintext is never the right default |
| Bind address      | `127.0.0.1` unless the thing needs to be reachable | `0.0.0.0` is an exposure, not a default |
| Retry count       | 3, with backoff                               | 0 retries is brittle; infinite is a DoS on yourself |

## Worked example

**Input:** "Generate a Python logging config — JSON to stdout in prod, human-readable to console in dev, nothing below INFO in prod."

**Output (`logging.yaml`, loaded via `logging.config.dictConfig`):**

```yaml
version: 1
disable_existing_loggers: false

formatters:
  human:
    format: "%(asctime)s %(levelname)-7s %(name)s — %(message)s"
  json:
    (): pythonjsonlogger.jsonlogger.JsonFormatter
    format: "%(asctime)s %(levelname)s %(name)s %(message)s"

handlers:
  console:
    class: logging.StreamHandler
    stream: ext://sys.stdout
    formatter: ${LOG_FORMAT:-human}   # set LOG_FORMAT=json in prod

root:
  level: ${LOG_LEVEL:-INFO}
  handlers: [console]
```

One file, two env vars switch behavior. Dev runs with no env set → human, INFO. Prod sets `LOG_FORMAT=json`.

## Edge cases

- **The tool has no config file — it's all CLI flags:** Generate a wrapper script or a Makefile target. The "config" is the invocation.
- **Config is read from a database / remote service:** You're generating seed data, not a file. Emit the INSERT statements or the API calls.
- **The user wants a Kubernetes ConfigMap:** It's still just a config file with YAML wrapping. Generate the inner content first, then wrap. → `config-consistency-checker` to ensure the ConfigMap matches the Deployment's env mappings.

## Do not

- **Do not** invent keys the consumer doesn't read. Every key must map to something the code actually loads. Unknown keys are silent lies.
- **Do not** put a real secret in an "example" config. Not even a "clearly fake" one like `password: changeme` — those get deployed.
- **Do not** emit a config without comments. The operator reading this at 3am needs to know what each section does.
- **Do not** bind to `0.0.0.0` by default. Make the operator opt into exposure.
- **Do not** generate every possible key with its default value. Emit only what *differs* from the tool's default — a 400-line config where 390 lines are defaults hides the 10 that matter.

## Output format

```
## Files
<path>
<code block with config — commented>

## Environment variables referenced
- <VAR>: <purpose>  (secret: <yes|no>)

## What you still need to set
- <anything that's a placeholder>
```
