# CI PR-Check Trigger Deduplication Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Remove the duplicate `pull_request` trigger from all 8 `ci-pr-*.yml` workflows so a commit pushed while a PR is open runs the PR-check pipeline once instead of twice, and update `CI-ARCHITECTURE.md` to document the corrected trigger model.

**Architecture:** Each `ci-pr-<service>.yml` currently declares two triggers (`push` on `feat-*/fix-*/chore-*` and `pull_request` targeting `develop`) that fire on the same commit SHA for the same code, running `_java-pr-check.yml` (or `_node-pr-check.yml` for notification) twice. Deleting the `pull_request:` block from the `on:` section leaves a single `push` trigger; GitHub still surfaces the resulting check on the PR because check runs are associated by commit SHA, not by triggering event, and every student's feat-branch → develop PR lives in one self-contained repo. No reusable workflow, build stage, or the full `ci-*.yml`/`promote-prod.yml` pipeline changes.

**Tech Stack:** GitHub Actions YAML, no application code changes.

---

## File Structure

Files modified (no files created):
- `.github/workflows/ci-pr-api-gateway.yml`
- `.github/workflows/ci-pr-auth-service.yml`
- `.github/workflows/ci-pr-drug-catalog.yml`
- `.github/workflows/ci-pr-inventory-service.yml`
- `.github/workflows/ci-pr-manufacturing-service.yml`
- `.github/workflows/ci-pr-supplier-service.yml`
- `.github/workflows/ci-pr-qc-service.yml`
- `.github/workflows/ci-pr-notification.yml`
- `CI-ARCHITECTURE.md`

Each workflow file gets the identical mechanical edit: delete the `pull_request:` block (4 lines) from its `on:` section. `CI-ARCHITECTURE.md` gets two section edits: the trigger matrix table and the Tier 1 diagram text.

---

### Task 1: Remove `pull_request` trigger from all 8 `ci-pr-*.yml` files

**Files:**
- Modify: `.github/workflows/ci-pr-api-gateway.yml:12-15`
- Modify: `.github/workflows/ci-pr-auth-service.yml:9-12`
- Modify: `.github/workflows/ci-pr-drug-catalog.yml:9-12`
- Modify: `.github/workflows/ci-pr-inventory-service.yml:9-12`
- Modify: `.github/workflows/ci-pr-manufacturing-service.yml:9-12`
- Modify: `.github/workflows/ci-pr-supplier-service.yml:9-12`
- Modify: `.github/workflows/ci-pr-qc-service.yml:9-12`
- Modify: `.github/workflows/ci-pr-notification.yml:9-12`

- [ ] **Step 1: Edit `ci-pr-api-gateway.yml`**

Before:
```yaml
on:
  push:
    branches: ['feat-*', 'fix-*', 'chore-*']
    paths:
      - 'api-gateway/**'
      - '.github/workflows/ci-pr-api-gateway.yml'
  pull_request:
    branches: [develop]
    paths:
      - 'api-gateway/**'
```

After:
```yaml
on:
  push:
    branches: ['feat-*', 'fix-*', 'chore-*']
    paths:
      - 'api-gateway/**'
      - '.github/workflows/ci-pr-api-gateway.yml'
```

- [ ] **Step 2: Edit `ci-pr-auth-service.yml`**

Before:
```yaml
on:
  push:
    branches: ['feat-*', 'fix-*', 'chore-*']
    paths:
      - 'auth-service/**'
      - '.github/workflows/ci-pr-auth-service.yml'
  pull_request:
    branches: [develop]
    paths:
      - 'auth-service/**'
```

After:
```yaml
on:
  push:
    branches: ['feat-*', 'fix-*', 'chore-*']
    paths:
      - 'auth-service/**'
      - '.github/workflows/ci-pr-auth-service.yml'
```

- [ ] **Step 3: Edit `ci-pr-drug-catalog.yml`**

Before:
```yaml
on:
  push:
    branches: ['feat-*', 'fix-*', 'chore-*']
    paths:
      - 'drug-catalog-service/**'
      - '.github/workflows/ci-pr-drug-catalog.yml'
  pull_request:
    branches: [develop]
    paths:
      - 'drug-catalog-service/**'
```

After:
```yaml
on:
  push:
    branches: ['feat-*', 'fix-*', 'chore-*']
    paths:
      - 'drug-catalog-service/**'
      - '.github/workflows/ci-pr-drug-catalog.yml'
```

- [ ] **Step 4: Edit `ci-pr-inventory-service.yml`**

Before:
```yaml
on:
  push:
    branches: ['feat-*', 'fix-*', 'chore-*']
    paths:
      - 'inventory-service/**'
      - '.github/workflows/ci-pr-inventory-service.yml'
  pull_request:
    branches: [develop]
    paths:
      - 'inventory-service/**'
```

