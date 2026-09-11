# SAST Tool Deduplication Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Remove the redundant Semgrep SAST step from all 4 reusable CI workflows, keeping CodeQL as the sole SAST tool, and update `CI-ARCHITECTURE.md` so it accurately documents the single-tool pipeline and the coverage trade-off.

**Architecture:** `_java-build.yml`, `_java-pr-check.yml`, `_node-build.yml`, and `_node-pr-check.yml` each currently run CodeQL then Semgrep as two separate, overlapping SAST steps, with Semgrep gated behind an optional `SEMGREP_APP_TOKEN` secret. Removing the Semgrep step, its secret declaration, and its header-comment stage listing from each file leaves CodeQL as the only SAST tool — no other stage (tests, coverage, OWASP, npm audit, Docker build, Trivy, ECR push, Cosign) changes. `CI-ARCHITECTURE.md` then needs every Semgrep/SEMGREP reference either removed or reworded to match, including 4 ASCII-diagram box lines where replacement text must be padded to preserve column alignment.

**Tech Stack:** GitHub Actions YAML, Markdown documentation. No application code changes.

---

## File Structure

Files modified (no files created):
- `.github/workflows/_java-build.yml`
- `.github/workflows/_java-pr-check.yml`
- `.github/workflows/_node-build.yml`
- `.github/workflows/_node-pr-check.yml`
- `CI-ARCHITECTURE.md`

---

### Task 1: Remove Semgrep from `_java-build.yml`

**Files:**
- Modify: `.github/workflows/_java-build.yml`

- [ ] **Step 1: Update the header comment stage list**

Before (lines 1-14):
```yaml
# Reusable — Java Build + SAST + Container Security + ECR Push + Cosign Sign
#
# Called by every Java/Spring Boot per-service ci-*.yml workflow.
# Pipeline stages (all in a single job, sequential):
#   1. Maven verify  — unit/integration tests + JaCoCo coverage gate (≥ 80%)
#   2. CodeQL        — Java security-extended queries (instruments Maven build)
#   3. Semgrep       — java · owasp-top-ten · spring-boot rules
#   4. OWASP Dep Chk — CVSS ≥ 7.0 reported (non-blocking)
#   5. Docker build  — multi-stage, non-root UID/GID 1000
#   6. Trivy         — image scan, fail on HIGH/CRITICAL CVEs
#   7. ECR push      — tagged as sha-<7chars>
#   8. Cosign sign   — keyless, GitHub OIDC → Fulcio CA → Rekor transparency log
#
# Outputs: image-tag (sha-<7chars>), registry (ECR registry URL)
```

After:
```yaml
# Reusable — Java Build + SAST + Container Security + ECR Push + Cosign Sign
#
# Called by every Java/Spring Boot per-service ci-*.yml workflow.
# Pipeline stages (all in a single job, sequential):
#   1. Maven verify  — unit/integration tests + JaCoCo coverage gate (≥ 80%)
#   2. CodeQL        — Java security-extended queries (instruments Maven build)
#   3. OWASP Dep Chk — CVSS ≥ 7.0 reported (non-blocking)
#   4. Docker build  — multi-stage, non-root UID/GID 1000
#   5. Trivy         — image scan, fail on HIGH/CRITICAL CVEs
#   6. ECR push      — tagged as sha-<7chars>
#   7. Cosign sign   — keyless, GitHub OIDC → Fulcio CA → Rekor transparency log
#
# Outputs: image-tag (sha-<7chars>), registry (ECR registry URL)
```

- [ ] **Step 2: Remove the `SEMGREP_APP_TOKEN` secret declaration**

Before:
```yaml
    secrets:
      AWS_ACCOUNT_ID:
        required: true
      SEMGREP_APP_TOKEN:
        required: false
      NVD_API_KEY:
        description: "NIST NVD API key — higher rate limits + faster OWASP NVD updates"
        required: false
```

After:
```yaml
    secrets:
      AWS_ACCOUNT_ID:
        required: true
      NVD_API_KEY:
        description: "NIST NVD API key — higher rate limits + faster OWASP NVD updates"
        required: false
```

