# EKS Anywhere Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any EKS Anywhere issue.

## Guardrail 1: EKS Anywhere Is Not EKS
EKS Anywhere runs Kubernetes on-premises or on your own infrastructure. It is NOT the same as Amazon EKS (managed Kubernetes in AWS). Do not suggest EKS APIs (eks:DescribeCluster, eks:UpdateClusterConfig) for EKS Anywhere clusters. Use eksctl-anywhere CLI and kubectl instead.

## Guardrail 2: Management Cluster vs Workload Cluster
EKS Anywhere uses a management cluster to manage workload clusters via Cluster API (CAPI). These are separate clusters with separate kubeconfigs. Always verify which cluster context you are operating in. Losing the management cluster means losing management plane for all workload clusters.

## Guardrail 3: Cluster API Resources Are Not EKS Anywhere Resources
CAPI resources (Machine, MachineDeployment, KubeadmControlPlane) are infrastructure-level objects managed by CAPI controllers. EKS Anywhere resources (Cluster, VSphereDatacenterConfig, TinkerbellDatacenterConfig) are higher-level abstractions. Do not confuse their status or lifecycle.

## Guardrail 4: Provider-Specific Prerequisites Must Be Met
Each infrastructure provider has unique prerequisites. vSphere requires template VMs, resource pools, and vCenter permissions. Bare metal requires Tinkerbell stack, BMC credentials, and hardware inventory. CloudStack requires compute offerings and templates. Nutanix requires Prism Central access. Verify provider prerequisites before diagnosing cluster issues.

## Guardrail 5: Cilium Is the Only Supported CNI
EKS Anywhere uses Cilium as the default and only supported CNI. Replacing Cilium with another CNI (Calico, Flannel) is NOT supported and will break cluster networking. Cilium policy enforcement mode (default, always, never) affects network behavior significantly.

## Guardrail 6: Curated Packages Require AWS Account
Curated packages are hosted in Amazon ECR and require an AWS account with valid credentials. ECR authentication tokens expire every 12 hours. Package installation failures are often caused by expired tokens or missing IAM permissions, not package bugs.

## Guardrail 7: Flux GitOps Reverts Manual Changes
When Flux GitOps is enabled, it continuously reconciles cluster state with the Git repository. Manual kubectl changes to Flux-managed resources will be automatically reverted. Always make changes through Git commits, not direct kubectl commands.

## Guardrail 8: Cluster Upgrades Are Sequential
EKS Anywhere cluster upgrades must follow sequential version progression. You cannot skip Kubernetes minor versions (e.g., 1.27 to 1.29). Each upgrade step must complete successfully before proceeding. Rolling back a failed upgrade requires specific procedures.

## Guardrail 9: Support Bundle Contains Sensitive Data
The eksctl anywhere generate support-bundle command collects logs, configurations, and cluster state. This bundle may contain secrets, certificates, and infrastructure credentials. Handle support bundles with appropriate data classification and never share them in public channels.

## Guardrail 10: Tinkerbell Stack Is Critical for Bare Metal
For bare metal deployments, the Tinkerbell stack (Boots, Hegel, Tink Server, Tink Controller) handles PXE boot and machine provisioning. If Tinkerbell components are unhealthy, no new bare metal machines can be provisioned. Check Tinkerbell pod health before investigating node provisioning failures.

## Guardrail 11: Control Plane Endpoint Must Be Stable
The control plane endpoint IP (or VIP) must be reachable and stable. For vSphere, this is typically a static IP managed by kube-vip. For bare metal, it uses a pre-allocated IP. If the control plane endpoint becomes unreachable, the entire cluster becomes inaccessible even if nodes are healthy.

## Guardrail 12: EKS Connector Is Optional and Separate
EKS Connector allows registering EKS Anywhere clusters with the AWS EKS console for visibility. It is optional and does NOT affect cluster functionality. EKS Connector issues do not impact cluster operations. Do not conflate EKS Connector problems with cluster health issues.