After:
```yaml
on:
  push:
    branches: ['feat-*', 'fix-*', 'chore-*']
    paths:
      - 'inventory-service/**'
      - '.github/workflows/ci-pr-inventory-service.yml'
```

- [ ] **Step 5: Edit `ci-pr-manufacturing-service.yml`**

Before:
```yaml
on:
  push:
    branches: ['feat-*', 'fix-*', 'chore-*']
    paths:
      - 'manufacturing-service/**'
      - '.github/workflows/ci-pr-manufacturing-service.yml'
  pull_request:
    branches: [develop]
    paths:
      - 'manufacturing-service/**'
```

After:
```yaml
on:
  push:
    branches: ['feat-*', 'fix-*', 'chore-*']
    paths:
      - 'manufacturing-service/**'
      - '.github/workflows/ci-pr-manufacturing-service.yml'
```

- [ ] **Step 6: Edit `ci-pr-supplier-service.yml`**

Before:
```yaml
on:
  push:
    branches: ['feat-*', 'fix-*', 'chore-*']
    paths:
      - 'supplier-service/**'
      - '.github/workflows/ci-pr-supplier-service.yml'
  pull_request:
    branches: [develop]
    paths:
      - 'supplier-service/**'
```

After:
```yaml
on:
  push:
    branches: ['feat-*', 'fix-*', 'chore-*']
    paths:
      - 'supplier-service/**'
      - '.github/workflows/ci-pr-supplier-service.yml'
```

- [ ] **Step 7: Edit `ci-pr-qc-service.yml`**

Before:
```yaml
on:
  push:
    branches: ['feat-*', 'fix-*', 'chore-*']
    paths:
      - 'qc-service/**'
      - '.github/workflows/ci-pr-qc-service.yml'
  pull_request:
    branches: [develop]
    paths:
      - 'qc-service/**'
```

After:
```yaml
on:
  push:
    branches: ['feat-*', 'fix-*', 'chore-*']
    paths:
      - 'qc-service/**'
      - '.github/workflows/ci-pr-qc-service.yml'
```

- [ ] **Step 8: Edit `ci-pr-notification.yml`**

Before:
```yaml
on:
  push:
    branches: ['feat-*', 'fix-*', 'chore-*']
    paths:
      - 'notification-service/**'
      - '.github/workflows/ci-pr-notification.yml'
  pull_request:
    branches: [develop]
    paths:
      - 'notification-service/**'
```

After:
```yaml
on:
  push:
    branches: ['feat-*', 'fix-*', 'chore-*']
    paths:
      - 'notification-service/**'
      - '.github/workflows/ci-pr-notification.yml'
```

- [ ] **Step 9: Validate YAML syntax on all 8 files**

Run:
```bash
python3 -c "
import yaml, sys
files = [
    '.github/workflows/ci-pr-api-gateway.yml',
    '.github/workflows/ci-pr-auth-service.yml',
    '.github/workflows/ci-pr-drug-catalog.yml',
    '.github/workflows/ci-pr-inventory-service.yml',
    '.github/workflows/ci-pr-manufacturing-service.yml',
    '.github/workflows/ci-pr-supplier-service.yml',
    '.github/workflows/ci-pr-qc-service.yml',
    '.github/workflows/ci-pr-notification.yml',
]
for f in files:
    yaml.safe_load(open(f))
    print(f, 'OK')
"
```
Expected: `OK` printed for all 8 files, no exceptions.

- [ ] **Step 10: Confirm no `pull_request` trigger remains in any `ci-pr-*.yml`**

Run: `grep -l "pull_request" .github/workflows/ci-pr-*.yml`
Expected: no output (no matches).

- [ ] **Step 11: Commit**

```bash
git add .github/workflows/ci-pr-api-gateway.yml \
        .github/workflows/ci-pr-auth-service.yml \
        .github/workflows/ci-pr-drug-catalog.yml \
        .github/workflows/ci-pr-inventory-service.yml \
        .github/workflows/ci-pr-manufacturing-service.yml \
        .github/workflows/ci-pr-supplier-service.yml \
        .github/workflows/ci-pr-qc-service.yml \
        .github/workflows/ci-pr-notification.yml
git commit -m "fix(ci): remove duplicate pull_request trigger from PR-check workflows

Each ci-pr-*.yml fired on both push to feat-*/fix-*/chore-* and
pull_request into develop, running the identical check twice per
commit while a PR is open. GitHub attaches check runs to the commit
SHA, so the push-triggered run still shows on the PR without a
second execution."
```

---

### Task 2: Update `CI-ARCHITECTURE.md` trigger documentation

**Files:**
- Modify: `CI-ARCHITECTURE.md:156-163` (Branch → Workflow Trigger Matrix)
- Modify: `CI-ARCHITECTURE.md:173-186` (Tier 1 section of the Three-Tier Promotion Model diagram)