- [ ] **Step 3: Remove the Semgrep SAST step**

Before:
```yaml
      # ── Semgrep SAST ──────────────────────────────────────────────────────────
      - name: Semgrep SAST (java · owasp-top-ten)
        uses: semgrep/semgrep-action@v1
        continue-on-error: true
        with:
          config: >-
            p/java
            p/owasp-top-ten
        env:
          SEMGREP_APP_TOKEN: ${{ secrets.SEMGREP_APP_TOKEN }}

      # ── OWASP Dependency Check ────────────────────────────────────────────────
```

After:
```yaml
      # ── OWASP Dependency Check ────────────────────────────────────────────────
```

(This removes the Semgrep step and its preceding blank line, leaving the blank line that already follows the CodeQL analyze step directly before the OWASP comment.)

- [ ] **Step 4: Validate YAML syntax**

Run:
```bash
python3 -c "import yaml; yaml.safe_load(open('.github/workflows/_java-build.yml')); print('OK')"
```
Expected: `OK`

- [ ] **Step 5: Confirm no Semgrep reference remains**

Run: `grep -n "Semgrep\|SEMGREP" .github/workflows/_java-build.yml`
Expected: no output.

- [ ] **Step 6: Commit**

```bash
git add .github/workflows/_java-build.yml
git commit -m "fix(ci): remove duplicate Semgrep SAST step from _java-build.yml

CodeQL and Semgrep ran as two overlapping, both-advisory SAST scans.
Keeping CodeQL only removes a vendor secret (SEMGREP_APP_TOKEN) and
consolidates SAST findings into the same Security tab Trivy uses."
```

---

### Task 2: Remove Semgrep from `_java-pr-check.yml`

**Files:**
- Modify: `.github/workflows/_java-pr-check.yml`

- [ ] **Step 1: Update the header comment stage list and fix the stale "and PRs" wording**

Before (lines 1-11):
```yaml
# Reusable — Lightweight Java PR / Feature Branch Check
#
# Used by ci-pr-<service>.yml workflows on feat-*, fix-*, chore-* branches and PRs.
# Intentionally stops before Docker — no image build, no ECR push, no Cosign.
# Goal: fast feedback (~5 min) for developers without burning ECR storage or runner time.
#
# Stages:
#   1. Maven verify  — unit/integration tests + JaCoCo coverage
#   2. CodeQL        — Java security-extended SAST
#   3. Semgrep       — java · owasp-top-ten · spring-boot rules
#   4. OWASP Dep Chk — CVSS ≥ 7.0 reported (non-blocking; add NVD_API_KEY for faster NVD sync)
```

After:
```yaml
# Reusable — Lightweight Java PR / Feature Branch Check
#
# Used by ci-pr-<service>.yml workflows on feat-*, fix-*, chore-* branches.
# Intentionally stops before Docker — no image build, no ECR push, no Cosign.
# Goal: fast feedback (~5 min) for developers without burning ECR storage or runner time.
#
# Stages:
#   1. Maven verify  — unit/integration tests + JaCoCo coverage
#   2. CodeQL        — Java security-extended SAST
#   3. OWASP Dep Chk — CVSS ≥ 7.0 reported (non-blocking; add NVD_API_KEY for faster NVD sync)
```

- [ ] **Step 2: Remove the `SEMGREP_APP_TOKEN` secret declaration**

Before:
```yaml
    secrets:
      SEMGREP_APP_TOKEN:
        required: false
      NVD_API_KEY:
        description: "NIST NVD API key — higher rate limits + faster OWASP NVD updates"
        required: false
```

After:
```yaml
    secrets:
      NVD_API_KEY:
        description: "NIST NVD API key — higher rate limits + faster OWASP NVD updates"
        required: false
```

- [ ] **Step 3: Remove the Semgrep SAST step**

