# CI/CD & GitHub Actions — Interview Questions

> Grounded in the zen-pharma-backend monorepo pipeline: reusable workflows, GitHub OIDC, 8-stage security gates, GitOps promotion.
> 18 questions across GitHub Actions mechanics, branching strategy, and real pipeline scenarios.

---

## Table of Contents

1. [GitHub Actions Concepts](#1-github-actions-concepts) — Q1–Q7
2. [Branching Strategy](#2-branching-strategy) — Q8–Q11
3. [Pipeline Scenarios from zen-infra](#3-pipeline-scenarios-from-zen-infra) — Q12–Q15
4. [Design Questions](#4-design-questions) — Q16–Q18

---

## 1. GitHub Actions Concepts

---

**Q1: What is a reusable workflow in GitHub Actions and why would you use it?**

- **What is being tested:** DRY principles in CI, `workflow_call` understanding.
- **Strong answer:** A reusable workflow uses `on: workflow_call` and is called from other workflows using `uses:` with a file path. It accepts inputs and secrets, and can return outputs. The benefit is that complex logic — build, scan, push — is defined once and called by each service workflow. Changes to the pipeline (e.g. upgrading Trivy version) are made in one place and automatically apply to all services. It also enforces consistency — every service goes through identical security gates with no accidental variation.
- **Project context:** `_java-build.yml` and `_node-build.yml` contain the full 8-stage pipeline. Each `ci-<service>.yml` is ~10 lines — it just calls the reusable workflow with the service-specific inputs.

---

**Q2: What is the difference between `workflow_call`, `workflow_dispatch`, and `workflow_run`?**

- **What is being tested:** Depth of GitHub Actions knowledge.
- **Strong answer:** `workflow_call` makes a workflow reusable — it can only be triggered by another workflow using `uses:`. `workflow_dispatch` adds a manual trigger with optional input parameters — shows up as a "Run workflow" button in the GitHub Actions UI. `workflow_run` triggers a workflow when another named workflow completes — useful for chaining workflows that are in separate files (e.g. run integration tests after the build workflow completes). The key difference from `needs:` is that `workflow_run` works across separate workflow files.
- **Project context:** `promote-prod.yml` uses `workflow_dispatch` with a service dropdown. The per-service `ci-*.yml` files use `workflow_call` to call `_java-build.yml`.

---

**Q3: How do you pass secrets to a reusable workflow?**

- **What is being tested:** Security mechanics of reusable workflows.
- **Strong answer:** There are two ways. The explicit way declares each secret in the `on.workflow_call.secrets` block and the caller passes them individually. The simpler way is `secrets: inherit` in the caller — this passes all secrets from the caller's context to the called workflow automatically. `secrets: inherit` is convenient but less explicit about what secrets the reusable workflow actually needs.
- **Project context:** All `ci-*.yml` files use `secrets: inherit` when calling `_java-build.yml` — so `AWS_ACCOUNT_ID`, `GITOPS_TOKEN`, and `SEMGREP_APP_TOKEN` are all inherited automatically.

---

**Q4: What are path filters in GitHub Actions and why are they important in a monorepo?**

- **What is being tested:** Monorepo CI efficiency.
- **Strong answer:** Path filters under `on.push.paths` restrict a workflow to only trigger when files matching the pattern have changed. In a monorepo without path filters, every push triggers all workflows — a change to `notification-service/` would rebuild all 7 services unnecessarily. With path filters, only the affected service's workflow triggers. Most monorepo CI setups also include the workflow file itself in the paths so that CI configuration changes are validated.

---

**Q5: What is GitHub OIDC and how does it work with AWS?**

- **What is being tested:** Modern secrets-free authentication understanding.
- **Strong answer:** GitHub Actions supports OpenID Connect. When a workflow runs, GitHub generates a short-lived OIDC token signed by GitHub's identity provider. This token contains claims about the repository, branch, and workflow. AWS IAM supports OIDC federation — you configure an IAM role with a trust policy that allows GitHub's OIDC provider. When the workflow runs `aws-actions/configure-aws-credentials`, it exchanges the GitHub OIDC token for short-lived AWS credentials scoped to that role. No static access keys are stored anywhere. The credentials expire when the workflow run ends.

---

**Q6: How do you prevent a workflow from running on certain branches or events?**

- **What is being tested:** Workflow control flow knowledge.
- **Strong answer:** Multiple approaches: (1) `on.push.branches` / `on.pull_request.branches` restrict triggers to specific branches, (2) `on.push.paths-ignore` excludes certain file paths, (3) job-level `if:` conditions evaluate expressions like `github.ref == 'refs/heads/develop'`, (4) `environment:` with Required Reviewers blocks a job until a human approves. The right choice depends on whether you want to prevent the workflow from starting at all (branch/path filters) or pause at a specific job (environment gate).

---

**Q7: What is a GitHub Environment and how does it enforce deployment gates?**

- **What is being tested:** GitHub's built-in deployment protection mechanisms.
- **Strong answer:** A GitHub Environment is a named deployment target (`dev`, `qa`, `prod`) configured in repo Settings. You can add Required Reviewers — the workflow pauses at any job referencing that environment until a reviewer approves it in the GitHub UI. You can also add wait timers, branch restrictions (only `main` can deploy to `prod`), and deployment history. The key thing is the gate happens at the job level — earlier jobs run, and the gated job waits for approval. This is how you implement a human sign-off without splitting into multiple separate workflows.
- **Project context:** Our `deploy-dev` job uses `environment: dev` (no gate). Our `promote-prod.yml` uses `environment: prod` with Required Reviewers — the workflow cannot even start until a Release Manager approves.

---

## 2. Branching Strategy

---

**Q8: Walk me through your branching strategy from a feature to production.**

- **What is being tested:** End-to-end understanding of Gitflow / trunk-based hybrid.
- **Strong answer:** Feature development starts on a branch cut from `develop` — named `feat/JIRA-101-description`. The developer pushes to their branch, gets fast CI feedback (SAST, tests, OWASP) in ~5 minutes. When ready, they open a PR targeting `develop`. The same CI runs on the PR. After code review and approval, it merges to `develop`. At sprint end, a release branch is cut from `develop` — `release/1.2.0`. Develop is now open for the next sprint. The release branch goes through QA testing. Any bugs found are fixed on branches off `release/1.2.0` and merged back. Once QA signs off, PROD promotion is triggered. After PROD is healthy, `release/1.2.0` is merged to `main` (tagged `v1.2.0`) and back-merged to `develop` to carry forward any hotfixes. The release branch is then deleted.

---

**Q9: Why do you back-merge the release branch into develop?**

- **What is being tested:** Awareness of the most common Gitflow mistake — losing hotfixes.
- **Strong answer:** Any bug fixes made directly on the release branch during QA must flow back to develop — otherwise the next sprint starts with develop not containing those fixes and they get lost in the next release. The back-merge ensures develop always contains everything that has ever been released to PROD.

---

**Q10: What is trunk-based development and how does it differ from Gitflow?**

- **What is being tested:** Awareness of the alternative model, common at Netflix, Google, Uber.
- **Strong answer:** In trunk-based development, all developers commit frequently (multiple times a day) to a single branch — `main` or `trunk`. Feature branches are very short-lived (hours, not days). Feature flags hide incomplete work from users. There are no long-lived `develop` or `release` branches. CI runs on every commit to trunk. This model reduces merge hell and integration pain but requires strong feature flag discipline and very high test coverage. Gitflow suits teams with scheduled releases and formal QA phases. Trunk-based suits teams doing continuous deployment.

---

**Q11: How do you handle a hotfix in production without disrupting the current sprint?**

- **What is being tested:** Practical Gitflow knowledge.
- **Strong answer:** Cut a hotfix branch directly from `main` (which reflects current PROD), not from develop (which may have unreleased sprint work). Fix the bug on the hotfix branch, get it reviewed, and merge it to `main`. Tag it as `v1.1.1`. Then back-merge `main` into `develop` so the fix is not lost in the next release. Do not merge from `develop` into `main` for a hotfix — `develop` may contain half-built features you don't want in production.

---

## 3. Pipeline Scenarios from zen-infra

---

**Q12: A developer pushed a feature branch and the GitHub Actions pipeline ran. The Terraform Plan showed the correct changes. But after they merged to `main`, the apply job was skipped entirely. Why?**

<details>
<summary>Expected answer</summary>

The `terraform.yml` workflow uses a **path filter**:

```yaml
on:
  push:
    branches: [main]
    paths:
      - 'envs/dev/**'
      - 'modules/**'
```

The apply job only triggers when files under `envs/dev/` or `modules/` are changed. If the developer only changed files in `.github/workflows/` or a documentation file, the paths filter does not match — GitHub skips the workflow entirely even after a merge to `main`.

To verify: go to Actions tab → the workflow should show as skipped with a note about the path filter. Ask the developer what files they changed.

</details>

---

**Q13: You are asked to allow a new GitHub repository `zen-pharma-reporting` to push Docker images to ECR. What Terraform change do you make?**

<details>
<summary>Expected answer</summary>

In `modules/iam/github-actions-oidc.tf`, the trust policy condition uses `StringLike` on the `sub` claim:

```hcl
condition {
  test     = "StringLike"
  variable = "token.actions.githubusercontent.com:sub"
  values = [
    "repo:${var.github_org}/zen-pharma-frontend:ref:refs/heads/main",
    "repo:${var.github_org}/zen-pharma-frontend:ref:refs/heads/develop",
    "repo:${var.github_org}/zen-pharma-backend:ref:refs/heads/main",
    "repo:${var.github_org}/zen-pharma-backend:ref:refs/heads/develop",
    # Add these two lines:
    "repo:${var.github_org}/zen-pharma-reporting:ref:refs/heads/main",
    "repo:${var.github_org}/zen-pharma-reporting:ref:refs/heads/develop",
  ]
}
```

Open a PR with this change. `terraform plan` will show an update to `aws_iam_role.github_actions_ci` (the trust policy changes). No new resources are created — the existing role now trusts the additional repo. After apply, add the role ARN to the new repo's GitHub Variables and use `aws-actions/configure-aws-credentials` in its workflow.

Note: the permission policy (`ECRPush`) already allows `arn:aws:ecr:*:ACCOUNT:repository/*` — all repos in the account. The trust policy is the real security gate.

</details>

---

**Q14: Explain the difference between `terraform fmt -check`, `terraform validate`, and `terraform plan`. Why does the pipeline run all three in that order?**

<details>
<summary>Expected answer</summary>

They catch different categories of errors, from cheap to expensive:

| Command | What it checks | Cost |
|---|---|---|
| `terraform fmt -check` | Code formatting only — indentation, spacing | Milliseconds, no AWS call |
| `terraform validate` | HCL syntax and type correctness — no AWS calls | Seconds, no AWS call |
| `terraform plan` | What will actually change in AWS — calls AWS APIs | 30–60 seconds, AWS API calls |

Running them in this order means:
- A formatting error (e.g. forgotten indent) fails in milliseconds instead of waiting 60 seconds for a plan
- A type error (e.g. passing a `string` where `list(string)` is expected) fails before any AWS calls
- Only well-formatted, syntactically valid code proceeds to the expensive plan step

In the `zen-infra` pipeline, `terraform fmt -check` failing blocks the PR from merging. This enforces consistent formatting across the whole team — no arguments about tabs vs spaces.

</details>

---

**Q15: The GitHub Actions OIDC role has `max_session_duration = 3600`. A CI job for building Docker images is taking 75 minutes because the image is very large. What happens and how do you fix it?**

<details>
<summary>Expected answer</summary>

After 60 minutes the STS credentials expire. The `docker push` step will fail with an authentication error — ECR returns `no basic auth credentials` or a 401.

Fix options:

1. **Increase `max_session_duration`** in Terraform:
   ```hcl
   max_session_duration = 7200  # 2 hours
   ```
   AWS allows up to 12 hours for OIDC roles. Open a PR, merge, apply. The change takes effect immediately for new workflow runs.

2. **Optimise the Docker build** — multi-stage builds, `.dockerignore` to exclude unnecessary files, layer caching (`actions/cache` for Docker layers). A 75-minute build is a sign the image needs refactoring regardless of the auth issue.

3. **Re-authenticate mid-job** — call `aws-actions/configure-aws-credentials` a second time before the push step. This mints a fresh token and extends the window.

Option 1 is the immediate fix. Option 2 is the right long-term fix.

</details>

---

## 4. Design Questions

---

**Q16: Design a CI/CD pipeline for a monorepo with 10 microservices. What are your key decisions?**

- **What is being tested:** System design thinking for CI/CD.
- **Strong answer:** Key decisions: (1) **Path filters** — each service's pipeline triggers only when its directory changes, (2) **Reusable workflows** — one `_java-build.yml` called by all Java services, not 10 copies of the same pipeline, (3) **Separate lightweight PR checks** from full build pipelines — fast feedback on feature branches, full security scan + Docker on merged code, (4) **GitOps promotion** — CI never talks to Kubernetes directly, updates a separate GitOps repo, ArgoCD reconciles, (5) **Build Once Deploy Many** — one ECR image per commit, promoted by changing values files, (6) **Image tag strategy** — `sha-<7chars>`, never `:latest`, (7) **Environment gates** — auto for DEV, PR-based for QA, manual dispatch for PROD.

---

**Q17: How would you implement a full audit trail for every production deployment?**

- **What is being tested:** Compliance and auditability design.
- **Strong answer:** In a GitOps model, the audit trail is built in: every deployment is a PR in zen-gitops with an author, timestamp, approvers, and the exact diff showing what image tag changed to what. The PR link is embedded in the Cosign signature in the Rekor transparency log. GitHub Actions run history shows who triggered `promote-prod.yml` and when. ArgoCD sync history shows every sync with the git commit SHA. Combined: you can trace any production deployment to: who approved the code (PR in zen-pharma-backend), who approved the promotion (PR in zen-gitops), who triggered the PROD workflow (GitHub Actions log), what image ran (ECR digest), and what code was in that image (git SHA). This satisfies SOC 2 and most compliance audit requirements.

---

**Q18: What would you change about this pipeline if the company needed to achieve SOC 2 Type II compliance?**

- **What is being tested:** Compliance awareness applied to real architecture.
- **Strong answer:** The pipeline is already well-positioned. Additions for SOC 2: (1) **Formalise the change management evidence** — the zen-gitops PR with approvals is change management evidence; export this to a ticketing system (Jira) per deployment, (2) **SBOM generation** — add Syft or Trivy SBOM output on every build, stored as an artifact, (3) **Enforce branch protection on zen-gitops** — require PR reviews, disallow direct pushes to main, require signed commits, (4) **Audit log retention** — GitHub Actions logs and ArgoCD sync history need to be retained for the compliance period (typically 1 year), export to S3 or a SIEM, (5) **Access reviews** — document who has access to `prod` environment approval in GitHub and the IAM role, review quarterly, (6) **Vulnerability SLA tracking** — CVSS ≥ 9.0 must be remediated within 24 hours, ≥ 7.0 within 30 days; export Trivy and OWASP findings to a tracking system.
