# CI/CD Architecture — zen-pharma-backend

> **Stack**: GitHub Actions (CI + Promotion) · AWS ECR · ArgoCD (CD) · AWS EKS  
> **Pattern**: GitOps · Build Once Deploy Many · PR-based Promotion Gates · Supply-Chain Security  
> **Services**: 7 microservices (6 Java/Spring Boot + 1 Node.js)

---

## Design Philosophy

Before diving into the files, these are the principles every decision in this architecture is built on:

**1. Build Once, Deploy Many**  
A Docker image is built and pushed to ECR exactly once, tagged with the git commit SHA. That same image — same bytes, same digest — flows through DEV → QA → PROD. No rebuilding per environment. This guarantees that what QA tested is exactly what runs in PROD.

**2. GitOps — Deployment is a git commit**  
Kubernetes state is declared in `chandika-s/zen-gitops`, not controlled by imperative `kubectl apply` commands in CI. ArgoCD watches that repo and reconciles the cluster toward the declared state. This means every deployment has a git history, an author, a timestamp, and is rollback-able by reverting a commit.

**3. Separation of concerns**  
Application code lives in this repo. Deployment configuration lives in zen-gitops. CI only writes to zen-gitops — it never talks to Kubernetes directly. This means: developers can't accidentally deploy by pushing code, and infra changes don't trigger application rebuilds.

**4. Shift security left**  
Security checks run on feature branches before code is reviewed, not after it's already in develop. A developer gets SAST and dependency CVE results in ~5 minutes on their own branch — at the point where it's cheapest to fix.

**5. Human gates at the right places**  
DEV: fully automatic — fast feedback loop. QA: PR review in zen-gitops — QA team decides when to deploy. PROD: manual workflow_dispatch + Required Reviewers — intentional, audited, at a maintenance window.

---

## How It Works — End-to-End Flow

```
 Developer                GitHub Actions               zen-gitops (GitOps repo)         EKS
 ─────────                ──────────────               ────────────────────────         ───
    │                           │                               │                         │
    │  git push feat-*          │                               │                         │
    ├──────────────────────────►│                               │                         │
    │                           │ ci-pr-<service>.yml           │                         │
    │                           │ ┌─────────────────────────┐   │                         │
    │                           │ │ Lint · Test             │   │                         │
    │                           │ │ CodeQL · OWASP          │   │                         │
    │                           │ │ (no Docker, no ECR)     │   │                         │
    │                           │ └─────────────────────────┘   │                         │
    │  ✓ Fast feedback (~5 min) │                               │                         │
    │◄──────────────────────────│                               │                         │
    │                           │                               │                         │
    │  PR merged → develop      │                               │                         │
    ├──────────────────────────►│                               │                         │
    │                           │ ci-<service>.yml              │                         │
    │                           │ ┌─────────────────────────┐   │                         │
    │                           │ │ Job: build              │   │                         │
    │                           │ │  Maven/npm              │   │                         │
    │                           │ │  CodeQL · OWASP         │   │                         │
    │                           │ │  Docker build           │   │                         │
    │                           │ │  Trivy scan             │   │                         │
    │                           │ │  ECR push → sha-abc1234 │   │                         │
    │                           │ │  Cosign keyless sign    │   │                         │
    │                           │ └──────────┬──────────────┘   │                         │
    │                           │            │ image-tag output  │                         │
    │                           │ ┌──────────▼──────────────┐   │                         │
    │                           │ │ Job: deploy-dev         │   │                         │
    │                           │ │  git commit:            │   │                         │
    │                           │ │  envs/dev/values-*.yaml │──►│ ArgoCD polls            │
    │                           │ │  image.tag: sha-abc1234 │   │ every 3 min             │
    │                           │ └──────────┬──────────────┘   │──────────────────────►  │
    │                           │            │                   │  DEV auto-sync          │
    │                           │            │                   │  new pod running        │
    │                           │ ┌──────────▼──────────────┐   │                         │
    │                           │ │ Job: open-qa-pr         │   │                         │
    │                           │ │  git checkout -b        │   │                         │
    │                           │ │  promote/qa/<svc>/tag   │   │                         │
    │                           │ │  yq patch qa values     │──►│ PR opened in zen-gitops │
    │                           │ │  gh pr create           │   │                         │
    │                           │ └─────────────────────────┘   │                         │
    │                           │                               │                         │
    │  QA team reviews PR       │                               │                         │
    ├──────────────────────────────────────────────────────────►│                         │
    │                           │                               │  PR merged              │
    │                           │                               │  ArgoCD auto-syncs ────►│
    │                           │                               │  QA auto-sync           │
    │                           │                               │  new pod running        │
    │                           │                               │                         │
    │  Release Manager triggers │                               │                         │
    │  promote-prod.yml manually│                               │                         │
    ├──────────────────────────►│                               │                         │
    │                           │ promote-prod.yml              │                         │
    │                           │ ┌─────────────────────────┐   │                         │
    │                           │ │ Read image tag from QA  │   │                         │
    │                           │ │ values file (yq)        │   │                         │
    │                           │ │ git checkout -b         │   │                         │
    │                           │ │ promote/prod/<svc>/tag  │   │                         │
    │                           │ │ yq patch prod values    │──►│ PR opened in zen-gitops │
    │                           │ │ gh pr create            │   │                         │
    │                           │ └─────────────────────────┘   │                         │
    │                           │                               │                         │
    │  Release Manager merges   │                               │                         │
    │  PROD PR after approvals  │                               │                         │
    ├──────────────────────────────────────────────────────────►│ PR merged               │
    │                           │                               │ ArgoCD: OutOfSync       │
    │                           │                               │ (manual sync required)  │
    │                           │                               │                         │
    │  Engineer syncs in ArgoCD UI at maintenance window        │──────────────────────►  │
    │                                                           │  PROD sync              │
    │                                                           │  new pod running        │
```

---

## Workflow File Map