Before:
```yaml
      - name: CodeQL — Analyze
        uses: github/codeql-action/analyze@v3
        with:
          category: /language:java

      - name: Semgrep SAST (java · owasp-top-ten)
        uses: semgrep/semgrep-action@v1
        continue-on-error: true
        with:
          config: >-
            p/java
            p/owasp-top-ten
        env:
          SEMGREP_APP_TOKEN: ${{ secrets.SEMGREP_APP_TOKEN }}

      # ── OWASP Dependency Check ────────────────────────────────────────────────
```

After:
```yaml
      - name: CodeQL — Analyze
        uses: github/codeql-action/analyze@v3
        with:
          category: /language:java

      # ── OWASP Dependency Check ────────────────────────────────────────────────
```

- [ ] **Step 4: Validate YAML syntax**

Run:
```bash
python3 -c "import yaml; yaml.safe_load(open('.github/workflows/_java-pr-check.yml')); print('OK')"
```
Expected: `OK`

- [ ] **Step 5: Confirm no Semgrep reference remains**

Run: `grep -n "Semgrep\|SEMGREP" .github/workflows/_java-pr-check.yml`
Expected: no output.

- [ ] **Step 6: Commit**

```bash
git add .github/workflows/_java-pr-check.yml
git commit -m "fix(ci): remove duplicate Semgrep SAST step from _java-pr-check.yml

Also fixes a stale header comment claiming this runs on PR events —
the pull_request trigger was already removed from the calling
ci-pr-*.yml workflows in a prior change."
```

---

### Task 3: Remove Semgrep from `_node-build.yml`

**Files:**
- Modify: `.github/workflows/_node-build.yml`

- [ ] **Step 1: Update the header comment stage list**

Before (lines 1-16):
```yaml
# Reusable — Node.js Build + SAST + Container Security + ECR Push + Cosign Sign
#
# Called by the notification-service ci-notification.yml workflow.
# Pipeline stages (all in a single job, sequential):
#   1. npm ci + ESLint      — install dependencies + lint (--max-warnings 0)
#   2. npm test             — unit tests with Jest + coverage gate (≥ 80%)
#   3. CodeQL               — JavaScript/TypeScript security-extended queries
#   4. Semgrep              — javascript · nodejs · owasp-top-ten rules
#   5. npm audit            — fail if any HIGH/CRITICAL vulnerabilities found
#   6. Docker build         — multi-stage, non-root UID/GID 1000
#   7. Trivy                — image scan, fail on HIGH/CRITICAL CVEs
#   8. ECR push             — tagged as sha-<7chars>
#   9. Cosign sign          — keyless, GitHub OIDC → Fulcio CA → Rekor transparency log
#
# Outputs: image-tag (sha-<7chars>), registry (ECR registry URL)
```

After:
```yaml
# Reusable — Node.js Build + SAST + Container Security + ECR Push + Cosign Sign
#
# Called by the notification-service ci-notification.yml workflow.
# Pipeline stages (all in a single job, sequential):
#   1. npm ci + ESLint      — install dependencies + lint (--max-warnings 0)
#   2. npm test             — unit tests with Jest + coverage gate (≥ 80%)
#   3. CodeQL               — JavaScript/TypeScript security-extended queries
#   4. npm audit            — fail if any HIGH/CRITICAL vulnerabilities found
#   5. Docker build         — multi-stage, non-root UID/GID 1000
#   6. Trivy                — image scan, fail on HIGH/CRITICAL CVEs
#   7. ECR push             — tagged as sha-<7chars>
#   8. Cosign sign          — keyless, GitHub OIDC → Fulcio CA → Rekor transparency log
#
# Outputs: image-tag (sha-<7chars>), registry (ECR registry URL)
```

- [ ] **Step 2: Remove the `SEMGREP_APP_TOKEN` secret declaration**

Before:
```yaml
    secrets:
      AWS_ACCOUNT_ID:
        required: true
      SEMGREP_APP_TOKEN:
        required: false
```

After:
```yaml
    secrets:
      AWS_ACCOUNT_ID:
        required: true
```

- [ ] **Step 3: Remove the Semgrep SAST step**

