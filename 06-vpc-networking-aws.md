# VPC, Networking & AWS Infrastructure — Interview Questions

> Grounded in the zen-infra VPC module: 3-tier subnet design, single NAT Gateway, EKS subnets with K8s tags, RDS in private subnets.
> 10 questions covering networking fundamentals, troubleshooting, and AWS-specific gotchas.

---

## Table of Contents

1. [VPC & Networking Scenarios](#1-vpc--networking-scenarios) — Q1–Q3
2. [Access & Connectivity Troubleshooting](#2-access--connectivity-troubleshooting) — Q4–Q6
3. [AWS Networking Concepts](#3-aws-networking-concepts) — Q7–Q10

---

## 1. VPC & Networking Scenarios

**Q1.** A developer says "I can't connect to the RDS database from my laptop even though I'm on the company VPN." Is this expected? How would you give them access temporarily for debugging?

<details>
<summary>Expected answer</summary>

Yes, this is **expected and correct behaviour**. The RDS security group only allows inbound on port 5432 from `eks_security_group_id`. A VPN puts the developer on the company network, not inside the EKS security group — so the rule never matches.

Options for temporary access:

**Option 1 — Port-forward via kubectl (recommended, no infra change):**
```bash
kubectl run pg-debug --image=postgres:15 --restart=Never -- sleep 3600
kubectl exec -it pg-debug -- psql \
  -h <rds_endpoint> -U pharmaadmin -d pharmadb
```
This works because the pod is running inside EKS and its network interface is in the EKS security group.

**Option 2 — Add a temporary SG rule (risky, needs cleanup):**
Add an ingress rule for the developer's VPN IP to the RDS security group. Remove it immediately after. In Terraform this is a change you commit, open PR, merge, and then immediately revert — to leave an audit trail.

**Never** set `publicly_accessible = true` on the RDS instance to solve this.

</details>

---

**Q2.** The EKS worker nodes in `us-east-1a` have stopped being able to pull Docker images from ECR. Nodes in `us-east-1b` are fine. What is the most likely cause?

<details>
<summary>Expected answer</summary>

The most likely cause is that the **NAT Gateway is down or has lost its Elastic IP**. In the current `modules/vpc` setup there is a **single NAT Gateway** in `public[0]` (us-east-1a). Both private EKS subnets in 1a and 1b route through this single NAT via the shared private route table.

Wait — if 1b nodes are fine but 1a nodes are not, it would suggest a subnet-level issue rather than the NAT (since both use the same NAT). In that case check:
- The route table association for `private_eks` subnet in 1a — it may have been accidentally disassociated
- The subnet's NACL (network ACL) — if someone added a deny rule
- The EKS security group — if a rule was modified

The deeper interview point: **for production HA** you would create one NAT Gateway per AZ, each with its own private route table pointing to the local NAT. Then losing the NAT in 1a would not affect 1b at all. The current `modules/vpc` code uses a single shared NAT to save ~$32/month, which is correct for dev but not for prod.

</details>

---

**Q3.** What are the Kubernetes tags on the private EKS subnets and why are they critical?

<details>
<summary>Expected answer</summary>

Two tags are set on `aws_subnet.private_eks`:

```hcl
"kubernetes.io/role/internal-elb" = "1"
"kubernetes.io/cluster/${var.project}-${var.env}-cluster" = "owned"
```

Without `kubernetes.io/role/internal-elb = 1`, the **AWS Load Balancer Controller** cannot discover which subnets to place internal load balancers in. Any Kubernetes `Service` of type `LoadBalancer` or `Ingress` will stay in `Pending` state indefinitely — it looks like a networking bug but it is actually a missing tag.

Without `kubernetes.io/cluster/…cluster = owned`, EKS cannot tag and manage the subnet for node placement. The cluster may fail to add new nodes or the VPC CNI plugin may have trouble allocating pod IPs.

These tags cost nothing and forgetting them is a very common real-world mistake.

</details>

---

## 2. Access & Connectivity Troubleshooting

**Q4.** You just ran `terraform apply` and the EKS cluster was created, but `kubectl get nodes` shows 0 nodes after 10 minutes. What do you check?

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

**Q5.** After a `terraform apply` succeeds, a developer runs `kubectl apply -f deployment.yaml` and the pod fails to start with `ImagePullBackOff`. Walk through your debugging steps.

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

**Q6.** You are paged at 2 AM: the pharma application is returning 503 errors. `kubectl get pods` shows all pods `Running`. `kubectl get nodes` shows all nodes `Ready`. Where do you look next?

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

## 3. AWS Networking Concepts

**Q7.** What is the difference between a Security Group and a Network ACL in the zen-infra VPC?

<details>
<summary>Expected answer</summary>

**Security Groups** are stateful, instance-level firewalls. "Stateful" means if you allow inbound on port 5432, the return traffic is automatically allowed — you don't need an outbound rule for it. In zen-infra, the RDS security group allows inbound 5432 from the EKS security group. That's the only rule needed.

**NACLs** are stateless, subnet-level firewalls. You must explicitly allow both inbound AND outbound for every flow. NACLs are evaluated before traffic reaches the instance — if a NACL denies traffic, the security group never sees it.

In zen-infra, NACLs are left at AWS defaults (allow all). All security is enforced via security groups. This is the standard pattern for most VPCs — NACLs add complexity without benefit unless you have regulatory requirements to block specific CIDR ranges at the subnet level.

Key interview point: security groups are the primary tool; NACLs are a secondary layer for defence-in-depth or compliance.

</details>

---

**Q8.** Why does zen-infra use a single NAT Gateway in dev? What would you change for production?

<details>
<summary>Expected answer</summary>

A NAT Gateway is required for private subnet resources (EKS nodes, RDS) to reach the internet — for pulling Docker images from ECR, downloading OS updates, and calling AWS APIs. Every AZ needs a route to the internet, but a NAT Gateway costs ~$32/month plus data transfer.

In dev, a **single NAT Gateway** in `us-east-1a` serves all private subnets across all AZs using a shared route table. This halves infrastructure cost at the expense of AZ resilience — if the NAT Gateway in 1a fails, all private subnets lose internet access simultaneously.

For production, you create **one NAT Gateway per AZ** (3 NAT Gateways for 3 AZs), each with its own private route table. Losing the NAT in 1a only affects subnets in 1a — 1b and 1c continue working.

In zen-infra: `modules/vpc/main.tf` creates `aws_nat_gateway.main` with `count = 1` for dev. A production-ready module would use `count = length(var.availability_zones)`.

</details>

---

**Q9.** What are the two required Kubernetes tags on the private EKS subnets and what breaks without each one?

<details>
<summary>Expected answer</summary>

```hcl
"kubernetes.io/role/internal-elb" = "1"
"kubernetes.io/cluster/${var.project}-${var.env}-cluster" = "owned"
```

Without `kubernetes.io/role/internal-elb = 1`: The AWS Load Balancer Controller cannot discover which subnets to use for internal load balancers. Any Kubernetes Service of type LoadBalancer or Ingress stays in Pending state indefinitely — the pod and service are healthy but no LB is provisioned. This is the most common silent failure on new EKS clusters.

Without `kubernetes.io/cluster/...-cluster = owned`: EKS cannot manage subnet resources for node placement, and the VPC CNI plugin may struggle to allocate pod IPs.

These tags cost nothing. Forgetting them is the most common networking mistake on new EKS clusters.

</details>

---

**Q10.** How does the EKS VPC CNI plugin assign IPs to pods, and what happens if a subnet runs out of IPs?

<details>
<summary>Expected answer</summary>

The AWS VPC CNI assigns each pod a **real VPC IP** from the node's subnet — not an overlay network address. The node pre-allocates a "warm pool" of IPs from the subnet. When a pod starts, it gets one of these pre-allocated IPs. This is why pod-to-pod and pod-to-RDS traffic works without NAT — every pod has a routable VPC IP.

The consequence: a /24 subnet (256 IPs) minus 5 reserved by AWS = 251 usable IPs. A t3.small node can hold ~4 pods. With 3 nodes you need ~12 pod IPs plus 3 for the node ENIs. That sounds fine — until you add more nodes or increase pod density.

If a subnet runs out of IPs: new pods fail to start with `Failed to allocate address` in the kubelet logs. The fix is larger subnets (/22 gives 1019 IPs) or **prefix delegation** (VPC CNI v1.9+) which assigns /28 blocks instead of individual IPs, multiplying capacity 16x without subnet changes.

In zen-infra, the private EKS subnets are /24 — fine for dev scale but undersized for production. A production sizing recommendation is /22 per AZ.

</details>