```
zen-pharma-backend/
└── .github/
    └── workflows/
        │
        │  ── Reusable building blocks ──────────────────────────────────────────
        ├── _java-build.yml          ← Full build: Maven + CodeQL + OWASP +
        │                                          Trivy + ECR + Cosign
        ├── _node-build.yml          ← Full build: npm + CodeQL + audit +
        │                                          Trivy + ECR + Cosign
        ├── _java-pr-check.yml       ← Lightweight: Maven + CodeQL + OWASP
        │                                           (no Docker, no ECR)
        ├── _node-pr-check.yml       ← Lightweight: npm + CodeQL + audit
        │                                           (no Docker, no ECR)
        │
        │  ── Feature branch checks (feat-*, fix-*, chore-*) ───────────────────
        ├── ci-pr-api-gateway.yml
        ├── ci-pr-auth-service.yml
        ├── ci-pr-drug-catalog.yml
        ├── ci-pr-inventory-service.yml
        ├── ci-pr-manufacturing-service.yml
        ├── ci-pr-supplier-service.yml
        ├── ci-pr-notification.yml
        │
        │  ── Full build + DEV deploy + QA PR (develop / release/**) ───────────
        ├── ci-api-gateway.yml
        ├── ci-auth-service.yml
        ├── ci-drug-catalog.yml
        ├── ci-inventory-service.yml
        ├── ci-manufacturing-service.yml
        ├── ci-supplier-service.yml
        ├── ci-notification.yml
        │
        │  ── PROD promotion (manual trigger) ───────────────────────────────────
        └── promote-prod.yml
```

> **Why reusable workflows (`_java-build.yml` etc.) instead of copying the steps into each `ci-*.yml`?**  
> The full build pipeline has 8 stages. With 7 services, copying those stages inline would mean 56 copies of the same logic. When you need to update a Trivy flag or change the Cosign version, you'd change it in one file instead of seven. Reusable workflows also guarantee consistency — every service goes through identical security gates with no accidental variation.

> **Why are there two lightweight variants (`_java-pr-check.yml`, `_node-pr-check.yml`) in addition to the full build?**  
> A developer may push 10–15 times a day to a feature branch while iterating. Running the full 15-minute pipeline on every push would: (1) burn GitHub Actions runner minutes on unreviewed code, (2) push dozens of unmerged images to ECR, and (3) slow down developer feedback. The PR check gives full security feedback (SAST, CVE scan) in ~5 minutes with no container overhead.

---

## Branch → Workflow Trigger Matrix

| Event | Branches | Workflow triggered | What runs |
|---|---|---|---|
| `push` | `feat-*`, `fix-*`, `chore-*` | `ci-pr-<service>.yml` | Lint · Test · CodeQL · OWASP |
| `push` | `develop`, `release/**` | `ci-<service>.yml` | Full build + ECR + DEV deploy + open QA PR |
| `workflow_dispatch` | any | `promote-prod.yml` | Read QA tag → open PROD PR |

> **Why is there no `pull_request` trigger for the PR-check workflow?**
> `ci-pr-<service>.yml` used to also trigger on `pull_request` into `develop`, in addition to `push` on `feat-*/fix-*/chore-*`. Since a student's PR is opened from a branch inside their own repo (not a cross-repo fork-to-upstream PR), both triggers fired for the same commit SHA — every push while a PR was open ran the identical check twice. GitHub Actions attaches check runs to the commit SHA, not to the event that triggered them, so removing the `pull_request` trigger loses no visibility: the push-triggered check still shows up on the PR automatically. One consequence: a branch that doesn't match `feat-*`, `fix-*`, or `chore-*` won't get any automatic CI check on its PR — make sure your branch name follows the convention.

> **Why path filters on every workflow?**  
> This is a monorepo with 7 services. Without path filters, pushing a one-line change to `notification-service/` would trigger all 7 pipelines — 6 unnecessary builds. Each workflow is scoped to its own service directory (`notification-service/**`) so only the affected service rebuilds. The exception: changes to the workflow file itself (`ci-notification.yml`) also trigger a run, so CI configuration changes are validated.

> **Why `develop` and `release/**` as the full-build trigger — not `main`?**  
> `main` is the stable branch — nothing should push directly to it except a merge. `develop` is the integration branch where features land. `release/**` branches are created at sprint end for hotfixes before a release. Both get the full pipeline. `main` intentionally has no workflow trigger — PROD is promoted via `promote-prod.yml`, not by pushing to main.

---

## Three-Tier Promotion Model

```
┌────────────────────────────────────────────────────────────────────────┐
│  TIER 1 — Feature Branch (ci-pr-*.yml)                                 │
│                                                                        │
│  Trigger: push to feat-* / fix-* / chore-*                            │
│           (check appears automatically on any PR from that branch)     │
│  Goal:    fast feedback to developer, no side-effects                  │
│                                                                        │
│  Maven verify (+ Postgres if needed)                                   │
│  → CodeQL  → OWASP Dependency Check                                   │
│                                                                        │
│  No Docker build.  No ECR push.  No GitOps update.  (~5 min)          │
└────────────────────────────────────────────────────────────────────────┘
                              │ PR merged to develop
                              ▼
┌────────────────────────────────────────────────────────────────────────┐
│  TIER 2 — develop / release push (ci-*.yml)                            │
│                                                                        │
│  Job 1 · build                                                         │
│    Maven verify / npm ci  (+ Postgres container if needed)             │
│    CodeQL · OWASP Dep Check                                            │
│    Docker build (non-root UID 1000)                                    │
│    Trivy — fail on HIGH / CRITICAL                                     │
│    ECR push  →  image tag: sha-<7chars>                                │
│    Cosign keyless sign  (OIDC → Fulcio → Rekor)                        │
│                                                                        │
│  Job 2 · deploy-dev  (GitHub environment: dev)                         │
│    git commit: envs/dev/values-<service>.yaml  ← image.tag updated    │
│    git push → zen-gitops main                                          │
│    ArgoCD (pharma-dev) auto-syncs within ~3 min                        │
│                                                                        │
│  Job 3 · open-qa-pr  (needs: build + deploy-dev)                       │
│    git checkout -b promote/qa/<service>/<image-tag>  in zen-gitops     │
│    yq patch: envs/qa/values-<service>.yaml                             │
│    gh pr create → chandika-s/zen-gitops                                     │
│    QA team reviews + merges the PR                                     │
│    ArgoCD (pharma-qa) auto-syncs on merge                              │
└────────────────────────────────────────────────────────────────────────┘
                              │ promote-prod.yml (workflow_dispatch)
                              ▼
┌────────────────────────────────────────────────────────────────────────┐
│  TIER 3 — PROD promotion (promote-prod.yml)                            │
│                                                                        │
│  Trigger: manual  (Release Manager selects service in dropdown)        │
│  GitHub environment: prod  (Required Reviewers gate before run)        │
│                                                                        │
│  Read image tag from envs/qa/values-<service>.yaml  (yq)              │
│    ↳ fails if QA values file missing — forces QA to exist first        │
│  Validate envs/prod/values-<service>.yaml exists                       │
│    ↳ exits with error if PROD file missing — PROD must be onboarded    │
│  git checkout -b promote/prod/<service>/<image-tag>  in zen-gitops     │
│  yq patch: envs/prod/values-<service>.yaml                             │
│  gh pr create → chandika-s/zen-gitops                                       │
│  After approvals: merge PR                                             │
│  ArgoCD (pharma-prod) shows OutOfSync → engineer syncs manually        │
│    at maintenance window                                               │
└────────────────────────────────────────────────────────────────────────┘
```

