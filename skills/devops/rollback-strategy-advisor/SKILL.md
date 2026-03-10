---
name: rollback-strategy-advisor
description: Advises on rollback strategies by analyzing what a deploy changes — recommending revert, roll-forward, feature-flag kill, or data repair depending on reversibility. Use during an incident when a deploy went bad, when designing a deploy pipeline and the user asks how to make it reversible, or when a migration needs an undo plan.
license: Apache-2.0
metadata:
  category: "devops"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "cd-pipeline-generator, config-consistency-checker"
---

# Rollback Strategy Advisor

"Just deploy the old version" only works if the new version didn't change anything the old version depends on. It usually did.

## The reversibility question

Before choosing a strategy, classify what the bad deploy changed:

| Change type                       | Reversible by redeploying old code? | Why / why not                                          |
| --------------------------------- | ----------------------------------- | ------------------------------------------------------ |
| Stateless code only               | ✅ Yes                               | Old code runs on old state; no state changed           |
| Additive schema (new column, new table) | ✅ Yes                         | Old code ignores the new column                        |
| Destructive schema (drop column, rename) | ❌ No                         | Old code expects the column that's gone                |
| Additive data (new rows)          | ⚠️  Usually                         | Unless the new rows confuse old code's queries         |
| Mutated data (UPDATE existing rows) | ❌ No                              | Old code expects old data shape; you need data repair  |
| New external side effects (emails sent, payments made) | ❌ Never        | Can't unsend. Compensating actions only.               |
| Config only                       | ✅ Yes                               | Revert the config                                      |
| Feature flag flip                 | ✅ Instantly                         | Flip it back — this is why flags exist                 |

## Decision tree

```
Is there a feature flag gating the bad behavior?
  ├─ YES → Kill the flag. Done in seconds. Investigate at leisure.
  └─ NO →
     Did the deploy change data/schema?
       ├─ NO → Redeploy previous artifact. Done.
       └─ YES →
          Was the change additive-only?
            ├─ YES → Redeploy previous artifact. Clean up schema later.
            └─ NO →
               Can you roll FORWARD (fix is small, well-understood)?
                 ├─ YES → Roll forward. Faster than untangling.
                 └─ NO → You're in data repair. See below.
```

## Data repair (the hard case)

When old code can't run on new data:

1. **Stop the bleeding.** Feature flag, maintenance page, or traffic drain — whatever stops more data from being mutated.
2. **Snapshot.** Before you touch anything. `pg_dump` / volume snapshot. You will want this when your repair script has a bug.
3. **Assess scope.** How many rows? `SELECT count(*) WHERE <mutated-condition>`. 10 rows is a manual fix. 10 million is a migration.
4. **Repair or compensate:**
   - Reversible mutation → write the inverse UPDATE
   - Irreversible mutation → restore affected rows from snapshot/backup
   - External side effects → compensating action (refund, apology email, manual ticket)
5. **Then** redeploy old code.

## Design-time advice — make rollbacks boring

If you're not mid-incident, the best advice is to make the next deploy reversible:

| Technique                    | Makes rollback trivial when                              |
| ---------------------------- | -------------------------------------------------------- |
| Feature flags                | You're shipping a behavior change — gate it              |
| Expand-contract migrations   | You're changing schema — add new alongside old, migrate, remove old in a *later* deploy |
| Dual-write period            | You're changing data format — write both formats until new code is stable |
| Immutable artifacts by SHA   | You're deploying — `image:abc123` can always be re-deployed; `image:latest` can't |
| Backward-compatible APIs     | You're changing an interface — new version reads old format |

## Worked example

**Situation:** Deployed v2.4.0 at 14:00. At 14:20, error rate spikes. v2.4.0 added a `NOT NULL` column `users.tenant_id` with a default, and the new code reads it.

**Reversibility check:** Schema change was additive (new column with default) → old code should ignore it. ✅ Reversible.

**Wait** — check the migration. It ran `ALTER TABLE users ADD COLUMN tenant_id ... NOT NULL DEFAULT 1`. But old code does `INSERT INTO users (...)` without `tenant_id`. Does `NOT NULL DEFAULT` allow that? Yes — the default fires. ✅ Still reversible.

**Action:** Redeploy v2.3.9. `kubectl rollout undo deployment/app`. The column stays; old code ignores it. Clean up never — the column is fine, the bug is elsewhere in v2.4.0.

**Post-mortem note:** If the migration had been `NOT NULL` *without* a default, old code's INSERTs would fail. That would have been a non-reversible schema change masquerading as additive.

## Do not

- **Do not** roll back before you understand what changed. Rolling back into a broken state is worse than being broken in a known state.
- **Do not** roll forward under pressure with an untested fix. Rolling forward is for when you *know* the fix; it's not a license to deploy a guess.
- **Do not** skip the snapshot before data repair. The repair will have a bug. It always does.
- **Do not** `DROP` the new column/table during rollback. Leave it. It's harmless, and the next deploy attempt will need it.
- **Do not** design a rollback strategy during an incident. Design it when you design the deploy. → `cd-pipeline-generator` should include the undo path.

## Output format

**Incident mode:**
```
## Reversibility
<change type> → <reversible: yes/no/partial>

## Recommended action
<flag kill | redeploy | roll forward | data repair>

## Steps
1. ...

## If this doesn't work
<next fallback>
```

**Design mode:**
```
## This deploy's reversibility class
<from the table>

## To make it cheaply reversible
<specific technique: flag / expand-contract / dual-write>

## Rollback command (pre-written — paste during incident)
<exact command>
```
