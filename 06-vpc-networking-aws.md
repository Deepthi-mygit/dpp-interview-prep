# VPC, Networking & AWS Infrastructure — Interview Questions

> Grounded in the zen-infra VPC module: 3-tier subnet design, single NAT Gateway, EKS subnets with K8s tags, RDS in private subnets.
> 18 questions covering networking fundamentals, troubleshooting, and AWS-specific gotchas.

---

## Table of Contents

1. [VPC & Networking Scenarios](#1-vpc--networking-scenarios) — Q1–Q3
2. [Access & Connectivity Troubleshooting](#2-access--connectivity-troubleshooting) — Q4–Q5
3. [AWS Networking Concepts](#3-aws-networking-concepts) — Q6–Q8
4. [AWS Networking Concepts — Advanced](#4-aws-networking-concepts--advanced) — Q9–Q13
5. [AWS Infrastructure Concepts (Beyond Networking)](#5-aws-infrastructure-concepts-beyond-networking) — Q14–Q18

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

**Q4.** After a `terraform apply` succeeds, a developer runs `kubectl apply -f deployment.yaml` and the pod fails to start with `ImagePullBackOff`. Walk through your debugging steps.

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

**Q5.** You are paged at 2 AM: the pharma application is returning 503 errors. `kubectl get pods` shows all pods `Running`. `kubectl get nodes` shows all nodes `Ready`. Where do you look next?

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

**Q6.** What is the difference between a Security Group and a Network ACL in the zen-infra VPC?

<details>
<summary>Expected answer</summary>

**Security Groups** are stateful, instance-level firewalls. "Stateful" means if you allow inbound on port 5432, the return traffic is automatically allowed — you don't need an outbound rule for it. In zen-infra, the RDS security group allows inbound 5432 from the EKS security group. That's the only rule needed.

**NACLs** are stateless, subnet-level firewalls. You must explicitly allow both inbound AND outbound for every flow. NACLs are evaluated before traffic reaches the instance — if a NACL denies traffic, the security group never sees it.

In zen-infra, NACLs are left at AWS defaults (allow all). All security is enforced via security groups. This is the standard pattern for most VPCs — NACLs add complexity without benefit unless you have regulatory requirements to block specific CIDR ranges at the subnet level.

Key interview point: security groups are the primary tool; NACLs are a secondary layer for defence-in-depth or compliance.

</details>

---

**Q7.** Why does zen-infra use a single NAT Gateway in dev? What would you change for production?

<details>
<summary>Expected answer</summary>

A NAT Gateway is required for private subnet resources (EKS nodes, RDS) to reach the internet — for pulling Docker images from ECR, downloading OS updates, and calling AWS APIs. Every AZ needs a route to the internet, but a NAT Gateway costs ~$32/month plus data transfer.

In dev, a **single NAT Gateway** in `us-east-1a` serves all private subnets across all AZs using a shared route table. This halves infrastructure cost at the expense of AZ resilience — if the NAT Gateway in 1a fails, all private subnets lose internet access simultaneously.

For production, you create **one NAT Gateway per AZ** (3 NAT Gateways for 3 AZs), each with its own private route table. Losing the NAT in 1a only affects subnets in 1a — 1b and 1c continue working.

In zen-infra: `modules/vpc/main.tf` creates `aws_nat_gateway.main` with `count = 1` for dev. A production-ready module would use `count = length(var.availability_zones)`.

</details>

---

**Q8.** How does the EKS VPC CNI plugin assign IPs to pods, and what happens if a subnet runs out of IPs?

<details>
<summary>Expected answer</summary>

The AWS VPC CNI assigns each pod a **real VPC IP** from the node's subnet — not an overlay network address. The node pre-allocates a "warm pool" of IPs from the subnet. When a pod starts, it gets one of these pre-allocated IPs. This is why pod-to-pod and pod-to-RDS traffic works without NAT — every pod has a routable VPC IP.

The consequence: a /24 subnet (256 IPs) minus 5 reserved by AWS = 251 usable IPs. A t3.small node can hold ~4 pods. With 3 nodes you need ~12 pod IPs plus 3 for the node ENIs. That sounds fine — until you add more nodes or increase pod density.

If a subnet runs out of IPs: new pods fail to start with `Failed to allocate address` in the kubelet logs. The fix is larger subnets (/22 gives 1019 IPs) or **prefix delegation** (VPC CNI v1.9+) which assigns /28 blocks instead of individual IPs, multiplying capacity 16x without subnet changes.

In zen-infra, the private EKS subnets are /24 — fine for dev scale but undersized for production. A production sizing recommendation is /22 per AZ.

</details>

---

## 4. AWS Networking Concepts — Advanced

**Q9.** What is a VPC Gateway Endpoint vs an Interface Endpoint (PrivateLink), and where would you use each?

<details>
<summary>Expected answer</summary>

**Gateway Endpoint** — supports only **S3** and **DynamoDB**. It works by adding a route in the route table that sends traffic for the service's public IP ranges to the endpoint instead of the NAT Gateway. It's free and adds no extra network hop cost.

**Interface Endpoint (AWS PrivateLink)** — creates an **ENI with a private IP** in your subnet for the target service (e.g. `ecr.api`, `ecr.dkr`, `secretsmanager`, `sts`, `logs`). Traffic goes over AWS's private backbone, never touching the internet or the NAT Gateway. Costs an hourly charge per AZ plus per-GB data processing.

Why this matters in zen-infra: EKS nodes in private subnets pull images from ECR and call Secrets Manager/STS through the **single NAT Gateway** today. Under load, or if the NAT Gateway is down, all of that traffic breaks. Adding interface endpoints for `ecr.api`, `ecr.dkr`, `s3` (as a gateway endpoint), `secretsmanager`, and `sts` removes that dependency entirely — pods keep working even if the NAT Gateway is unavailable, and you cut NAT data-processing costs for what is usually the bulk of private-subnet egress traffic.

</details>

---

**Q10.** What is the difference between VPC Peering and a Transit Gateway, and when would zen-infra need to move from one to the other?

<details>
<summary>Expected answer</summary>

**VPC Peering** connects exactly two VPCs directly. It is not transitive — if VPC A peers with B, and B peers with C, A cannot reach C through B. Each peering connection needs its own route table entries in both VPCs, and CIDR ranges must not overlap.

**Transit Gateway (TGW)** is a regional hub-and-spoke router. Every attached VPC (and VPN/Direct Connect connection) gets one attachment to the TGW, and TGW route tables control which spokes can reach which. It scales to hundreds of VPCs without the N² peering-connection problem.

zen-infra today is a single VPC, so neither is in play yet. The trigger to introduce one: a second VPC appears — e.g. a shared-services VPC for CI runners, or a separate VPC per environment (dev/staging/prod) that still needs to reach a central logging or artifact-registry VPC. With 2–3 VPCs, peering is simplest. Once you're adding a third or fourth VPC, or need transitive routing (dev → shared-services → on-prem via Direct Connect), switch to TGW — retrofitting peering into TGW later means re-doing every route table.

</details>

---

**Q11.** How does DNS resolution work inside a VPC, and what do `enableDnsSupport` and `enableDnsHostnames` actually control?

<details>
<summary>Expected answer</summary>

Every VPC has an **Amazon-provided DNS resolver** reachable at the base of the VPC CIDR + 2 (e.g. `10.0.0.2` for a `10.0.0.0/16` VPC). It resolves both public DNS names and internal VPC/Route 53 private hosted zone names.

- **`enableDnsSupport`** — controls whether that resolver is even reachable from instances in the VPC. If disabled, nothing can resolve DNS at all inside the VPC.
- **`enableDnsHostnames`** — controls whether EC2 instances (and other resources) get **DNS hostnames** assigned alongside their IPs (e.g. `ip-10-0-1-23.ec2.internal`), and whether public IPs get a resolvable public DNS name.

Both must be `true` for EKS: the cluster API server endpoint, RDS endpoint hostnames, and pod-to-service DNS (via CoreDNS, which itself forwards upstream to the VPC resolver) all depend on this. If a developer reports "the RDS endpoint hostname won't resolve from my pod," and everything else looks fine, checking these two VPC attributes is a five-second check before going deeper.

For cross-VPC or on-prem DNS resolution, **Route 53 Resolver** (inbound/outbound endpoints) extends this — e.g. resolving on-prem AD DNS names from within the VPC, or letting on-prem resolve private hosted zone records.

</details>

---

**Q12.** Two teams want to peer their VPCs and discover both were created with the default `10.0.0.0/16` CIDR. Why is this a problem, and how do you prevent it going forward?

<details>
<summary>Expected answer</summary>

**VPC peering (and Transit Gateway attachments) require non-overlapping CIDR ranges.** If both VPCs use `10.0.0.0/16`, AWS will simply refuse to create the peering connection — routing is ambiguous because a destination IP could exist in either VPC.

This is a very common real-world mistake because `10.0.0.0/16` is the default suggested range in the AWS console and in most Terraform module examples (including a naive copy of zen-infra's `modules/vpc` for a second environment).

Fix once it's already happened: there is no clean fix short of **re-IP'ing one VPC** — recreating it with a different CIDR and migrating resources, which is disruptive for anything with hardcoded IPs (RDS, on-prem firewall rules).

Prevention: maintain a **CIDR allocation plan** before creating any new VPC — e.g. dev `10.0.0.0/16`, staging `10.1.0.0/16`, prod `10.2.0.0/16`, shared-services `10.3.0.0/16`. In zen-infra's `modules/vpc`, the VPC CIDR is a `var.vpc_cidr` input specifically so each environment's `terraform.tfvars` can assign a distinct block instead of everyone defaulting to the same value.

</details>

---

**Q13.** What is the difference between an Internet Gateway (IGW) and a NAT Gateway, and why can't a private subnet just attach an IGW directly?

<details>
<summary>Expected answer</summary>

**Internet Gateway (IGW)** — a horizontally scaled, VPC-wide component that does **1:1 NAT** between a resource's private IP and its assigned public/Elastic IP. It allows **both inbound and outbound** internet traffic. Anything with a public IP and a route to the IGW is directly reachable from the internet — this is what makes a subnet "public."

**NAT Gateway** — sits in a public subnet, itself has an Elastic IP, and performs **source NAT (SNAT)** for outbound-only traffic from private subnets. It is stateful: it tracks connections it initiated, so unsolicited inbound traffic from the internet is dropped. This is what makes a subnet "private but internet-capable."

Why a private subnet can't just route to the IGW directly: the IGW would make every instance in that subnet directly reachable inbound from the internet the moment it has a public IP — that's the definition of a public subnet, not a security boundary you can opt out of selectively. There's no "outbound only" mode on an IGW; that behavior is exactly what the NAT Gateway exists to provide.

In zen-infra's `modules/vpc`: `public` subnets route `0.0.0.0/0` → IGW (bidirectional), while `private_eks` and `private_rds` subnets route `0.0.0.0/0` → NAT Gateway (outbound-only). RDS subnets don't strictly need this route at all — the deeper point is that removing the NAT route from `private_rds` entirely, since RDS never initiates outbound internet calls, is a legitimate hardening step.

</details>

---

## 5. AWS Infrastructure Concepts (Beyond Networking)

**Q14.** What is the difference between an IAM User, an IAM Role, and an IAM Policy — and why does zen-infra avoid IAM users almost entirely?

<details>
<summary>Expected answer</summary>

**IAM Policy** — a JSON document that defines permissions: which actions on which resources, allow or deny. A policy by itself does nothing until it's attached to an identity.

**IAM User** — a long-lived identity, typically for a human or a legacy application, with permanent credentials (password and/or access keys). Policies attach directly to a user or via a group.

**IAM Role** — an identity with **no permanent credentials**. Something (a person, an EC2 instance, an EKS pod via IRSA, a GitHub Actions run via OIDC) **assumes** the role and gets temporary credentials via STS that expire (typically 1 hour, or ~15 minutes for GitHub OIDC). The same policy document shape applies to both users and roles — the difference is entirely in how credentials are obtained.

zen-infra uses roles almost exclusively: the EKS node group has a role, IRSA gives pods roles, GitHub Actions assumes a role via OIDC for `terraform apply` and ECR push. The only reason to use an IAM user is a break-glass emergency account with MFA enforced, kept out of normal workflows. The interview point: **long-lived credentials are a liability** — every IAM user with an access key is a secret that can leak, needs manual rotation, and doesn't expire on its own. Roles remove that liability by design.

</details>

---

**Q15.** What is the difference between an Application Load Balancer (ALB) and a Network Load Balancer (NLB), and which one is fronting the EKS ingress in zen-infra?

<details>
<summary>Expected answer</summary>

**ALB** — operates at **Layer 7 (HTTP/HTTPS)**. It can route based on host header, path, and query string, terminate TLS, and inspect request content. Best fit for HTTP microservices where you want path-based routing (`/api/*` → one target group, `/static/*` → another) without running your own ingress logic.

**NLB** — operates at **Layer 4 (TCP/UDP)**. It just forwards packets to targets with extremely low latency and can handle millions of requests per second, but has no visibility into HTTP semantics. It also supports a **static IP per AZ**, which ALB does not.

In zen-infra, the **NGINX Ingress Controller** in EKS is fronted by an **NLB**, not an ALB. This is the standard pattern for Kubernetes: NGINX Ingress already does the Layer 7 routing (host/path rules, TLS termination) inside the cluster, so the load balancer in front of it only needs to forward TCP traffic to the ingress controller's pods — an NLB is cheaper and faster for that job. Using an ALB here would mean paying for Layer 7 features (the AWS Load Balancer Controller's ALB Ingress mode) that NGINX Ingress already provides, effectively duplicating the routing layer.

</details>

---

**Q16.** What is the difference between RDS Multi-AZ and RDS Read Replicas — do they solve the same problem?

<details>
<summary>Expected answer</summary>

No — they solve two different problems, and it's a common interview trap to conflate them.

**Multi-AZ** — a **synchronous standby** in a different AZ, invisible as a separate endpoint. Every write to the primary is synchronously replicated to the standby before the write is acknowledged. Purpose is **availability**: if the primary fails or during patching, RDS automatically fails over to the standby (~60–120 seconds), and the DNS endpoint just starts pointing at it. You cannot query the standby directly, and it does not help with read scaling — write latency is even slightly higher due to sync replication.

**Read Replica** — an **asynchronous copy**, exposed as its own separate endpoint that you can query directly. Purpose is **read scaling and workload isolation**: point reporting queries or a read-heavy service at the replica so they don't compete with the primary for resources. Because replication is async, replicas can lag behind the primary (replication lag), so they're not suitable for reads that must be immediately consistent. A read replica is not automatically promoted on primary failure — that requires a manual (or scripted) promotion.

You can have both at once: a Multi-AZ primary (for HA) with one or more read replicas off of it (for read scaling). In zen-infra, the current RDS instance is Multi-AZ for HA (per the DR strategy in `04-gitops-argocd.md`) but has no read replicas — there's no read-heavy reporting workload yet to justify one.

</details>

---

**Q17.** How do you stop an ECR repository from growing unbounded, and how do you catch vulnerable base images before they reach EKS?

<details>
<summary>Expected answer</summary>

Two separate ECR features address this:

**Lifecycle policies** — a JSON rule set on the repository that automatically expires old images, e.g. "keep only the last 10 tagged images" or "expire untagged images older than 14 days." Without this, every CI build pushes a new image and the repository grows forever, increasing storage cost and making it harder to find the image you actually want. Untagged images (left behind after a tag is overwritten, e.g. repeated pushes to `latest`) are the usual biggest source of waste.

**Image scanning** — ECR can scan on push (`scanOnPush`) using either basic scanning (CVEs from the Clair database) or enhanced scanning (continuous rescanning via Amazon Inspector, which also catches newly disclosed CVEs in images that haven't changed). Findings appear in the ECR console/API and can gate a pipeline: fail the deploy step if any `CRITICAL` or `HIGH` severity finding is found.

In zen-infra's DevSecOps pipeline (`05-security-devsecops.md`), Trivy already scans images in CI before push — ECR's own scan-on-push is a second, always-on layer that catches CVEs disclosed *after* the image was already pushed, which a one-time CI scan would miss.

</details>

---

**Q18.** What is the difference between EBS, EFS, and S3, and where would each fit into zen-infra's stack?

<details>
<summary>Expected answer</summary>

**EBS (Elastic Block Store)** — block storage attached to a **single EC2 instance** (or a single EKS node) at a time. This is what backs an EKS worker node's root volume and any `PersistentVolume` using the `ebs.csi.aws.com` driver — e.g. a StatefulSet needing its own dedicated disk. It cannot be mounted by multiple nodes simultaneously (with the newer Multi-Attach exception, which is niche and io1/io2-only).

**EFS (Elastic File System)** — a managed **NFS** file system that **multiple EC2 instances or pods across AZs can mount concurrently**. Fits a workload where several pod replicas need to read/write the *same* files — e.g. a shared upload directory, or ReadWriteMany PVCs in Kubernetes. Slower per-operation latency than EBS, but that's the tradeoff for concurrent multi-writer access.

**S3** — object storage, not a filesystem at all — accessed via HTTP API (`PUT`/`GET`/`DELETE` on keys), not mounted as a block or file device. Used in zen-infra for the **Terraform state backend** (with versioning and native locking) and would be the right place for things like build artifacts, ALB/NLB access logs, or backups — anything accessed by key rather than needing POSIX file semantics.

The zen-pharma stack today is mostly stateless (EKS pods) plus RDS (which manages its own storage internally, invisible as EBS/EFS to the operator) — the interview point is knowing *why* you'd reach for EBS vs EFS vs S3 rather than needing to have all three wired up already.

</details>
