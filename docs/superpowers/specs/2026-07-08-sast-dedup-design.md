# SAST Tool Deduplication — Design

## Problem

As part of simplifying the CI pipeline for batch 2 students, the previous pass removed a duplicate PR-check trigger. This pass addresses a second source of complexity: every reusable build workflow runs **two** SAST tools — CodeQL and Semgrep — covering overlapping ground. Neither currently gates the pipeline (Semgrep has `continue-on-error: true`; CodeQL's `analyze` step only uploads SARIF to the Security tab, it doesn't fail the job), so today this is two advisory scanners, not one blocking + one informational.

## Decision

Keep CodeQL, drop Semgrep, across all four reusable workflows: `_java-build.yml`, `_java-pr-check.yml`, `_node-build.yml`, `_node-pr-check.yml`.

**Why CodeQL over Semgrep:** CodeQL is GitHub-native — no extra secret (`SEMGREP_APP_TOKEN` is removed entirely), and its findings land in the same GitHub Security tab that Trivy's container scan already uses, so students learn one security-scanning pattern (SARIF → Security tab) instead of two different vendor UIs.

**Acknowledged trade-off:** `CI-ARCHITECTURE.md` itself documents that CodeQL and Semgrep are not fully redundant — Semgrep's `p/spring-boot` ruleset catches framework misconfigurations (CSRF disabled, actuator endpoints exposed without auth, missing `@PreAuthorize`) that CodeQL's semantic/data-flow analysis does not. This is a conscious, documented trade-off for a training curriculum where reducing tool count matters more than that specific coverage — not an oversight. The updated doc must say so explicitly.

**Out of scope:** No change to what gates the pipeline. CodeQL keeps its current advisory-only behavior (uploads to Security tab, does not fail the job) — no new blocking logic is added in this pass.

## Changes

### 1. Reusable workflow files (4)

For each of `_java-build.yml`, `_java-pr-check.yml`, `_node-build.yml`, `_node-pr-check.yml`:
- Remove the Semgrep step entirely (including its `SEMGREP_APP_TOKEN` env reference).
- Remove the `SEMGREP_APP_TOKEN` entry from the `secrets:` block (for `_node-pr-check.yml`, this removes the entire `secrets:` block since it contained nothing else).
- Renumber the stage list in the header comment to remove the Semgrep line.

### 2. `CI-ARCHITECTURE.md`

Every one of the following needs updating (full inventory, grep-verified against the current file):
- 2 ASCII-diagram box lines in "How It Works — End-to-End Flow" (lines ~41, ~53) — content replaced with padding preserved so box borders stay aligned.
- "Workflow File Map" — 4 tree entries describing the reusable workflows, reworded to drop Semgrep.
- "Branch → Workflow Trigger Matrix" — one table cell ("What runs" column for the PR-check row).
- "Three-Tier Promotion Model" — 2 ASCII-diagram box lines (Tier 1, Tier 2), padding preserved.
- "Security tooling" — "Implemented in zen-pharma-backend" table: remove the Semgrep row. (The "Industry reference" table, which describes general industry practice, is unchanged.)
- "DevSecOps Pipeline — Every Stage Explained," Stage 2 section: update the intro sentence (was "Both SAST tools..."), remove the `#### Semgrep` subsection and its table, replace the "why two SAST tools" blockquote with one explaining the CodeQL-only decision and the acknowledged trade-off, remove the Semgrep-specific `p/spring-boot` blockquote.
- "Summary — What Runs Where" — remove the `Semgrep SAST` row.
- "GitHub Secrets — Required Setup" table — remove the `SEMGREP_APP_TOKEN` row.
- "AWS OIDC + ECR Setup" Step 6 secrets table — remove the `SEMGREP_APP_TOKEN` row.

### 3. Bonus fixes (confirmed in scope)

- `CI-ARCHITECTURE.md`'s CodeQL "Fail condition" row currently reads "Any finding from `security-extended` queries," which is inaccurate — CodeQL does not actually fail the job. Correct it to state findings are advisory (uploaded to Security tab, do not fail the build), consistent with the "leave gating out of scope" decision.
- `_java-pr-check.yml` and `_node-pr-check.yml` header comments still say "...on feat-\*, fix-\*, chore-\* branches **and PRs**" — stale since the prior session removed the `pull_request` trigger from the calling `ci-pr-*.yml` workflows. Same category of fix as the `ci-pr-api-gateway.yml` comment corrected in that prior pass.

## Testing / Verification

1. YAML validity of all 4 edited reusable workflow files.
2. `grep -rn "Semgrep\|SEMGREP" .github/workflows/` returns nothing after the change.
3. `grep -n "Semgrep\|SEMGREP" CI-ARCHITECTURE.md` returns nothing except the "Industry reference" table row (general industry context, intentionally unchanged) and the new explanatory blockquote referencing Semgrep by name as part of explaining the trade-off.
4. ASCII-diagram box lines verified character-length-equal to their bordering lines (alignment check), not just visually similar.
5. Manual verification (user-run, not automated): trigger one Java service and the Node service's PR-check pipeline, confirm CodeQL still runs and uploads to the Security tab, confirm no Semgrep step appears in the run.

## Non-goals

- Not adding new blocking/gating logic for CodeQL findings.
- Not touching OWASP Dependency Check, npm audit, Trivy, Cosign, or any other pipeline stage.
- Not adding SBOM generation (still a separate, later spec if pursued).
