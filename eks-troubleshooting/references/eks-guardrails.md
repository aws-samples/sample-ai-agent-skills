# Amazon EKS Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any EKS issue.

## Guardrail 1: aws-auth ConfigMap Is the Bridge Between IAM and RBAC
The aws-auth ConfigMap in kube-system maps IAM roles/users to Kubernetes RBAC groups. Misconfiguration can lock you out of the cluster entirely. ALWAYS back up before editing: `kubectl get configmap aws-auth -n kube-system -o yaml > aws-auth-backup.yaml`. The cluster creator's IAM identity always has system:masters access and is NOT listed in aws-auth. If locked out, use the cluster creator's credentials or contact AWS support.

## Guardrail 2: IRSA Requires Three Components
IAM Roles for Service Accounts (IRSA) requires ALL three: (1) OIDC provider associated with the cluster (`aws eks describe-cluster --query cluster.identity.oidc`), (2) Kubernetes service account annotated with `eks.amazonaws.com/role-arn: arn:aws:iam::<account>:role/<role>`, (3) IAM role trust policy allowing `sts:AssumeRoleWithWebIdentity` from the OIDC provider with the correct service account condition. Missing any one component causes silent permission failures — pods get default node role permissions instead.

## Guardrail 3: VPC CNI Assigns Pod IPs from VPC Subnets
The VPC CNI plugin assigns real VPC IP addresses to pods via secondary IPs on ENIs. Each instance type has a maximum number of ENIs and IPs per ENI. Max pods = `(ENIs × (IPs_per_ENI - 1)) + 2`. When subnets run out of IPs, new pods go Pending with "failed to assign an IP address." Check subnet CIDR utilization, not just node count. Consider using prefix delegation to increase IP density.

## Guardrail 4: CoreDNS Runs as Pods, Not a Managed Service
CoreDNS is deployed as a Kubernetes Deployment (typically 2 replicas) on worker nodes. If worker nodes are unhealthy, CoreDNS pods fail, and ALL DNS resolution in the cluster breaks. This cascades to every pod that resolves service names, including pods that call AWS APIs via endpoints. Always check CoreDNS pod health early in any "connectivity" investigation.

## Guardrail 5: ALB Controller Requires Subnet Tags
The AWS Load Balancer Controller discovers subnets using tags. Public subnets need `kubernetes.io/role/elb: 1`. Private subnets need `kubernetes.io/role/internal-elb: 1`. Both need `kubernetes.io/cluster/<cluster-name>: owned` or `shared`. Missing tags = no ALB/NLB created, and the Ingress resource shows no errors — it just silently does nothing. Always check subnet tags first.

## Guardrail 6: EBS CSI Driver Is Required Since EKS 1.23
The in-tree EBS provisioner is deprecated. EKS 1.23+ requires the EBS CSI driver add-on for dynamic EBS volume provisioning. Without it, PVCs using `gp2`/`gp3` StorageClasses stay Pending indefinitely. The CSI driver also needs an IAM role (via IRSA or pod identity) with `ec2:CreateVolume`, `ec2:AttachVolume`, etc. Check both the add-on status AND its IAM permissions.

## Guardrail 7: Cluster Endpoint Public vs Private Access
EKS has two endpoint access toggles. Public-only: kubectl works from anywhere, but nodes route through the internet to reach the API server. Private-only: kubectl only works from within the VPC or connected networks (VPN, Direct Connect, peered VPC). Public+private (recommended): kubectl works from anywhere, nodes use private VPC endpoint. Changing endpoint access can lock you out if you're not in the VPC.

## Guardrail 8: Managed Node Groups vs Self-Managed vs Fargate
Each compute type has different troubleshooting paths. Managed node groups: AWS manages the ASG, launch template, and AMI updates — check `aws eks describe-nodegroup` for health issues. Self-managed nodes: you own the ASG, launch template, AMI, and bootstrap script — check the ASG and EC2 instances directly. Fargate: serverless, no nodes to manage — check Fargate profile selectors (namespace + labels) and pod annotations.

## Guardrail 9: kubeconfig Context Is the #1 "It's Not Working" Cause
Before any investigation, verify kubectl is pointing at the right cluster: `kubectl config current-context`. Refresh credentials with `aws eks update-kubeconfig --name <cluster> --region <region>`. Common issues: expired tokens, wrong AWS profile, wrong region, stale kubeconfig from a deleted cluster. If `kubectl get nodes` returns "error: You must be logged in to the server," it's almost always a kubeconfig or IAM issue.

## Guardrail 10: Pod Security Standards Replaced PodSecurityPolicy in 1.25
PodSecurityPolicy (PSP) was removed in Kubernetes 1.25. PSP resources are silently ignored — they don't cause errors, they just don't enforce anything. Use Pod Security Standards (PSS) with Pod Security Admission (PSA) instead. Apply labels to namespaces: `pod-security.kubernetes.io/enforce: restricted|baseline|privileged`. Migrating from PSP to PSS requires auditing existing workloads.

## Guardrail 11: Karpenter and Cluster Autoscaler Are Mutually Exclusive
Cluster Autoscaler scales by adjusting ASG desired count — it works with node groups. Karpenter provisions nodes directly via EC2 fleet API, bypassing ASGs entirely. Running both simultaneously causes conflicts: both try to scale, leading to over-provisioning or race conditions. Choose one. Karpenter is generally preferred for new clusters due to faster scaling and better bin-packing.

## Guardrail 12: Control Plane Logging Has 5 Separate Log Types
EKS control plane logging sends logs to CloudWatch under `/aws/eks/<cluster>/cluster`. The 5 log types are: api (API server requests), audit (who did what), authenticator (IAM authentication), controllerManager (controller loops), scheduler (pod scheduling decisions). Each must be explicitly enabled. They are NOT enabled by default. Enable at least api and audit for production clusters.
