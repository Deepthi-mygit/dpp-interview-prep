# EKS & Kubernetes — Interview Questions

> Grounded in the zen-pharma EKS 1.33 cluster with IRSA, External Secrets, ArgoCD, and NGINX Ingress.
> 25 questions across EKS setup, Kubernetes concepts, incident response, and live troubleshooting.

---

## Table of Contents

1. [EKS Concepts & IRSA](#1-eks-concepts--irsa) — Q1–Q3
2. [Kubernetes Concepts](#2-kubernetes-concepts) — Q4–Q7
3. [Incident Response Scenarios](#3-incident-response-scenarios) — Q8–H9
4. [Troubleshooting Commands Reference](#4-troubleshooting-commands-reference) — I1–I7
5. [Scenario Troubleshooting](#5-scenario-troubleshooting) — Q24–Q25

---

## 1. EKS Concepts & IRSA

---

**Q1.** You just ran `terraform apply` and the EKS cluster was created, but `kubectl get nodes` shows 0 nodes after 10 minutes. What do you check?

<details>
<summary>Expected answer</summary>

Check in this order:

1. **Node group IAM role** — the three policy attachments in `modules/eks/main.tf` must exist before the node group. If `terraform apply` failed between creating the role and attaching the policies, nodes will spin up but cannot join the cluster (no `AmazonEKSWorkerNodePolicy`).

2. **`depends_on` in `aws_eks_node_group`** — the code has:
   ```hcl
   depends_on = [
     aws_iam_role_policy_attachment.node_AmazonEKSWorkerNodePolicy,
     aws_iam_role_policy_attachment.node_AmazonEKS_CNI_Policy,
     aws_iam_role_policy_attachment.node_AmazonEC2ContainerRegistryReadOnly,
   ]
   ```
   If a partial apply happened, this dependency chain may not have completed.

3. **Subnet IDs** — check that `var.subnet_ids` received the private EKS subnet IDs (not the RDS subnets, not the public subnets).

4. **`aws-auth` ConfigMap** — the cluster was created with `bootstrap_cluster_creator_admin_permissions = true`. If a different IAM identity created the cluster vs. what you are using for `kubectl`, you may need to add your IAM role to `aws-auth`.

```bash
aws eks update-kubeconfig --region us-east-1 --name zen-pharma-dev-cluster
kubectl get nodes
kubectl describe node <node-name>  # check events
```

</details>

---

**Q2.** Explain IRSA in plain English. Why is it better than putting `AWS_ACCESS_KEY_ID` in a Kubernetes Secret?

<details>
<summary>Expected answer</summary>

**IRSA (IAM Roles for Service Accounts)** lets a Kubernetes pod get temporary AWS credentials automatically, without storing any long-lived keys anywhere.

Here is how it works step by step in the zen-pharma project:

1. When Terraform runs `modules/eks`, it creates an **OIDC provider** (`aws_iam_openid_connect_provider`) registered with AWS IAM. This tells AWS: *"trust JWT tokens signed by this EKS cluster."*

2. When Terraform runs `modules/iam`, it creates the ESO IRSA role with a **trust policy condition**:
   ```
   system:serviceaccount:external-secrets:external-secrets
   ```
   This means: *"only the service account named `external-secrets` in the `external-secrets` namespace can assume this role."*

3. At runtime, Kubernetes **projects a signed JWT token** into the ESO pod's filesystem automatically.

4. The ESO pod calls AWS STS with that token. STS validates it against the OIDC provider and returns **1-hour temporary credentials**.

5. ESO uses those credentials to call `secretsmanager:GetSecretValue` on `/pharma/*`.

**Why it is better than a static key:**
- A static `AWS_ACCESS_KEY_ID` stored in a K8s Secret can be read by anyone with `kubectl get secret` in that namespace — or anyone with S3/etcd access
- Static keys never expire — a leaked key is valid until manually rotated
- IRSA credentials expire in 1 hour and are scoped to exactly the permissions in the IAM role
- There is nothing to rotate, nothing to leak, nothing to store

</details>

---

**Q3.** The `argocd-application-controller` pod is crashing with an `Unauthorized` error when trying to sync resources. What would you check?

<details>
<summary>Expected answer</summary>

The ArgoCD IRSA role (`modules/iam/main.tf`) has a trust policy condition:

```hcl
"system:serviceaccount:argocd:argocd-application-controller"
```

Check in this order:

1. **Namespace name** — is ArgoCD actually installed in the `argocd` namespace? If it was installed in `argocd-system` or another namespace, the trust condition never matches and STS rejects the token.

2. **Service account annotation** — the Helm values for ArgoCD must annotate the service account with the role ARN:
   ```yaml
   serviceAccount:
     annotations:
       eks.amazonaws.com/role-arn: arn:aws:iam::ACCOUNT:role/zen-pharma-dev-argocd-role
   ```
   If this annotation is missing, Kubernetes does not project the OIDC token into the pod.

3. **OIDC provider ARN** — verify `var.oidc_provider_arn` in `modules/iam` received the correct value from `module.eks.oidc_provider_arn`. A mismatch means the trust policy references a different cluster's OIDC provider.

4. **ArgoCD role has no AWS permissions attached** — this is correct. The role exists purely for EKS authentication. ArgoCD's access to Kubernetes resources is controlled by K8s RBAC (the `aws-auth` ConfigMap), not IAM policies.

</details>

---

## 2. Kubernetes Concepts

---

**Q4.** What is Helm and how does it relate to your GitOps values files?

<details>
<summary>Expected answer</summary>

Helm is a Kubernetes package manager. A Helm chart is a parameterised template for Kubernetes manifests — Deployments, Services, ConfigMaps, Ingresses. Values files (`values-<service>.yaml`) provide the runtime parameters — image repository, image tag, replica count, resource limits, environment variables. ArgoCD renders the Helm chart with the values file to produce the final Kubernetes manifests and applies them to the cluster. In a GitOps model, you never run `helm upgrade` manually — ArgoCD does it when values files change.

**Project context:** `zen-gitops` has a shared Helm chart (`helm-charts/`) and per-service values files in `envs/`. CI patches only `image.tag` in the values file. Everything else (replicas, resources, ingress) is managed separately in the values files by the platform team.

</details>

---

**Q5.** What happens in Kubernetes during a rolling update?

<details>
<summary>Expected answer</summary>

A rolling update gradually replaces old pods with new ones. Kubernetes creates a new ReplicaSet for the new version. It brings up new pods (controlled by `maxSurge` — how many extra pods can exist during the update) and only terminates old pods once new ones pass their readiness probe (controlled by `maxUnavailable` — how many old pods can be terminated before new ones are ready). This ensures zero downtime — at no point during the rollout are all pods unavailable. If the new pods fail their readiness probe, the rollout stalls and does not terminate old pods, limiting the blast radius.

</details>

---

**Q6.** What is a Kubernetes readiness probe vs a liveness probe?

<details>
<summary>Expected answer</summary>

A readiness probe determines whether a pod is ready to receive traffic — if it fails, the pod is removed from the Service endpoint list but continues running. A liveness probe determines whether a pod is alive — if it fails, Kubernetes restarts the pod. Readiness is used to handle slow startup (Spring Boot apps can take 30–60 seconds to be ready) and temporary unreadiness (e.g. during a dependency outage). Liveness is used to detect deadlocks or hung processes that are still running but not functioning. You typically need both: readiness for traffic management, liveness for self-healing.

</details>

---

**Q7.** What is a Kyverno policy and how does it enforce security at the cluster level?

<details>
<summary>Expected answer</summary>

Kyverno is a Kubernetes-native policy engine that runs as a validating and mutating admission webhook. It uses CRDs (`ClusterPolicy`, `Policy`) written in YAML to define rules. Validating policies reject resources that violate rules. Mutating policies automatically modify resources (e.g. add default labels or resource limits). Common policies: require non-root containers (`runAsNonRoot: true`), require resource limits on all containers, require image signature verification (Cosign), disallow `latest` image tags, require specific labels for cost allocation. Unlike OPA/Gatekeeper which uses Rego, Kyverno policies are YAML-native and easier to read.

</details>

---

## 3. Incident Response Scenarios

---

## H1

### Question
> "A pod in the prod namespace is stuck in `CrashLoopBackOff`. Walk me through your exact debugging steps — what commands do you run and in what order?"

### What the interviewer is really testing
- Systematic debugging vs guessing
- Exact kubectl command knowledge
- Ability to interpret common error patterns

---

### Model Answer

**CrashLoopBackOff means:** the container started, ran briefly, crashed, and Kubernetes keeps restarting it with exponential backoff (10s → 20s → 40s → 80s → 5min cap).

---

### Step-by-step with exact commands

**Step 1: Get the pod name and quick overview**
```bash
kubectl get pods -n prod
# NAME                           READY   STATUS             RESTARTS   AGE
# auth-service-7d9f8c-xk2pv      0/1     CrashLoopBackOff   5          8m
```

**Step 2: Describe the pod — read Events section first**
```bash
kubectl describe pod auth-service-7d9f8c-xk2pv -n prod
# Focus on:
# - Image: 516209541629.dkr.ecr.us-east-1.amazonaws.com/auth-service:sha-xxx
# - Last State: Terminated / Exit Code
# - Events: OOMKilled? ImagePullBackOff? Liveness failed?
```

Common exit codes:
| Exit Code | Meaning |
|-----------|---------|
| 0 | Clean exit (unexpected for a server) |
| 1 | Application error (check logs) |
| 137 | OOMKilled (memory limit exceeded) |
| 139 | Segfault |
| 143 | SIGTERM — graceful shutdown not completing |

**Step 3: Read the logs from the CRASHED container (--previous flag)**
```bash
# Logs from the container that just crashed (not the new one starting)
kubectl logs auth-service-7d9f8c-xk2pv -n prod --previous

# Common findings:
# - "Cannot create bean... DataSource" → DB not reachable
# - "Caused by: java.lang.OutOfMemoryError" → OOM before K8s kills it
# - "Failed to load secret: jwt-secret key not found" → Missing secret
# - "Port 8081 already in use" → Port conflict (rare in containers)
```

**Step 4: Read the logs of the currently running (starting) container**
```bash
kubectl logs auth-service-7d9f8c-xk2pv -n prod
# See if current startup attempt shows the same error
# If it's in the backoff window, it may not have started yet
```

**Step 5: Check env vars and mounted secrets**
```bash
# Only possible during the brief window when pod is Running before crash
# Or exec into an identical pod in dev
kubectl exec -n prod auth-service-7d9f8c-xk2pv -- printenv | grep -E "DB_|JWT_|SPRING_"

# Check if secrets exist and have values
kubectl get secret db-credentials -n prod -o jsonpath='{.data}' | jq
kubectl get secret jwt-secret -n prod -o jsonpath='{.data.JWT_SECRET}' | base64 -d
```

**Step 6: Check recent events at namespace level**
```bash
kubectl get events -n prod --sort-by='.lastTimestamp' | tail -20
# Look for: FailedScheduling, BackOff, OOMKilling, FailedMount
```

**Step 7: Check resource usage just before crash**
```bash
kubectl top pod -n prod
# If you're too late, check Prometheus:
# container_memory_working_set_bytes{pod="auth-service-..."}
```

---

### Decision tree by exit code

```
CrashLoopBackOff
       │
       ├── Exit Code 137 (OOMKilled)
       │       → kubectl describe pod: look for "OOMKilled" in Last State
       │       → Fix: increase memory limit in values-auth-service.yaml
       │         or set JVM heap: -Xmx400m (lower than 512Mi limit)
       │
       ├── Exit Code 1 (App crash)
       │       → kubectl logs --previous: look for Java exception
       │       │
       │       ├── DB connection refused → check DB host, network policy, RDS status
       │       ├── Secret not found → check ESO sync status
       │       └── Port already in use → check SERVICE_PORT vs server.port config
       │
       ├── Exit Code 0 (Clean exit)
       │       → Application is exiting immediately (treating crash as success)
       │       → Check if entrypoint is wrong or app has no daemon process
       │
       └── ImagePullBackOff (different status)
               → check image tag, ECR permissions, imagePullSecrets
```

---

## H2

### Question
> "The auth-service pod is in `Pending` state for 10 minutes after a deployment. What are all the possible causes and how do you isolate each one?"

### What the interviewer is really testing
- Comprehensive Kubernetes scheduling knowledge
- Systematic narrowing from many possibilities
- `kubectl describe pod` reading skills

---

### Model Answer

**Pending means:** the pod exists but the Kubernetes scheduler has not assigned it to a node yet (or a node assignment failed).

---

### All possible causes

**Step 1: Always start with `kubectl describe pod`**
```bash
kubectl describe pod auth-service-<hash> -n prod
# The Events section at the bottom tells you EXACTLY why it's pending
# If Events is empty — the scheduler hasn't even tried yet
```

---

### Cause 1: Insufficient resources (most common)

```
Events:
  Warning  FailedScheduling  10m  default-scheduler  
    0/3 nodes are available: 3 Insufficient cpu, 3 Insufficient memory.
```

```bash
# Check node capacity vs current usage
kubectl top nodes
kubectl describe nodes | grep -A5 "Allocated resources"

# Check what the pod is requesting
kubectl describe pod auth-service-<hash> -n prod | grep -A5 "Requests:"
# Requests: cpu: 100m, memory: 256Mi (from values-auth-service.yaml)

# Fix: scale up the node group (ASG) or reduce requests
```

---

### Cause 2: ResourceQuota exceeded (namespace-level limit)

```bash
kubectl describe resourcequota -n prod
# If quota is set and the new pod exceeds it → Pending

# Check if a quota is blocking
kubectl get resourcequota -n prod
kubectl describe resourcequota -n prod
```

---

### Cause 3: PersistentVolumeClaim not bound

From auth-service values — it has a `tmp` emptyDir, not a PVC. But if it were:
```
Events:
  Warning  FailedScheduling  pod has unbound immediate PersistentVolumeClaims
```

```bash
kubectl get pvc -n prod
# STATUS should be Bound; if Pending → storage class provisioner issue
kubectl describe pvc <pvc-name> -n prod
```

---

### Cause 4: Node selector / affinity not matching

```bash
kubectl describe pod auth-service-<hash> -n prod | grep -A3 "Node-Selectors"
# If nodeSelector is set and no node matches:
# "0/3 nodes are available: 3 node(s) didn't match node selector"

# Check node labels
kubectl get nodes --show-labels
```

---

### Cause 5: Taints and tolerations

```bash
# Check if nodes have taints that the pod doesn't tolerate
kubectl describe nodes | grep -A3 "Taints:"
kubectl describe pod auth-service-<hash> -n prod | grep -A3 "Tolerations:"
```

Common example: a node tainted `node.kubernetes.io/not-ready:NoSchedule` means the node itself has a problem.

---

### Cause 6: Image pull issue (pod goes Pending then ImagePullBackOff)

```bash
kubectl describe pod auth-service-<hash> -n prod
# Events: Failed to pull image "516209541629.dkr.ecr.us-east-1.amazonaws.com/auth-service:sha-xxx"
# Check: does imagePullSecrets exist? Is ECR auth working?
kubectl get secret -n prod | grep ecr
```

---

### Cause 7: Topology spread constraints too strict

From `helm-charts/values.yaml`, `topologySpreadConstraints: []` by default. But if set:
```bash
# If maxSkew: 1 and only 2 zones available with 3 pods → can't schedule
# "0/3 nodes are available: 3 node(s) didn't match pod topology spread constraints"
kubectl describe pod auth-service-<hash> -n prod | grep -A5 "TopologySpreadConstraints"
```

---

### Quick isolation flowchart

```
Pod Pending for >2 min
       │
       └── kubectl describe pod → check Events
               │
               ├── "Insufficient cpu/memory" → check node capacity, resize requests
               ├── "unbound PVC" → check storage class, PVC status
               ├── "didn't match node selector" → check node labels
               ├── "didn't match taint" → check node taints + pod tolerations
               ├── "didn't match topology spread" → relax maxSkew or add nodes
               ├── "quota exceeded" → check ResourceQuota
               └── "no events at all" → scheduler overwhelmed, check kube-scheduler logs
```

---


## H3

### Question
> "The drug-catalog-service pod is in `ImagePullBackOff`. You can see the image in ECR. Walk me through every reason this can happen and how you fix each one."

### What the interviewer is really testing
- ECR authentication mechanics on EKS
- IRSA understanding for ECR pull
- Systematic elimination of causes

---

### Model Answer

**ImagePullBackOff means:** the kubelet tried to pull the image and failed. Kubernetes backs off exponentially before retrying.

---

### Step 1: Get the exact error

```bash
kubectl describe pod drug-catalog-service-<hash> -n prod
# Events section:
# Failed to pull image "516209541629.dkr.ecr.us-east-1.amazonaws.com/drug-catalog-service:sha-abc123"
# Error: rpc error: code = Unknown desc = failed to pull and unpack image:
#   failed to resolve reference "...":
#   unexpected status code 401 Unauthorized

# Or:
# Error: rpc error: code = Unknown desc = failed to pull and unpack image:
#   failed to resolve reference "...":
#   repository "drug-catalog-service" not found
```

---

### Possible causes and fixes

**Cause 1: ECR authorization token expired (most common)**

ECR tokens are valid for 12 hours. EKS nodes use the `ec2:ecr-token-manager` via the node IAM role to refresh them automatically — but the role must have the right permissions.

```bash
# Check the node IAM role has ECR pull permissions
aws iam get-role-policy \
  --role-name pharma-prod-cluster-node-group-role \
  --policy-name AmazonEC2ContainerRegistryReadOnly

# Or check the attached managed policies
aws iam list-attached-role-policies \
  --role-name pharma-prod-cluster-node-group-role
# Expected: AmazonEC2ContainerRegistryReadOnly is attached
```

**Fix:** Attach `AmazonEC2ContainerRegistryReadOnly` to the node group IAM role in Terraform and apply.

---

**Cause 2: Image tag or SHA doesn't exist in ECR**

The values file references a tag that was never pushed or was overwritten.

```bash
# List all tags for the repository
aws ecr list-images \
  --repository-name drug-catalog-service \
  --region us-east-1 \
  --query 'imageIds[*].imageTag' \
  --output table

# Check what tag ArgoCD is trying to pull
kubectl get deployment drug-catalog-service -n prod \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
```

**Fix:** Check the zen-gitops values file. If the tag is missing from ECR, re-run the CI pipeline for that commit.

---

**Cause 3: Cross-account image (wrong account ID in ECR URI)**

```bash
# The image URI in the pod spec
kubectl get pod drug-catalog-service-<hash> -n prod \
  -o jsonpath='{.spec.containers[0].image}'
# 516209541629.dkr.ecr.us-east-1.amazonaws.com/drug-catalog-service:sha-abc123
# ^^^^^^^^^^^^ This must be YOUR account ID
```

**Fix:** Correct the account ID in the Helm values file and redeploy.

---

**Cause 4: Wrong region**

```bash
# ECR region must match where the image was pushed
# us-east-1 in URI vs node in us-west-2 → will fail
kubectl get pod drug-catalog-service-<hash> -n prod \
  -o jsonpath='{.spec.containers[0].image}' | grep -o 'ecr\.[^.]*'
# ecr.us-east-1 ← confirm this matches your cluster region
```

---

**Cause 5: Network policy blocking egress to ECR**

```bash
# Check NetworkPolicy in prod namespace
kubectl get networkpolicy -n prod

# If egress is restricted, nodes may not reach ECR endpoint
# ECR uses regional endpoints: api.ecr.us-east-1.amazonaws.com
# For private ECR: VPC endpoints for ECR (ecr.api, ecr.dkr, s3) must exist
```

---

### Quick diagnosis flowchart

```
ImagePullBackOff
       │
       └── kubectl describe pod → Events → exact error message
               │
               ├── 401 Unauthorized → node IAM role missing ECR policy
               ├── repository not found → wrong account ID or region
               ├── manifest unknown → tag doesn't exist in ECR
               ├── timeout / no route → network policy or VPC endpoint missing
               └── name resolution failed → DNS issue or private endpoint misconfigured
```

---


## H4

### Question
> "ArgoCD shows the auth-service application as `OutOfSync` even though no one changed the Helm values. What causes this and how do you handle it without blindly syncing?"

### What the interviewer is really testing
- Understanding of ArgoCD's sync algorithm and what it compares
- Drift awareness — not everything in `OutOfSync` is a problem
- Judgment: investigate before sync, not sync first

---

### Model Answer

**OutOfSync means:** ArgoCD computed the desired manifests from the Git repo and compared them to the live cluster state, and found a difference. This does NOT always mean something is wrong.

---

### Common causes of unexpected OutOfSync

**Cause 1: Resource mutation by admission controllers (most common)**

Kyverno or other admission webhooks can mutate resources on create/update, adding annotations, labels, or fields that are not in the Helm values. ArgoCD sees the cluster object as different from what Helm would render.

```bash
# Check what ArgoCD sees as the diff
argocd app diff auth-service-prod

# Example output:
# + metadata.annotations:
# +   kyverno.io/last-applied-configuration: ...
# +   kubectl.kubernetes.io/last-applied-configuration: ...
```

**Fix:** Add an ArgoCD `ignoreDifferences` rule for this annotation in the Application spec.

```yaml
spec:
  ignoreDifferences:
  - group: apps
    kind: Deployment
    jsonPointers:
    - /metadata/annotations/kyverno.io~1last-applied-configuration
```

---

**Cause 2: Horizontal Pod Autoscaler changed replica count**

If an HPA is active, it modifies `spec.replicas` on the Deployment. ArgoCD then sees the replica count in the cluster differs from the Helm values.

```bash
kubectl get hpa -n prod
# NAME           REFERENCE                        REPLICAS
# auth-service   Deployment/auth-service          3        ← HPA scaled to 3
# Helm values say: replicaCount: 1 → OutOfSync

argocd app diff auth-service-prod | grep replicas
```

**Fix:** Either ignore `spec.replicas` in ArgoCD, or remove `replicaCount` from the Helm values when HPA is enabled.

```yaml
spec:
  ignoreDifferences:
  - group: apps
    kind: Deployment
    jsonPointers:
    - /spec/replicas
```

---

**Cause 3: Someone ran `kubectl apply` or `kubectl edit` directly**

This is drift — the cluster state was changed outside GitOps. This is the case where you SHOULD sync (after reviewing the diff).

```bash
# See what changed and who changed it
kubectl describe deployment auth-service -n prod | grep -A5 "Annotations:"
# kubectl.kubernetes.io/last-applied-configuration shows the last kubectl apply

# Check ArgoCD audit log for when OutOfSync appeared
argocd app history auth-service-prod
```

**Fix:** Review the diff (`argocd app diff`), determine if the manual change is intentional (if so, add it to git), then sync ArgoCD.

---

**Cause 4: Helm rendering is non-deterministic**

Some Helm charts use `randAlphaNum` or `now` in templates — these generate different values on each render. ArgoCD re-renders on every sync cycle and sees a difference every time.

```bash
argocd app diff auth-service-prod | grep -i rand
```

**Fix:** Replace dynamic values in Helm templates with static values or use a Kubernetes Secret with a stable value set at install time.

---

### Decision flow before syncing

```
ArgoCD shows OutOfSync
       │
       └── argocd app diff auth-service-prod
               │
               ├── Only annotations/metadata → likely admission controller → add ignoreDifferences
               ├── Replica count only → HPA active → add ignoreDifferences for /spec/replicas
               ├── Actual config change → check git log, someone may have manually changed cluster
               │         → review diff, if OK, add to git, then sync
               └── Random-looking value → non-deterministic Helm template → fix template
```

**Never blindly sync OutOfSync.** Always run `argocd app diff` first to understand what changed.

---


## H5

### Question
> "A new deployment of auth-service succeeded (all pods running) but `/actuator/health/readiness` keeps returning 503. Traffic is not routing to the new pods. Debug this."

### What the interviewer is really testing
- Understanding of Kubernetes readiness probe + service endpoint mechanics
- Knowing that readiness failure = removed from Service endpoints
- Systematic multi-layer debugging

---

### Model Answer

**Key mechanics:** When a pod's readiness probe fails, Kubernetes removes it from the Service's Endpoints object. The nginx ingress → Service → no endpoints → 503 or traffic only going to old pods.

---

### Step-by-step

**Step 1: Confirm readiness probe is failing**
```bash
kubectl describe pod auth-service-<hash> -n prod | grep -A10 "Readiness:"
# Readiness:  http-get http://:8081/actuator/health/readiness
# Failure threshold: 3 times in a row → pod not ready

kubectl get pods -n prod
# READY column shows 0/1 for affected pods
```

**Step 2: Manually hit the readiness endpoint to see the actual response**
```bash
kubectl exec -n prod auth-service-<hash> -- \
  curl -v http://localhost:8081/actuator/health/readiness

# Possible responses:
# 200 {"status":"UP"} → probe passing now but wasn't before? Timing issue
# 200 {"status":"DOWN"} → app is reporting itself down
# 503 {"status":"OUT_OF_SERVICE"} → Spring component is not ready
# Connection refused → app not listening on 8081
# Timeout → app started but is stuck
```

**Step 3: Read the full health response to see which component is DOWN**
```bash
kubectl exec -n prod auth-service-<hash> -- \
  curl -s http://localhost:8081/actuator/health | python3 -m json.tool

# Example:
# {
#   "status": "DOWN",
#   "components": {
#     "db": {
#       "status": "DOWN",
#       "details": {
#         "error": "Unable to acquire JDBC Connection: Timeout after 30000ms"
#       }
#     },
#     "diskSpace": { "status": "UP" }
#   }
# }
```

**Step 4: Check if the Service has endpoints**
```bash
kubectl get endpoints auth-service -n prod
# NAME           ENDPOINTS         AGE
# auth-service   <none>            5m   ← no endpoints! All pods failing readiness

# Once pods pass readiness:
# auth-service   10.0.1.45:8081    5m
```

---

### Common root causes for readiness 503

| Cause | Symptom in `/actuator/health` | Fix |
|-------|------------------------------|-----|
| DB not reachable | `"db": {"status": "DOWN"}` | Check DB_HOST config, RDS security group, network policy |
| DB credential wrong after rotation | `"db": {"status": "DOWN", "error": "Auth failed"}` | Check ESO sync, restart pods after secret update |
| Missing secret (env var empty) | App fails to start DataSource bean | Check ESO sync status |
| Port mismatch | Connection refused | Verify SERVICE_PORT env var matches values file |
| Slow JVM startup (within initialDelaySeconds) | Probe fires too early | Increase `readinessProbe.initialDelaySeconds` in values file |
| App deployed with wrong Spring profile | Different config loaded | Check SPRING_PROFILES_ACTIVE env var |

---

### Specific to this repo — check configmap values

```bash
kubectl get configmap auth-service -n prod -o yaml
# Verify:
# DB_HOST: pharma-prod-postgres.cs3c424yurej.us-east-1.rds.amazonaws.com (correct)
# SERVER_PORT: "8081" (matches readiness probe port)
# SPRING_PROFILES_ACTIVE: prod
```

---


## H6

### Question
> "A Kubernetes NetworkPolicy was applied to the prod namespace and now inter-service calls are failing. The drug-catalog-service can no longer reach the auth-service. How do you diagnose and fix it without removing the NetworkPolicy entirely?"

### What the interviewer is really testing
- Understanding of NetworkPolicy mechanics (default-deny model)
- How to test connectivity inside the cluster
- Writing targeted allow rules rather than giving up and disabling security

---

### Model Answer

**NetworkPolicy mechanics:** By default, pods accept all traffic. Once ANY NetworkPolicy selects a pod via `podSelector`, all traffic not explicitly allowed by a policy is denied. This is the "default deny" effect.

---

### Step 1: Identify which NetworkPolicy is selecting the affected pods

```bash
# List all NetworkPolicies in prod
kubectl get networkpolicy -n prod

# Describe to see pod selectors and rules
kubectl describe networkpolicy -n prod
# Look for:
# PodSelector: app.kubernetes.io/name=auth-service
# Ingress:     <allowed sources>
# Egress:      <allowed destinations>
```

---

### Step 2: Test connectivity directly

```bash
# Exec into the drug-catalog-service and try to reach auth-service
kubectl exec -n prod deployment/drug-catalog-service -- \
  curl -v http://auth-service:8081/actuator/health --max-time 5

# Expected if blocked:
# * connect to 10.100.x.x port 8081 failed: Connection timed out
# (timeout, not refused — NetworkPolicy drops packets silently)
```

---

### Step 3: Understand the problem — what is missing

NetworkPolicy operates on label selectors. The drug-catalog-service must be listed as an allowed ingress source for the auth-service policy.

```bash
# Get labels on drug-catalog-service pods
kubectl get pods -n prod -l app.kubernetes.io/name=drug-catalog-service --show-labels

# Get the auth-service NetworkPolicy ingress rules
kubectl get networkpolicy auth-service-policy -n prod -o yaml
# Look for ingress.from.podSelector
# If drug-catalog-service is not in the from list → traffic is blocked
```

---

### Step 4: Fix — add a targeted ingress rule (do NOT remove the policy)

Add a rule that allows traffic from drug-catalog-service pods to auth-service on port 8081:

```yaml
# Patch the existing NetworkPolicy or update via Helm values:
# In helm-charts/templates/networkpolicy.yaml:

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: auth-service-policy
  namespace: prod
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: auth-service
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app.kubernetes.io/name: drug-catalog-service   # ← add this
    ports:
    - protocol: TCP
      port: 8081
  - from:
    - podSelector:
        matchLabels:
          app.kubernetes.io/name: ingress-nginx          # NGINX ingress
    ports:
    - protocol: TCP
      port: 8081
```

---

### Step 5: Verify after fix

```bash
# Re-test connectivity
kubectl exec -n prod deployment/drug-catalog-service -- \
  curl -s http://auth-service:8081/actuator/health

# Expected:
# {"status":"UP"}
```

---

### Common NetworkPolicy mistakes

| Mistake | Effect |
|---------|--------|
| No `namespaceSelector` when calling cross-namespace | Block all cross-namespace traffic even with correct podSelector |
| Policy only has `Ingress` type but pod needs egress to DB | Egress to DB still works (egress not selected), ingress blocked |
| Policy added after readiness probe configured | Kubelet's readiness probe (from node, not pod) gets blocked → pod stuck not-ready |
| Wrong label in `podSelector` | All traffic blocked: policy selects no pods, so nothing is allowed |

---

### Key rule: ingress to auth-service must also allow kube-system

```bash
# The kubelet and kube-system components need to reach pods for health probes
# If you forget namespaceSelector for kube-system:
kubectl get networkpolicy auth-service-policy -n prod -o yaml | \
  grep -A10 "from:"
# Ensure kube-system namespace is allowed for probe traffic
```

---


## H7

### Question
> "You get an OOMKilled event on the manufacturing-service. The pod restarts every 30 minutes. How do you investigate root cause and what is your immediate vs permanent fix?"

### What the interviewer is really testing
- OOM debugging methodology
- JVM-specific memory knowledge (for Spring Boot services)
- Distinguishing memory leak from under-provisioning

---

### Model Answer

OOMKilled = Linux kernel OOM killer terminated the process because it exceeded the container's memory limit (currently `512Mi` in this repo).

---

### Step 1: Confirm OOMKilled

```bash
kubectl describe pod manufacturing-service-<hash> -n prod
# Last State:     Terminated
#   Reason:       OOMKilled
#   Exit Code:    137
#   Started:      Thu, 07 May 2026 02:00:00 +0000
#   Finished:     Thu, 07 May 2026 02:28:00 +0000
```

---

### Step 2: Get the memory usage trend BEFORE the kill

```bash
# In Prometheus/Grafana:
container_memory_working_set_bytes{
  pod=~"manufacturing-service-.*",
  namespace="prod",
  container="pharma-service"
}
```

Two distinct patterns tell you different stories:

```
Pattern A — Memory leak:
  MB
 512 │                      ██ ← killed
     │                  ████
     │            ██████
     │        ████
 256 │   █████
     └─────────────────────── time
     (gradual linear increase over 30 min → memory leak)

Pattern B — Under-provisioning:
  MB
 512 │ ██ ← killed immediately on startup
     │
     └─────────────────────── time
     (instant spike → JVM heap set too high for limit)
```

---

### Step 3: Check JVM heap configuration

For Spring Boot services, the JVM needs explicit heap sizing:

```bash
kubectl exec -n prod deployment/manufacturing-service -- \
  java -XX:+PrintFlagsFinal -version 2>&1 | grep -i "heapsize\|xmx"

# Common problem:
# JVM defaults to 25% of container memory as max heap = 128Mi (512Mi * 0.25)
# But JVM also uses metaspace, thread stacks, native memory
# Total JVM memory = heap + metaspace + threads + etc. can easily exceed 512Mi
```

**Fix for JVM memory management:**
```yaml
# In configmap of manufacturing-service values:
configmap:
  JAVA_OPTS: "-Xms256m -Xmx384m -XX:MaxMetaspaceSize=96m -XX:+UseContainerSupport"
  # Xmx384m + MaxMetaspace96m + overhead ≈ 480m < 512Mi limit
```

Or use `XX:+UseContainerSupport` (Java 11+) which automatically respects container limits:
```yaml
JAVA_OPTS: "-XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0"
# 75% of 512Mi = 384Mi for heap → safer
```

---

### Step 4: Investigate memory leak (if Pattern A)

```bash
# Get heap dump before next OOM kill
kubectl exec -n prod deployment/manufacturing-service -- \
  jcmd 1 GC.heap_info

kubectl exec -n prod deployment/manufacturing-service -- \
  jcmd 1 VM.native_memory summary

# For full heap dump (warning: large file):
kubectl exec -n prod deployment/manufacturing-service -- \
  jmap -dump:format=b,file=/tmp/heap.hprof 1
kubectl cp prod/manufacturing-service-<pod>:/tmp/heap.hprof ./heap.hprof
# Analyze with Eclipse MAT or VisualVM
```

Common Spring Boot memory leak causes:
- `@Cacheable` with no eviction policy and unbounded keys
- Database ResultSet not closed (connection leak → memory accumulates)
- Static collections growing indefinitely
- Hibernate L2 cache misconfiguration

---

### Immediate vs Permanent fix

| Timeframe | Action |
|-----------|--------|
| **Immediate** | Increase memory limit to `768Mi` or `1Gi` (buys time for investigation) |
| **Immediate** | Set explicit `-Xmx` flag to prevent JVM from consuming all available memory |
| **Short-term** | Enable GC logging to see if GC is running frequently before OOM |
| **Permanent** | Find and fix the leak: heap dump analysis, add `spring.cache.caffeine.spec=maximumSize=1000,expireAfterWrite=10m` |
| **Permanent** | Add Prometheus alert: memory usage > 80% of limit for 5 minutes → page before OOM |

---


## H8

### Question
> "A rolling update of the manufacturing-service is stuck. Old pods are terminating but new pods stay in `ContainerCreating` for 15 minutes. The deployment shows 0 available replicas. What do you do?"

### What the interviewer is really testing
- Rolling update mechanics — maxUnavailable, maxSurge
- Distinguishing scheduling failure from image pull from volume mount failure
- Safe intervention: pause, investigate, resume or roll back

---

### Model Answer

**ContainerCreating means:** the pod is scheduled to a node but the container has not started. The kubelet is preparing the container — pulling image, mounting volumes, setting up networking.

If it's stuck here for more than 5 minutes, something is blocking container startup.

---

### Step 1: Pause the rollout immediately

```bash
# Before you investigate, stop the rollout from creating more stuck pods
kubectl rollout pause deployment/manufacturing-service -n prod

# Check the current rollout status
kubectl rollout status deployment/manufacturing-service -n prod
# "Waiting for rollout to finish: 1 out of 3 new replicas have been updated..."
```

---

### Step 2: Describe a stuck new pod

```bash
# Find new pods (ContainerCreating)
kubectl get pods -n prod -l app.kubernetes.io/name=manufacturing-service

# Describe the stuck one
kubectl describe pod manufacturing-service-<new-hash> -n prod
# Focus on Events:
# - "pulling image..." → still pulling (check if this is a large image on slow network)
# - "failed to pull image" → ImagePullBackOff (see H3)
# - "MountVolume.SetUp failed" → volume mount issue
# - "RunContainerError" → container started but failed immediately (different from ContainerCreating)
# - No events at all → kubelet hasn't processed it yet
```

---

### Step 3: Investigate by event type

**If stuck on image pull:**
```bash
# How large is the new image?
aws ecr describe-images \
  --repository-name manufacturing-service \
  --region us-east-1 \
  --query 'imageDetails[?contains(imageTags,`sha-newver`)].[imageSizeInBytes]'
# If >1GB on a t3.small node → pulls can take 10-15 min on first pull

# Is the image already cached on the node?
# (no direct command, but if all pods land on same node they share the cache)
```

**If stuck on volume mount (InitContainer or ConfigMap/Secret):**
```bash
# Check if ConfigMap/Secret referenced in new version exists
kubectl get configmap manufacturing-service -n prod
kubectl get secret db-credentials -n prod

# Check if ExternalSecret has synced the new secret version
kubectl get externalsecret -n prod
kubectl describe externalsecret db-credentials -n prod | grep -A5 "Conditions:"
```

**If stuck due to init container:**
```bash
# Init containers must complete before the main container starts
kubectl get pod manufacturing-service-<new-hash> -n prod -o jsonpath='{.status.initContainerStatuses}'
# Look for: "state": {"waiting": {"reason": "CrashLoopBackOff"}}
kubectl logs manufacturing-service-<new-hash> -n prod -c init-<name>
```

---

### Step 4: Decision — resume or roll back

```bash
# If you identified and fixed the issue (e.g., pushed the missing secret):
kubectl rollout resume deployment/manufacturing-service -n prod

# If the new image is fundamentally broken and fixing takes time:
kubectl rollout undo deployment/manufacturing-service -n prod
# This rolls back to the previous ReplicaSet revision

# Verify rollback completed
kubectl rollout status deployment/manufacturing-service -n prod
# "deployment "manufacturing-service" successfully rolled out"

kubectl get pods -n prod -l app.kubernetes.io/name=manufacturing-service
# All pods should be Running and 1/1 READY again
```

---

### maxUnavailable and maxSurge context

From this repo's Helm chart default strategy:
```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 0    # Never reduce below desired replicas
    maxSurge: 1          # Can create 1 extra pod during rollout
```

With `maxUnavailable: 0`, a stuck new pod blocks the entire rollout — no old pods are terminated until new ones are ready. This is the safe default for production but means a broken new image can leave the deployment stuck indefinitely.

---


## H9

### Question
> "Nginx ingress is returning 502 Bad Gateway for the pharma-ui. The pods are running and healthy. Walk through every layer you'd check."

### What the interviewer is really testing
- Full network path knowledge: external → ingress → service → pod
- Nginx-specific knowledge (502 vs 503 vs 504 meanings)
- Reading ingress configuration

---

### Model Answer

**502 Bad Gateway means:** nginx received a bad/invalid response from the upstream (backend pod). It reached the pod but got something unexpected.

Compare: 503 = no upstream available, 504 = upstream timed out.

---

### Layer-by-layer investigation

**Layer 1: Verify pods are actually running AND ready**
```bash
kubectl get pods -n prod -l app.kubernetes.io/name=pharma-ui
# READY must be 1/1, not 0/1
# If 0/1 → readiness probe failing → see H5

kubectl get endpoints pharma-ui -n prod
# If <none> → all pods failing readiness → service has no backends → 503 not 502
# If endpoints exist → pods are ready, but nginx is getting bad response
```

**Layer 2: Check if nginx can reach the pod directly**
```bash
# Get pod IP
POD_IP=$(kubectl get pod -n prod -l app.kubernetes.io/name=pharma-ui \
  -o jsonpath='{.items[0].status.podIP}')

# Exec into nginx pod and try the backend directly
kubectl exec -n ingress-nginx deployment/ingress-nginx-controller -- \
  curl -v http://${POD_IP}:8080/
# If this returns 200 → nginx routing config is wrong
# If this returns something unexpected → app is returning bad response
```

**Layer 3: Check ingress configuration**
```bash
kubectl describe ingress pharma-ui -n prod
# Check:
# - Backend service name and port match actual Service
# - ingressClassName: nginx (matches the nginx controller)
# - Path: correct?

kubectl get ingress pharma-ui -n prod -o yaml
```

From this repo's pattern (`envs/prod/values-pharma-ui.yaml`):
```yaml
ingress:
  enabled: true
  className: nginx
  host: prod.pharma.internal
  path: /
  pathType: Prefix
```

**Layer 4: Check the Service**
```bash
kubectl describe service pharma-ui -n prod
# Selector must match pod labels
# Port must match targetPort

kubectl get service pharma-ui -n prod -o yaml
# spec.ports[].targetPort should match the port pharma-ui listens on
```

Common mistake: `service.port: 8080` but `service.targetPort: 3000` (UI is on 3000) — nginx reaches the pod but nothing is listening on 3000.

**Layer 5: Check nginx logs**
```bash
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller --since=5m | \
  grep -i "pharma-ui\|502\|bad gateway"

# Common nginx upstream error messages:
# "connect() failed (111: Connection refused)"  → wrong port
# "upstream sent invalid header"               → app not speaking HTTP
# "recv() failed (104: Connection reset by peer)" → app crashed mid-response
```

**Layer 6: Check TLS / SSL passthrough**

From this repo's ArgoCD ingress pattern:
```yaml
annotations:
  nginx.ingress.kubernetes.io/ssl-passthrough: "true"
```

If pharma-ui has ssl-passthrough set but the pod is not serving HTTPS, nginx gets garbage back → 502.

```bash
kubectl get ingress pharma-ui -n prod -o yaml | grep -A5 annotations
# If ssl-passthrough: "true" → pod must serve HTTPS
# If not set → nginx handles TLS, pod serves plain HTTP
```

**Layer 7: Check if it's a protocol mismatch (HTTP/2 vs HTTP/1.1)**
```bash
# If pharma-ui is gRPC or HTTP/2 but nginx defaults to HTTP/1.1:
# Add annotation:
# nginx.ingress.kubernetes.io/backend-protocol: "GRPC"  or "HTTP2"
kubectl describe ingress pharma-ui -n prod | grep backend-protocol
```

---

### 502 investigation summary

```
502 Bad Gateway
       │
       ├── Pods not ready? (0/1) → readiness probe issue (see H5)
       │
       ├── No endpoints? → all pods failing readiness
       │
       ├── Nginx can't reach pod? → network policy blocking ingress → pod
       │
       ├── Wrong port in Service? → connection refused
       │
       ├── SSL mismatch? → ssl-passthrough on non-HTTPS backend
       │
       └── App crashing mid-response? → check pod logs during the 502
```

---

## 4. Troubleshooting Commands Reference

---

## I1

### Question
> "What is the exact sequence of `kubectl` commands you run when a pod is not starting? Give me the actual commands."

### What the interviewer is really testing
- Command fluency — do you hesitate or go straight to the right command?
- Systematic approach vs jumping to conclusions

---

### The exact sequence

```bash
# 1. Get the pod name and see the status at a glance
kubectl get pods -n prod
# Look at: STATUS (Pending/CrashLoopBackOff/OOMKilled/ImagePullBackOff/Terminating)
# Look at: RESTARTS (high number = CrashLoop)
# Look at: READY (0/1 = not ready)

# 2. Get the full picture — Events section is the most important part
kubectl describe pod <pod-name> -n prod
# Scroll to bottom: Events section tells you exactly what happened
# Also check: Limits/Requests, node assigned, volumes mounted

# 3. Read the logs of the crashed container (MUST use --previous for CrashLoop)
kubectl logs <pod-name> -n prod --previous
# --previous = logs from the container that just exited, not the new one starting

# 4. Read the logs of the currently starting container (if available)
kubectl logs <pod-name> -n prod
# May be empty if still in backoff window

# 5. Check namespace-level events sorted by time
kubectl get events -n prod --sort-by='.lastTimestamp' | tail -20
# Catches: scheduling failures, OOM events, image pull failures

# 6. If Pending, check node availability
kubectl top nodes
kubectl describe nodes | grep -E "MemoryPressure|DiskPressure|PIDPressure"

# 7. Check if the relevant secrets/configmaps exist
kubectl get secret db-credentials -n prod
kubectl get configmap auth-service -n prod

# 8. For CrashLoopBackOff — exec in during the brief running window
# (or use a debug pod with same image)
kubectl debug pod/<pod-name> -n prod \
  --image=busybox \
  --copy-to=debug-pod \
  -- sh
```

---

### The 30-second cheatsheet

```bash
# Quick status
kubectl get pods -n prod -o wide

# What happened
kubectl describe pod <name> -n prod | grep -A20 Events

# Why it crashed
kubectl logs <name> -n prod --previous 2>&1 | tail -30

# Namespace-level events
kubectl get events -n prod --sort-by='.lastTimestamp' | tail -10
```

---

## I2

### Question
> "How do you check if an ExternalSecret has successfully synced from AWS Secrets Manager? What does a failed sync look like?"

---

### Commands

```bash
# 1. Quick status of all ExternalSecrets in a namespace
kubectl get externalsecret -n prod
# NAME             STORE                REFRESH INTERVAL   STATUS          READY
# db-credentials   aws-secrets-manager  1h                 SecretSynced    True   ← good
# jwt-secret       aws-secrets-manager  1h                 SecretSyncedError False ← problem

# 2. Full status with conditions and last sync time
kubectl describe externalsecret db-credentials -n prod
# Look for:
# Status:
#   Refresh Time: 2026-05-07T09:45:00Z   ← when last synced
#   Conditions:
#     Type: Ready
#     Status: True
#     Reason: SecretSynced
#     Message: Secret was synced

# 3. Verify the resulting K8s Secret was created and has values
kubectl get secret db-credentials -n prod
kubectl get secret db-credentials -n prod -o jsonpath='{.data}' | jq 'keys'
# Should show: ["DB_PASSWORD","DB_USERNAME"]

# Check values are non-empty (decode base64)
kubectl get secret db-credentials -n prod \
  -o jsonpath='{.data.DB_USERNAME}' | base64 -d && echo

# 4. Check ESO controller logs for errors
kubectl logs -n kube-system deployment/external-secrets --since=1h | \
  grep -i "error\|db-credentials\|failed"
```

---

### What a failed sync looks like

```yaml
# kubectl describe externalsecret db-credentials -n prod

Status:
  Conditions:
  - Last Transition Time: "2026-05-07T09:00:00Z"
    Message: 'could not get secret value: AccessDenied: User: arn:aws:sts::516209541629:assumed-role/external-secrets-role/...
      is not authorized to perform: secretsmanager:GetSecretValue on resource: arn:aws:secretsmanager:us-east-1:516209541629:secret:/pharma/prod/db-credentials*'
    Reason:      SecretSyncedError
    Status:      "False"
    Type:        Ready
```

---

## I3

### Question
> "A node is NotReady. How do you safely drain and cordon it without dropping traffic from running pods?"

---

### Exact procedure

```bash
# 1. Confirm the node is NotReady
kubectl get nodes
# NAME                STATUS     ROLES    AGE
# ip-10-0-1-45.ec2    NotReady   <none>   5d

# 2. Understand WHY before touching it
kubectl describe node ip-10-0-1-45.ec2
# Look for:
# - Conditions: MemoryPressure/DiskPressure/NetworkUnavailable = True
# - Events: kubelet stopped posting status

# 3. CORDON first — prevents NEW pods from being scheduled here
# (existing pods keep running, just no new scheduling)
kubectl cordon ip-10-0-1-45.ec2
# Node ip-10-0-1-45.ec2 cordoned
# Node STATUS becomes: NotReady,SchedulingDisabled

# 4. Check what pods are running on this node
kubectl get pods --all-namespaces \
  --field-selector spec.nodeName=ip-10-0-1-45.ec2

# 5. Check PodDisruptionBudgets before draining
# If a PDB says minAvailable: 1 and there's only 1 pod, drain will BLOCK
kubectl get pdb --all-namespaces

# 6. DRAIN — evicts all pods with grace period
kubectl drain ip-10-0-1-45.ec2 \
  --ignore-daemonsets \          # DaemonSet pods can't be moved — ignore them
  --delete-emptydir-data \       # pods using emptyDir lose their data — accept this
  --grace-period=60 \            # give pods 60s to finish in-flight requests
  --timeout=300s                 # drain must complete within 5 min or fail

# 7. Verify no more non-daemonset pods on the node
kubectl get pods --all-namespaces \
  --field-selector spec.nodeName=ip-10-0-1-45.ec2 | grep -v daemonset

# 8. Now safe to: reboot the node, replace it, investigate the issue

# 9. If node recovers: uncordon to allow scheduling again
kubectl uncordon ip-10-0-1-45.ec2
```

---

### Why cordon before drain?

Cordon prevents new pods from landing on the node. Drain evicts existing pods. Without cordoning first, evicted pods could immediately reschedule onto the same bad node before the drain command removes them.

---

## I4

### Question
> "How do you exec into a running container in the prod namespace if `kubectl exec` is blocked by RBAC? How do you debug network issues from inside a pod without installing tools?"

---

### If kubectl exec is blocked

From this repo's prod RBAC — `pods/exec` is NOT in the prod role. So developers cannot exec into prod pods. Options:

**Option 1: Ephemeral debug container (Kubernetes 1.25+, no RBAC exec needed)**
```bash
# Attach a debug container to a running pod without exec
kubectl debug -it <pod-name> -n prod \
  --image=busybox:latest \
  --target=pharma-service   # share PID/net/etc. namespaces with main container

# This creates a temporary container in the pod's namespace
# You can inspect processes, network, filesystem from here
```

**Option 2: Debug copy of the pod**
```bash
# Create a copy of the pod with a debug shell, without replacing it
kubectl debug <pod-name> -n prod \
  --copy-to=debug-auth-service \
  --image=eclipse-temurin:21-jre \
  -it -- bash
# Delete when done
kubectl delete pod debug-auth-service -n prod
```

---

### Debug network without installing tools (busybox is pre-installed)

```bash
# From inside a running pod (if exec allowed) or ephemeral container:

# 1. Check DNS resolution
cat /etc/resolv.conf
nslookup pharma-prod-postgres.cs3c424yurej.us-east-1.rds.amazonaws.com

# 2. Check if a port is reachable (nc / wget in busybox)
nc -zv pharma-prod-postgres.cs3c424yurej.us-east-1.rds.amazonaws.com 5432
# Connected → DB port open
# refused → network policy or security group blocking

# 3. Check HTTP connectivity (wget in busybox, curl if available)
wget -q -O- http://catalog-service:8080/actuator/health

# 4. Check env vars (no tools needed)
env | grep DB_

# 5. Check filesystem (read-only root issue)
ls -la /tmp
touch /tmp/test && echo "tmp is writable" || echo "tmp is read-only"

# 6. From Java process perspective (if JDK tools available)
jcmd 1 VM.system_properties | grep -i "db\|datasource"
```

---


## I5

### Question
> "How do you check what resources a namespace is consuming right now — CPU, memory, pods — and compare that to its limits? Give exact commands."

---

### Commands

```bash
# 1. Live resource usage by pod in the namespace (requires metrics-server)
kubectl top pods -n prod
# NAME                               CPU(cores)   MEMORY(bytes)
# auth-service-7d9f8c-xk2pv         12m          221Mi
# drug-catalog-service-9b3f4d-lm8k   8m           198Mi
# manufacturing-service-2c6a1e-pq5r  45m          389Mi

# 2. Live resource usage summed by node (for capacity planning)
kubectl top nodes
# NAME                   CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
# ip-10-0-1-45.ec2       312m         15%    1842Mi          73%

# 3. Resource requests and limits defined in pod specs (what is reserved)
kubectl get pods -n prod \
  -o custom-columns="NAME:.metadata.name,CPU_REQ:.spec.containers[*].resources.requests.cpu,MEM_REQ:.spec.containers[*].resources.requests.memory,CPU_LIM:.spec.containers[*].resources.limits.cpu,MEM_LIM:.spec.containers[*].resources.limits.memory"
# NAME                               CPU_REQ   MEM_REQ   CPU_LIM   MEM_LIM
# auth-service-7d9f8c-xk2pv         100m      256Mi     500m      512Mi
# drug-catalog-service-9b3f4d-lm8k   100m      256Mi     500m      512Mi

# 4. Check if a ResourceQuota exists and how much is used
kubectl get resourcequota -n prod
kubectl describe resourcequota -n prod
# Name:       prod-quota
# Namespace:  prod
# Resource              Used   Hard
# --------              ----   ----
# pods                  6      20
# requests.cpu          600m   4
# requests.memory       1536Mi 8Gi
# limits.cpu            3      8
# limits.memory         3072Mi 16Gi

# 5. Check LimitRange (default requests/limits for pods without explicit values)
kubectl get limitrange -n prod
kubectl describe limitrange -n prod
# Useful if pods are created without requests — LimitRange sets defaults

# 6. Nodes: total allocatable resources vs total requested
kubectl describe nodes | grep -A10 "Allocated resources:"
# Allocated resources:
#   (Total limits may be over 100 percent, i.e., overcommitted.)
#   Resource           Requests    Limits
#   cpu                1250m (62%) 3500m (175%)   ← overcommitted on CPU limits
#   memory             3584Mi (71%) 7168Mi (143%) ← overcommitted on memory limits

# 7. See how many pods are running per node
kubectl get pods -n prod -o wide | awk '{print $7}' | sort | uniq -c | sort -rn
# 3 ip-10-0-1-45.ec2   ← 3 pods on node 1
# 2 ip-10-0-2-67.ec2   ← 2 pods on node 2
# 1 ip-10-0-3-89.ec2   ← 1 pod on node 3
```

---

### What to look for

| Signal | Meaning |
|--------|---------|
| `kubectl top pods` memory near limit | Pod is close to OOMKill — investigate or raise limit |
| `Allocated resources` CPU requests > 80% | Cluster is near full — add a node before the next spike |
| ResourceQuota `Used` near `Hard` | Next pod create will be rejected by quota |
| CPU limits > 100% (overcommitted) | OK for CPU (throttling, not kill) — but watch for latency |
| Memory limits > 100% (overcommitted) | Dangerous — OOMKill can cascade across pods if all spike together |

---


## I6

### Question
> "A Helm upgrade failed halfway through and left the release in a broken state. Walk me through diagnosing and recovering it."

---

### Diagnosis

```bash
# 1. Check release status
helm list -n prod
# NAME            NAMESPACE  REVISION  STATUS          CHART
# auth-service    prod       5         failed          pharma-service-1.0.0

helm status auth-service -n prod
# STATUS: failed
# LAST DEPLOYED: Thu May 07 02:15:00 2026
# NOTES: ...

# 2. See the full history
helm history auth-service -n prod
# REVISION  UPDATED     STATUS      CHART                    DESCRIPTION
# 1         ...         superseded  pharma-service-1.0.0    Install complete
# 2         ...         superseded  pharma-service-1.0.0    Upgrade complete
# 3         ...         superseded  pharma-service-1.0.0    Upgrade complete
# 4         ...         superseded  pharma-service-1.0.0    Upgrade complete
# 5         ...         failed      pharma-service-1.0.0    Upgrade failed: ...

# 3. See what changed in the failed revision
helm get manifest auth-service -n prod --revision 5 > /tmp/failed.yaml
helm get manifest auth-service -n prod --revision 4 > /tmp/previous.yaml
diff /tmp/previous.yaml /tmp/failed.yaml

# 4. Check the actual K8s resources for inconsistency
kubectl get deployments,services,configmaps -n prod -l app.kubernetes.io/instance=auth-service
```

---

### Recovery options

**Option A: Rollback to last good revision (most common)**
```bash
helm rollback auth-service 4 -n prod
# Rolls back to revision 4

# Verify
helm status auth-service -n prod
# STATUS: deployed
```

**Option B: Force upgrade with --cleanup-on-fail (if rollback not working)**
```bash
helm upgrade auth-service ./helm-charts \
  -n prod \
  -f envs/prod/values-auth-service.yaml \
  --cleanup-on-fail \    # delete new resources if upgrade fails
  --atomic               # rollback automatically if upgrade fails
```

**Option C: Nuclear option — uninstall and reinstall**
```bash
# ONLY if rollback is not working
# WARNING: brief downtime

helm uninstall auth-service -n prod
# Wait for resources to clean up
kubectl get pods -n prod | grep auth-service

helm install auth-service ./helm-charts \
  -n prod \
  -f envs/prod/values-auth-service.yaml
```

---

### Why do Helm upgrades fail halfway?

Common causes in this repo's setup:
1. **Job hook failed** — if a pre-upgrade hook (e.g., DB migration) fails, upgrade is marked failed but some resources were already updated
2. **Readiness timeout** — new pod never becomes ready within `--timeout` (default 5min); Helm marks as failed but deployment exists
3. **Immutable field change** — tried to change a field like `selector.matchLabels` which can't be updated; the Deployment object is stuck
4. **Resource conflict** — a resource already exists that Helm didn't create (not in its release history)

For the immutable field case:
```bash
kubectl delete deployment auth-service -n prod
helm upgrade auth-service ./helm-charts -n prod -f envs/prod/values-auth-service.yaml
```

---

## I7

### Question
> "How do you verify that the correct container image SHA is running in prod and it matches what ArgoCD deployed? Give exact commands."

---

### Commands

```bash
# 1. What ArgoCD thinks should be deployed
argocd app get auth-service-prod
# Look for: Summary → Images: 516209541629.dkr.ecr.us-east-1.amazonaws.com/auth-service:v1.0.0

# 2. What the Deployment spec says (desired state)
kubectl get deployment auth-service -n prod \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
# 516209541629.dkr.ecr.us-east-1.amazonaws.com/auth-service:v1.0.0

# 3. What is ACTUALLY running in the pod (imageID contains the SHA256 digest)
kubectl get pods -n prod -l app.kubernetes.io/name=auth-service \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.containerStatuses[0].imageID}{"\n"}{end}'
# auth-service-7d9f8c-xk2pv   docker-pullable://...auth-service@sha256:abc123def456...

# 4. Cross-check the SHA against ECR (what was actually pushed)
aws ecr describe-images \
  --repository-name auth-service \
  --region us-east-1 \
  --query 'imageDetails[?contains(imageTags,`v1.0.0`)].[imageTags,imageDigest]' \
  --output table
# Expected: sha256:abc123def456... (should match imageID from step 3)

# 5. Compare Git tag to image digest (full chain verification)
# What commit is tagged v1.0.0?
git log --oneline -1 v1.0.0   # if tag exists in app repo

# What SHA is in the prod values file?
cat envs/prod/values-auth-service.yaml | grep tag
# tag: v1.0.0

# 6. One-liner to print all running images in prod
kubectl get pods -n prod \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{range .status.containerStatuses[*]}  {.name}: {.imageID}{"\n"}{end}{end}'
```

---

### Why imageID matters more than image tag

Docker tags are **mutable**. Someone can push a new image to ECR with the same tag `v1.0.0`, overwriting the previous one. The `imageID` contains the immutable SHA256 content hash of the image — it changes if even one byte of the image changes. Verifying `imageID` is the only way to be certain what code is running.

In a pharma GxP environment, the `imageID` SHA is what you record in your audit trail — not the tag.

---

## 5. Scenario Troubleshooting

---

**Q24.** After a `terraform apply` succeeds, a developer runs `kubectl apply -f deployment.yaml` and the pod fails to start with `ImagePullBackOff`. Walk through your debugging steps.

<details>
<summary>Expected answer</summary>

`ImagePullBackOff` means the node cannot pull the Docker image from ECR.

Step-by-step:

```bash
kubectl describe pod <pod-name> -n <namespace>
# Look at Events section — it will say something like:
# Failed to pull image "123456789012.dkr.ecr.us-east-1.amazonaws.com/auth-service:latest"
# unauthorized: authentication required
```

Possible causes and checks:

1. **Wrong ECR URL** — confirm the image URI in the deployment matches the ECR repository URL from `module.ecr.repository_urls["auth-service"]`

2. **Node group IAM role missing ECR policy** — the `AmazonEC2ContainerRegistryReadOnly` policy must be attached to the node group role. Check in AWS Console → IAM → Roles → `zen-pharma-dev-eks-node-group-role` → Permissions tab

3. **Image does not exist in ECR** — the CI pipeline may not have pushed the image yet. Check ECR → repository → Images tab. If the tag `latest` does not exist, the CI job failed or was never run

4. **Region mismatch** — the image URI must use the same region as the cluster (`us-east-1`)

5. **`imagePullSecrets` missing** — for private ECR, nodes use their IAM role to authenticate. No `imagePullSecrets` are needed if the node role has the ECR policy. If someone added an incorrect `imagePullSecrets` it may be interfering

</details>

---

**Q25.** You are paged at 2 AM: the pharma application is returning 503 errors. `kubectl get pods` shows all pods `Running`. `kubectl get nodes` shows all nodes `Ready`. Where do you look next?

<details>
<summary>Expected answer</summary>

All pods and nodes are healthy, so the issue is likely at the networking or routing layer.

Check in this order:

```bash
# 1. Check the Ingress
kubectl get ingress -A
kubectl describe ingress <ingress-name>
# Look for: Address (should be the NLB DNS name), and any events

# 2. Check the NLB in AWS Console
# EC2 → Load Balancers → find the NLB created by NGINX Ingress
# Target Group → check target health — are the targets healthy?

# 3. Check NGINX Ingress Controller pods
kubectl get pods -n ingress-nginx
kubectl logs -n ingress-nginx <ingress-nginx-controller-pod>

# 4. Check the API Gateway service
kubectl get svc -A
kubectl describe svc api-gateway
# Is the ClusterIP resolving? Are the endpoint slices populated?

# 5. Check RDS connectivity from a pod
kubectl run debug --image=postgres:15 --restart=Never -- sleep 300
kubectl exec -it debug -- pg_isready -h <rds_endpoint> -p 5432
# If this fails, the RDS security group or subnet routing may have changed
```

The most common 2 AM cause after "everything looks fine": the NLB target group health checks are failing (even though pods are Running), or a recent deployment changed a service selector that no longer matches the pod labels.

</details>

---

*Sources: zen-infra/Interview.md · zen-infra/docs/INTERVIEW-PREP.md · zen-gitops/docs/interview/interview-questions-part2.md*
*Stack: AWS EKS 1.33 · IRSA · External Secrets Operator · ArgoCD · NGINX Ingress · Helm*