- [ ] **Step 1: Replace the Branch → Workflow Trigger Matrix table**

Before (`CI-ARCHITECTURE.md:156-163`):
```markdown
## Branch → Workflow Trigger Matrix

| Event | Branches | Workflow triggered | What runs |
|---|---|---|---|
| `push` | `feat-*`, `fix-*`, `chore-*` | `ci-pr-<service>.yml` | Lint · Test · CodeQL · Semgrep · OWASP |
| `pull_request` | target: `develop` | `ci-pr-<service>.yml` | Same as above |
| `push` | `develop`, `release/**` | `ci-<service>.yml` | Full build + ECR + DEV deploy + open QA PR |
| `workflow_dispatch` | any | `promote-prod.yml` | Read QA tag → open PROD PR |
```

After:
```markdown
## Branch → Workflow Trigger Matrix

| Event | Branches | Workflow triggered | What runs |
|---|---|---|---|
| `push` | `feat-*`, `fix-*`, `chore-*` | `ci-pr-<service>.yml` | Lint · Test · CodeQL · Semgrep · OWASP |
| `push` | `develop`, `release/**` | `ci-<service>.yml` | Full build + ECR + DEV deploy + open QA PR |
| `workflow_dispatch` | any | `promote-prod.yml` | Read QA tag → open PROD PR |

> **Why is there no `pull_request` trigger for the PR-check workflow?**
> `ci-pr-<service>.yml` used to also trigger on `pull_request` into `develop`, in addition to `push` on `feat-*/fix-*/chore-*`. Since a student's PR is opened from a branch inside their own repo (not a cross-repo fork-to-upstream PR), both triggers fired for the same commit SHA — every push while a PR was open ran the identical check twice. GitHub Actions attaches check runs to the commit SHA, not to the event that triggered them, so removing the `pull_request` trigger loses no visibility: the push-triggered check still shows up on the PR automatically.
```

- [ ] **Step 2: Update the Tier 1 diagram trigger line**

Before (`CI-ARCHITECTURE.md:173-186`, showing only the changed line in context):
```
┌────────────────────────────────────────────────────────────────────────┐
│  TIER 1 — Feature Branch (ci-pr-*.yml)                                 │
│                                                                        │
│  Trigger: push to feat-* / fix-* / chore-*,  or PR → develop          │
│  Goal:    fast feedback to developer, no side-effects                  │
│                                                                        │
│  Maven verify (+ Postgres if needed)                                   │
│  → CodeQL  → Semgrep  → OWASP Dependency Check                        │
│                                                                        │
│  No Docker build.  No ECR push.  No GitOps update.  (~5 min)          │
└────────────────────────────────────────────────────────────────────────┘
```

After:
```
┌────────────────────────────────────────────────────────────────────────┐
│  TIER 1 — Feature Branch (ci-pr-*.yml)                                 │
│                                                                        │
│  Trigger: push to feat-* / fix-* / chore-*                            │
│           (check appears automatically on any PR from that branch)     │
│  Goal:    fast feedback to developer, no side-effects                  │
│                                                                        │
│  Maven verify (+ Postgres if needed)                                   │
│  → CodeQL  → Semgrep  → OWASP Dependency Check                        │
│                                                                        │
│  No Docker build.  No ECR push.  No GitOps update.  (~5 min)          │
└────────────────────────────────────────────────────────────────────────┘
```

- [ ] **Step 3: Verify no other reference to the removed `pull_request` trigger remains**

Run: `grep -n "pull_request" CI-ARCHITECTURE.md`
Expected: no output (no matches) — if any remain from other sections describing the old behavior, review and update them consistently with this change.

- [ ] **Step 4: Commit**

```bash
git add CI-ARCHITECTURE.md
git commit -m "docs: reflect single-trigger PR-check workflow in CI-ARCHITECTURE.md

Removes the pull_request row from the trigger matrix and updates the
Tier 1 diagram now that ci-pr-*.yml only triggers on push."
```

---

### Task 3: Manual end-to-end verification (run by user, not automated)

This step requires pushing to a real branch and opening a PR against a live GitHub repo — do not run this automatically; hand off to the user to confirm in their own environment.

- [ ] **Step 1: Pick one service to verify (e.g. auth-service)**

- [ ] **Step 2: Push a trivial commit to a `feat-*` branch touching `auth-service/`**

Confirm exactly one workflow run appears in the Actions tab for `ci-pr-auth-service.yml`.

- [ ] **Step 3: Open a PR from that branch into `develop`**

Confirm the PR's checks tab shows the existing run from Step 2 — no new run should start.

- [ ] **Step 4: Push an additional commit to the same branch while the PR is open**

Confirm exactly one new workflow run starts (not two).

- [ ] **Step 5: Report back**

Confirm to the team that the duplicate run is gone before closing out this change.
