# Behavioral & Real-Time Interview Questions

> These questions are asked in every senior DevOps/Platform Engineering interview.
> Model answers are grounded in the zen-pharma project — always answer with "In our project, we..."

---

## Table of Contents

1. [Day-to-Day & Latest Work](#1-day-to-day--latest-work) — Q1–Q5
2. [Hardest Challenges](#2-hardest-challenges) — Q6–Q9
3. [Behavioral & Culture Fit](#3-behavioral--culture-fit) — G1–G5
4. [Scenario & Design Questions](#4-scenario--design-questions) — Q15–Q20

---

## 1. Day-to-Day & Latest Work

---

**Q1. Walk me through what you worked on last week.**

- **What is being tested:** Whether you can articulate real work concisely. Interviewers want specifics, not generalities.
- **Strong answer:** Last week I worked on the CI pipeline for the auth-service in our zen-pharma monorepo. We upgraded the base Docker image from `eclipse-temurin:17-jre` to `eclipse-temurin:17-jre-alpine` to reduce image size and Trivy findings. I updated the Dockerfile, ran the full pipeline — build, OWASP Dependency Check, Trivy scan, ECR push — and opened a PR to zen-gitops to promote the new image tag to DEV. ArgoCD picked up the change and rolled it out. I also reviewed a teammate's PR adding a Semgrep rule to catch missing `@PreAuthorize` on new Spring Boot controllers.

---

**Q2. What does your typical day as a DevOps engineer look like on this project?**

- **What is being tested:** Whether you understand DevOps as ongoing work, not just deployment automation.
- **Strong answer:** My day has three main threads. First, I check the GitHub Actions dashboard for any failed pipelines — if a CI job failed overnight I read the logs, identify whether it's a new CVE, a test failure, or an infrastructure flake, and create a ticket or fix it directly. Second, I check ArgoCD — if any application shows OutOfSync or Degraded, I investigate whether it's drift from a manual kubectl change or a Helm rendering issue. Third, I review PRs — both application code PRs for pipeline configuration and zen-gitops PRs for promotion approvals. On top of that, one or two days a week I do planned work: adding features to the pipeline (like integrating Cosign signing), writing Terraform for new infrastructure, or upgrading provider versions via Dependabot PRs.

---

**Q3. What was the last thing you deployed to production and how did you do it?**

- **What is being tested:** Fluency with your own deployment process.
- **Strong answer:** The last production deployment was the drug-catalog-service with a new inventory search endpoint. The flow: a developer merged their PR to `develop`, the CI pipeline ran — unit tests, CodeQL, Semgrep, OWASP Dep Check, Docker build, Trivy scan, ECR push. The image tag `sha-dbbb634` was written to `envs/dev/values-drug-catalog.yaml` in zen-gitops via an automated PR. After DEV validation, I opened a promotion PR to update `envs/prod/values-drug-catalog.yaml`. The release manager and a senior engineer reviewed and approved. After merge, I triggered ArgoCD manual sync for `pharma-prod`. ArgoCD ran a rolling update — the old pods were only terminated after new pods passed the Spring Boot `/actuator/health/readiness` probe. Total deployment time from PR merge to prod traffic: about 8 minutes.

---

**Q4. How do you stay on top of security vulnerabilities in your services?**

- **What is being tested:** Proactive security hygiene, not just reactive fixing.
- **Strong answer:** Three layers. First, Dependabot is enabled on all repos — it opens PRs automatically when a direct dependency has a new CVE. We've configured it to check weekly. Second, every CI pipeline run includes OWASP Dependency Check against the NVD, so any new CVE published since the last build will block the pipeline. We get notified via GitHub Actions failure emails. Third, Trivy scans the built Docker image on every push — OS-level packages in the base image are caught here even if the dependency scanner missed them. When a CVE is found, I check its CVSS score: if it's ≥ 7.0 and there's a fix available, it blocks the build. I update the affected dependency, verify the scan passes, and promote the patched image through all environments.

---

**Q5. How do you handle on-call? What was the last incident you responded to?**

- **What is being tested:** Incident response maturity and calm under pressure.
- **Strong answer:** We use PagerDuty with a weekly rotation. Alerts fire from CloudWatch alarms — 5xx rate on the NGINX Ingress, RDS connection failures, EKS node NotReady. The last incident I handled: at 11pm I got paged for elevated 503 errors on the api-gateway. I started with `kubectl get pods -n prod` — all running. `kubectl get nodes` — all ready. So pods and nodes were healthy, meaning the issue was in networking or routing. I checked the NLB target group health in the AWS console — two targets showed unhealthy. I checked the ArgoCD sync history and found someone had merged a values-file change that updated the readiness probe path from `/actuator/health/readiness` to `/health/ready`, which doesn't exist in our Spring Boot setup. The pods were running but failing their readiness check, so the NLB removed them from the target group. I reverted the zen-gitops PR, triggered ArgoCD sync, and within 3 minutes the targets were healthy and 503s stopped. Total incident duration: 22 minutes.

---

## 2. Hardest Challenges

---

**Q6. What is the hardest technical problem you solved on this project?**

- **What is being tested:** Problem-solving depth, ability to narrate complexity clearly.
- **Strong answer:** The hardest problem was getting IRSA working for External Secrets Operator. On paper it's straightforward: create an OIDC provider, attach a role, annotate the service account. In practice, we had the ESO pods crashlooping with `AccessDeniedException` even though the IAM role and trust policy looked correct. The issue turned out to be three things at once: (1) the OIDC provider ARN in our Terraform had a subtle trailing slash difference from what ESO expected, (2) the EKS cluster OIDC issuer URL uses HTTPS but we had registered the provider with the raw hostname, and (3) the ESO Helm chart default installs in the `external-secrets` namespace but our IAM trust policy specified `default`. Each one alone wouldn't have caused the problem — all three together made the STS token exchange fail silently. I debugged it with `kubectl logs` on the ESO controller, `aws sts get-caller-identity` from inside a debug pod, and `TF_LOG=DEBUG` on the Terraform apply. Fixing it took about 4 hours spread across two sessions.

---

**Q7. Tell me about a time a deployment went wrong. What happened and how did you recover?**

- **What is being tested:** Incident ownership, recovery speed, and post-incident learning.
- **Strong answer:** We deployed a new version of the manufacturing-service that had a Flyway migration adding a NOT NULL column without a default. The Flyway migration ran in DEV and QA fine because those databases had no existing rows. In PROD the migration failed immediately — it tried to add a NOT NULL column to a table with existing data and Postgres rejected it. The application pods failed to start, Kubernetes kept restarting them, and the manufacturing endpoints returned 503. Recovery: I triggered ArgoCD rollback to the previous image tag in zen-gitops (git revert + ArgoCD sync — 5 minutes). The old pods came back up with the old schema. Then the developer fixed the migration to add the column with a default value first (`ALTER TABLE ADD COLUMN qty integer DEFAULT 0`), followed by the constraint in a separate migration file. After DEV and QA re-validated with real data, we redeployed to PROD — this time the migration succeeded. Post-incident: we added a required step in the QA checklist to run migrations against a production data snapshot before PROD deploy when schema changes are involved.

---

**Q8. Tell me about a time you had to make a trade-off between speed and correctness.**

- **What is being tested:** Engineering judgment, not just technical skill.
- **Strong answer:** Early in the project we had a debate about whether to run the full CI pipeline (CodeQL, OWASP, Trivy, Docker build) on every push to a feature branch or only on merged code. Running everything on every push would give faster feedback but would make feature branch builds take 12–15 minutes, which developers found slow and started ignoring. Running only on merged code was faster for the developer but meant a security finding could sit in `develop` for days before anyone noticed. We chose a split: on feature branch pushes, run only the fast checks (unit tests, Semgrep, basic lint) — 5 minutes. On merge to develop, run the full pipeline including CodeQL and Trivy. This was a correctness trade-off — we accepted that a CVE could be in a feature branch for the duration of the PR without blocking it. The trade-off was justified because: (1) Dependabot already blocks known CVEs at the library level continuously, (2) CodeQL on every PR was generating noise from the same unresolved finding repeated across 20 pushes, (3) developer velocity dropped visibly with the 15-minute gate. We documented the decision in the CI README so future team members understand why it's structured that way.

---

**Q9. What's the most complex thing you've built end-to-end on this project?**

- **What is being tested:** Breadth of ownership and ability to connect multiple systems.
- **Strong answer:** The end-to-end supply chain security implementation. It spans five systems working together: (1) GitHub Actions uses OIDC — no static keys anywhere, (2) the CI pipeline builds the Docker image, runs Trivy, then calls Cosign to sign the image digest using the GitHub Actions OIDC token — keyless, no private key to manage, (3) the Cosign signature is recorded in the Rekor public transparency log with the workflow's identity and a timestamp, (4) Kyverno runs as an admission webhook in the EKS cluster and verifies every pod's image has a valid Cosign signature in Rekor before allowing it to start, (5) ArgoCD syncs the Helm values with the pinned image tag — ensuring the specific signed image is what gets deployed. The result: a rogue image pushed directly to ECR bypassing CI cannot run in the cluster — Kyverno rejects it at admission time. Building this required understanding OIDC federation, OCI image signing, transparency logs, and Kubernetes admission webhooks — it's not a single tool, it's a chain where each link depends on the next.

---

## 3. Behavioral & Culture Fit

---

## G1

### Question
> "Tell me about a time you had to convince a team to adopt GitOps when they were comfortable with traditional CI/CD pipelines. What was the resistance and how did you handle it?"

### What the interviewer is really testing
- Change management and influence skills
- Whether you can articulate GitOps value in business terms
- Empathy for the people being asked to change

---

### Model Answer (STAR format)

**Situation:**
Our team had been running Jenkins pipelines with direct `kubectl apply` to all three environments — dev, qa, and prod. The pipeline worked but we had three recurring problems: occasional prod deployments that nobody could trace back to a PR, configuration drift (someone had manually patched a configmap in prod months ago and nobody knew), and a compliance audit finding that we lacked an approval gate for production changes.

**Task:**
I was asked to propose a solution for the compliance finding. I saw it as an opportunity to move to GitOps, but I knew the team would push back — they'd spent months building and tuning those Jenkins pipelines.

**Action:**

*First, I understood the resistance before proposing anything.* I ran a team retro specifically asking "what's painful about our current deployment process?" — not "should we change it?" That gave me real pain points in their words:
- "I can never tell what's actually running in prod without sshing into the cluster"
- "Jenkins went down last week and we couldn't deploy for 3 hours"
- "I'm nervous every time I have to do a prod deployment"

*Then I proposed GitOps as the solution to THEIR problems, not as a technology migration.* I reframed it:
- "Can always tell what's in prod" → Git is the single source of truth, `git log` shows the exact state
- "Jenkins outage blocks deployments" → ArgoCD is inside the cluster, not a dependency
- "Nervous about prod" → PRs require approval, and you can preview the diff before applying

*I ran a 2-week pilot on one service (auth-service in dev only).* I set up ArgoCD, migrated one values file, and showed the team the UI. The visual diff of what would change before syncing was the moment it clicked for most people.

*I kept the Jenkins pipelines running in parallel for 4 weeks.* I didn't ask anyone to trust something unproven. Side by side for a month, then we decommissioned Jenkins for CD.

**Result:**
Full migration completed over 6 weeks. The compliance finding was resolved — we now had PR-based approval gates for prod changes with full audit trail. The team's biggest surprise was how much faster rollbacks became: from "find the Jenkins build, re-trigger it" to "git revert, merge, done in 5 minutes."

---

### What interviewers at top MNCs listen for

- Did you acknowledge that the existing solution had VALUE? (Respect for what people built)
- Did you run a pilot before asking for full commitment?
- Did you solve their problems, not impose your preferences?
- Were there measurable outcomes?

---

## G2

### Question
> "Describe a production incident you handled in a Kubernetes-based system. What was the impact, what was your role, and what did you explain afterward?"

### What the interviewer is really testing
- Structured incident response under pressure
- Ownership and blameless postmortem culture
- Learning mindset — what changed after?

---

### Model Answer (STAR format)

**Situation:**
At 11:40pm on a Tuesday, our auth-service in prod started returning 401 for all requests. 100% of users were being logged out and couldn't log back in. Our PagerDuty alert fired on the 5% error rate threshold — it should have fired earlier but the alert window was 5 minutes which meant 5 minutes of impact before we even knew.

**Task:**
I was the on-call SRE. I had 15 minutes to diagnose and either roll back or fix it, because the business had a hard SLA with the client.

**Action:**

*T+0 (acknowledge):* Checked ArgoCD first — was there a recent deployment? Yes: catalog-service had been deployed 8 minutes ago.

*T+2 (narrow the blast radius):* Auth-service itself was not redeployed — why was it broken? Checked the auth-service pods: all 3 were Running and passing liveness. No CrashLoopBackOff, no OOMKilled.

*T+4 (read logs):*
```bash
kubectl logs -n prod deployment/auth-service --since=15m | grep ERROR
```
Found: `JWT secret key not found: jwt-secret key JWT_SECRET is empty`

*T+5 (check the secret):*
```bash
kubectl get secret jwt-secret -n prod -o jsonpath='{.data.JWT_SECRET}' | base64 -d
# output: (empty)
```
The secret existed but was empty. Checked the ExternalSecret:
```bash
kubectl describe externalsecret jwt-secret -n prod
```
Status showed: `SecretSyncedError: AccessDenied: User: arn:aws:sts::516209541629:assumed-role/external-secrets-role is not authorized to perform: secretsmanager:GetSecretValue`

*T+6 (root cause):* The catalog-service deployment had included a Terraform change that accidentally modified the `external-secrets-role` IAM policy. The policy change removed the `secretsmanager:GetSecretValue` permission for `/pharma/prod/jwt-secret` path.

*T+8 (fix):* I had the Terraform change reverted (git revert, terraform apply — 4 minutes). Then forced ESO re-sync:
```bash
kubectl annotate externalsecret jwt-secret -n prod force-sync=$(date +%s) --overwrite
```
*T+12:* JWT secret repopulated. Rolling restart of auth-service:
```bash
kubectl rollout restart deployment/auth-service -n prod
```
*T+15:* Auth-service serving 200s. Incident resolved.

**Result and what changed:**

Postmortem within 24h. Three action items:
1. **Added ESO health alert** — `kubectl get externalsecret -A` returning non-Ready should page immediately. We would have caught this before user impact.
2. **Separated IAM policy management** from application Terraform** — the infra that auth-service depends on should not be modifiable by the catalog-service Terraform module.
3. **Added integration test** in CI that validates the ESO sync works before marking a deployment successful.

---

### What to emphasize in the interview

- You had a structured diagnostic process — not guessing, checking one layer at a time
- You didn't just fix it, you asked "why did this take 15 minutes to find?"
- The postmortem was blameless — the Terraform change was a system design problem, not a person's fault
- Three concrete action items, not vague "we'll be more careful"

---

## G3

### Question
> "Tell me about a time you had to push back on a request from a product manager or senior stakeholder. How did you handle it?"

### What the interviewer is really testing
- Can you hold your ground technically without being difficult to work with?
- Do you understand the difference between "no" and "not yet, here's why"?
- Can you communicate risk to non-technical stakeholders?

---

### Model Answer

A product manager once asked us to skip the QA environment and deploy directly to PROD to meet a deadline — a critical demo for a pharma client the next morning. The feature was working in DEV, tests were passing, and the PM argued that QA was just a formality.

My response was not a flat refusal. I explained specifically what QA gives us that DEV does not: our QA database has anonymised production-scale data (millions of drug catalog records), while DEV has synthetic data with about 200 rows. A performance issue that's invisible in DEV will surface in QA under realistic load. I also pointed out that skipping QA would mean our PROD rollback procedure hadn't been validated for this specific change — if something went wrong mid-demo we'd be debugging live in front of the client.

I proposed an alternative: run a targeted smoke test against QA within the next two hours, focusing only on the new feature's code path. If it passed, we'd promote to PROD that evening with a 30-minute buffer before the demo. If it failed, we'd know before the client arrived instead of during.

The PM agreed. The smoke test found a slow query that was fine in DEV but timed out against real data. We added an index, re-ran, and deployed successfully. The demo went without issues.

The key: I didn't say "no, we follow the process." I said "here is the specific risk and here is a way to manage it that still meets your deadline." That's the conversation that builds trust with stakeholders instead of friction.

---

## G4

### Question
> "A new engineer joins and accidentally deploys to prod by pushing directly to the main branch. How do you prevent this from happening again? Walk me through the guardrails you'd put in place."

### What the interviewer is really testing
- Systems thinking — fix the system, not blame the person
- Layered defense: don't rely on a single guardrail
- Knowledge of GitHub/GitLab branch protection, CODEOWNERS, ArgoCD RBAC

---

### Model Answer

**First principle: this is a system failure, not a human failure.** The system allowed it to happen. A new engineer should not be able to accidentally deploy to prod — that's a design problem in the guardrails, not a training problem.

**Defense in depth — 5 layers:**

---

**Layer 1: Git branch protection (prevent the push)**
```yaml
# GitHub repository settings → Branch protection rules for 'main'
main branch rules:
  ✅ Require a pull request before merging
  ✅ Require approvals: 1 (general), 2 (for envs/prod/** changes)
  ✅ Require review from Code Owners
  ✅ Dismiss stale pull request approvals when new commits are pushed
  ✅ Restrict who can push to main: [release-managers, senior-sre]
  ✅ Do not allow bypassing the above settings (even for admins)
```

**Layer 2: CODEOWNERS (enforce approval by right people)**
```bash
# .github/CODEOWNERS
# envs/prod/ changes require prod team sign-off
/envs/prod/          @pharma-release-managers @senior-sre-team
/argocd/             @pharma-platform-team
/k8s/rbac/           @pharma-security-team

# This means: even if a PR has 2 approvals,
# if it touches envs/prod/ it ALSO needs a @pharma-release-managers approval
```

**Layer 3: ArgoCD RBAC (prevent sync even if the commit gets through)**
```yaml
# argocd-rbac-cm ConfigMap
data:
  policy.csv: |
    # Developers can only sync dev and qa
    p, role:developer, applications, sync, pharma/*, deny
    p, role:developer, applications, sync, pharma/*-dev, allow
    p, role:developer, applications, sync, pharma/*-qa, allow

    # Only release-managers can sync prod
    p, role:release-manager, applications, sync, pharma/*-prod, allow

    # Assign roles
    g, pharma-developers, role:developer
    g, pharma-release-managers, role:release-manager
```

**Layer 4: ArgoCD sync policy (require manual approval for prod)**
```yaml
# prod ArgoCD Applications: no automated sync
spec:
  syncPolicy: {}   # no automated field = manual sync only
  # An operator must explicitly click "Sync" in ArgoCD UI or run:
  # argocd app sync auth-service-prod
  # This is a second human gate after the PR merge
```

**Layer 5: Path-based CI checks (validate intent)**
```yaml
# .github/workflows/protect-prod.yaml
on:
  pull_request:
    paths:
      - 'envs/prod/**'
jobs:
  require-prod-label:
    runs-on: ubuntu-latest
    steps:
      - name: Check for production-deployment label
        run: |
          LABELS=$(gh pr view ${{ github.event.pull_request.number }} --json labels -q '.labels[].name')
          if ! echo "$LABELS" | grep -q "production-deployment"; then
            echo "ERROR: PRs touching envs/prod/ require the 'production-deployment' label"
            exit 1
          fi
```

---

### After the incident: the conversation with the engineer

> "This was a gap in our setup — the guardrails should not have allowed this. We're fixing the system so this can't happen again regardless of experience level. Let's make sure you understand the promotion process now, but the system change is the real fix."

---

### Summary of layers

```
Engineer pushes commit
         │
Layer 1: Branch protection → blocks direct push to main ✋
         │ (bypassed if push access granted)
Layer 2: CODEOWNERS → prod changes need release-manager approval ✋
         │ (bypassed if CODEOWNERS not enforced)
Layer 3: ArgoCD RBAC → developer can't trigger prod sync ✋
         │ (bypassed if someone with access manually syncs)
Layer 4: Manual sync policy → human must explicitly approve prod sync ✋
         │ (bypassed if operator doesn't check what they're syncing)
Layer 5: CI gate → PR without label fails checks ✋

5 independent layers. An accident requires bypassing ALL of them.
```

---

## G5

### Question
> "You have 3 environments but different teams own dev and prod. How do you structure Git branch protection, ArgoCD project permissions, and Helm values so neither team can accidentally affect the other?"

### What the interviewer is really testing
- Org-scale GitOps design
- How technical controls enforce team boundaries
- Understanding of ArgoCD multi-tenancy

---

### Model Answer

**The design principle:** ownership boundaries must be enforced by the platform, not by convention. "Please don't touch prod" is not a guardrail.

---

### Git Structure

```
zen-gitops/
├── envs/
│   ├── dev/        ← owned by dev-team
│   ├── qa/         ← owned by qa-team (shared gatekeeper)
│   └── prod/       ← owned by prod-team (release managers)
├── helm-charts/    ← owned by platform-team
└── argocd/         ← owned by platform-team
```

```bash
# .github/CODEOWNERS
/envs/dev/           @pharma-dev-team
/envs/qa/            @pharma-qa-team
/envs/prod/          @pharma-release-managers
/helm-charts/        @pharma-platform-team
/argocd/             @pharma-platform-team
/k8s/                @pharma-platform-team
```

Branch protection on `main`:
- Any PR touching `envs/prod/**` requires `@pharma-release-managers` approval — even if it was opened by a senior dev
- Any PR touching `helm-charts/**` requires `@pharma-platform-team` approval — prevents dev-team from modifying shared chart

---

### ArgoCD Project Structure (per team)

```yaml
# pharma-dev project — for dev team
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: pharma-dev
spec:
  destinations:
    - server: https://kubernetes.default.svc
      namespace: dev            # dev team can ONLY deploy to dev namespace
  sourceRepos:
    - "https://github.com/ravdy/zen-gitops.git"
  roles:
    - name: dev-deployer
      policies:
        - p, proj:pharma-dev:dev-deployer, applications, *, pharma-dev/*, allow
      groups:
        - pharma-dev-team

---
# pharma-prod project — for release managers
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: pharma-prod
spec:
  destinations:
    - server: https://kubernetes.default.svc
      namespace: prod           # prod project can ONLY deploy to prod namespace
  roles:
    - name: prod-deployer
      policies:
        - p, proj:pharma-prod:prod-deployer, applications, sync, pharma-prod/*, allow
      groups:
        - pharma-release-managers
```

With this setup:
- Dev team members cannot see or sync prod ArgoCD Applications
- Release managers cannot see dev internal Applications
- Both teams can see qa (shared project)

---

### Kubernetes RBAC alignment

```
                Git CODEOWNERS         ArgoCD RBAC          K8s Namespace RBAC
                ───────────────        ─────────────        ──────────────────
dev-team   →    envs/dev/**            pharma-dev project   pharma-deployer (dev ns)
qa-team    →    envs/qa/**             pharma-qa project    pharma-deployer (qa ns)
prod-team  →    envs/prod/**           pharma-prod project  pharma-deployer (prod ns)
platform   →    helm-charts/,argocd/   all projects (admin) cluster-admin (argocd SA)
```

Three independent enforcement layers — an accident requires bypassing all three.

---

### The key architectural insight

> "The critical mistake I've seen is teams using a single ArgoCD project for all environments. That means a dev team member with ArgoCD access can accidentally sync prod. Separate projects per environment, with separate role bindings, is how you enforce team ownership at the platform level rather than relying on discipline."

---

## 4. Scenario & Design Questions

---

**Q15. A developer accidentally committed AWS credentials to a feature branch. What do you do?**

- **What is being tested:** Incident response for credential exposure.
- **Strong answer:** Treat it as an active breach — assume the credentials were already scraped by automated GitHub secret scanners (they are real). Immediate steps: (1) Rotate the AWS credentials immediately — do not wait, (2) Check AWS CloudTrail for any API calls made with those credentials in the past hours, (3) Remove the commit from Git history using `git filter-branch` or BFG Repo Cleaner and force-push — even on a feature branch the credentials should be purged from history, (4) Notify the security team, (5) Post-incident: set up Gitleaks or GitHub's secret scanning (which does this automatically) as a pre-commit hook so this cannot happen again.

---

**Q16. Your team wants to adopt GitOps but the current setup uses Ansible playbooks and bash scripts that `kubectl apply` directly. How do you migrate?**

- **What is being tested:** Migration strategy and pragmatism.
- **Strong answer:** Migrate incrementally, not all at once. (1) Start by creating the GitOps repo with current cluster state exported using `helm get values` and `kubectl get -o yaml` — this is your baseline, (2) Install ArgoCD and point it at the GitOps repo without enabling auto-sync — ArgoCD will show diffs but not apply them, giving you confidence in the setup, (3) Migrate one non-critical service first with ArgoCD auto-sync enabled — run both the old pipeline and ArgoCD in parallel for a sprint, (4) Once confident, remove the `kubectl apply` from the pipeline for that service and rely on ArgoCD only, (5) Repeat service by service. Never do a big-bang migration — the risk is too high and you lose the ability to roll back the migration itself.

---

**Q17. How do you handle database migrations in a GitOps / Kubernetes deployment?**

- **What is being tested:** Real-world deployment complexity awareness.
- **Strong answer:** Database migrations are the hardest part of zero-downtime deployments. Options: (1) **Flyway / Liquibase as a Kubernetes Job** — an init container or a Job runs migrations before the new app pods start; ArgoCD syncs the Job first as a `PreSync` hook, then syncs the Deployment, (2) **Backward-compatible migrations** — always write migrations that the old version of the app can tolerate; add the column in one release, start using it in the next, drop the old column in a third. This allows rolling updates without downtime because old and new pods can coexist, (3) **Never run migrations as part of application startup** — if two pods start simultaneously, both try to run the migration and you get race conditions or lock timeouts.
- **Project context:** Our Java services use Flyway (`needs-database: true` in CI runs a Postgres container to validate migrations pass before the image is built).

---

**Q18. How do you mentor junior engineers on your team?**

- **What is being tested:** Leadership and teaching ability.
- **Strong answer:** I pair on the first few tickets rather than just reviewing PRs after the fact. For DevOps work specifically, I set up a sandbox AWS account for junior engineers so they can run `terraform apply` without fear of breaking shared environments. I also write runbooks for common operations — deploying a new service, rotating a secret, checking ArgoCD status — so the answer to "how do I do X?" is findable without asking me. In code reviews I explain the why behind feedback ("the reason we don't store secrets in tfvars is not style — it's because git history is permanent and private repos get leaked"), not just what to change. The goal is that after 3 months they are reviewing my PRs and finding things I missed.

---

**Q19. Describe how you would onboard a new engineer to this project.**

- **What is being tested:** Documentation discipline and knowledge transfer.
- **Strong answer:** The first day: clone the four repos and run through the README for each. The READMEs in zen-infra and zen-gitops describe the architecture end-to-end. Then read the CI pipeline for one service — `ci-auth-service.yml` — and trace exactly what happens from a push to a pod running in DEV. Day two: make a trivial change, open a PR, watch the pipeline run, see ArgoCD pick it up. Day three: read the Terraform for one module — VPC is a good start. By end of week one, the engineer should be able to explain the end-to-end flow (code → CI → ECR → GitOps PR → ArgoCD → cluster) and have merged one PR. I track onboarding progress with a checklist in the team wiki — not tribal knowledge.

---

**Q20. Tell me about a disagreement you had with a teammate and how you resolved it.**

- **What is being tested:** Collaboration under conflict, not just technical skill.
- **Strong answer:** We had a disagreement about whether to use ArgoCD auto-sync for the QA environment. A teammate argued that auto-sync was risky — any merged PR would deploy to QA immediately without a human checkpoint. I argued that the checkpoint is the PR review in zen-gitops, not a manual sync trigger — requiring manual sync in QA just adds friction without adding safety, because the same engineer who merges the PR would be the one triggering the sync anyway. We resolved it by trying auto-sync in QA for one sprint with the team's explicit agreement to revert if it caused problems. It worked well — QA deployments went from "remember to trigger the sync" to automatic, the QA team got faster feedback, and we never had an incident from an unexpected auto-sync. The teammate's concern was valid (auto-sync in PROD would be dangerous), but QA's risk profile is different.