> **Why is QA promotion a PR instead of a direct git commit?**  
> A direct commit to the zen-gitops QA branch would deploy immediately with no human review. A PR gives: (1) a review gate where the QA team can inspect exactly what's changing, (2) a discussion thread per promotion for sign-off comments, (3) the ability to close the PR if the build shouldn't go to QA yet, and (4) a permanent audit trail — GitHub records who approved, when, and the exact diff.

> **Why is PROD promotion a separate `workflow_dispatch` file instead of an inline job in `ci-*.yml`?**  
> If PROD were an inline job in `ci-*.yml`, it would run in the same workflow as DEV deployment. This causes two problems: (1) the GitHub `prod` environment gate (Required Reviewers) would block every DEV deployment waiting for PROD approval, and (2) PROD would be tied to the same push event as DEV, removing the ability to promote to PROD independently or at a different time. A separate file means DEV deploys automatically while PROD is a conscious, standalone decision.

> **Why does PROD use `workflow_dispatch` with a service dropdown, not one file per service?**  
> Seven separate PROD promotion files would duplicate identical logic. The dropdown in a single `promote-prod.yml` lets the Release Manager pick the service, trigger once, and get a PR — one workflow, all services covered. Each trigger creates a separate PR in zen-gitops, so per-service audit trail is preserved.

> **Why does PROD read the image tag from the QA values file instead of accepting it as an input?**  
> If the Release Manager had to manually type the image tag, they could make a typo or promote a tag that never went through QA. Reading from `envs/qa/values-<service>.yaml` ensures that whatever is running in QA right now is exactly what gets promoted to PROD — no human error in the tag selection.

> **Why is ArgoCD auto-sync for DEV and QA, but manual for PROD?**  
> DEV needs fast feedback — developers push to develop and expect to see results in a few minutes. QA auto-syncs because the QA team controls deployment by deciding when to merge the promotion PR — once they merge, they want it live immediately. PROD is different: production changes should happen at a scheduled maintenance window, after CAB approval, with an engineer watching the rollout in ArgoCD. Auto-sync at midnight because someone merged a PR would be dangerous.

---

## Core Principle — Build Once, Deploy Many

The same Docker image (`sha-abc1234`) is pushed to ECR exactly once. Every environment promotion updates only a values file — no rebuild, no re-tag.

```
  git push to develop
        │
        ▼
  Docker build + ECR push
  image: sha-abc1234  ──────────────────────────────────┐
                                                        │ same bytes
   envs/dev/values.yaml  ──► DEV pod running sha-abc1234│
   envs/qa/values.yaml   ──► QA  pod running sha-abc1234│
   envs/prod/values.yaml ──► PROD pod running sha-abc1234
```

What changes per promotion: one line in one YAML file (`image.tag`).  
What never changes: the image digest.

> **Why not rebuild the image per environment?**  
> If you rebuild per environment, the image that runs in PROD is technically different from the one QA tested — even if the source code is identical, dependency resolution, base image layers, or build tooling could produce a different binary. "It passed QA" means nothing if you rebuilt for PROD. Building once and promoting the same digest eliminates this class of problem entirely.

> **Why `sha-<7chars>` as the image tag format instead of `:latest` or a version number?**  
> `:latest` is mutable — it gets overwritten on every push, so you lose the ability to know which commit is running in any environment. Semver requires someone to manually bump the version and tag a release. `sha-<7chars>` is: automatically derived from the git commit, immutable, and directly traceable — given a tag like `sha-a3f2c1d`, you can run `git show a3f2c1d` and see exactly what code is in that image.

---

## Service Matrix

| Service | Stack | Reusable workflow | DB in tests | ECR repo | zen-gitops name |
|---|---|---|---|---|---|
| api-gateway | Java 17 / Spring Cloud Gateway | `_java-build.yml` | No | `api-gateway` | `api-gateway` |
| auth-service | Java 17 / Spring Boot + JWT | `_java-build.yml` | Yes | `auth-service` | `auth-service` |
| drug-catalog-service | Java 17 / Spring Boot + Flyway | `_java-build.yml` | Yes | `drug-catalog-service` | `catalog-service` |
| inventory-service | Java 17 / Spring Boot + Flyway | `_java-build.yml` | Yes | `inventory-service` | `inventory-service` |
| manufacturing-service | Java 17 / Spring Boot + Flyway | `_java-build.yml` | Yes | `manufacturing-service` | `manufacturing-service` |
| supplier-service | Java 17 / Spring Boot + Flyway | `_java-build.yml` | Yes | `supplier-service` | `supplier-service` |
| notification-service | Node.js 20 / Express + Jest | `_node-build.yml` | No | `notification-service` | `notification-service` |

