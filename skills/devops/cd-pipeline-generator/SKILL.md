---
name: cd-pipeline-generator
description: Generates deployment pipelines with environment promotion, approval gates, and rollback triggers based on target infrastructure. Use when wiring automated deployments from CI to staging/production, when the user asks for a release pipeline, or when adding promotion gates to an existing deploy workflow.
license: Apache-2.0
metadata:
  category: "devops"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "ci-pipeline-synthesizer, rollback-strategy-advisor, containerization-assistant"
---

# CD Pipeline Generator

CD is CI plus **consequences**. The artifact from CI flows through environments with gates between them. The shape is always the same; only the deploy mechanism changes.

## The promotion ladder

```
CI artifact ──▶ dev ──▶ staging ──[gate]──▶ prod
```

Each arrow is a deploy. Each `[gate]` is a decision point. The pipeline's job is to make the arrows automatic and the gates explicit.

| Environment | Trigger                  | Gate before it                | Rollback urgency |
| ----------- | ------------------------ | ----------------------------- | ---------------- |
| dev         | Every green CI on main   | None — auto                   | Don't bother     |
| staging     | Every green CI on main   | None (or: schedule, daily)    | Low              |
| prod        | Tag / release / manual   | Human approval + smoke test   | High             |

## Step 1 — Determine the deploy mechanism

| Target                        | Deploy command                                                  | Artifact shape         |
| ----------------------------- | --------------------------------------------------------------- | ---------------------- |
| Kubernetes                    | `kubectl apply` / `helm upgrade` / ArgoCD sync                  | Image tag in manifest  |
| Serverless (Lambda, Cloud Functions) | `aws lambda update-function-code` / framework CLI        | Zip / image            |
| VM / bare metal               | `rsync` + restart, or Ansible playbook                          | Tarball / package      |
| PaaS (Heroku, Fly, Render)    | `git push <remote>` or platform CLI                             | Git ref / image        |
| Static site                   | `aws s3 sync` / `netlify deploy` / push to `gh-pages`           | Built `dist/` folder   |

Read the repo. A `Dockerfile` + `k8s/` dir means Kubernetes. A `serverless.yml` means serverless. A `Procfile` means PaaS.

## Step 2 — One artifact, many environments

**The artifact is built once** (in CI) and **promoted unchanged.** If you rebuild per environment, your staging test means nothing about prod.

- Containers: tag with git SHA. `staging` deploys `:abc123`; prod deploys the *same* `:abc123` after staging passes.
- Packages: build once, stash in an artifact store, download per environment.

Environment-specific config comes from the **environment**, not from the artifact. Env vars, mounted configs, secret stores.

## Step 3 — Gates

| Gate type          | Implementation                                         | When to use                     |
| ------------------ | ------------------------------------------------------ | ------------------------------- |
| Human approval     | GitHub `environment:` protection rules; GitLab `when: manual` | Before prod, always       |
| Smoke test         | A pipeline step that hits the deployed service's health endpoint | After every deploy, auto  |
| Soak time          | `sleep` / scheduled job — deploy staging, wait N hours, auto-promote if no alerts | Mature systems only |
| Canary/percentage  | Platform-specific (Argo Rollouts, traffic splitting)   | High-traffic prod only    |

Start simple. Human approval before prod, smoke test after every deploy. Add soak/canary only when there's evidence they're needed.

## Worked example

**Repo:** Node app, `Dockerfile`, `k8s/deployment.yaml`, deploys to GKE.

```yaml
# .github/workflows/cd.yml
name: CD

on:
  workflow_run:
    workflows: [CI]
    types: [completed]
    branches: [main]

jobs:
  deploy-staging:
    if: github.event.workflow_run.conclusion == 'success'
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - uses: actions/checkout@v4
      - run: |
          gcloud auth configure-docker
          kubectl set image deployment/app app=gcr.io/proj/app:${{ github.sha }} -n staging
      - run: ./scripts/smoke-test.sh https://staging.example.com

  deploy-prod:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment: production   # ← GitHub env protection = human approval gate
    steps:
      - uses: actions/checkout@v4
      - run: |
          gcloud auth configure-docker
          kubectl set image deployment/app app=gcr.io/proj/app:${{ github.sha }} -n prod
      - run: ./scripts/smoke-test.sh https://example.com
```

Same SHA both times. The approval gate lives in GitHub's environment protection rules — not in the YAML.

## Edge cases

- **No staging environment exists:** Don't invent one in YAML. Either the user sets one up, or you generate a prod-only pipeline with a louder approval gate. Flag the gap.
- **Database migrations:** They can't be rolled back by redeploying the old image. Put migration in a separate gated job. → `rollback-strategy-advisor`.
- **Multiple services, one repo:** Deploy jobs per service, with `paths:` triggers so touching service A doesn't redeploy service B.
- **The smoke test can't run from the CI runner** (private network): Run it from inside the cluster as a Job, and have the pipeline poll its result.

## Do not

- **Do not** rebuild the artifact per environment. Build once in CI, promote the immutable artifact.
- **Do not** put the approval logic in a `if: github.actor == 'alice'` condition. Use platform environment protection — it's auditable.
- **Do not** `|| true` a failing smoke test. A smoke test that can't block is theater.
- **Do not** store deploy credentials as plaintext env vars. Reference `${{ secrets.* }}` and list what must be configured.
- **Do not** generate a canary/blue-green pipeline for a system that gets 10 requests a day. Match sophistication to traffic.

## Output format

```
## Promotion ladder
<env> → <env> → [gate] → <env>

## Deploy mechanism
<detected target + command>

## Pipeline
<file path>
<code block>

## Required configuration
- Secrets: <list>
- Environment protection rules: <which envs need approval>

## Gaps
- <missing smoke test / no staging env / migrations not handled>
```