Before:
```yaml
      # ── Semgrep SAST ──────────────────────────────────────────────────────────
      - name: Semgrep SAST (javascript · nodejs · owasp-top-ten)
        uses: semgrep/semgrep-action@v1
        continue-on-error: true
        with:
          config: >-
            p/javascript
            p/nodejs
            p/owasp-top-ten
        env:
          SEMGREP_APP_TOKEN: ${{ secrets.SEMGREP_APP_TOKEN }}

      # ── npm audit (fail on HIGH/CRITICAL) ─────────────────────────────────────
```

After:
```yaml
      # ── npm audit (fail on HIGH/CRITICAL) ─────────────────────────────────────
```

- [ ] **Step 4: Validate YAML syntax**

Run:
```bash
python3 -c "import yaml; yaml.safe_load(open('.github/workflows/_node-build.yml')); print('OK')"
```
Expected: `OK`

- [ ] **Step 5: Confirm no Semgrep reference remains**

Run: `grep -n "Semgrep\|SEMGREP" .github/workflows/_node-build.yml`
Expected: no output.

- [ ] **Step 6: Commit**

```bash
git add .github/workflows/_node-build.yml
git commit -m "fix(ci): remove duplicate Semgrep SAST step from _node-build.yml"
```

---

### Task 4: Remove Semgrep from `_node-pr-check.yml`

**Files:**
- Modify: `.github/workflows/_node-pr-check.yml`

- [ ] **Step 1: Update the header comment stage list and fix the stale "and PRs" wording**

Before (lines 1-11):
```yaml
# Reusable — Lightweight Node.js PR / Feature Branch Check
#
# Used by ci-pr-notification.yml on feat-*, fix-*, chore-* branches and PRs.
# No Docker, no ECR, no Cosign. Fast feedback for developers.
#
# Stages:
#   1. ESLint        — lint (--max-warnings 0)
#   2. Jest          — unit tests + coverage
#   3. CodeQL        — JavaScript security-extended SAST
#   4. Semgrep       — javascript · nodejs · owasp-top-ten rules
#   5. npm audit     — fail on HIGH/CRITICAL vulnerabilities
```

After:
```yaml
# Reusable — Lightweight Node.js PR / Feature Branch Check
#
# Used by ci-pr-notification.yml on feat-*, fix-*, chore-* branches.
# No Docker, no ECR, no Cosign. Fast feedback for developers.
#
# Stages:
#   1. ESLint        — lint (--max-warnings 0)
#   2. Jest          — unit tests + coverage
#   3. CodeQL        — JavaScript security-extended SAST
#   4. npm audit     — fail on HIGH/CRITICAL vulnerabilities
```

- [ ] **Step 2: Remove the `secrets:` block entirely**

`SEMGREP_APP_TOKEN` is the only secret this workflow declares. Removing the Semgrep step (Step 3) removes its only use, so the whole block goes.

Before:
```yaml
      node-version:
        required: false
        type: string
        default: '20'
    secrets:
      SEMGREP_APP_TOKEN:
        required: false

jobs:
```

After:
```yaml
      node-version:
        required: false
        type: string
        default: '20'

jobs:
```

- [ ] **Step 3: Remove the Semgrep SAST step**

Before:
```yaml
      - name: Semgrep SAST (javascript · nodejs · owasp-top-ten)
        uses: semgrep/semgrep-action@v1
        continue-on-error: true
        with:
          config: >-
            p/javascript
            p/nodejs
            p/owasp-top-ten
        env:
          SEMGREP_APP_TOKEN: ${{ secrets.SEMGREP_APP_TOKEN }}

      - name: npm audit (HIGH/CRITICAL → fail)
```

After:
```yaml
      - name: npm audit (HIGH/CRITICAL → fail)
```

- [ ] **Step 4: Validate YAML syntax**

Run:
```bash
python3 -c "import yaml; yaml.safe_load(open('.github/workflows/_node-pr-check.yml')); print('OK')"
```
Expected: `OK`

- [ ] **Step 5: Confirm no Semgrep reference remains**

