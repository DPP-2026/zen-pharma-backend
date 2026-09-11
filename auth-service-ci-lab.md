# Lab: Run the zen-pharma-backend CI/CD Pipeline After Forking

You forked `zen-pharma-backend`. This guide gets its GitHub Actions pipeline running end
to end — from a push on your laptop to a pod running on your EKS cluster, promoted through
DEV → QA → PROD. **`auth-service` is the fully worked example.** Part 10 gives you the same
recipe, condensed, for the other 7 services in this repo.

This guide covers **CI/CD only** — the GitHub Actions side. It assumes your AWS
infrastructure (EKS, RDS, ECR, IAM) and ArgoCD are already running. If they aren't yet, do
that first — see [Part 0](#part-0--things-that-must-already-be-true) below.

---

## Architecture at a glance

```
feature branch push (feat-*/fix-*/chore-*)
      │
      ▼
ci-pr-auth-service.yml           (~5 min, no Docker, no ECR)
  Maven verify → CodeQL
      │
      │  merge to develop / release/**
      ▼
ci-auth-service.yml              (~12-15 min)
  ├─ build            Maven verify → CodeQL → Docker → Trivy → ECR push → Cosign sign
  ├─ deploy-dev  ───► git commit  envs/dev/values-auth-service.yaml   in zen-gitops
  │                        │
  │                        ▼
  │                   ArgoCD auto-syncs DEV (polls every ~3 min)
  │
  └─ open-qa-pr  ───► PR opened  envs/qa/values-auth-service.yaml    in zen-gitops
                            │
                       (you review + merge)
                            ▼
                       ArgoCD auto-syncs QA
                            │
                  promote-prod.yml (workflow_dispatch, manual)
                            ▼
                       PR opened  envs/prod/values-auth-service.yaml  in zen-gitops
                            │
                     (2 approvals + merge)
                            ▼
                       ArgoCD PROD — you trigger sync manually
```

CI never touches Kubernetes. CI talks to `zen-gitops`. ArgoCD watches `zen-gitops` and talks
to Kubernetes. See `CI-ARCHITECTURE.md` in this repo for the full design rationale behind
every stage — this guide is the "do it yourself" companion to that reference.

---

## Part 0 — Things that must already be true

This lab only sets up **GitHub Actions**. Before you start, the following must exist —
these live in other repos and are out of scope here:

| Prerequisite | Where it comes from |
|---|---|
| AWS account, VPC, EKS cluster, RDS, ECR repos, IAM OIDC role | `zen-infra` — Terraform. See `zen-infra/docs/FULL-DEPLOYMENT-GUIDE.md` Stages 1–2 |
| ArgoCD installed + `pharma` AppProject + Application manifests for dev/qa/prod | `zen-infra/scripts/01-install-prerequisites.sh` and `02-bootstrap-argocd.sh` |
| External Secrets Operator wired to Secrets Manager (`db-credentials`, `jwt-secret`) | `zen-infra/scripts/03-setup-external-secrets.sh` |
| `zen-gitops` fork personalised with **your** AWS account ID and RDS instance ID | `zen-gitops/README.md` → "After Forking — Required Personalisation" |

If any of these are missing, do them first (or ask your instructor which mode your course
uses — see below) — otherwise `deploy-dev` will push a commit but the pod will sit in
`ImagePullBackOff` or `CrashLoopBackOff` forever, with nothing wrong in this repo's CI.

**Why External Secrets Operator instead of native Kubernetes Secrets:**

- **Single source of truth** — the real value lives in AWS Secrets Manager, not in the
  cluster or in git. A plain Kubernetes Secret has no upstream source, so drift and stale
  values creep in over time.
- **No static AWS credentials in the cluster** — ESO authenticates via IRSA, so no long-lived
  access keys sit in a Secret or env var the way a hand-rolled sync script would need.
- **Automatic rotation** — ESO polls Secrets Manager on an interval and updates the
  Kubernetes Secret in place; rotating the value upstream doesn't require a manual
  `kubectl apply` or a redeploy.
- **Audit trail** — reads go through AWS Secrets Manager, logged in CloudTrail. Kubernetes
  Secrets are only base64-encoded and carry no equivalent access log.
- **Consistent across environments** — dev, qa, and prod all sync from the same Secrets
  Manager paths, so promoting a secret change is an AWS-side update, not a per-namespace
  `kubectl` command.
- **No app changes needed** — Pods still consume an ordinary Kubernetes Secret; ESO only
  changes how that Secret's contents are populated and kept fresh.

---

## Course / instructor setup

Unlike an earlier version of this lab (`zen-pharma-backend-lab1`), this repo's IAM trust
policy is **not** a shared role that an instructor edits per student. It is created by
Terraform (`zen-infra/modules/iam/github-actions-oidc.tf`) scoped to `var.github_org` —
whatever GitHub account/org owns the fork that ran `terraform apply`. There are two ways to
run this course:

### Mode A — Self-service (recommended, matches this repo's design)

Each student forks all four repos (`zen-infra`, `zen-gitops`, `zen-pharma-backend`,
`zen-pharma-frontend`) and provisions their **own** AWS account end to end by following
`zen-infra/docs/FULL-DEPLOYMENT-GUIDE.md` Stages 1–2. Terraform automatically scopes the
`pharma-dev-github-actions-role` trust policy to that student's own fork
(`var.github_org` = their GitHub username). **No instructor action is required per
student** — this is the main advantage over the lab1 shared-cluster model.

Instructor's one-time job: make sure `DPP-2026/zen-infra`, `zen-gitops`,
`zen-pharma-backend`, and `zen-pharma-frontend` are visible/forkable (public, or students
added as collaborators), and that Actions are allowed to run on forks at the org level
(**Organization Settings → Actions → General → Fork pull request workflows**, if the
upstream org restricts this).

### Mode B — Shared classroom cluster (legacy / lab1-style)

If you'd rather run one shared EKS cluster for the whole cohort (closer to
`zen-pharma-backend-lab1`), you need to redo, per student fork, what Terraform otherwise
automates:

