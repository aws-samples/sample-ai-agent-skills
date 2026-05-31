---
title: "E2 — Cloud Map EKS Integration Issues"
description: "Diagnose EKS service discovery integration with Cloud Map"
status: active
severity: MEDIUM
triggers:
  - "EKS Cloud Map"
  - "EKS service discovery"
  - "external-dns"
  - "cloud map controller"
owner: devops-agent
objective: "Identify and resolve EKS integration issues with Cloud Map"
context: "EKS integrates with Cloud Map via the AWS Cloud Map MCS Controller or external-dns. These controllers sync Kubernetes services with Cloud Map for cross-cluster or cross-service discovery."
---

## Phase 1 — Triage

MUST:
- Check if Cloud Map controller is deployed: `kubectl get pods -n kube-system | grep cloudmap`
- Check controller logs: `kubectl logs -n kube-system -l app=aws-cloud-map-mcs-controller --tail=100`
- Verify Cloud Map namespace exists: `aws servicediscovery list-namespaces`
- Check Kubernetes services with Cloud Map annotations: `kubectl get svc -A -o yaml | grep cloudmap`
- Verify IRSA for the controller: `kubectl get sa -n kube-system aws-cloud-map-mcs-controller -o yaml`

SHOULD:
- Check controller RBAC permissions
- Verify Cloud Map service and instances match Kubernetes endpoints
- Check for controller errors or reconciliation failures

MAY:
- Check external-dns if using that instead of MCS controller
- Verify cross-cluster discovery if using multi-cluster setup

## Phase 2 — Remediate

MUST:
- Deploy and configure the Cloud Map controller or external-dns
- Configure IRSA with Cloud Map permissions
- Annotate Kubernetes services for Cloud Map registration

SHOULD:
- Use the latest controller version
- Monitor controller health and reconciliation
- Test discovery from consumer services

MAY:
- Set up multi-cluster service discovery
- Implement controller monitoring and alerting

## Common Issues

- symptoms: "Kubernetes services not appearing in Cloud Map"
  diagnosis: "Cloud Map controller not deployed or lacking permissions."
  resolution: "Deploy the controller and configure IRSA with Cloud Map permissions."

- symptoms: "Controller running but not syncing services"
  diagnosis: "Services missing required annotations or controller RBAC insufficient."
  resolution: "Add Cloud Map annotations to services and verify controller RBAC."

- symptoms: "Cloud Map instances stale after pod scaling"
  diagnosis: "Controller reconciliation delayed or failing."
  resolution: "Check controller logs for errors. Restart controller if stuck."

## Output Format

```yaml
root_cause: "eks_integration_issue — <specific_cause>"
evidence:
  - type: controller_status
    content: "<controller pod status and logs>"
  - type: kubernetes_services
    content: "<annotated services>"
  - type: cloudmap_state
    content: "<Cloud Map services and instances>"
severity: MEDIUM
mitigation:
  immediate: "Fix controller configuration or IRSA permissions"
  long_term: "Automate Cloud Map integration in EKS cluster setup"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Deploy and configure Cloud Map controller | YELLOW |
| Configure IRSA with Cloud Map permissions | YELLOW |
| Annotate Kubernetes services for registration | GREEN |
| Use latest controller version | GREEN |
| Monitor controller health | GREEN |
| Set up multi-cluster service discovery | YELLOW |

## Escalation Conditions

- Fix requires modifying production EKS cluster configurations
- Fix involves deploying or updating controllers in production clusters

## Data Sensitivity

- **MEDIUM**: EKS integration configurations expose cluster topology and service mesh architecture
- IRSA configurations reveal cross-service trust relationships

## Prohibited Actions

- NEVER suggest deleting namespaces with active service registrations

## Phase 3 — Rollback

1. If controller was deployed or updated, rollback to previous version: `kubectl rollout undo deployment -n kube-system aws-cloud-map-mcs-controller`
2. If IRSA was modified, restore previous service account annotations
3. If service annotations were changed, remove Cloud Map annotations to stop syncing
4. Verify rollback by confirming Cloud Map instances match expected Kubernetes service state

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Data Sensitivity

data_sensitivity:
  - command: "describe-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
