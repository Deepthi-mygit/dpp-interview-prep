# GitOps & ArgoCD — Interview Questions

> Grounded in the zen-gitops repo: ArgoCD app-of-apps, Helm values promotion, multi-env sync policies, External Secrets.
> 22 questions — conceptual openers through full architecture and disaster recovery scenarios.

---

## Table of Contents

1. [GitOps Concepts](#1-gitops-concepts) — Q1–Q6
2. [Deep Scenario Questions](#2-deep-scenario-questions) — Q7–Q14
3. [Architecture & DR Questions](#3-architecture--dr-questions) — Q15–Q16
4. [Quick Conceptual Checks](#4-quick-conceptual-checks) — Q17–Q22

---

## 1. GitOps Concepts

---

**Q: What is GitOps and how does it differ from traditional CI/CD?**

- **What is being tested:** Whether you understand GitOps as a pull-based declarative model — not just "we use Git."
- **Strong answer:** In traditional CI/CD, the pipeline pushes changes to the cluster using `kubectl apply` or Helm commands. Once the pipeline finishes, there is no ongoing guarantee that the cluster stays in the desired state — someone could `kubectl edit` a deployment manually and drift from what CI deployed. GitOps inverts this. Git is the single source of truth for desired state. ArgoCD runs inside the cluster and continuously pulls from Git, comparing desired state against actual cluster state. Any drift is detected and self-healed automatically. Every change to the cluster is a Git commit — auditable, reversible, and author-attributed.
- **Project context:** In our project, GitHub Actions never runs `kubectl` directly. It only commits to `chandika-s/zen-gitops`. ArgoCD watches that repo and syncs the cluster. If someone manually changes a deployment in Kubernetes, ArgoCD detects the drift and reverts it to what's in Git.

---

**Q: Explain ArgoCD's core components and how they work together.**

- **What is being tested:** Internal architecture knowledge, not just "ArgoCD deploys things."
- **Strong answer:** ArgoCD has four main components. The **API Server** exposes the gRPC/REST interface used by the UI and CLI. The **Repository Server** clones Git repos and renders Helm/Kustomize manifests into raw Kubernetes manifests. The **Application Controller** is a Kubernetes controller that runs the reconciliation loop — it continuously compares the live cluster state against the rendered desired state and marks apps as Synced, OutOfSync, or Degraded. **Redis** caches rendered manifests and cluster state to reduce API server load. ArgoCD uses Kubernetes CRDs — `Application`, `AppProject`, `ApplicationSet` — so its own state is stored in etcd, making it self-healing and HA-capable.

---

**Q: What is the difference between ArgoCD auto-sync and manual sync? When do you use each?**

- **What is being tested:** Whether you understand the operational implications of auto vs. manual sync.
- **Strong answer:** Auto-sync means ArgoCD applies changes to the cluster as soon as it detects a diff between Git and the cluster — typically within the poll interval (default 3 minutes). Manual sync means ArgoCD detects the diff but waits for a human to click Sync in the UI or run `argocd app sync`. Use auto-sync for lower environments where fast feedback is the goal. Use manual sync for production — production changes should happen at a planned maintenance window with an engineer watching the rollout, not automatically at 2am because someone merged a PR.
- **Project context:** DEV uses auto-sync (`pharma-dev`), QA uses auto-sync (`pharma-qa`) because the gate is the PR merge in zen-gitops — once the QA team merges the PR they want it deployed immediately. PROD uses manual sync (`pharma-prod`) — after the PROD PR merges, ArgoCD shows OutOfSync, and an engineer triggers sync at the maintenance window.

---

**Q: How do you handle a rollback in a GitOps model?**

- **What is being tested:** Understanding that rollback is a git revert, not a kubectl rollout undo.
- **Strong answer:** In GitOps, a rollback is simply reverting the Git commit that updated the image tag in the values file. You `git revert` the promotion commit in zen-gitops, merge it, and ArgoCD syncs the cluster back to the previous image. This is safer than `kubectl rollout undo` because it goes through the same reconciliation loop as a forward deployment, and the rollback is also captured in Git history with an author and timestamp. You can also roll back to any arbitrary previous version by changing the image tag to any previously-pushed SHA.

---

**Q: What is ArgoCD ApplicationSet and when would you use it?**

- **What is being tested:** Whether you know ArgoCD beyond the basics.
- **Strong answer:** ApplicationSet is an ArgoCD CRD that generates multiple Application resources from a template, using generators like List, Cluster, Git directory, or Pull Request generators. Instead of maintaining 7 separate ArgoCD Application YAMLs for 7 services, you write one ApplicationSet with a list generator and it creates and manages all 7 automatically. It is particularly useful in a monorepo pattern where each subdirectory is a service — the Git directory generator discovers them automatically.

---

**Q: How does ArgoCD handle secrets? Can you store secrets in Git?**

- **What is being tested:** Security awareness around GitOps and secrets.
- **Strong answer:** You should never store plain secrets in Git, even in a private repo. ArgoCD integrates with external secrets managers. The common patterns are: **Sealed Secrets** (Bitnami) — secrets are encrypted with a cluster-specific key and the encrypted blob is safe to commit; **External Secrets Operator** — ArgoCD syncs a CRD that tells the operator to fetch the secret from AWS Secrets Manager or HashiCorp Vault; **Vault Agent Injector** — secrets are injected into pods as environment variables or files at runtime without ever touching Git.

---

## 2. Deep Scenario Questions

---

## A1

### Question
> "Walk me through how your GitOps pipeline works end-to-end — from a developer pushing code to it running in production."

### What the interviewer is really testing
- Do you understand the full pipeline or just one slice of it?
- Can you articulate the separation between CI (build) and CD (deploy)?
- Do you know how Git becomes the single source of truth?

---

### Model Answer

Our pipeline has two distinct halves — **CI** (build, test, publish) and **CD** (deploy via GitOps). They are intentionally decoupled.

---

### Full end-to-end flow

```
Developer
    │
    ├─ 1. Writes code, opens PR to application repo (e.g. auth-service)
    │
    ▼
CI Pipeline (GitHub Actions / GitLab CI)
    ├─ 2. Build: docker build → image tagged sha-<commit>
    ├─ 3. Test: unit + integration tests
    ├─ 4. Scan: Trivy image scan, SAST
    ├─ 5. Push: docker push → ECR
    │        516209541629.dkr.ecr.us-east-1.amazonaws.com/auth-service:sha-dbbb634
    │
    └─ 6. Promote: open PR to zen-gitops repo
              Updates: envs/dev/values-auth-service.yaml
              image.tag: sha-dbbb634
    │
    ▼
zen-gitops repo (THIS REPO)
    ├─ 7. PR reviewed, merged to main
    │
    ▼
ArgoCD (watching zen-gitops main branch)
    ├─ 8. Detects change in envs/dev/values-auth-service.yaml
    ├─ 9. Runs: helm template pharma-service + dev values
    ├─ 10. Applies diff to dev namespace in cluster
    ├─ 11. Kubernetes rolls out new Deployment (rolling update)
    │
    ▼
Promotion to QA
    ├─ 12. After dev validation, CI opens PR: promote/qa/auth-service/sha-dbbb634
    │        Updates: envs/qa/values-auth-service.yaml → tag: sha-dbbb634
    ├─ 13. QA team reviews + approves PR
    ├─ 14. Merge triggers ArgoCD sync to qa namespace
    │
    ▼
Promotion to Prod (same pattern)
    ├─ 15. Release manager opens PR: promote/prod/auth-service/sha-dbbb634
    │        Updates: envs/prod/values-auth-service.yaml
    ├─ 16. Senior engineer + release manager approve (2-person rule)
    └─ 17. ArgoCD syncs to prod namespace
```

---

### Architecture Diagram

```
┌──────────────────┐     push image      ┌──────────────────────────────────┐
│  App Repo        │ ──────────────────► │  AWS ECR                         │
│  auth-service    │                     │  :sha-dbbb634                    │
└──────────────────┘                     └──────────────────────────────────┘
        │                                              ▲
        │ CI updates tag                               │ pull image
        ▼                                              │
┌──────────────────┐                     ┌────────────┴─────────────────────┐
│  zen-gitops      │ ◄── ArgoCD polls ── │  ArgoCD                          │
│  (this repo)     │     every 3 min     │                                  │
│                  │                     │  pharma project                  │
│  envs/dev/       │                     │  ├── dev  namespace apps         │
│  envs/qa/        │                     │  ├── qa   namespace apps         │
│  envs/prod/      │                     │  └── prod namespace apps         │
└──────────────────┘                     └──────────────────────────────────┘
                                                       │
                                                       │ kubectl apply
                                                       ▼
                                         ┌─────────────────────────────────┐
                                         │  EKS Cluster                    │
                                         │  ├── dev namespace              │
                                         │  ├── qa  namespace              │
                                         │  └── prod namespace             │
                                         └─────────────────────────────────┘
```

---

### Key GitOps principles to state

1. **Git is the single source of truth** — nobody runs `kubectl apply` manually; everything flows through a PR
2. **CI and CD are separated** — CI touches the app repo; CD is triggered by changes to zen-gitops
3. **Declarative** — the repo describes desired state, not imperative steps
4. **Auditable** — every change to every environment has a PR, a reviewer, a merge commit
5. **Rollback = git revert** — reverting a PR is a rollback, no special tooling needed

---

## A2

### Question
> "How does ArgoCD detect and handle configuration drift? What happens when someone manually changes a resource in the cluster?"

### What the interviewer is really testing
- Understanding of desired state vs live state reconciliation
- How `selfHeal` and `prune` work
- Real-world war story awareness — drift is a common incident cause

---

### Model Answer

**Drift** = the cluster's live state no longer matches what Git says it should be.

---

### How ArgoCD detects drift

ArgoCD runs two continuous processes:

**1. Git polling (every 3 minutes by default)**
```
ArgoCD repo-server → fetches latest commit from zen-gitops
                   → renders Helm templates with env values
                   → produces desired manifests
```

**2. Live state watch (via Kubernetes informers)**
```
ArgoCD application-controller → watches all managed resources via K8s API
                               → caches live state in memory
```

ArgoCD compares desired (from Git) vs live (from cluster) using a **three-way diff**:
- Last applied state (stored in `kubectl.kubernetes.io/last-applied-configuration` annotation)
- Current live state
- Desired state from Git

If they differ, the app status becomes `OutOfSync`.

---

### What happens when someone manually changes a resource

**Scenario:** A developer panics at 3am and runs:
```bash
kubectl set image deployment/auth-service auth-service=...auth-service:hotfix -n prod
```

**With `selfHeal: false` (manual sync — this repo's likely default for prod):**
```
1. ArgoCD detects the image tag mismatch within ~3 minutes
2. App status → OutOfSync
3. ArgoCD UI shows the diff (desired: v1.0.0 vs live: hotfix)
4. Sends notification (if configured)
5. Cluster stays in drifted state until an operator manually syncs
6. The manual change persists until someone acts
```

**With `selfHeal: true` (automatic reconciliation):**
```
1. ArgoCD detects drift within ~3 minutes
2. Immediately reverts: applies the manifest from Git
3. The manual change is OVERWRITTEN
4. App returns to Synced state
5. Developer's hotfix is gone
```

---

### Sync policy options

```yaml
# In ArgoCD Application manifest
spec:
  syncPolicy:
    automated:
      prune: true      # delete resources removed from Git
      selfHeal: true   # revert manual cluster changes
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
```

| Policy | Meaning | Recommended for |
|--------|---------|-----------------|
| `automated` disabled | Manual sync only | prod — human gate required |
| `automated` + `selfHeal: false` | Auto-sync new changes, tolerate manual tweaks | qa |
| `automated` + `selfHeal: true` | Full reconciliation loop | dev |
| `prune: true` | Remove resources deleted from Git | All envs |

---

### Detecting drift proactively

```bash
# Check all apps for drift
argocd app list | grep OutOfSync

# See exactly what drifted
argocd app diff auth-service-prod

# Example output:
# === apps/Deployment/prod/auth-service ===
# -  image: ...auth-service:v1.0.0
# +  image: ...auth-service:hotfix
```

---

### The key insight to give in the interview

> "Drift is not just about image tags. ConfigMaps, resource limits, labels, annotations — anything can drift. In a pharma environment where every prod change needs an audit trail, selfHeal on prod is actually essential compliance tooling: it guarantees the cluster always reflects what's in Git, which is what your change management system approved."

---

## A3

### Question
> "How do you handle multi-environment promotions in GitOps — how does a change go from dev → qa → prod in your setup?"

### What the interviewer is really testing
- Understanding of the promote/* branch pattern (visible in this repo's branches)
- How to enforce environment gates
- Separation of concerns between environments

---

### Model Answer

This repo uses a **promote branch pattern** — visible in the remote branches:
```
remotes/origin/promote/qa/auth-service/sha-c11e73c
remotes/origin/promote/qa/auth-service/sha-d0430b8
remotes/origin/promote/prod/auth-service/sha-a0017b8
remotes/origin/promote/prod/auth-service/latest
```

Each branch name encodes: `promote/<env>/<service>/<image-sha>`

---

### How it works step by step

**Phase 1: Dev deployment (automated)**
```
1. CI builds auth-service image → sha-dbbb634
2. CI script runs:
   git checkout -b promote/dev/auth-service/sha-dbbb634
   # updates envs/dev/values-auth-service.yaml: tag: sha-dbbb634
   git commit -m "promote(dev): auth-service → sha-dbbb634"
   git push origin promote/dev/auth-service/sha-dbbb634
   gh pr create --base main --title "promote(dev): auth-service → sha-dbbb634"
3. PR auto-merges (no approval required for dev)
4. ArgoCD detects main branch change → syncs dev namespace
```

**Phase 2: QA promotion (semi-automated)**
```
5. After dev smoke tests pass (manual or automated E2E):
6. CI or engineer runs promote script:
   git checkout -b promote/qa/auth-service/sha-dbbb634
   # updates envs/qa/values-auth-service.yaml: tag: sha-dbbb634
   git push → PR created
7. QA lead reviews + approves PR
8. Merge → ArgoCD syncs qa namespace
```

**Phase 3: Prod promotion (gated)**
```
9. After QA sign-off:
10. Release manager creates:
    promote/prod/auth-service/sha-dbbb634
11. Requires: 2 approvals (release manager + senior engineer)
12. CODEOWNERS for envs/prod/ enforces this
13. Merge → ArgoCD syncs prod namespace (manual sync policy)
14. Operator watches rollout:
    kubectl rollout status deployment/auth-service -n prod
```

---

### Environment comparison (auth-service actual values)

| Setting | dev | qa | prod |
|---------|-----|----|------|
| Image tag | `sha-dbbb634` | `sha-c11e73c` | `v1.0.0` |
| Spring profile | `dev` | `qa` | `prod` |
| Log level | `DEBUG` | `INFO` | `DEBUG` |
| DB host | `pharma-dev-postgres...` | `<RDS_ENDPOINT>` | `pharma-prod-postgres...` |
| Autoscaling | disabled | enabled (1-3) | disabled |
| fullnameOverride | set | not set | set |
| tmp volume | present | absent | present |

---

### Architecture Diagram

```
            ┌─────────────────────────────────────┐
            │  zen-gitops repo (main branch)       │
            │                                     │
            │  envs/dev/values-auth-service.yaml  │ ← auto-merged
            │  envs/qa/values-auth-service.yaml   │ ← QA approval
            │  envs/prod/values-auth-service.yaml │ ← 2-person approval
            └──────────────┬──────────────────────┘
                           │  ArgoCD watches
              ┌────────────┼────────────┐
              ▼            ▼            ▼
           dev ns        qa ns       prod ns
         (sync: auto)  (sync: auto) (sync: manual)
```

---

### Branch protection rules (what you'd add to GitHub)

```yaml
# .github/CODEOWNERS
envs/prod/   @pharma-release-managers @senior-sre-team
envs/qa/     @pharma-qa-team
envs/dev/    @pharma-developers

# Branch protection on main:
# - require PR review
# - require CODEOWNERS approval
# - require CI to pass
# - prod/* paths require 2 approvals
```

---

## A5

### Question
> "Your ArgoCD project uses `clusterResourceWhitelist: group:* kind:*`. What are the security implications and how would you harden this in a regulated environment like pharma?"

### What the interviewer is really testing
- Security awareness in GitOps
- Understanding of ArgoCD's project-level isolation model
- Pragmatic approach to security hardening in regulated environments

---

### Model Answer

From `argocd/projects/pharma-project.yaml`:
```yaml
clusterResourceWhitelist:
  - group: "*"
    kind: "*"
namespaceResourceWhitelist:
  - group: "*"
    kind: "*"
```

**This means:** Any ArgoCD Application within the `pharma` project can create, modify, or delete **any Kubernetes resource** — including `ClusterRole`, `ClusterRoleBinding`, `PersistentVolume`, `CustomResourceDefinition`, `ValidatingWebhookConfiguration`, and even `Namespace`.

---

### Security implications

| Risk | Scenario |
|------|----------|
| **Privilege escalation** | A compromised app in zen-gitops creates a `ClusterRole` with admin rights and binds it to a new SA |
| **Namespace escape** | An app deploys a `ClusterRoleBinding` that lets it access prod secrets from dev |
| **Persistence** | Attacker adds a `ValidatingWebhookConfiguration` that intercepts all cluster traffic |
| **Data exfiltration** | App creates a `PersistentVolume` mounting sensitive node paths |
| **Supply chain** | A malicious PR to zen-gitops installs a CRD that exfiltrates data |

In pharma, these risks are amplified because:
- Patient data may be in the cluster
- GxP validation requires you to prove no unauthorized changes were made
- A ClusterRole escalation could bypass all the namespace-scoped RBAC controls we built

---

### How to harden it

**Step 1: Restrict cluster-level resources to only what's needed**

```yaml
clusterResourceWhitelist:
  # ArgoCD needs namespaces to manage its apps
  - group: ""
    kind: Namespace
  # External Secrets Operator ClusterSecretStore
  - group: "external-secrets.io"
    kind: ClusterSecretStore
  # Cert-manager ClusterIssuer
  - group: "cert-manager.io"
    kind: ClusterIssuer
  # Prometheus cluster-level rules
  - group: "monitoring.coreos.com"
    kind: ClusterRole
  # Explicitly NO: ClusterRole, ClusterRoleBinding, ValidatingWebhookConfiguration
```

**Step 2: Lock down namespace resources per project**

```yaml
namespaceResourceWhitelist:
  - group: "apps"
    kind: Deployment
  - group: "apps"
    kind: ReplicaSet
  - group: ""
    kind: Service
  - group: ""
    kind: ConfigMap
  - group: ""
    kind: ServiceAccount
  - group: "networking.k8s.io"
    kind: Ingress
  - group: "autoscaling"
    kind: HorizontalPodAutoscaler
  - group: "external-secrets.io"
    kind: ExternalSecret
  - group: "monitoring.coreos.com"
    kind: ServiceMonitor
  # Explicitly excluded: Secret (managed by ESO), Role, RoleBinding
```

**Step 3: Create separate projects per environment with different whitelist**

```yaml
# pharma-prod project — most restrictive
clusterResourceWhitelist: []   # no cluster-level changes in prod
namespaceResourceWhitelist:
  - group: "apps"
    kind: Deployment
  # ... only what prod needs
```

**Step 4: Enable ArgoCD audit logging**
```yaml
# In argocd-cm ConfigMap
data:
  resource.customizations: |
    # log all resource changes
  server.rbac.log.enforce.enable: "true"
```

---

### What to say to close the answer

> "The wildcard is a pragmatic shortcut that works fine when you have full trust in everything that goes into zen-gitops. In a regulated pharma environment that's not enough — you need the platform to enforce least-privilege even if a PR slips through review. I'd start with the namespace whitelist restriction, which stops 90% of the attack surface, and layer in the cluster-level restriction after cataloguing all the CRDs we actually deploy."

---

## A6

### Question
> "How do you do a rollback in GitOps? Walk me through both an ArgoCD UI rollback and a Git-based rollback."

### What the interviewer is really testing
- Practical rollback knowledge — a question always asked at senior/staff level
- Understanding that in GitOps, rollback = changing Git state
- Awareness of the two approaches and when each is appropriate

---

### Model Answer

**There are two rollback mechanisms. They are NOT equivalent.**

---

### Method 1: ArgoCD UI / CLI Rollback (fast, temporary)

This reverts the cluster state to a previous ArgoCD sync revision — **without changing Git**.

```bash
# List history of an application
argocd app history auth-service-prod

# Output:
# ID  DATE                           REVISION
# 3   2026-05-07 10:22:15 +0000 UTC  main (7ad8ac0)  ← current (broken)
# 2   2026-05-06 14:10:03 +0000 UTC  main (4de3cd7)  ← last known good
# 1   2026-05-05 09:15:00 +0000 UTC  main (12c197e)

# Rollback to revision 2
argocd app rollback auth-service-prod 2

# ArgoCD re-applies the manifests from that revision to the cluster
# Status becomes: OutOfSync (because Git still shows newer broken state)
```

**When to use:** Immediate production fire. Gets you stable in 60 seconds.

**Danger:** Git still has the broken commit. If ArgoCD auto-syncs or someone manually syncs, the broken version comes back. You MUST follow up with a Git rollback.

---

### Method 2: Git Revert (permanent, compliant)

This is the correct GitOps rollback — it changes the source of truth.

```bash
# Identify the bad commit
git log --oneline envs/prod/values-auth-service.yaml

# 7ad8ac0 promote(prod): auth-service → sha-a0017b8   ← broke prod
# 4de3cd7 promote(prod): auth-service → sha-c11e73c   ← last good

# Revert the bad commit
git revert 7ad8ac0 --no-edit
# Creates new commit: "Revert promote(prod): auth-service → sha-a0017b8"

# Push and create PR
git push origin revert/prod-auth-service-rollback
gh pr create --base main \
  --title "revert(prod): auth-service rollback" \
  --body "Rolling back sha-a0017b8 due to [incident-link]"

# After PR merges → ArgoCD auto-detects change → syncs prod → rollback complete
```

**When to use:** After the immediate fire is out. This is the permanent fix and the audit trail entry.

---

### Real incident from this repo

Looking at the git log:
```
7ad8ac0 Merge pull request #64 - Revert "promote(prod): auth-service → sha-a0017b8"
6132a31 Revert "promote(prod): auth-service → sha-a0017b8"
1e9e1a2 Merge pull request #63 - promote(prod): auth-service → sha-a0017b8
4de3cd7 promote(prod): auth-service → sha-a0017b8
```

This repo has a real rollback in its history — PR #63 deployed `sha-a0017b8` to prod, then PR #64 reverted it. This is the Git-based rollback pattern in action.

---

### Decision flowchart

```
Production incident — need rollback
           │
           ├─ Is prod actively broken RIGHT NOW?
           │      YES → argocd app rollback <app> <last-good-revision>
           │             (cluster stable in 60s)
           │             THEN → follow up with git revert
           │
           └─ Can wait 10-15 min for PR process?
                  YES → git revert → PR → merge
                         (ArgoCD syncs automatically)
                         (preferred — creates audit trail)
```

---

### Key point to make in interview

> "In GitOps, the ArgoCD rollback is an emergency brake — it buys you time. The Git revert is the actual rollback — it's what your change management audit will reference. In a pharma environment under GxP, you want the Git revert to be the primary record: who approved rolling back, what was reverted, and why."

---

## A7

### Question
> "How would you implement a blue-green or canary deployment strategy using ArgoCD?"

### What the interviewer is really testing
- Knowledge of Argo Rollouts (the natural companion to ArgoCD)
- Understanding of traffic splitting mechanisms (weighted services, ingress annotations)
- Knowing when to use blue-green vs canary

---

### Model Answer

Standard ArgoCD + Helm only supports **rolling updates** (replace pods incrementally). For blue-green or canary, you need **Argo Rollouts**.

---

### Blue-Green Deployment

Two full environments run simultaneously; traffic switches atomically.

```yaml
# Replace Deployment with Rollout
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: auth-service
  namespace: prod
spec:
  replicas: 3
  selector:
    matchLabels:
      app: auth-service
  template:
    # ... same pod spec as Deployment
  strategy:
    blueGreen:
      activeService: auth-service-active      # receives 100% traffic
      previewService: auth-service-preview    # new version goes here first
      autoPromotionEnabled: false             # require manual promotion
      scaleDownDelaySeconds: 30              # keep old version for 30s after switch
```

```yaml
# Two services required
---
apiVersion: v1
kind: Service
metadata:
  name: auth-service-active    # Nginx ingress points here
spec:
  selector:
    app: auth-service
    # Argo Rollouts adds: rollouts-pod-template-hash: <blue-hash>
---
apiVersion: v1
kind: Service
metadata:
  name: auth-service-preview   # QA tests against this
spec:
  selector:
    app: auth-service
    # Argo Rollouts adds: rollouts-pod-template-hash: <green-hash>
```

**Promotion flow:**
```bash
# New image deployed to preview (green)
# Run smoke tests against auth-service-preview
# Promote: switch all traffic to green
kubectl argo rollouts promote auth-service -n prod

# If bad, abort: switch back to blue instantly
kubectl argo rollouts abort auth-service -n prod
```

---

### Canary Deployment

Gradually shift traffic to new version, monitor, then proceed or rollback.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: catalog-service
  namespace: prod
spec:
  strategy:
    canary:
      canaryService: catalog-service-canary
      stableService: catalog-service-stable
      trafficRouting:
        nginx:
          stableIngress: catalog-service-ingress
      steps:
        - setWeight: 10         # send 10% to canary
        - pause: {duration: 5m} # watch for 5 min
        - setWeight: 30         # increase to 30%
        - pause: {duration: 10m}
        - analysis:             # automated gate: check error rate
            templates:
              - templateName: error-rate-check
        - setWeight: 100        # full rollout
```

```yaml
# AnalysisTemplate — automated quality gate using Prometheus
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: error-rate-check
spec:
  metrics:
    - name: error-rate
      interval: 1m
      failureLimit: 1
      provider:
        prometheus:
          address: http://prometheus-operated.monitoring.svc:9090
          query: |
            sum(rate(http_server_requests_seconds_count{
              app="catalog-service", status=~"5.."
            }[5m])) /
            sum(rate(http_server_requests_seconds_count{
              app="catalog-service"
            }[5m])) < 0.01
```

---

### Blue-Green vs Canary — when to use which

| Scenario | Strategy | Why |
|----------|----------|-----|
| auth-service (critical, fast failover needed) | Blue-Green | Instant rollback, zero traffic on bad version |
| catalog-service (high traffic, gradual risk) | Canary | Limit blast radius to 10% of users |
| pharma-ui (user-facing, A/B testing) | Canary | Test UX changes with subset of users |
| DB schema changes | Blue-Green | Need old version running during migration |

---

## A8

### Question
> "You have a hotfix that must go to prod immediately but the normal PR process takes hours. How do you handle this in GitOps without breaking the process?"

### What the interviewer is really testing
- Maturity in handling the GitOps vs urgency tension
- Whether you have a documented break-glass procedure
- Do you know when to bypass vs when to hold firm

---

### Model Answer

**First principle: the GitOps process exists for good reasons. A break-glass procedure is not an excuse to bypass it permanently — it is a documented, audited exception.**

---

### The break-glass procedure

**Step 1: Declare an incident formally**
```
Incident declared: P1 - auth-service prod returning 500s
Incident commander: [name]
Communication channel: #incident-prod-auth
Time: 2026-05-07 02:14 UTC
```

**Step 2: Choose the fastest safe option**

Option A — ArgoCD rollback (if the current version is broken):
```bash
# Fastest: 60 seconds. No Git change needed.
argocd app rollback auth-service-prod <last-good-revision>
# This is not bypassing GitOps — it's using ArgoCD's built-in rollback
```

Option B — Emergency direct push (true break-glass, new fix needed):
```bash
# Create a branch directly from prod state
git checkout -b hotfix/prod-auth-service-null-ptr main

# Make minimal fix to envs/prod/values-auth-service.yaml
# (e.g., pointing to a pre-built hotfix image)

# Push and create PR with emergency label
gh pr create --base main \
  --title "hotfix(prod): auth-service null pointer in token validation" \
  --label "emergency,break-glass" \
  --body "P1 incident #123. Approved by: @cto @release-manager"
```

**Step 3: Emergency approval**
- Minimum 1 senior approver (relaxed from normal 2)
- Verbal approval logged in incident channel
- PR merged immediately

**Step 4: ArgoCD syncs prod** (if manual sync policy, trigger manually)
```bash
argocd app sync auth-service-prod --force
kubectl rollout status deployment/auth-service -n prod
```

**Step 5: Post-incident (within 24h)**
- Write postmortem
- Create a follow-up PR with proper testing if the hotfix was rushed
- Update runbook if the break-glass procedure needs adjustment

---

### What guardrails you keep even in break-glass

| Guardrail | Keep? | Reason |
|-----------|-------|--------|
| All changes go through Git | ✅ YES | Audit trail is non-negotiable in pharma |
| PR required (no direct push to main) | ✅ YES | One approver minimum |
| CI must pass | ⚠️ OPTIONAL | May skip if emergency image already tested |
| 2-person approval rule | ⚠️ RELAXED | 1 senior approver acceptable, documented |
| Manual prod sync | ✅ YES | Human confirms before ArgoCD applies |

---

### Key line for the interview

> "In GitOps, there's no such thing as bypassing the process — there's only making the process faster. The break-glass procedure is a documented, auditable fast lane, not an exit from the guardrails. The difference between a mature and an immature GitOps setup is whether that fast lane is designed in advance or improvised under pressure at 2am."

---

## 3. Architecture & DR Questions

---

## BS1

### Question
> "Explain the cloud architecture for the Zen Pharma application — from the user request to the database. Walk me through each layer and why you designed it that way."

### What the interviewer is really testing
- Can you explain a real system end-to-end, not just one tool?
- Do you understand separation of concerns (network, compute, data, secrets, delivery)?
- Can you justify trade-offs (single cluster vs multi-cluster, GitOps, IRSA)?

---

### Model Answer

Zen Pharma is a **microservices platform** for pharmaceutical operations: authentication, drug catalog, inventory, manufacturing, QC, suppliers, notifications, and a React UI. The cloud stack is **AWS-centric** with **Kubernetes as the runtime** and **GitOps for delivery**.

---

### High-level architecture (north–south flow)

```
Internet / corporate VPN
         │
         ▼
┌────────────────────────────────────────────────────────────┐
│  AWS (us-east-1)                                           │
│                                                            │
│  Route 53 / internal DNS  →  prod.pharma.internal          │
│         │                                                  │
│         ▼                                                  │
│  NGINX Ingress Controller (in EKS)                         │
│         │  TLS termination, path routing                   │
│         ▼                                                  │
│  api-gateway (ClusterIP)  ──►  backend microservices       │
│         │                      (auth, catalog, inventory,  │
│         │                       manufacturing, qc, etc.)   │
│         ▼                                                  │
│  pharma-ui (React, ClusterIP)                              │
│         │                                                  │
│         ├──────────────────►  AWS RDS PostgreSQL           │
│         │                      (per-env instances/schemas) │
│         │                                                  │
│         └──────────────────►  AWS Secrets Manager          │
│                                via External Secrets (IRSA) │
└────────────────────────────────────────────────────────────┘
```

**Companion repos (split by concern):**

| Repo | Responsibility |
|------|----------------|
| `zen-infra` | Terraform — VPC, EKS, RDS, ECR, IAM, OIDC for IRSA |
| `zen-pharma-backend` | Spring Boot microservices (build + CI) |
| `zen-pharma-frontend` | React UI |
| `zen-gitops` | Desired cluster state — Helm values, ArgoCD apps, RBAC, ESO |

---

### Layer-by-layer breakdown

**1. Edge and ingress**
- Users hit **NGINX Ingress** inside the cluster (`ingress-nginx`), not individual pod IPs.
- Host-based routing (e.g. `prod.pharma.internal`) sends `/` to **pharma-ui** and API paths to **api-gateway**.
- Ingress gives a **stable entry point** while pods are ephemeral.

**2. Application tier (EKS)**
- **One EKS cluster**, **three namespaces**: `dev`, `qa`, `prod` — logical isolation, shared control plane cost.
- **Nine workloads**: api-gateway, auth-service, catalog-service, inventory-service, manufacturing-service, qc-service, supplier-service, notification-service, pharma-ui.
- Each service is a **Deployment + Service + ConfigMap** (from shared Helm chart `helm-charts/`).
- **api-gateway** routes to internal services via Kubernetes DNS (`http://auth-service:8081`, etc.).
- **prod** runs **2+ replicas**, **HPA** (2–5), and **pod anti-affinity** so pods spread across nodes/AZs.

**3. Data tier**
- **PostgreSQL on RDS** — one RDS endpoint per environment (e.g. `pharma-prod-postgres....rds.amazonaws.com`).
- Schemas separated in DB (`pharmacy`, `inventory`, `procurement`, `manufacturing` — see `db-init/01-schemas.sql`).
- Services connect via **ConfigMap** (`DB_HOST`) + **Secret** (`DB_USERNAME`, `DB_PASSWORD` from ESO).
- RDS is in **private subnets**; only the cluster security group can reach port 5432.

**4. Secrets and identity**
- **No secrets in Git.** Credentials live in **AWS Secrets Manager** (`/pharma/<env>/db-credentials`, `jwt-secret`, etc.).
- **External Secrets Operator** syncs them into Kubernetes Secrets every hour.
- **IRSA** — ESO's ServiceAccount assumes `external-secrets-role` via OIDC; no long-lived AWS keys in the cluster.
- Per-service **ServiceAccounts** where apps need AWS API access (annotated with `eks.amazonaws.com/role-arn`).

**5. Container images and registry**
- CI builds images → pushes to **ECR** (`516209541629.dkr.ecr.us-east-1.amazonaws.com/<service>:<tag>`).
- EKS nodes pull via IAM/node role or `imagePullSecrets`; tags are immutable SHAs in lower envs, semver in prod.

**6. Delivery (GitOps)**
- **ArgoCD** watches `zen-gitops` `main` branch.
- **dev**: 8–9 individual ArgoCD Applications, **automated sync + selfHeal**.
- **qa/prod**: **app-of-apps** for atomic multi-service deploys; **prod = manual sync** (human gate after PR merge).
- Promotion: CI opens PRs updating `envs/<env>/values-*.yaml` image tags — audit trail in Git.

**7. Observability**
- **Prometheus** scrapes Spring Boot `/actuator/prometheus` endpoints.
- **Grafana** dashboards (values in `k8s/monitoring/`).
- Health: liveness `/actuator/health`, readiness `/actuator/health/readiness` — failed readiness removes pod from Service endpoints.

**8. Security and governance (cross-cutting)**
- **Namespace RBAC** — dev team cannot `exec` into prod pods.
- **ArgoCD AppProject** scopes which namespaces/repos an app can touch.
- **Branch protection + CODEOWNERS** on `envs/prod/`.
- **Network policies** (where enabled) restrict east–west traffic between namespaces.

---

### Why this design (talking points for the interview)

| Decision | Rationale |
|----------|-----------|
| EKS + namespaces vs 3 clusters | Lower cost and ops burden for training/small prod; namespaces + RBAC give env isolation |
| Microservices + api-gateway | Independent deploy/scale per domain; gateway centralizes auth routing and cross-cutting concerns |
| GitOps (ArgoCD) | Declarative state, PR audit trail, rollback = git revert — important in regulated pharma |
| RDS vs in-cluster Postgres | Managed backups, Multi-AZ option, patching — don't run stateful DB on Kubernetes for prod |
| ESO + Secrets Manager | Secrets rotation and audit in AWS; Git stays free of credentials |
| Shared Helm chart | DRY — one chart, per-service values; consistent probes, labels, HPA patterns |

---

### One-minute elevator version

> "Zen Pharma runs nine containerized services on AWS EKS in us-east-1, split across dev, qa, and prod namespaces. Traffic enters through NGINX Ingress to api-gateway and pharma-ui. Backends are Spring Boot microservices talking to environment-specific RDS PostgreSQL. Secrets come from AWS Secrets Manager via External Secrets and IRSA. Images live in ECR; ArgoCD in the cluster syncs desired state from the zen-gitops repo after CI updates image tags through PRs. Prometheus and Grafana give us observability, and prod changes require CODEOWNERS approval plus manual ArgoCD sync."

---

## BS2

### Question
> "Explain the disaster recovery (DR) architecture for Zen Pharma. What are you protecting against, what are your RTO/RPO targets, and how would you recover from a regional outage or database failure?"

### What the interviewer is really testing
- Do you know DR fundamentals (RTO, RPO, backup, restore, failover)?
- Can you map DR concepts to this specific stack (EKS, RDS, GitOps)?
- Pragmatism — what is actually implemented vs what you would add for true enterprise DR?

---

### Model Answer

**Disaster recovery** is the set of people, processes, and technology that lets you **restore acceptable service** after a failure — from a single pod crash (handled by Kubernetes) to losing an entire AWS Availability Zone or Region.

---

### DR fundamentals (define these first in the interview)

| Term | Definition | Zen Pharma example |
|------|------------|-------------------|
| **RTO** (Recovery Time Objective) | Max acceptable **downtime** before service is restored | Prod target: **4 hours** (business); dev/qa: **24 hours** |
| **RPO** (Recovery Point Objective) | Max acceptable **data loss** measured in time | Prod DB: **15 minutes** (RDS PITR window); app config: **0** (Git is source of truth) |
| **RPO vs backup frequency** | If you backup hourly, worst case you lose ~1 hour of data unless continuous replication | RDS automated backups + transaction logs enable finer RPO |
| **HA** (High Availability) | Stay up during **component** failure (single node, single AZ) | Multi-AZ RDS, EKS across 3 AZs, multiple pod replicas |
| **DR** (Disaster Recovery) | Recover after **large** failure (region loss, corruption, ransomware) | Cross-region backups, runbook, secondary cluster |
| **Backup** | Point-in-time copy of data/config | RDS snapshots, Secrets Manager versioning, Git history |
| **Restore** | Rebuild from backup into a working system | Restore RDS snapshot; ArgoCD re-syncs cluster from Git |
| **Failover** | Switch traffic/workload to standby | DNS cutover to DR region; promote RDS read replica |
| **Failback** | Return to primary after DR event | Sync data back, cut DNS back, decommission DR resources |
| **Cold / Warm / Hot standby** | How ready the DR environment is before disaster | This platform: **warm** app config (Git), **cold/warm** infra (Terraform), RDS cross-region replica optional |

---

### What we protect (assets and failure modes)

| Asset | Failure modes | Primary protection | DR mechanism |
|-------|---------------|-------------------|--------------|
| **Application state** (Deployments, Services) | Cluster loss, bad deploy | Git (`zen-gitops`) + ArgoCD | Re-provision EKS; ArgoCD syncs from Git — **RPO ≈ 0** for manifests |
| **Container images** | ECR regional outage | ECR in us-east-1 | Replicate ECR to `us-west-2`; pull from DR registry |
| **Database** | AZ failure, corruption, accidental DROP | RDS Multi-AZ (HA) | Automated backups + **PITR**; cross-region snapshot copy |
| **Secrets** | Secrets Manager outage, accidental delete | Versioning + IAM | Replicate secrets to DR region; ESO re-sync |
| **Terraform state** | State bucket loss | S3 versioning + locking | Restore state object; `terraform apply` rebuilds infra |
| **ArgoCD config** | argocd namespace lost | Git (`argocd/` in zen-gitops) | Reinstall ArgoCD; re-apply Applications from Git |

**What Kubernetes handles without a DR plan:** single pod crash (ReplicaSet), node drain (scheduler + PDB), rolling deploy failure (readiness probe blocks bad rollout).

**What Kubernetes does NOT handle:** entire region unavailable, RDS deleted without backup, ransomware encrypting backups, Git repo deleted without mirror.

---

### Current-state architecture (single region — us-east-1)

```
                    PRIMARY REGION: us-east-1
┌─────────────────────────────────────────────────────────────┐
│  VPC (3 AZs: us-east-1a, 1b, 1c)                            │
│                                                             │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐        │
│  │   AZ-1a     │   │   AZ-1b     │   │   AZ-1c     │        │
│  │ EKS nodes   │   │ EKS nodes   │   │ EKS nodes   │        │
│  │ (pods spread│   │             │   │             │        │
│  │  via anti-  │   │             │   │             │        │
│  │  affinity)  │   │             │   │             │        │
│  └──────┬──────┘   └──────┬──────┘   └──────┬──────┘        │
│         └─────────────────┼─────────────────┘               │
│                           │                                 │
│              ┌────────────▼────────────┐                    │
│              │  RDS PostgreSQL         │                    │
│              │  Multi-AZ (sync standby)│  ← HA within region │
│              │  Automated backups      │                    │
│              │  PITR (35-day window)   │                    │
│              └─────────────────────────┘                    │
└─────────────────────────────────────────────────────────────┘

DR REGION (target): us-west-2  [documented / partially implemented]
┌─────────────────────────────────────────────────────────────┐
│  Cross-region RDS snapshot copy (daily)                     │
│  ECR replication (optional)                                 │
│  Terraform-ready VPC + EKS (cold/warm — provision on declare) │
│  ArgoCD → same zen-gitops repo (region-agnostic desired state)│
└─────────────────────────────────────────────────────────────┘
```

---

### Tiered DR strategy (match response to scenario)

**Tier 1 — Component failure (minutes, automated)**
- Pod OOM / crash → Kubernetes restarts; HPA scales if needed.
- Single AZ degradation → EKS schedules pods on healthy AZs; RDS Multi-AZ fails over DB to standby AZ (**~60–120 seconds** for RDS failover).
- Bad deployment → readiness probe fails; rollout pauses; **Git revert + ArgoCD sync** or `argocd app rollback`.

**Tier 2 — Data corruption or logical error (hours, semi-automated)**
- Wrong migration / bad data → **RDS point-in-time restore** to timestamp before incident.
- Steps:
  1. Identify incident time → choose restore time (RPO).
  2. Restore RDS to **new instance** (never overwrite prod in place).
  3. Update `DB_HOST` in `envs/prod/values-*.yaml` via emergency PR.
  4. ArgoCD sync → pods reconnect to restored DB.
  5. Validate data; cut over; decommission broken instance after root-cause analysis.

**Tier 3 — Cluster loss, AZ-wide EKS failure (hours)**
- EKS control plane is regional — usually AWS restores it.
- If worker nodes are gone but control plane OK → scale node group; ArgoCD re-syncs apps from Git (**RTO: 1–2 hours**).
- If entire cluster unusable:
  1. `terraform apply` in `zen-infra` for new EKS (or restore from IaC).
  2. Install ArgoCD, ESO, ingress, Prometheus from `zen-gitops`.
  3. Point Applications at existing RDS endpoint (if RDS survived).
  4. DNS still points to same ALB/ingress once NLB is recreated.

**Tier 4 — Regional disaster (us-east-1 down) (hours to days)**
- Declare disaster; execute **regional DR runbook**.
- **RTO target: 4–8 hours** (depends on whether DR EKS is warm or cold).
- **RPO target: 15 min – 24 hours** (depends on cross-region RDS replica vs snapshot-only).

Runbook outline:
```
1. Incident commander declares Tier-4 DR
2. Restore RDS from latest cross-region snapshot → us-west-2
   (or promote cross-region read replica if configured)
3. Provision DR EKS (Terraform) OR activate warm standby cluster
4. Update env-specific values: DB_HOST, ECR region, ingress DNS
5. ArgoCD sync all prod apps from zen-gitops (unchanged image tags)
6. Route 53 failover: prod.pharma.internal → DR ingress ALB
7. Smoke tests: auth login, catalog read, manufacturing write path
8. Communicate RTO met / known gaps (e.g. in-flight transactions lost)
9. Post-incident: failback plan when primary region returns
```

---

### Backup inventory (what to back up and how often)

| Component | Backup method | Frequency | Retention | Restore test |
|-----------|---------------|-----------|-----------|----------------|
| RDS PostgreSQL | Automated snapshots + PITR | Continuous (WAL) + daily snapshot | 35 days (configurable) | Quarterly restore to QA |
| AWS Secrets Manager | Automatic versioning | On every change | Per AWS policy | Rotate + verify ESO sync |
| zen-gitops | GitHub (main + tags) | Every commit | Indefinite | Clone repo; ArgoCD bootstrap |
| ECR images | Immutable tags + lifecycle | Per CI push | 90 days untagged | Pull by digest in DR |
| Terraform state | S3 versioning | Per apply | 90 days | Restore version; re-apply |
| Prometheus metrics | Short retention in-cluster | N/A for DR | 15 days | Not critical for DR; use CloudWatch/long-term store for audit |

**3-2-1 rule (state in interview):** 3 copies of data, 2 different media, 1 offsite — here: primary RDS + automated snapshot + cross-region snapshot copy.

---

### RTO / RPO summary table (recommended targets)

| Scenario | RPO | RTO | Mechanism |
|----------|-----|-----|-----------|
| Pod / node failure | 0 | < 5 min | K8s self-healing, Multi-AZ |
| RDS AZ failover | ~0 (sync replica) | 1–2 min | RDS Multi-AZ automatic |
| Accidental table drop | 5–15 min | 1–4 hours | RDS PITR to new instance |
| EKS cluster rebuild | 0 (Git) | 2–4 hours | Terraform + ArgoCD bootstrap |
| Full region loss | 15 min – 24 h | 4–8 hours | Cross-region RDS + DR EKS + DNS failover |

---

### GitOps advantage in DR

> "Our **desired application state is in Git**, not only in the cluster. If we lose the entire EKS cluster but still have RDS and ECR, recovery is: rebuild cluster from Terraform, reinstall platform components, point ArgoCD at zen-gitops — and we're back to the last approved prod manifests. That's why RPO for **configuration** is effectively zero. Data RPO is governed by RDS backup strategy."

---

### Testing DR (what mature teams do)

- **Game days** twice a year: simulate RDS restore, cluster rebuild, DNS failover.
- **Chaos engineering** in QA: kill AZ, drain nodes, verify PDB + anti-affinity.
- **Documented runbooks** in Confluence/wiki with exact commands (not tribal knowledge).
- **Measure actual RTO** during tests — if restore took 6 hours, update the SLA or improve automation.

---

### Honest gap analysis (shows maturity)

For this training platform, **full active-active multi-region** is not implemented — that would double cost. What we have:
- Multi-AZ RDS and multi-AZ EKS worker spread
- GitOps config survivability
- RDS automated backup + PITR
- Cross-region DR is **documented/runbook-based** — enable ECR replication + RDS cross-region snapshots in `zen-infra` for production hardening

**Closing line for the interview:**

> "HA keeps us running when something small breaks; DR is how we come back when something big breaks. For Zen Pharma, GitOps gives us fast application recovery; RDS PITR gives us data recovery; and regional DR is the layer we'd activate only for a true region-wide outage — with RTO measured in hours, not seconds, unless we invest in a warm standby cluster."

---

## 4. Quick Conceptual Checks

---

### Q17 — `[7 min]`
> "What are the four principles of GitOps? Give me a real example of each."

**Why this opens the session:** Every other question connects back to these four. If students can't anchor answers here, they'll sound like they're reciting definitions instead of speaking from experience.

**Model answer:**

| Principle | What it means | Real example |
|-----------|--------------|--------------|
| **Declarative** | Describe WHAT, not HOW | Helm values file says `replicas: 3` — we never run `kubectl scale` |
| **Versioned & immutable** | Git is the audit trail | Every image tag change is a commit — `git log` tells you who deployed what and when |
| **Pulled automatically** | Agent inside cluster pulls, CI never pushes | ArgoCD polls GitHub every 3 min — CI pipeline has zero cluster credentials |
| **Continuously reconciled** | Agent detects and corrects drift | Manual `kubectl edit` on prod gets reverted by ArgoCD's selfHeal within minutes |

**Discussion prompt:** "What breaks in your project if you skip the 'pull' principle?" → CI pipeline needs cluster credentials → compromised runner = compromised cluster.

---

### Q18 — `[8 min]`
> "You have auto-sync enabled on dev/qa but want manual approval for prod. How do you configure this in ArgoCD without maintaining 27 separate Application YAMLs?"

**Why this question:** Tests practical ArgoCD knowledge (ApplicationSet) and whether they think about operational safety vs speed.

**Model answer — use ApplicationSet with a matrix generator:**

```yaml
generators:
  - matrix:
      generators:
        - list:
            elements:
              - service: api-gateway
              - service: auth-service
              - service: drug-catalog-service
              # ... 6 more
        - list:
            elements:
              - env: dev
                autoSync: "true"
              - env: qa
                autoSync: "true"
              - env: prod
                autoSync: "false"
template:
  metadata:
    name: "{{env}}-{{service}}"
  spec:
    source:
      path: helm-charts
      helm:
        valueFiles:
          - "../../envs/{{env}}/values-{{service}}.yaml"
    syncPolicy:
      automated:
        enabled: "{{autoSync}}"
        selfHeal: true
        prune: true
    destination:
      namespace: "{{env}}"
```

27 Applications from one YAML. Dev/QA auto-sync. Prod is OutOfSync notification only — a human reviews in ArgoCD UI and clicks Sync.

**Additional prod hardening:** CODEOWNERS on `envs/prod/` requiring 2 reviewer approvals, sync windows restricting prod syncs to business hours only.

---

### Q19 — `[7 min]`
> "ArgoCD shows OutOfSync but every time you sync it goes back to OutOfSync immediately. What's happening?"

**Why this question:** Real-world scenario every ArgoCD user hits eventually.

**Model answer — it's a drift loop:**

Something is modifying the resource AFTER ArgoCD applies it, so the actual state never matches the desired state.

**Most common causes:**
1. **HPA modifies `spec.replicas`** — ArgoCD says `replicas: 1`, HPA scales it to 3, ArgoCD sees drift, syncs back to 1, HPA scales back to 3. Loop.
2. **Admission webhook adds annotations** — a mutating webhook injects a sidecar annotation that's not in your manifests.
3. **Kubernetes sets default fields** — e.g., `imagePullPolicy` gets defaulted by the API server.

**Fix — `ignoreDifferences`:**
```yaml
ignoreDifferences:
  - group: apps
    kind: Deployment
    jsonPointers:
      - /spec/replicas          # let HPA own this
```

**Teaching moment:** This is why you should never set `spec.replicas` in a Deployment manifest when HPA is managing it. Let HPA own replicas entirely.

---

### Q20 — `[7 min]`
> "ArgoCD shows an app as Synced and Healthy but users are getting 500 errors. How is that possible?"

**Why this question:** Trips up most candidates. They assume ArgoCD's status = application health. It doesn't.

**Model answer:**

`Synced` = Git state matches cluster state. `Healthy` = Kubernetes health checks pass. Neither means the app is functionally working.

**Common real reasons:**
- `/actuator/health` returns 200 (Spring Boot is alive) but DB connection pool is exhausted → every request fails
- App started (pod is Ready) but it's trying to connect to a wrong DB schema — requests fail at runtime
- Rolling update completed but Service endpoints still include a terminating pod
- Config difference between dev and prod values — a wrong env var was promoted, app starts but every request fails

**Lesson:** Always monitor application-level SLOs (error rate, latency) in Grafana separately from ArgoCD status. ArgoCD tells you what's deployed, not whether it works.

---

### Q21 — `[7 min]`
> "You're mid-deployment on production — 50% of pods are on the new version — and error rate starts climbing. What do you do right now?"

**Why this question:** Tests decision-making under pressure. Most candidates jump straight to rollback. The right answer is: pause first.

**Model answer:**

```bash
# Step 1: PAUSE — don't rollback yet, don't panic
kubectl rollout pause deployment/api-gateway -n prod
# Traffic now splits 50/50 between old and new pods. Status is frozen.

# Step 2: Identify which pods are causing errors
kubectl get pods -n prod -l app=api-gateway -o wide
# See which pods are new (check AGE or image tag)

kubectl logs -n prod <new-pod-name> --tail=50 | grep -i error
```

**Decision tree:**
- **New pods are erroring** → errors are from the new version → rollback:
  ```bash
  kubectl rollout undo deployment/api-gateway -n prod
  # THEN: revert image tag in envs/prod/values-api-gateway.yaml in git
  ```
- **Old pods are erroring** → the deploy wasn't the cause → investigate independently
- **Both are erroring** → something else is wrong (DB, downstream service) → don't rollback

**Critical GitOps detail:** If `selfHeal: true` is on and you do `kubectl rollout undo` without reverting Git, ArgoCD will re-deploy the bad version within minutes. Always update Git to match what you want running.

---

### Q22 — `[5 min]` *(quick fire)*
> "GitHub goes down. You have a P0 production fix that must ship in 20 minutes. What do you do?"

**Why this question:** Tests whether they understand GitOps is a process, not a hard dependency.

**Model answer — break glass, document everything:**

1. Check ArgoCD's last cached Git state — if the fix is already cached, trigger sync manually
2. Apply directly: `kubectl set image deployment/api-gateway pharma-service=<hotfix-image> -n prod`
3. Write down exactly what you did and why
4. The moment GitHub is back: open a PR with the same change, merge immediately
5. If ArgoCD `selfHeal` is on — it will fight your kubectl change. Temporarily suspend the Application, apply fix, update Git, unsuspend.

**One-liner answer:** "GitOps is the default workflow, not a cage. In a genuine emergency, kubectl works. The rule is: document it and fix Git immediately after."