1. Manually update the trust policy on the shared `pharma-dev-github-actions-role` to add
   each student's fork to the `sub` condition (same pattern as
   `zen-pharma-backend-lab1/auth-service-ci-lab.md` → "Instructor Setup — OIDC and IAM
   role", just with this repo's actual role name and repos):
   ```bash
   aws iam update-assume-role-policy \
     --role-name pharma-dev-github-actions-role \
     --policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Federated":"arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"},"Action":"sts:AssumeRoleWithWebIdentity","Condition":{"StringEquals":{"token.actions.githubusercontent.com:aud":"sts.amazonaws.com"},"StringLike":{"token.actions.githubusercontent.com:sub":["repo:DPP-2026/zen-pharma-backend:ref:refs/heads/develop","repo:DPP-2026/zen-pharma-backend:ref:refs/heads/main","repo:<STUDENT-USERNAME>/zen-pharma-backend:*"]}}}]}'
   ```
   Remember: `update-assume-role-policy` **replaces** the whole `sub` list — always include
   every previous student's entry when adding a new one.
2. Give each student a separate ArgoCD `Application` (or namespace) so their deployments
   don't collide, and give them the shared cluster's kubeconfig / ArgoCD URL / RDS host.
3. Create the `dev` and `prod` GitHub Environments (see Part 2.4) in **each student's
   fork** — GitHub Environments are not inherited from the upstream template repo by forks.

The rest of this guide is identical either way — it only talks to your own
`zen-pharma-backend` fork and your own `zen-gitops` fork.

---

## Prerequisites