Run: `grep -n "Semgrep\|SEMGREP" .github/workflows/_node-pr-check.yml`
Expected: no output.

- [ ] **Step 6: Confirm the caller workflow still works with no declared secrets**

Run: `grep -n "secrets" .github/workflows/ci-pr-notification.yml`
Expected: shows `secrets: inherit` — this remains valid even though `_node-pr-check.yml` no longer declares any `secrets:` schema (GitHub Actions allows `secrets: inherit` regardless of whether the called workflow declares secrets).

- [ ] **Step 7: Commit**

```bash
git add .github/workflows/_node-pr-check.yml
git commit -m "fix(ci): remove duplicate Semgrep SAST step from _node-pr-check.yml

Also fixes a stale header comment claiming this runs on PR events —
the pull_request trigger was already removed from ci-pr-notification.yml
in a prior change."
```

---

### Task 5: Update `CI-ARCHITECTURE.md`

**Files:**
- Modify: `CI-ARCHITECTURE.md`

- [ ] **Step 1: Fix the "End-to-End Flow" ASCII box for the PR-check pipeline**

Before:
```
    │                           │ │ CodeQL · Semgrep · OWASP│   │                         │
```

After:
```
    │                           │ │ CodeQL · OWASP          │   │                         │
```

(Padding computed to keep the box border `│` characters aligned — verify with Step 9 below.)

- [ ] **Step 2: Fix the "End-to-End Flow" ASCII box for the full-build pipeline**

Before:
```
    │                           │ │  CodeQL · Semgrep · OWASP│   │                         │
```

After:
```
    │                           │ │  CodeQL · OWASP          │   │                         │
```

- [ ] **Step 3: Reword the "Workflow File Map" tree entries**

Before:
```
        ├── _java-build.yml          ← Full build: Maven + CodeQL + Semgrep +
        │                                          OWASP + Trivy + ECR + Cosign
        ├── _node-build.yml          ← Full build: npm + CodeQL + Semgrep +
        │                                          audit + Trivy + ECR + Cosign
        ├── _java-pr-check.yml       ← Lightweight: Maven + CodeQL + Semgrep +
        │                                           OWASP  (no Docker, no ECR)
        ├── _node-pr-check.yml       ← Lightweight: npm + CodeQL + Semgrep +
        │                                           audit  (no Docker, no ECR)
```

After:
```
        ├── _java-build.yml          ← Full build: Maven + CodeQL + OWASP +
        │                                          Trivy + ECR + Cosign
        ├── _node-build.yml          ← Full build: npm + CodeQL + audit +
        │                                          Trivy + ECR + Cosign
        ├── _java-pr-check.yml       ← Lightweight: Maven + CodeQL + OWASP
        │                                           (no Docker, no ECR)
        ├── _node-pr-check.yml       ← Lightweight: npm + CodeQL + audit
        │                                           (no Docker, no ECR)
```

- [ ] **Step 4: Fix the "Branch → Workflow Trigger Matrix" table cell**

Before:
```
| `push` | `feat-*`, `fix-*`, `chore-*` | `ci-pr-<service>.yml` | Lint · Test · CodeQL · Semgrep · OWASP |
```

After:
```
| `push` | `feat-*`, `fix-*`, `chore-*` | `ci-pr-<service>.yml` | Lint · Test · CodeQL · OWASP |
```

- [ ] **Step 5: Fix the "Three-Tier Promotion Model" Tier 1 box**

Before:
```
│  → CodeQL  → Semgrep  → OWASP Dependency Check                        │
```

After:
```
│  → CodeQL  → OWASP Dependency Check                                   │
```

- [ ] **Step 6: Fix the "Three-Tier Promotion Model" Tier 2 box**

Before:
```
│    CodeQL · Semgrep · OWASP Dep Check                                  │
```

After:
```
│    CodeQL · OWASP Dep Check                                            │
```

- [ ] **Step 7: Remove the Semgrep row from the "Implemented in zen-pharma-backend" table**