> **Why does `drug-catalog-service` have a different zen-gitops name (`catalog-service`)?**  
> The GitOps repo was bootstrapped with `catalog-service` as the Helm release name before the application repo settled on `drug-catalog-service`. Renaming it in zen-gitops would require migrating ArgoCD application definitions, Helm release history, and all existing values files. The `ci-drug-catalog.yml` workflow bridges this with a `GITOPS_SERVICE_NAME: catalog-service` env var. The ECR repository and Docker image remain `drug-catalog-service`.

---

## Security tooling — industry categories vs this repository

Common **DevSecOps categories** and **widely used tools** (examples), followed by what **this repo** actually runs in CI (or has wired but disabled).

### Industry reference

| Category | Typical tools (examples) | Role |
|----------|--------------------------|------|
| SAST (static application security) | CodeQL, Semgrep, Checkmarx, Veracode, Snyk Code | Find vulnerable patterns and misconfigurations in source code |
| SCA (software composition analysis) | Snyk, Mend, OWASP Dependency Check, Dependabot, `npm audit` | Find known CVEs in third-party libraries and dependencies |
| Secrets in repository | Gitleaks, TruffleHog, GitHub secret scanning, git-secrets | Detect API keys, tokens, and credentials committed to Git |
| Container / artifact scanning | Trivy, Grype, Clair, AWS ECR native scanning | Scan OS packages and layers in built container images |
| Supply-chain signing | Cosign, Notation, Sigstore | Cryptographically sign images for provenance and policy |
| DAST / runtime testing | OWASP ZAP, Burp Suite | Test *running* deployments for vulnerabilities (usually separate from build CI) |

### Implemented in zen-pharma-backend

