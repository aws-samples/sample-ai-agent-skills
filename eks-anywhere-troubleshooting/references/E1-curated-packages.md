---
title: "E1 — Curated Packages"
description: "Diagnose EKS Anywhere curated package installation and management issues"
status: active
severity: MEDIUM
triggers: ["package install failed", "curated package error", "ECR authentication failed", "helm release failed", "package controller error"]
owner: devops-agent
objective: "Resolve curated package issues including ECR authentication, Helm releases, and package controller problems"
context: "Curated packages are vetted Helm charts hosted in Amazon ECR. They require AWS credentials and periodic ECR token refresh. The package controller manages lifecycle."
---

## Phase 1 — Triage

MUST:
- Check package controller: `kubectl get pods -n eksa-packages --kubeconfig <workload-kubeconfig>`
- List installed packages: `kubectl get packages -A --kubeconfig <workload-kubeconfig>`
- Check HelmRelease status: `kubectl get helmreleases -A --kubeconfig <workload-kubeconfig>`

SHOULD:
- Verify ECR authentication: `kubectl get secret -n eksa-packages ecr-token --kubeconfig <workload-kubeconfig>`
- Check package bundle availability: `kubectl get packagebundles -A --kubeconfig <workload-kubeconfig>`
- Review package controller logs: `kubectl logs -n eksa-packages deployment/eks-anywhere-packages --tail=200`

MAY:
- Verify AWS credentials for ECR access: `aws ecr get-login-password --region us-west-2`
- Check available package versions: `eksctl anywhere list packages`

## Phase 2 — Remediate

MUST:
- Refresh ECR authentication token if expired (tokens expire every 12 hours)
- Fix package controller crashes or resource issues
- Resolve Helm chart pull failures from ECR

SHOULD:
- Update AWS credentials if IAM permissions are insufficient
- Verify network connectivity from cluster nodes to ECR endpoints

MAY:
- Manually trigger package reconciliation by restarting the package controller
- Upgrade package versions to resolve known bugs

## Safety Ratings

```
safety_ratings:
  - "kubectl get packages/helmreleases: GREEN — read-only status check"
  - "kubectl logs package-controller: GREEN — read-only log retrieval"
  - "aws ecr get-login-password: GREEN — generates temporary token"
  - "eksctl anywhere install package: YELLOW — installs software on cluster"
```

## Escalation Conditions

- Package controller crash-loops and cannot be recovered
- ECR authentication fails despite valid AWS credentials
- Critical packages (cert-manager, metrics-server) fail to install blocking other operations
- Package version incompatibility with current EKS Anywhere version

## Data Sensitivity

- `kubectl get packages`: LOW — returns package metadata
- `kubectl get secret ecr-token`: HIGH — contains ECR authentication credentials
- `aws ecr get-login-password`: HIGH — generates ECR authentication token
- `package controller logs`: MEDIUM — may contain registry URLs and authentication details

## Prohibited Actions

- NEVER expose ECR tokens or AWS credentials in logs or support bundles
- NEVER force-delete package resources while Helm releases are in progress
- NEVER install packages from untrusted registries as curated package replacements
- NEVER modify package controller deployment directly — use eksctl anywhere CLI

## Phase 3 — Rollback

1. If a package was installed incorrectly: `eksctl anywhere delete package <name> --cluster <cluster-name>`
2. If package controller was modified, restore from original deployment
3. Clean up orphaned Helm releases: `kubectl delete helmrelease <name> -n <namespace>`
4. Verify remaining packages are healthy: `kubectl get packages -A`
5. Refresh ECR token to ensure future operations succeed

## Common Issues

- symptoms: "Package install fails — ImagePullBackOff from ECR"
  diagnosis: "ECR authentication token expired or AWS credentials invalid."
  resolution: "Refresh ECR token. Verify AWS credentials have ecr:GetAuthorizationToken permission."

## Output Format

```yaml
root_cause: "curated_packages — <specific_cause>"
evidence:
  - type: package_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix package installation"
  long_term: "Automate ECR token refresh"
```

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Data Sensitivity

data_sensitivity:
  - command: "describe-cluster"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "kubectl get configmap aws-auth"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "kubectl get pods"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest system:masters group for troubleshooting"
  - "NEVER suggest disabling RBAC or Pod Security Standards"
  - "NEVER suggest running pods as root to fix permissions"