Before:
```
| SAST | **CodeQL** (`security-extended`) | `_java-pr-check.yml`, `_java-build.yml`, `_node-pr-check.yml`, `_node-build.yml` | SARIF uploaded to GitHub **Code scanning** when org/repo settings allow |
| SAST | **Semgrep** (`p/java`, `p/spring-boot`, `p/javascript`, `p/nodejs`, `p/owasp-top-ten`, etc.) | Same reusable workflows | Optional `SEMGREP_APP_TOKEN` for Semgrep Cloud dashboards |
```

After:
```
| SAST | **CodeQL** (`security-extended`) | `_java-pr-check.yml`, `_java-build.yml`, `_node-pr-check.yml`, `_node-build.yml` | SARIF uploaded to GitHub **Code scanning** when org/repo settings allow |
```

(Do not touch the "Industry reference" table above this one — it lists Semgrep as a general industry example and stays unchanged.)

- [ ] **Step 8: Rewrite the "Stage 2 — SAST" section**

Before (from the section heading through the end of the two blockquotes that follow the Semgrep subsection):
```markdown
### Stage 2 — SAST (Static Application Security Testing)

Both SAST tools run on every push — feature branch checks and full builds alike. Finding a security issue on a feature branch is far cheaper than finding it after it's merged.

#### CodeQL

| | Detail |
|---|---|
| Tool | GitHub CodeQL (`github/codeql-action`) |
| Query suite | `security-extended` — GitHub's high-confidence security rules |
| How it works | Instruments the Maven/npm **compilation** to build a complete call graph and data flow model, then queries it for vulnerabilities spanning multiple files and method calls |
| What it catches | SQL injection flowing through 3 service layers, path traversal assembled across methods, XSS sinks reached via indirect calls, insecure deserialization |
| What it misses | Framework-specific misconfigurations — Spring Boot CSRF disabled, actuator endpoints exposed — these are config patterns, not data flow issues |
| Results | Uploaded as SARIF to the GitHub Security tab |
| Fail condition | Any finding from `security-extended` queries |

> **Why must CodeQL initialize BEFORE the Maven build step?**  
> CodeQL instruments the compilation process to collect call graph and type information. If you run CodeQL after Maven has already compiled, the instrumentation hooks were never in place and CodeQL produces a much weaker analysis — or fails entirely. The `codeql-action/init` step must run before `mvn verify`.

#### Semgrep

| | Detail |
|---|---|
| Tool | Semgrep (`semgrep/semgrep-action`) |
| Rule sets | `p/java` + `p/owasp-top-ten` + `p/spring-boot` (Java) / `p/nodejs` + `p/owasp-top-ten` (Node) |
| How it works | Pattern matching on the Abstract Syntax Tree — fast, no compilation needed |
| What it catches | Spring Boot CSRF disabled, actuator endpoints without auth, missing `@PreAuthorize`, hardcoded secrets, CORS wildcard, insecure HTTP configs, OWASP Top 10 anti-patterns |
| What it misses | Complex multi-step data flow vulnerabilities — it matches patterns, not how data flows across method boundaries |
| Results | Build log + Semgrep cloud dashboard (if `SEMGREP_APP_TOKEN` configured) |
| Fail condition | Any rule violation |

> **Why two SAST tools? Is that overkill?**  
> They cover genuinely different surfaces with minimal overlap. CodeQL builds a full semantic model of your code and finds complex vulnerabilities that span multiple files — a SQL injection that's assembled across three service layers. Semgrep matches patterns on the AST and excels at framework-specific misconfigurations — things like Spring Boot having CSRF protection disabled or an actuator endpoint exposed without authentication. Neither tool catches what the other specialises in. For a pharma application with compliance requirements, both are warranted. If you want to simplify, drop Semgrep and keep CodeQL — it is the more thorough of the two.

> **Why `p/spring-boot` specifically?**  
> Spring Boot has many security footguns that are not obvious: actuator endpoints enabled by default, CSRF disabled in REST API configurations, permissive CORS configs, missing method-level security annotations. The `p/spring-boot` ruleset is maintained by the Semgrep community specifically for these patterns. Generic Java rules would miss them.
```