| Category | Tool / mechanism | Where it runs | Notes |
|----------|------------------|---------------|--------|
| SAST | **CodeQL** (`security-extended`) | `_java-pr-check.yml`, `_java-build.yml`, `_node-pr-check.yml`, `_node-build.yml` | SARIF uploaded to GitHub **Code scanning** when org/repo settings allow |
| SCA (Node) | **`npm audit`** (fail on HIGH/CRITICAL) | `_node-pr-check.yml`, `_node-build.yml` | |
| SCA (Java) | **OWASP Dependency Check** (Maven plugin) | All Java `ci-pr-*.yml` / `ci-*.yml` via `_java-pr-check.yml`, `_java-build.yml` | Add optional repo secret **`NVD_API_KEY`** for faster NVD API sync (see [NVD API key](#nvd-api-key-owasp-dependency-check)) |
| Secrets in repo | **GitHub Secret scanning** | GitHub platform (enabled under repo Code security settings) | Scans all commits and PRs automatically; no CI runner time or step required |
| Container scan | **Trivy** | `_java-build.yml`, `_node-build.yml` only | No image in PR check — Trivy runs after Docker build |
| Image signing | **Cosign** (keyless via GitHub OIDC) | `_java-build.yml`, `_node-build.yml` | |
| Automated dependency PRs | **Dependabot** | — | Optional: add `.github/dependabot.yml` (not present in this repo today) |
| DAST (dynamic application security) | **OWASP ZAP** (baseline scan) | `ci-*.yml` — `dast` job, runs after `deploy-dev` | Currently disabled (`if: false`). Enable per-service by setting `DEV_<SERVICE>_URL` in repo variables and removing `if: false`. Uses `zaproxy/action-baseline@v0.12.0` with `fail_action: warn`. |

> **DAST activation checklist:** (1) Set `DEV_<SERVICE>_URL` in GitHub repo variables. (2) Remove `if: false` from the `dast` job in the relevant `ci-*.yml`. (3) Tune `.zap/rules.tsv` after first run to suppress false positives.

---

## DevSecOps Pipeline — Every Stage Explained

### Stage 1 — Unit Tests + Code Coverage

| | Java | Node.js |
|---|---|---|
| Tool | Maven (`mvn verify`) + JaCoCo | Jest (`jest --coverage`) |
| What it does | Compiles code, runs all unit and integration tests, measures line/branch coverage | Runs all Jest specs, measures statement/branch/function coverage |
| Coverage gate | Configured in `pom.xml` via `jacoco-maven-plugin` | Configured in `jest.config.js` `coverageThreshold` |
| Fail condition | Any test failure, or coverage below threshold | Any test failure, or coverage below threshold |
| Artifact | JaCoCo HTML report uploaded to GitHub Actions artifacts (7 days) | Jest HTML report |

> **Why run tests in CI at all if developers run them locally?**  
> Local test results are not auditable and depend on the developer's machine state. CI tests run on a clean, reproducible environment on every push — they catch "works on my machine" issues and ensure no one accidentally pushes failing code.

> **Why a coverage gate?**  
> Without a coverage threshold, teams gradually stop writing tests as pressure mounts. The gate makes the degradation visible immediately — a PR that drops coverage below threshold fails the build, prompting a conversation about whether the test debt is acceptable.

> **Why a real Postgres container for integration tests instead of mocking the database?**  
> Database mocks test that your code calls the mock correctly — they don't test that your SQL queries, JPA mappings, or Flyway migrations work against a real database engine. Real database bugs (index violations, constraint errors, dialect differences) are only caught with a real database. The Postgres 15 container starts fresh per run and is thrown away — there is no shared state between runs.

---

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

---

### Stage 3 — Dependency Vulnerability Scan (SCA)

| | Detail |
|---|---|
| Tool | OWASP Dependency Check (`org.owasp:dependency-check-maven` via `mvn`) |
| What it scans | All declared dependencies in `pom.xml` / `package.json` against the NIST National Vulnerability Database (NVD) |
| What it catches | Known CVEs in third-party libraries — Log4Shell in log4j, Spring4Shell in Spring Framework, prototype pollution in npm packages |
| NVD cache | Cached keyed on `pom.xml` hash — avoids re-downloading the ~200 MB database on every run |
| Maven gate | `-DfailBuildOnCVSS=7` — Maven exits non-zero if any dependency has CVSS ≥ 7.0 |
| Pipeline | **`continue-on-error: true`** on the OWASP step — findings do **not** fail the job; review the HTML artifact and logs |
| Artifact | HTML report — 14 days on full build, 3 days on PR check |

> **Why CVSS ≥ 7.0 as the threshold and not ≥ 9.0 (Critical only)?**  
> CVSS 7.0 is the bottom of the High range. A High severity CVE is exploitable under realistic conditions. Critical (≥ 9.0) covers only the most severe cases but misses a large class of exploitable vulnerabilities. 7.0 is the standard threshold used in most security compliance frameworks (PCI-DSS, SOC 2). Teams sometimes start at 9.0 and lower it over time as they clear the backlog.

> **Why cache the NVD database?**  
> The NVD database is approximately 200 MB and takes 5+ minutes to download on a cold run. Without caching, every pipeline run pays this cost. The cache is keyed on `pom.xml` hash — so it is invalidated and refreshed whenever dependencies change, ensuring you always check against a recent copy of the vulnerability database when dependencies are updated.

> **How is this different from Trivy?**  
> OWASP Dependency Check scans your *declared source dependencies* — what's listed in `pom.xml` or `package.json`. Trivy scans the *built container image* — OS packages installed by the Dockerfile base image, native libraries, and all runtime dependencies bundled inside the image. A vulnerable OpenSSL version in the Alpine base image would only be caught by Trivy. A vulnerable Spring Boot version in `pom.xml` would be caught by both, but OWASP Dep Check catches it earlier (before Docker build). Both scans are needed.

#### NVD API key (OWASP Dependency Check)

NIST’s NVD 2.0 API applies **strict rate limits** to unauthenticated requests. Without an API key, OWASP Dependency Check can spend a long time waiting between requests when refreshing CVE data. With a key, you get a **higher rate limit**, so the first run (and cache refreshes) complete much faster.

**Create an NVD API key (you do this once):**

1. Open the NIST NVD developer page: [https://nvd.nist.gov/developers/request-an-api-key](https://nvd.nist.gov/developers/request-an-api-key).
2. Submit the form (valid email). NIST emails you a link to verify and receive the key.
3. In the GitHub repo: **Settings → Secrets and variables → Actions → New repository secret**.
4. Name: **`NVD_API_KEY`** (exact name — workflows pass it to the Maven plugin via the `NVD_API_KEY` environment variable).
5. Value: paste the key from NIST. Save.

Reusable workflows use **`secrets: inherit`** on service workflows, so the secret is available to `_java-pr-check.yml` and `_java-build.yml` without extra wiring.

---

### Stage 4 — Docker Build

| | Detail |
|---|---|
| What | Multi-stage Docker build |
| Non-root | Container runs as UID/GID 1000 — never as root inside the pod |
| Build args | `--build-arg UID=1000 --build-arg GID=1000` passed at build time |
| Image tag | `sha-<7chars>` — immutable, traceable to the exact commit |

> **Why multi-stage Docker build?**  
> A single-stage build would include Maven, the full JDK, all build dependencies, and source code in the final image. Multi-stage separates the build environment from the runtime environment — the final image contains only the compiled JAR and the JRE, not Maven or the JDK. This reduces image size (smaller attack surface, faster pulls) and removes build tools that have no business being in a production container.

> **Why non-root UID 1000?**  
> If a container escape vulnerability is exploited in a root container, the attacker gets root on the host node — they can read secrets from other pods, modify node configuration, or pivot to the control plane. A non-root container limits the blast radius of a container escape to the permissions of UID 1000. Kyverno policies in EKS can also enforce non-root as an admission requirement, rejecting any pod spec that tries to run as root.

---

### Stage 5 — Container Image Scan

| | Detail |
|---|---|
| Tool | Trivy (`aquasecurity/trivy-action`) |
| What it scans | The built Docker image — OS packages (apt/apk), language runtime libraries bundled in the image |
| Fail condition | Any HIGH or CRITICAL CVE with a fix available |
| `ignore-unfixed: true` | CVEs with no available fix are reported but do not fail the build |
| Results | SARIF uploaded to GitHub Security tab under `trivy-<service-name>` category |
| Runs after | Docker build, before ECR push — rejects a vulnerable image before it ever reaches the registry |

> **Why `ignore-unfixed: true`?**  
> Some CVEs in base OS packages (Alpine, Debian) have no patched version available yet. If you fail on unfixed CVEs, the build is permanently blocked with no actionable path to resolve it — you can't update a package that has no update. `ignore-unfixed` means the build fails only when a fix exists (i.e., there is something you can actually do: update the base image or the package). Unfixed CVEs still appear in the SARIF report for visibility.

> **Why scan the image and not just the source dependencies (OWASP already does that)?**  
> The container image contains more than your application's dependencies. It includes the OS, the JRE/Node runtime, and any OS-level libraries installed by the Dockerfile. These are invisible to OWASP Dependency Check which only reads `pom.xml`. A vulnerable `libssl` in the Alpine base layer or a vulnerable `glibc` would only be caught by scanning the image.

> **Why upload results to the GitHub Security tab?**  
> The SARIF format integrates with GitHub's security dashboard, which aggregates findings across all services in one place. Security engineers can triage and track findings without reading raw pipeline logs. The `category: trivy-<service-name>` tag keeps each service's findings separate in the dashboard.

---

### Stage 6 — ECR Push

| | Detail |
|---|---|
| Auth | AWS OIDC — no long-lived access keys |
| Role | `pharma-github-actions-role` — scoped to ECR push permissions only |
| Image tag | `sha-<7chars>` pushed to `<account>.dkr.ecr.us-east-1.amazonaws.com/<service>` |
| Digest | SHA256 content-addressable digest captured after push for Cosign signing |

> **Why AWS OIDC instead of storing `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` as secrets?**  
> Long-lived AWS access keys are a major security risk. If they are accidentally committed to code, exposed in build logs, or leaked from GitHub's secrets store, an attacker has persistent AWS access until someone notices and rotates them. OIDC tokens are short-lived (~15 minutes), scoped to a specific workflow run, and automatically expire. There are no credentials to leak, rotate, or audit. The trust is established by GitHub's OIDC provider — the role only trusts tokens from this specific repository.

> **Why sign the digest rather than the tag?**  
> A Docker tag (`sha-abc1234`) is a pointer — it could theoretically be overwritten in ECR to point to a different image. The digest (`sha256:...`) is content-addressable and immutable — it is the cryptographic hash of the image manifest. Signing the digest means the signature is tied to the exact image bytes, not a mutable pointer.

---

### Stage 7 — Supply Chain Signing

| | Detail |
|---|---|
| Tool | Cosign (`sigstore/cosign-installer`) — keyless mode |
| How it works | GitHub OIDC token → Fulcio CA issues a short-lived signing certificate → image digest signed → signature + certificate stored in Rekor public transparency log |
| No key management | No long-lived signing keys exist. Signing identity = GitHub Actions OIDC claim for this repo |
| Enforcement | Kyverno policy in EKS rejects any pod whose image has no valid Cosign signature in Rekor |
| Fail condition | Cosign sign command fails |

> **Why keyless signing (no signing keys)?**  
> Traditional image signing requires managing a long-lived private key — storing it securely, rotating it, revoking it if compromised. Keyless signing uses GitHub's OIDC token as the signing identity. The Fulcio CA issues a certificate that expires in minutes. The signature is anchored in the Rekor public transparency log. There is nothing to store, rotate, or leak. The signing identity is cryptographically tied to the GitHub repository and workflow.

> **Why sign images at all? ECR is already private.**  
> ECR access controls prevent external actors from pushing images. But they don't prevent: (1) a compromised AWS credential pushing a malicious image directly to ECR, bypassing CI entirely, or (2) a misconfigured Kubernetes workload pulling the wrong image. Cosign + Kyverno means that even if someone pushes a rogue image to ECR, it will be rejected by the cluster at pod admission time because it lacks a valid signature from the CI pipeline. Defence in depth — ECR access control and image signing are complementary, not redundant.

---

### Stage 8 — GitOps Update (DEV)

> **Why commit directly to zen-gitops main for DEV instead of opening a PR?**  
> DEV is a fast feedback environment — the goal is for developers to see their changes running within minutes of merging to develop. A PR review step would introduce a human gate that slows down DEV unnecessarily. DEV is also a low-trust environment — a broken DEV deployment is expected occasionally and is easily fixed. The value of a PR gate increases as you move toward production, not in the lowest environment.

> **Why `yq` to patch the values file instead of `sed`?**  
> `sed` operates on text patterns. If the YAML file structure or indentation changes, the `sed` pattern can match the wrong line or silently corrupt the file. `yq` is a YAML-aware tool — it understands the structure and patches `.image.tag` as a typed YAML key, regardless of formatting. It is also idempotent: running it twice produces the same result.

> **Why the `git diff --staged --quiet && exit 0` guard before committing?**  
> If the same commit SHA is somehow processed twice (e.g. a workflow re-run), the values file already has the correct tag and there is nothing to commit. Without the guard, `git commit` would fail with "nothing to commit" and exit non-zero, failing the pipeline for a false reason. The guard makes the step idempotent.

---

### Stage 9 — QA Promotion PR

> **Why open a PR in zen-gitops rather than a PR in this repo (zen-pharma-backend)?**  
> The change being reviewed is a deployment configuration change — `image.tag` in a Helm values file. That belongs in zen-gitops, not in the application source repo. Keeping deployment PRs in zen-gitops also means: (1) the QA team can review and approve without having access to the application source code, and (2) the deployment history in zen-gitops is clean and contains only deployment changes, not application code.

> **Why does the open-qa-pr job check if the values file exists before proceeding?**  
> Not all services are onboarded to QA yet (inventory, manufacturing, supplier, catalog don't have QA values files). Without the guard, the workflow would fail with a cryptic `yq` error when the file is missing. The guard exits cleanly with a `::warning::` annotation — the pipeline succeeds, DEV still gets the new image, and the warning tells the operator exactly what to do to enable QA promotion.

> **Why `printf '%s\n'` + `--body-file` for the PR body instead of `--body "..."`?**  
> Multiline strings containing `:` characters or markdown bold (`**text**`) inside a YAML workflow file cause YAML parser errors. The parser interprets `**Image:**` as a YAML anchor reference and lines starting at column 0 as top-level YAML properties. Writing the PR body to a temp file with `printf` sidesteps YAML parsing entirely — the PR body content never appears as a literal string in the workflow YAML.

---

### Summary — What Runs Where

```
                          Feature branch    develop / release
Stage                     ci-pr-*.yml       ci-*.yml
─────                     ───────────       ────────
Unit tests                     ✓                ✓
Code coverage (JaCoCo/Jest)    ✓                ✓
CodeQL SAST                    ✓                ✓
OWASP Dependency Check (Java)  ✓                ✓
npm audit (Node)               ✓                ✓
Docker build                   ✗                ✓
Trivy image scan               ✗                ✓
ECR push                       ✗                ✓
Cosign sign                    ✗                ✓
GitOps DEV update              ✗                ✓
QA promotion PR                ✗                ✓
DAST — ZAP baseline            ✗                ✗ (disabled — if: false)

Approx. runtime                ~5 min           ~15 min
```

---

## Reusable Workflows — Inputs

### `_java-build.yml` and `_java-pr-check.yml`

| Input | Type | Required | Default | Description |
|---|---|---|---|---|
| `service-name` | string | yes | — | Used in artifact names |
| `service-dir` | string | yes | — | Directory relative to repo root |
| `ecr-repository` | string | yes | — | ECR repo name (`_java-build.yml` only) |
| `aws-region` | string | no | `us-east-1` | AWS region (`_java-build.yml` only) |
| `needs-database` | boolean | no | `false` | Starts a Postgres 15 sidecar for tests |

**Secrets (reusable):** `NVD_API_KEY` (optional) — NVD API key for faster OWASP Dependency Check NVD sync (repository secret, not a workflow input).

**Outputs (`_java-build.yml` only):** `image-tag` (`sha-<7chars>`), `registry` (ECR URL)

### `_node-build.yml` and `_node-pr-check.yml`

Same inputs as Java equivalents with `node-version` (default `20`) replacing `needs-database`.

---

## GitHub Environments — Required Setup

**Settings → Environments** in the `zen-pharma-backend` repo:

| Environment | Protection rule | Reviewers |
|---|---|---|
| `dev` | None (auto-deploys) | — |
| `prod` | Required reviewers | Release Manager + QA Lead |

> **Why does QA have no GitHub environment gate?**  
> QA promotion is gated by the PR review in zen-gitops. The QA team reviews and merges the PR — that is the gate. Adding a GitHub environment on top would be a second gate on the same decision, creating friction without adding safety. The zen-gitops PR is actually a stronger gate: it shows the exact diff (what tag is changing to what), allows inline comments, and requires a separate account to approve (not the bot that opened it).

> **Why does PROD have a GitHub environment gate in addition to the zen-gitops PR?**  
> PROD has two gates by design: (1) the GitHub environment `prod` with Required Reviewers blocks the `promote-prod.yml` workflow from running at all until a Release Manager approves — this prevents even the PR from being opened by an unauthorised trigger, and (2) the zen-gitops PR itself requires approval before merge. The double gate reflects the sensitivity of PROD changes.

---

## GitHub Secrets — Required Setup

**Settings → Secrets and variables → Actions:**

| Secret | Used by | Description |
|---|---|---|
| `AWS_ACCOUNT_ID` | all `ci-*.yml` | 12-digit AWS account ID for ECR URL construction |
| `GITOPS_TOKEN` | all `ci-*.yml`, `promote-prod.yml` | GitHub PAT or App token with `contents: write` on `chandika-s/zen-gitops` |
| `NVD_API_KEY` | `_java-pr-check.yml`, `_java-build.yml` (when OWASP enabled) | NIST NVD API key — higher rate limits for OWASP Dependency Check (optional but recommended) |

> **Why a dedicated `GITOPS_TOKEN` instead of using `GITHUB_TOKEN`?**  
> `GITHUB_TOKEN` is automatically generated per workflow run and scoped to the current repository only. It cannot write to a different repository (`chandika-s/zen-gitops`). A separate PAT or GitHub App token with `contents: write` permission on zen-gitops is required for cross-repo operations. Using a GitHub App token (instead of a personal PAT) is preferred in production — App tokens are not tied to a specific user account and don't expire when someone leaves the organisation.

---

## AWS OIDC + ECR Setup — Step by Step

GitHub Actions authenticates to AWS using **OpenID Connect (OIDC)** — no long-lived AWS access keys stored as secrets. Each workflow run gets a short-lived token (valid ~1 hour) via the GitHub OIDC provider.

> **Why OIDC instead of storing `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` as secrets?**  
> Long-lived keys are a security risk: they don't expire, can be leaked via logs, and give permanent access if compromised. OIDC tokens are ephemeral (1 hour), scoped to a specific workflow run, and automatically rotated. If a token leaks, it expires before an attacker can use it. OIDC is the AWS and GitHub recommended approach for CI/CD.

### Step 1 — Create GitHub OIDC Identity Provider in AWS

This tells AWS to trust tokens issued by GitHub Actions.

```bash
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list 6938fd4d98bab03faadb97b34396831e3780aea1
```

> Only needs to be done once per AWS account. If you get `EntityAlreadyExists`, it's already set up.

### Step 2 — Get your AWS Account ID

```bash
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
echo "Account ID: $AWS_ACCOUNT_ID"
```

### Step 3 — Create the IAM Trust Policy

This policy allows GitHub Actions from **this specific repo** to assume the role. The `sub` condition ensures only workflows from `chandika-s/zen-pharma-backend` can authenticate — no other repo can assume this role.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:chandika-s/zen-pharma-backend:*"
        }
      }
    }
  ]
}
```

Save as `/tmp/trust-policy.json` (replace `<ACCOUNT_ID>` with your actual account ID), then create the role:

```bash
aws iam create-role \
  --role-name pharma-github-actions-role \
  --assume-role-policy-document file:///tmp/trust-policy.json \
  --description "GitHub Actions OIDC role for zen-pharma-backend CI/CD"