- `git` — `git --version`
- `kubectl` — `kubectl version --client`
- `argocd` CLI — `brew install argocd` / [Windows install](https://argo-cd.readthedocs.io/en/stable/cli_installation/)
- `aws` CLI — `aws --version`
- `gh` CLI (optional, handy for watching Actions runs) — `gh --version`

```bash
aws eks update-kubeconfig --region us-east-1 --name <your-cluster-name>
kubectl get nodes   # should list cluster nodes
```

---

## Part 1 — Fork the repos

### Step 1.1 — Fork zen-pharma-backend

1. Go to `https://github.com/DPP-2026/zen-pharma-backend` → **Fork** → **Create fork**
2. Clone it:

```bash
git clone https://github.com/<YOUR-USERNAME>/zen-pharma-backend.git
cd zen-pharma-backend
```

Relevant contents:

```
zen-pharma-backend/
├── auth-service/ … api-gateway/ … drug-catalog-service/ … (8 services total)
└── .github/workflows/
    ├── _java-build.yml, _java-pr-check.yml     ← reusable Java pipeline
    ├── _node-build.yml, _node-pr-check.yml     ← reusable Node pipeline
    ├── ci-pr-<service>.yml                     ← feature-branch check, per service
    ├── ci-<service>.yml                        ← full build + DEV + open QA PR, per service
    └── promote-prod.yml                        ← manual PROD promotion, all services
```

You don't create any workflow files — they already exist. **Important:** GitHub disables
Actions on forks by default. Go to your fork's **Actions** tab and click
**"I understand my workflows, go ahead and enable them"** — otherwise nothing below will
run.

### Step 1.2 — Fork zen-gitops (if you haven't already)

1. Go to `https://github.com/DPP-2026/zen-gitops` → **Fork** → **Create fork**
2. Clone it in a separate directory and follow `zen-gitops/README.md` → **"After
   Forking — Required Personalisation"** to replace the instructor's AWS account ID and RDS
   instance ID with your own throughout `envs/`. Do this before continuing — otherwise
   `ImagePullBackOff` is guaranteed regardless of what you do in Part 4.
3. Also enable Actions on this fork if it has any workflows you plan to use.

---

## Part 2 — Configure GitHub Actions on zen-pharma-backend

All of this is done in GitHub Settings — no code changes.

### Step 2.1 — Create a Personal Access Token for GitOps writes

CI commits to your `zen-gitops` fork after every build and opens PRs there. It needs a
token to do that.

1. GitHub → profile icon → **Settings** → **Developer settings** → **Personal access
   tokens** → **Fine-grained tokens** → **Generate new token**
2. **Token name**: `zen-pharma-gitops-token`; **Expiration**: 90 days; **Resource owner**:
   your account; **Repository access**: only `zen-gitops` (your fork)
3. **Permissions** → **Contents**: Read and write; **Pull requests**: Read and write
4. **Generate token** — copy it now, you won't see it again

### Step 2.2 — Add repository secrets

`zen-pharma-backend` fork → **Settings → Secrets and variables → Actions → Secrets** →
**New repository secret**:

| Secret | Value |
|---|---|
| `AWS_ACCOUNT_ID` | Your 12-digit AWS account ID |
| `GITOPS_TOKEN` | The PAT from Step 2.1 |
| `NVD_API_KEY` | *(optional)* NIST NVD API key — see `CI-ARCHITECTURE.md` § NVD API key. Not required today: the OWASP Dependency Check step is currently `if: false` (disabled) in `_java-build.yml`/`_java-pr-check.yml` in this repo. |

> No `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` — auth to AWS is via OIDC
> (`role-to-assume: arn:aws:iam::<account>:role/pharma-dev-github-actions-role`), no
> long-lived keys stored anywhere.

### Step 2.3 — Add a repository variable

Same page, **Variables** tab → **New repository variable**:

| Variable | Value |
|---|---|
| `GITOPS_REPO` | `<YOUR-USERNAME>/zen-gitops` |

### Step 2.4 — Create GitHub Environments

**Settings → Environments → New environment**:

| Environment | Protection rule | Used by |
|---|---|---|
| `dev` | None | `deploy-dev` job in every `ci-<service>.yml` |
| `prod` | **Required reviewers** — add yourself and/or a classmate; enable **Prevent self-review** | `promote-prod.yml` |

Without the `prod` environment, `promote-prod.yml` will fail with an environment-not-found
error the first time you run it.

---

## Part 3 — Confirm the zen-gitops values files for auth-service

If you completed Part 1.2, these already exist (this repo isn't `-lab1`; the values files
ship with the fork, they don't need to be authored by hand):

```
zen-gitops/envs/dev/values-auth-service.yaml
zen-gitops/envs/qa/values-auth-service.yaml
zen-gitops/envs/prod/values-auth-service.yaml
```

Sanity-check that the account-ID/RDS-host substitution from the `zen-gitops` README
actually landed in all three:

```bash
cd zen-gitops
grep -rn "image:\|DB_HOST" envs/dev/values-auth-service.yaml envs/qa/values-auth-service.yaml envs/prod/values-auth-service.yaml
```

You should see your own AWS account ID in every `image.repository` line and your own RDS
endpoint in every `DB_HOST` line — not the instructor's.

---

## Part 4 — Trigger the CI pipeline

### Step 4.1 — Create the develop branch

```bash
cd zen-pharma-backend
git checkout -b develop
git push -u origin develop
```

### Step 4.2 — Push a feature branch and watch the PR check

```bash
git checkout -b feat-first-build
echo "" >> auth-service/pom.xml
git add auth-service/pom.xml
git commit -m "test: trigger PR check"
git push -u origin feat-first-build
```

Go to **Actions** in your fork. `ci-pr-auth-service.yml` runs (calls the reusable
`_java-pr-check.yml`):

```
✓ Checkout
✓ Setup Java 17 (Temurin)
✓ Start PostgreSQL (integration tests)   ← auth-service needs a DB
✓ Initialize CodeQL
✓ Maven verify (tests + JaCoCo coverage)
✓ Upload JaCoCo coverage report
✓ CodeQL — Analyze
```

No Docker, no AWS credentials, no ECR — ~5 minutes. Wait for the green check before
continuing.

### Step 4.3 — Merge to develop and watch the full CI

```bash
git checkout develop
git merge feat-first-build
git push origin develop
```

Go to **Actions** → **CI/CD — auth-service**. Three jobs run:

**Job `build`** (`_java-build.yml`, ~12–15 min):

```
✓ Checkout
✓ Set image tag             →  sha-abc1234
✓ Setup Java 17 (Temurin)
✓ Start PostgreSQL (integration tests)
✓ Initialize CodeQL
✓ Maven verify (tests + JaCoCo coverage gate ≥ 80%)
✓ Upload JaCoCo coverage report
✓ CodeQL — Analyze
✓ Configure AWS credentials (OIDC)
✓ Login to Amazon ECR
✓ Build Docker image
✓ Trivy — Image vulnerability scan   (fails on HIGH/CRITICAL with a fix available)
✓ Upload Trivy SARIF to GitHub Security tab
✓ Push image to ECR                 →  sha-abc1234
✓ Install Cosign
✓ Cosign — Sign image (keyless, GitHub OIDC → Fulcio → Rekor)
```

**Job `deploy-dev`** (needs `build`, environment `dev`) — commits directly to your
`zen-gitops` fork's `main` branch:

```
Run yq e ".image.tag = \"sha-abc1234\"" -i "_gitops/envs/dev/values-auth-service.yaml"
[main xyz5678] ci(dev): update auth-service → sha-abc1234
```

Open your `zen-gitops` fork — you'll see this new commit on `main`.

**Job `open-qa-pr`** (needs `build` + `deploy-dev`) — opens or updates a PR in your
`zen-gitops` fork:

```
promote(qa): auth-service → sha-abc1234
```

The branch is always `promote/qa/auth-service/latest` — every subsequent push to
`develop` force-pushes over the same branch and updates the same PR (it doesn't open a
new PR per commit). Leave it open — you'll merge it in Part 8.

> A `dast` job also exists in this workflow but is disabled (`if: false`) by default. See
> the comment in `ci-auth-service.yml` if you want to turn on the ZAP baseline scan later.

### Step 4.4 — Verify the image landed in ECR

```bash
aws ecr describe-images \
  --repository-name auth-service \
  --region us-east-1 \
  --query 'sort_by(imageDetails, &imagePushedAt)[-1].imageTags' \
  --output table
```

You should see your `sha-abc1234` tag.

---

## Part 5 — Watch ArgoCD deploy to DEV

The `deploy-dev` commit is picked up by ArgoCD's poll loop (~3 min).

```bash
argocd login <ARGOCD-URL> --username admin --password <PASSWORD>
argocd app get auth-service-dev
```

Expect `Sync Status: Synced`, `Health Status: Progressing → Healthy`.

```bash
kubectl get pods -n dev -w
```

```
auth-service-7d9f8c-xxxx   0/1   ContainerCreating   0   5s
auth-service-7d9f8c-xxxx   0/1   Running             0   20s
auth-service-7d9f8c-xxxx   1/1   Running             0   55s
```

Ctrl+C to exit the watch. Confirm the running image and hit the health endpoint:

```bash
kubectl get pods -n dev -l app.kubernetes.io/name=auth-service \
  -o jsonpath='{.items[0].spec.containers[0].image}'
# expect: <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/auth-service:sha-abc1234

kubectl port-forward -n dev deploy/auth-service 8081:8081 &
curl http://localhost:8081/actuator/health   # expect {"status":"UP"}
kill %1
```

---

## Part 6 — ArgoCD self-heal demo (optional)

The `auth-service-dev` Application ships with `selfHeal` commented out. Turn it on to see
GitOps drift-correction live:

```bash
argocd app set auth-service-dev --self-heal
argocd app get auth-service-dev | grep -A5 "Sync Policy"
# Auto-Sync: enabled (Prune=true, SelfHeal=true)

kubectl delete deployment auth-service -n dev
watch argocd app get auth-service-dev
```

Within ~3 minutes ArgoCD notices `OutOfSync / Missing`, re-applies the manifest, and the
pod comes back — nobody ran `kubectl apply`. Same story if you `kubectl scale
deployment auth-service -n dev --replicas=3`: it reverts to whatever `replicaCount` the
Helm values say.

---

## Part 7 — Promote to QA

QA is a **single shared** ArgoCD app (`pharma-qa`) watching the whole `envs/qa/`
directory — not one app per service.

1. Open your `zen-gitops` fork → **Pull requests** → `promote(qa): auth-service →
   sha-abc1234` (opened by Step 4.3's `open-qa-pr` job)
2. Review the diff — should be a single `image.tag` line
3. **Merge pull request**

```bash
argocd app get pharma-qa
kubectl get pods -n qa -w
```

`pharma-qa` auto-syncs (with `selfHeal: true`) within ~3 minutes of the merge.

```bash
kubectl get pods -n qa -l app.kubernetes.io/name=auth-service \
  -o jsonpath='{.items[0].spec.containers[0].image}'

kubectl port-forward -n qa deploy/auth-service 8082:8081 &
curl http://localhost:8082/actuator/health
kill %1
```

---

## Part 8 — Promote to PROD

PROD promotion is a **separate, manually-triggered** workflow — nothing you did in Parts
4–7 pushes to PROD by itself.

### Step 8.1 — Run promote-prod.yml

1. **Actions → Promote to PROD → Run workflow**
2. Choose `auth-service` from the **service** dropdown (this triggers the `prod`
   environment's required-reviewer gate from Step 2.4 first)
3. Once approved, the job:
   - Reads `.image.tag` out of your `zen-gitops` fork's `envs/qa/values-auth-service.yaml`
     — whatever is running in QA right now is exactly what gets promoted, no manual tag entry
   - Fails loudly if `envs/prod/values-auth-service.yaml` doesn't exist yet
   - Opens `promote(prod): auth-service → sha-abc1234` in your `zen-gitops` fork
     (branch `promote/prod/auth-service/<tag>`), with a pre-merge checklist in the PR body

### Step 8.2 — Merge and sync manually

The PROD ArgoCD app (`pharma-prod`) has **no automated sync policy** — this is intentional.
Merging the PR only updates git; nothing deploys until you say so:

```bash
argocd app get pharma-prod
# Sync Status:   OutOfSync
# Health Status: Missing   (first time) / Healthy (subsequent promotions)

argocd app diff pharma-prod        # inspect exactly what will change
argocd app sync pharma-prod --prune
argocd app wait pharma-prod --health --timeout 300
```

```bash
kubectl get pods -n prod -w
kubectl port-forward -n prod deploy/auth-service 8083:8081 &
curl http://localhost:8083/actuator/health
kill %1
```

`pharma-prod` also has `ignoreDifferences` on `spec.replicas` — an on-call engineer can
`kubectl scale` in prod during an incident without ArgoCD fighting them; it won't
auto-revert like DEV would.

---

## Part 9 — GitHub Secrets / Variables / Environments reference

| Kind | Name | Used by | Notes |
|---|---|---|---|
| Secret | `AWS_ACCOUNT_ID` | all `ci-*.yml` | 12-digit account ID, builds the ECR URL and role ARN |
| Secret | `GITOPS_TOKEN` | all `ci-*.yml`, `promote-prod.yml` | PAT with `contents:write` + `pull_requests:write` on your `zen-gitops` fork |
| Secret | `NVD_API_KEY` | `_java-build.yml`, `_java-pr-check.yml` | optional — OWASP step is currently disabled repo-wide, so this has no effect yet |
| Variable | `GITOPS_REPO` | all `ci-*.yml`, `promote-prod.yml` | `<your-username>/zen-gitops` |
| Environment | `dev` | `deploy-dev` job | no protection rule |
| Environment | `prod` | `promote-prod.yml` job | required reviewers + prevent self-review |

---

## Part 10 — Repeat for the other 7 services (outline)

Each service follows the **exact same recipe** as auth-service in Parts 4, 7, 8 — you don't
repeat Parts 1–3 or the secrets/variables/environments in Part 2/9, those are repo-wide.
Push to `feat-*` → PR check runs; merge to `develop` → full build + deploy-dev + open-qa-pr
run; run `promote-prod.yml` and pick the service from the dropdown to go to PROD. What
differs per service:

| Service | `service-dir` | Reusable workflow | DB in tests | ECR repo | zen-gitops values name | In `promote-prod.yml` dropdown? |
|---|---|---|---|---|---|---|
| api-gateway | `api-gateway` | `_java-build.yml` | No | `api-gateway` | `api-gateway` | Yes |
| auth-service | `auth-service` | `_java-build.yml` | Yes | `auth-service` | `auth-service` | Yes |
| drug-catalog-service | `drug-catalog-service` | `_java-build.yml` | Yes | `drug-catalog-service` | **`catalog-service`** ⚠️ | Yes (as `catalog-service`) |
| inventory-service | `inventory-service` | `_java-build.yml` | Yes | `inventory-service` | `inventory-service` | Yes |
| manufacturing-service | `manufacturing-service` | `_java-build.yml` | Yes | `manufacturing-service` | `manufacturing-service` | Yes |
| supplier-service | `supplier-service` | `_java-build.yml` | Yes | `supplier-service` | `supplier-service` | Yes |
| notification-service | `notification-service` | `_node-build.yml` (Node 20) | No | `notification-service` | `notification-service` | Yes |
| qc-service | `qc-service` | `_java-build.yml` | No | `qc-service` | `qc-service` | **No** ⚠️ — not yet added to the dropdown in `promote-prod.yml` |

⚠️ Two things to watch for:

- **`drug-catalog-service`** — its `ci-drug-catalog.yml` sets `GITOPS_SERVICE_NAME:
  catalog-service`, because the GitOps repo was bootstrapped with that Helm release name
  before the source repo settled on `drug-catalog-service`. Its QA/PROD PR branches and
  values file paths use `catalog-service`, not `drug-catalog-service`. The ECR repo and
  Docker image are still `drug-catalog-service`.
- **`qc-service`** — CI (`ci-qc-service.yml` / `ci-pr-qc-service.yml`) exists and runs DEV
  deploy + open-qa-pr like every other service, but it's missing from the `options:` list
  in `promote-prod.yml`. To promote it to PROD, add `- qc-service` to that dropdown first
  (see `QC-SERVICE-DEPLOYMENT.md` for the full onboarding story of this service).

To onboard an entirely new service beyond these 8, see `CI-ARCHITECTURE.md` → FAQ → "How do
I onboard a new microservice?".

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Workflow doesn't start on push | Actions disabled on the fork | Fork → **Actions** tab → enable workflows |
| Job fails: `Error assuming role` / `AccessDenied` on `sts:AssumeRoleWithWebIdentity` | `AWS_ACCOUNT_ID` wrong, or your fork isn't in the role's trust policy | Confirm the account ID; in Mode B check the trust policy's `sub` list includes your fork |
| Job fails: `denied: requested access to the resource is denied` on ECR push | ECR repo doesn't exist yet, or wrong region | `aws ecr describe-repositories --region us-east-1`; create it via `zen-infra` Terraform if missing |
| `deploy-dev` / `open-qa-pr` fails: `Authentication failed` against `zen-gitops` | `GITOPS_TOKEN` expired or wrong repo scope | Regenerate the PAT with Contents + Pull requests write scoped to your `zen-gitops` fork |
| `deploy-dev` / `open-qa-pr` fails: file not found (`envs/dev/values-*.yaml`) | You skipped Part 1.2's personalisation, or the values file genuinely doesn't exist for that service in your fork | Check `zen-gitops/envs/<env>/values-<service>.yaml` exists; `open-qa-pr` skips cleanly with a `::warning::` if the QA file is missing, it does not fail the build |
| Maven tests fail: `Connection refused` to DB | Postgres sidecar not up, or `needs-database: true` missing for that service | Check the `ci-<service>.yml` passes `needs-database: true` to the reusable workflow |
| Pod `CrashLoopBackOff` in dev/qa/prod | Spring Boot can't reach RDS | `kubectl logs -n <env> deploy/<service> --previous`; check `DB_HOST` in that env's values file matches your RDS endpoint |
| Pod `ImagePullBackOff` | ECR auth issue, or `zen-gitops` still has the instructor's AWS account ID | Re-check Part 1.2/Part 3 — `image.repository` must contain **your** account ID |
| `promote-prod.yml` run fails immediately, no reviewer prompt shown | `prod` GitHub Environment not created in your fork yet | Settings → Environments → create `prod` with required reviewers (Part 2.4) |
| `promote-prod.yml` fails: `QA values file not found` | Service never had a QA promotion merged, so `envs/qa/values-<service>.yaml` was never created/updated | Merge the QA promotion PR for that service first |
| `promote-prod.yml` service missing from dropdown | Not yet added to `options:` in `promote-prod.yml` (currently true for `qc-service`) | Edit `promote-prod.yml`, add the service name used in `zen-gitops` (e.g. `catalog-service`, not `drug-catalog-service`) |
| ArgoCD shows `OutOfSync` and never heals | `selfHeal` not enabled (dev is `selfHeal: false` by default) | `argocd app set auth-service-dev --self-heal`, or just re-run `argocd app sync` |
| `pharma-prod` stuck `OutOfSync` after merge | Prod uses **manual sync** by design | `argocd app sync pharma-prod --prune` |
| `argocd: command not found` | CLI not installed | `brew install argocd` (Mac) or see the [ArgoCD CLI docs](https://argo-cd.readthedocs.io/en/stable/cli_installation/) |