After:
```markdown
### Stage 2 — SAST (Static Application Security Testing)

SAST runs on every push — feature branch checks and full builds alike. Finding a security issue on a feature branch is far cheaper than finding it after it's merged.

#### CodeQL

| | Detail |
|---|---|
| Tool | GitHub CodeQL (`github/codeql-action`) |
| Query suite | `security-extended` — GitHub's high-confidence security rules |
| How it works | Instruments the Maven/npm **compilation** to build a complete call graph and data flow model, then queries it for vulnerabilities spanning multiple files and method calls |
| What it catches | SQL injection flowing through 3 service layers, path traversal assembled across methods, XSS sinks reached via indirect calls, insecure deserialization |
| What it misses | Framework-specific misconfigurations — Spring Boot CSRF disabled, actuator endpoints exposed — these are config patterns, not data flow issues |
| Results | Uploaded as SARIF to the GitHub Security tab |
| Fail condition | None — findings are advisory. They're uploaded to the Security tab for visibility but do not fail the job |

> **Why must CodeQL initialize BEFORE the Maven build step?**  
> CodeQL instruments the compilation process to collect call graph and type information. If you run CodeQL after Maven has already compiled, the instrumentation hooks were never in place and CodeQL produces a much weaker analysis — or fails entirely. The `codeql-action/init` step must run before `mvn verify`.

> **Why CodeQL only, not both CodeQL and Semgrep?**  
> CodeQL and Semgrep genuinely covered different surfaces — CodeQL's semantic data-flow model catches multi-file vulnerabilities like SQL injection assembled across service layers; Semgrep's pattern matching caught framework-specific misconfigurations like Spring Boot CSRF disabled or an exposed actuator endpoint. Keeping only CodeQL means one less external vendor, one less optional secret (`SEMGREP_APP_TOKEN`), and every SAST finding lands in the same GitHub Security tab that Trivy's container scan already uses — one consistent scanning pattern instead of two. The trade-off: Semgrep's Spring Boot misconfiguration rules are no longer checked by this pipeline. For a training curriculum, the reduction in tool count and mental overhead outweighs that narrower slice of coverage — teams with stricter compliance requirements should keep both.
```

- [ ] **Step 9: Remove the Semgrep row from the "Summary — What Runs Where" table**

Before:
```
Unit tests                     ✓                ✓
Code coverage (JaCoCo/Jest)    ✓                ✓
CodeQL SAST                    ✓                ✓
Semgrep SAST                   ✓                ✓
OWASP Dependency Check (Java)  ✓                ✓
```

After:
```
Unit tests                     ✓                ✓
Code coverage (JaCoCo/Jest)    ✓                ✓
CodeQL SAST                    ✓                ✓
OWASP Dependency Check (Java)  ✓                ✓
```

- [ ] **Step 10: Remove the `SEMGREP_APP_TOKEN` row from "GitHub Secrets — Required Setup"**

Before:
```
| `AWS_ACCOUNT_ID` | all `ci-*.yml` | 12-digit AWS account ID for ECR URL construction |
| `GITOPS_TOKEN` | all `ci-*.yml`, `promote-prod.yml` | GitHub PAT or App token with `contents: write` on `chandika-s/zen-gitops` |
| `SEMGREP_APP_TOKEN` | `_java-build.yml`, `_node-build.yml` | Semgrep cloud token (optional — OSS rules work without it) |
| `NVD_API_KEY` | `_java-pr-check.yml`, `_java-build.yml` (when OWASP enabled) | NIST NVD API key — higher rate limits for OWASP Dependency Check (optional but recommended) |
```

After:
```
| `AWS_ACCOUNT_ID` | all `ci-*.yml` | 12-digit AWS account ID for ECR URL construction |
| `GITOPS_TOKEN` | all `ci-*.yml`, `promote-prod.yml` | GitHub PAT or App token with `contents: write` on `chandika-s/zen-gitops` |
| `NVD_API_KEY` | `_java-pr-check.yml`, `_java-build.yml` (when OWASP enabled) | NIST NVD API key — higher rate limits for OWASP Dependency Check (optional but recommended) |
```