```

### Step 4 — Create and Attach ECR Permission Policy

The role only needs ECR permissions — nothing else. This follows the principle of least privilege.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ECRAuth",
      "Effect": "Allow",
      "Action": "ecr:GetAuthorizationToken",
      "Resource": "*"
    },
    {
      "Sid": "ECRPushPull",
      "Effect": "Allow",
      "Action": [
        "ecr:BatchCheckLayerAvailability",
        "ecr:BatchGetImage",
        "ecr:CompleteLayerUpload",
        "ecr:DescribeImages",
        "ecr:DescribeRepositories",
        "ecr:GetDownloadUrlForLayer",
        "ecr:InitiateLayerUpload",
        "ecr:ListImages",
        "ecr:PutImage",
        "ecr:UploadLayerPart"
      ],
      "Resource": "arn:aws:ecr:us-east-1:<ACCOUNT_ID>:repository/*"
    }
  ]
}
```

Save as `/tmp/ecr-policy.json`, then attach:

```bash
aws iam put-role-policy \
  --role-name pharma-github-actions-role \
  --policy-name pharma-ecr-access \
  --policy-document file:///tmp/ecr-policy.json
```

> **Why scope the IAM role to ECR permissions only?**  
> If the role also had S3, Lambda, or other permissions, a compromised workflow run (e.g. through a malicious dependency) could exfiltrate data or modify infrastructure. The role can only push images to ECR — the blast radius of a compromise is limited to ECR writes.

