# CI PR-Check Trigger Deduplication — Design

## Problem

Batch 2 students find the CI pipeline heavy to digest. Part of the friction is a genuine defect, not just perceived complexity: every `ci-pr-<service>.yml` workflow (all 8 services: api-gateway, auth-service, drug-catalog, inventory-service, manufacturing-service, supplier-service, qc-service, notification) triggers on **both**:

- `push` to `feat-*`, `fix-*`, `chore-*` branches, and
- `pull_request` targeting `develop`

Because students work within their own self-contained fork (feat-branch → PR → develop, all in one repo they own), any commit pushed while a PR is open fires the identical PR-check pipeline twice for the same SHA — once from the push event, once from the PR sync event. Students see the same checks run twice and don't know why, and it burns double the Actions minutes.

`CI-ARCHITECTURE.md` (lines 156–163, "Branch → Workflow Trigger Matrix") currently documents this duplication as if it were two intentional, distinct rows — reinforcing the confusion rather than flagging it.

## Decision

Remove the `pull_request` trigger from all 8 `ci-pr-*.yml` files. Keep only the `push` trigger on `feat-*/fix-*/chore-*`.

**Why this is safe:** GitHub Actions associates check runs with a commit SHA, not with the event that triggered the run. Since every student's PR (feat-branch → develop) lives inside a single repo they own — not a cross-repo fork-to-upstream PR — the checks produced by the push-triggered workflow run automatically appear on the PR for that SHA. No visibility is lost; only the duplicate execution is removed.

**Out of scope for this pass:** `release/**` trigger on the full `ci-*.yml` files stays as-is (confirmed in active use). No changes to `_java-pr-check.yml`, `_node-pr-check.yml`, the full `ci-*.yml` build/deploy files, or `promote-prod.yml` — no pipeline stages, tools, or environments change. This is a pure trigger-config fix; the real-world pipeline content is fully preserved. SAST tool deduplication (CodeQL + Semgrep overlap) and SBOM generation are a separate, later spec.

## Changes

### 1. Workflow trigger files (8 files)

In each of:
- `.github/workflows/ci-pr-api-gateway.yml`
- `.github/workflows/ci-pr-auth-service.yml`
- `.github/workflows/ci-pr-drug-catalog.yml`
- `.github/workflows/ci-pr-inventory-service.yml`
- `.github/workflows/ci-pr-manufacturing-service.yml`
- `.github/workflows/ci-pr-supplier-service.yml`
- `.github/workflows/ci-pr-qc-service.yml`
- `.github/workflows/ci-pr-notification.yml`

Remove the `pull_request:` block from the `on:` section. Example (auth-service), before:

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

No other part of these files changes.

### 2. `CI-ARCHITECTURE.md` documentation updates

- **Branch → Workflow Trigger Matrix** (currently lines 156–163): remove the separate `pull_request` → `ci-pr-<service>.yml` row (it no longer exists as a trigger). Add a short note explaining that PR checks still show up on the PR automatically, because GitHub associates check runs with the commit SHA rather than the triggering event — this is a teaching point worth keeping, not just removing.
- **Tier 1 diagram** (in "Three-Tier Promotion Model," currently reads `Trigger: push to feat-* / fix-* / chore-*, or PR → develop`): update to reflect the single trigger, with a one-line callout that the same checks are visible on the PR.

## Testing / Verification

This is GitHub Actions trigger configuration — verified by:
1. YAML validity of each edited workflow file (`yamllint` or GitHub's own workflow parser via a test push).
2. Manual verification on one service (auth-service) end-to-end: push to a `feat-*` branch, confirm the PR-check workflow runs exactly once; open a PR from that branch to `develop`, confirm the existing check appears on the PR without a second run being kicked off; push an additional commit to the branch, confirm exactly one new run (not two).
3. Visual diff review of the other 7 files to confirm identical mechanical change.

## Non-goals

- Not changing coverage gates, SAST tools, dependency scanning, or any other pipeline stage.
- Not changing the `release/**` trigger or the full build/deploy pipeline.
- Not addressing SAST tool overlap (CodeQL + Semgrep) or missing SBOM generation — tracked as a follow-up spec.