- [ ] **Step 11: Remove the `SEMGREP_APP_TOKEN` row from the "AWS OIDC + ECR Setup" Step 6 table**

Before:
```
| Secret | Value | Required |
|--------|-------|----------|
| `AWS_ACCOUNT_ID` | Your 12-digit AWS account ID | Yes |
| `GITOPS_TOKEN` | GitHub PAT with `repo` scope on `chandika-s/zen-gitops` | Yes |
| `SEMGREP_APP_TOKEN` | Token from semgrep.dev (for dashboard integration) | Optional |
```

After:
```
| Secret | Value | Required |
|--------|-------|----------|
| `AWS_ACCOUNT_ID` | Your 12-digit AWS account ID | Yes |
| `GITOPS_TOKEN` | GitHub PAT with `repo` scope on `chandika-s/zen-gitops` | Yes |
```

- [ ] **Step 12: Verify no unintended Semgrep references remain**

Run: `grep -n "Semgrep\|SEMGREP" CI-ARCHITECTURE.md`
Expected: exactly 2 matches — the "Industry reference" table row (`SAST (static application security) | CodeQL, Semgrep, Checkmarx, ...`), and the new "Why CodeQL only, not both CodeQL and Semgrep?" blockquote from Step 8. If anything else matches, find and fix it before proceeding.

- [ ] **Step 13: Verify ASCII box alignment**

Run:
```bash
python3 -c "
with open('CI-ARCHITECTURE.md', encoding='utf-8') as f:
    lines = f.readlines()
# End-to-End Flow box: check the 5 lines of the PR-check box all have equal length
box1 = [lines[i] for i in range(38, 43)]  # 0-indexed lines 39-43
lens1 = set(len(l.rstrip(chr(10))) for l in box1)
print('PR-check box lengths:', lens1)
box2 = [lines[i] for i in range(50, 58)]  # 0-indexed lines 51-58
lens2 = set(len(l.rstrip(chr(10))) for l in box2)
print('Full-build box lengths:', lens2)
"
```
Expected: each printed set contains exactly one length value (all lines in that box are the same length — confirms the `│` borders still align). If a set has more than one value, find the mismatched line and fix its padding.

Also visually check the Tier 1 and Tier 2 boxes in "Three-Tier Promotion Model" render with `│` characters in a straight vertical line — read the file back and eyeball columns 0 and 74.

- [ ] **Step 14: Commit**

```bash
git add CI-ARCHITECTURE.md
git commit -m "docs: update CI-ARCHITECTURE.md for CodeQL-only SAST

Reflects removal of the duplicate Semgrep step across all 4 reusable
workflows: trigger matrix, ASCII diagrams, workflow file map, the
security tooling tables, Stage 2 section (with the coverage trade-off
now documented explicitly), summary table, and secrets setup tables.
Also corrects the CodeQL 'Fail condition' row, which incorrectly
claimed findings block the build."
```

---

### Task 6: Manual end-to-end verification (run by user, not automated)

This step requires triggering a real workflow run on a live GitHub repo — do not run this automatically; hand off to the user.

- [ ] **Step 1: Push a trivial commit to a `feat-*` branch touching a Java service (e.g. auth-service) and the Node service (notification-service)**

- [ ] **Step 2: In the Actions tab, open each resulting workflow run**

Confirm a CodeQL step still runs and completes. Confirm no step named "Semgrep SAST" appears anywhere in the run.

- [ ] **Step 3: Check the repo's Security tab → Code scanning alerts**

Confirm CodeQL findings (if any) still appear there as before.

- [ ] **Step 4: Confirm no workflow run fails due to a missing `SEMGREP_APP_TOKEN` secret**

(It should never have been required — this just confirms removing it didn't break anything.)

- [ ] **Step 5: Report back**

Confirm to the team that CodeQL-only SAST is working as expected before closing out this change.