### Step 5 — Create ECR Repositories (one per microservice)

```bash
for svc in api-gateway auth-service drug-catalog-service inventory-service \
           manufacturing-service notification-service supplier-service; do
  aws ecr create-repository \
    --repository-name "$svc" \
    --region us-east-1 \
    --image-scanning-configuration scanOnPush=true \
    --encryption-configuration encryptionType=AES256
done
```

> `scanOnPush=true` enables AWS-native image scanning alongside Trivy in the pipeline — defense in depth.

### Step 6 — Add Secrets to GitHub Repository

Go to **Settings → Secrets and variables → Actions → New repository secret**:

| Secret | Value | Required |
|--------|-------|----------|
| `AWS_ACCOUNT_ID` | Your 12-digit AWS account ID | Yes |
| `GITOPS_TOKEN` | GitHub PAT with `repo` scope on `chandika-s/zen-gitops` | Yes |

### Step 7 — Create GitHub Environments

Go to **Settings → Environments**:

| Environment | Protection Rules |
|-------------|-----------------|
| `dev` | None — auto-deploys on merge to develop |
| `prod` | Add required reviewers for manual approval gate |

### Step 8 — Verify Setup

```bash
# Verify OIDC provider
aws iam list-open-id-connect-providers

# Verify role
aws iam get-role --role-name pharma-github-actions-role --query 'Role.Arn'

# Verify ECR repos
aws ecr describe-repositories --region us-east-1 \
  --query 'repositories[].repositoryName' --output table

# Verify GitHub secrets (requires gh CLI)
gh secret list --repo chandika-s/zen-pharma-backend
```

