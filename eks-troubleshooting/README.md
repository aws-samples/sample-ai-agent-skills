# EKS Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon EKS problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for EKS clusters when the AWS Console alone isn't enough — cluster lifecycle, node health, pod scheduling, networking (VPC CNI, CoreDNS, ALB), IAM/RBAC, storage (EBS/EFS CSI), autoscaling (Cluster Autoscaler, Karpenter, HPA), add-on management, and upgrade planning.

### Activate When

- Cluster creation or deletion fails
- Cluster API server is unreachable (endpoint access issues)
- Cluster upgrade fails or gets stuck
- Nodes are NotReady or failing to join the cluster
- Managed node group creation/update fails
- Fargate pods are not scheduling
- Pods are stuck in Pending, CrashLoopBackOff, or OOMKilled
- Image pull errors (ImagePullBackOff)
- VPC CNI issues or IP address exhaustion
- CoreDNS failures (DNS resolution broken)
- Service connectivity problems (ClusterIP, NodePort, LoadBalancer)
- Ingress/ALB not creating or routing traffic
- aws-auth ConfigMap misconfiguration (locked out of cluster)
- IRSA not working (pods can't assume IAM roles)
- RBAC permission denied errors
- EBS/EFS CSI driver issues or PVC stuck in Pending
- Cluster Autoscaler or Karpenter not scaling
- HPA not triggering or scaling incorrectly
- Add-on installation, update, or compatibility failures
- Container Insights or logging not working
- The user says something is wrong with EKS without naming specific symptoms

---

## Skill Structure

```
eks-troubleshooting/
├── SKILL.md                              # Main skill definition and investigation workflow
├── README.md                             # This file
└── references/
    ├── eks-guardrails.md                 # 12 anti-misdiagnosis guardrails
    ├── eks-hallucination-patterns.yaml   # 8 common LLM hallucination patterns
    ├── A1-cluster-creation-failures.md
    ├── A2-cluster-endpoint-access.md
    ├── A3-cluster-upgrade-failures.md
    ├── B1-node-not-ready.md
    ├── B2-managed-nodegroup-failures.md
    ├── B3-fargate-profile-issues.md
    ├── B4-node-scaling-problems.md
    ├── C1-pod-pending.md
    ├── C2-crashloopbackoff.md
    ├── C3-oomkilled.md
    ├── C4-image-pull-errors.md
    ├── D1-vpc-cni-ip-exhaustion.md
    ├── D2-coredns-failures.md
    ├── D3-service-connectivity.md
    ├── D4-ingress-alb-issues.md
    ├── E1-aws-auth-configmap.md
    ├── E2-irsa-failures.md
    ├── E3-rbac-denied.md
    ├── E4-pod-identity.md
    ├── F1-ebs-csi-driver.md
    ├── F2-efs-csi-driver.md
    ├── F3-pvc-pending.md
    ├── G1-cluster-autoscaler.md
    ├── G2-karpenter-issues.md
    ├── G3-hpa-issues.md
    ├── H1-addon-failures.md
    ├── H2-addon-version-compatibility.md
    ├── H3-custom-addons.md
    ├── I1-container-insights.md
    ├── I2-logging-issues.md
    └── Z1-general-troubleshooting.md
```

---

## Runbook Library (34 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Cluster** | A1–A3 | Cluster creation failures, endpoint access issues, cluster upgrade failures |
| **B — Nodes** | B1–B4 | Node NotReady, managed node group failures, Fargate profile issues, node scaling problems |
| **C — Pods** | C1–C4 | Pending pods, CrashLoopBackOff, OOMKilled, image pull errors |
| **D — Networking** | D1–D4 | VPC CNI / IP exhaustion, CoreDNS failures, service connectivity, ingress/ALB issues |
| **E — IAM & RBAC** | E1–E4 | aws-auth ConfigMap issues, IRSA failures, RBAC denied, pod identity problems |
| **F — Storage** | F1–F3 | EBS CSI driver issues, EFS CSI driver issues, PVC pending |
| **G — Scaling** | G1–G3 | Cluster Autoscaler issues, Karpenter issues, HPA not scaling |
| **H — Add-ons** | H1–H3 | Add-on installation/update failures, version compatibility, custom add-ons |
| **I — Observability** | I1–I2 | Container Insights setup/issues, control plane and application logging |
| **Z — Catch-All** | Z1 | General EKS troubleshooting |

---

## Guardrails & Anti-Hallucination

The skill includes two guardrail files that prevent common misdiagnosis:

### EKS Guardrails (`eks-guardrails.md`)
12 rules covering: aws-auth ConfigMap backup requirements, IRSA three-component dependency, VPC CNI IP allocation and ENI limits, CoreDNS pod dependency on node health, ALB Controller subnet tag requirements, EBS CSI driver add-on requirement, cluster endpoint public/private access implications, managed vs self-managed vs Fargate troubleshooting paths, kubeconfig context verification, Pod Security Standards migration, Karpenter vs Cluster Autoscaler mutual exclusivity, and control plane log types.

### Hallucination Patterns (`eks-hallucination-patterns.yaml`)
8 patterns that LLMs commonly get wrong about EKS, including:
- Claiming direct access to EKS control plane components (etcd, kube-apiserver)
- Suggesting DaemonSets on Fargate (not supported)
- Claiming cluster upgrades can skip minor versions (must be sequential)
- Confusing aws-auth ConfigMap with IAM policies (different layers)
- Assuming VPC CNI has unlimited IPs (bounded by ENI limits)
- Claiming CoreDNS is a managed service (it runs as pods)
- Suggesting PodSecurityPolicy on 1.25+ (removed, use PSS/PSA)
- Assuming Karpenter and Cluster Autoscaler can coexist (mutually exclusive)

---

## Investigation Workflow

Each runbook follows a consistent phased structure:

### Phase 1 — Triage
Collect initial evidence using kubectl (`get nodes`, `get pods`, `describe`, `logs`, `get events`) and AWS CLI (`describe-cluster`, `describe-nodegroup`, `describe-addon`). Classify the failure domain.

### Phase 2 — Remediate
Deep dive into the specific domain using targeted commands, CloudWatch metrics, Container Insights, CloudTrail events, or VPC flow logs. Apply the fix.

### Output Format
Every runbook produces structured YAML output:
```yaml
root_cause: "<category> — <detail>"
evidence:
  - type: <source>
    content: "<specific finding>"
severity: CRITICAL | HIGH | MEDIUM
mitigation:
  immediate: "<action>"
  long_term: "<prevention>"
```

---

## Prerequisites

- AWS CLI with EKS, EC2, CloudWatch, CloudTrail, and IAM permissions
- kubectl configured with cluster access (`aws eks update-kubeconfig`)
- eksctl (optional, for eksctl-managed clusters)
- For Container Insights: CloudWatch Agent and Fluent Bit deployed
- For IRSA debugging: IAM permissions to describe OIDC providers and roles

---

## Usage Examples

### Nodes Not Ready
```
My EKS cluster "prod-cluster" in us-east-1 has 3 nodes showing NotReady.
Check node conditions, kubelet status, VPC CNI health, and recent events.
```

### Pod Scheduling Failures
```
Pods in the "api" namespace are stuck in Pending. The cluster has 10 nodes
but nothing is scheduling. Check resource requests, node capacity, taints,
and tolerations.
```

### IRSA Not Working
```
My pod can't access S3 even though I set up IRSA. The service account has
the role annotation. Check the OIDC provider, trust policy, and token
mounting.
```

### Cluster Upgrade
```
I need to upgrade my EKS cluster from 1.27 to 1.28. Check add-on
compatibility, deprecated APIs, and node group AMI versions before
proceeding.
```

### General Triage
```
Something is wrong with my EKS cluster but I'm not sure what. Pods are
failing, some nodes look unhealthy. Run a general investigation and
follow whichever runbook matches the symptoms.
```

---

## License

MIT-0