### How OIDC works at runtime

```
GitHub Actions Runner                    AWS
─────────────────────                    ───
1. Workflow starts
2. Runner requests OIDC token ──────────→
   (contains repo, branch, workflow)     3. IAM validates token against
                                            trust policy (repo match,
                                            audience match)
                                         4. Returns temporary credentials
                                            (AccessKeyId, SecretAccessKey,
                                            SessionToken — expires ~1hr)
5. Runner uses temp creds ──────────────→ 6. ECR authenticates, allows push
```

No long-lived keys ever touch GitHub Secrets. If a workflow run is compromised, the attacker gets a token that expires in ~1 hour and can only push to ECR.

---

## GitOps Repo Layout (`username/zen-gitops`)

```
zen-gitops/
├── argocd/apps/
│   ├── dev/                         ← One ArgoCD Application per service (auto-sync)
│   ├── qa/pharma-qa-app.yaml        ← Single app watching envs/qa/ (auto-sync)
│   └── prod/pharma-prod-app.yaml    ← Single app watching envs/prod/ (manual sync)
└── envs/
    ├── dev/
    │   ├── values-api-gateway.yaml
    │   ├── values-auth-service.yaml
    │   ├── values-catalog-service.yaml      ← drug-catalog-service
    │   ├── values-inventory-service.yaml
    │   ├── values-manufacturing-service.yaml
    │   ├── values-supplier-service.yaml
    │   └── values-notification-service.yaml
    ├── qa/                                  ← only services onboarded to QA
    │   ├── values-api-gateway.yaml
    │   ├── values-auth-service.yaml
    │   └── values-notification-service.yaml
    └── prod/                                ← only services onboarded to PROD
        ├── values-api-gateway.yaml
        ├── values-auth-service.yaml
        └── values-notification-service.yaml
```

> **Why a separate GitOps repo instead of a `gitops/` directory in this repo?**  
> If deployment config lived in the same repo as application code: (1) a developer committing code could accidentally change deployment config, (2) CI triggers would be hard to separate — a code push and a deployment change would both be in the same repo, and (3) access control is coarser — you can't give a QA engineer write access to `envs/qa/` without also giving them write access to application source code. A separate repo gives clean separation of access, history, and trigger logic.

> **Why one ArgoCD app per service for DEV, but a single shared app for QA and PROD?**  
> DEV needs per-service independent deployments — a change to `auth-service` should deploy immediately without waiting for or affecting `api-gateway`. QA and PROD are more controlled — typically you promote a set of services together as a release, and a single app watching the whole `envs/qa/` directory makes rollbacks and environment state easier to manage.

Values file structure patched by CI:
```yaml
image:
  repository: <aws-account>.dkr.ecr.us-east-1.amazonaws.com/<service>
  tag: sha-abc1234   # ← patched by yq on every promotion
  pullPolicy: IfNotPresent
```

Services not yet in `envs/qa/` or `envs/prod/` (inventory, manufacturing, supplier, catalog): CI skips the promotion PR with a `::warning::` annotation. Create the values file in zen-gitops to enable promotion for that service.

---

## FAQ

**Q: Why is there no ArgoCD CLI or smoke test in the pipeline?**  
ArgoCD polls zen-gitops every ~3 minutes and syncs automatically — no pipeline intervention is needed. Smoke tests require network access from GitHub-hosted runners into the private VPC (`dev.pharma.internal`), which is not configured. Post-deploy health validation can be added later as an ArgoCD `PostSync` hook once the infrastructure is in place — this keeps CI and post-deploy validation cleanly separated.

**Q: Why does `drug-catalog-service` use `catalog-service` in zen-gitops?**  
The GitOps repo was bootstrapped with `catalog-service` as the Helm release name before the source repo settled on `drug-catalog-service`. The `ci-drug-catalog.yml` workflow carries `GITOPS_SERVICE_NAME: catalog-service` to bridge this. ECR and the Docker image remain `drug-catalog-service`.

**Q: What happens if I push to `develop` but the QA values file doesn't exist yet?**  
The `open-qa-pr` job exits with `::warning::` and exit code 0 — the pipeline succeeds. DEV still gets the new image. Create `envs/qa/values-<service>.yaml` in zen-gitops to enable QA promotion for that service.

**Q: Does PROD ever auto-sync?**  
No. `pharma-prod` has `syncPolicy: Manual`. After the PROD PR merges, ArgoCD shows `OutOfSync`. An engineer syncs in the ArgoCD UI at the maintenance window.

**Q: How do I onboard a new microservice?**  
1. Copy `ci-api-gateway.yml` (no DB) or `ci-auth-service.yml` (with DB) and rename throughout  
2. Copy `ci-pr-api-gateway.yml` for the feature branch check  
3. Add the service to the `options` list in `promote-prod.yml`  
4. Create `envs/dev/values-<service>.yaml` in zen-gitops  
5. Create an ArgoCD Application manifest in `zen-gitops/argocd/apps/dev/`  
6. Add `envs/qa/` and `envs/prod/` values files when the service is ready for those environments

**Q: What is the image tag format?**  
`sha-<first 7 chars of git SHA>` — e.g. `sha-abc1234`. Set in the reusable build workflows via:
```bash
echo "image_tag=sha-${GITHUB_SHA::7}" >> $GITHUB_OUTPUT
```
