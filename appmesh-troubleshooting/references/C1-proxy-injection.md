---
title: "C1 — Envoy Proxy Injection Issues"
description: "Diagnose Envoy sidecar injection failures"
status: active
severity: HIGH
triggers:
  - "sidecar not injected"
  - "envoy missing"
  - "no proxy"
  - "injection failed"
  - "sidecar injection"
owner: devops-agent
objective: "Identify why Envoy sidecars are not being injected and fix the injection mechanism"
context: "Envoy sidecars must be present in every service pod/task for mesh routing. On EKS, the App Mesh controller handles injection via annotations. On ECS, the Envoy container must be in the task definition."
---

## Phase 1 — Triage

MUST:
- For EKS, check pod containers: `kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.containers[*].name}'`
- Check namespace annotations: `kubectl get namespace <namespace> -o yaml | grep appmesh`
- Check pod annotations: `kubectl get pod <pod-name> -n <namespace> -o yaml | grep appmesh`
- For ECS, check task definition: `aws ecs describe-task-definition --task-definition <task-def> --query 'taskDefinition.containerDefinitions[*].name'`
- Verify App Mesh controller is running (EKS): `kubectl get pods -n appmesh-system`

SHOULD:
- Check controller logs: `kubectl logs -n appmesh-system -l app.kubernetes.io/name=appmesh-controller --tail=50`
- Verify mutating webhook configuration: `kubectl get mutatingwebhookconfigurations | grep appmesh`
- Check if init container (proxyinit) is present

MAY:
- Check controller version compatibility with cluster version
- Verify IRSA (IAM Roles for Service Accounts) for the controller

## Phase 2 — Remediate

MUST:
- For EKS: add namespace annotation `appmesh.k8s.aws/sidecarInjectorWebhook: enabled`
- For EKS: add pod annotations for mesh name and virtual node name
- For ECS: add Envoy container and proxyConfiguration to task definition
- Ensure App Mesh controller is running and healthy

SHOULD:
- Use the AWS-recommended Envoy image version
- Configure Envoy container resource limits
- Set up proper init container for traffic interception (iptables rules)

MAY:
- Customize Envoy configuration via annotations
- Set up Envoy access logging

## Common Issues

- symptoms: "Pod running but no Envoy sidecar container"
  diagnosis: "Namespace or pod missing App Mesh injection annotations."
  resolution: "Add appmesh.k8s.aws/sidecarInjectorWebhook annotation to namespace."

- symptoms: "Envoy container present but not starting"
  diagnosis: "Envoy image pull failure or resource limits too low."
  resolution: "Verify Envoy image is accessible and increase resource limits."

- symptoms: "ECS task missing Envoy container"
  diagnosis: "Task definition doesn't include Envoy container definition."
  resolution: "Add Envoy container and proxyConfiguration to the ECS task definition."

## Output Format

```yaml
root_cause: "proxy_injection_issue — <specific_cause>"
evidence:
  - type: pod_containers
    content: "<container list showing missing Envoy>"
  - type: annotations
    content: "<namespace and pod annotations>"
  - type: controller_status
    content: "<App Mesh controller health>"
severity: HIGH
mitigation:
  immediate: "Add injection annotations or Envoy container definition"
  long_term: "Automate sidecar injection configuration in deployment templates"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Add namespace annotation for injection | YELLOW | Affects all new pods in namespace |
| Add pod annotations for mesh/node | YELLOW | Changes pod networking |
| Add Envoy container to ECS task def | YELLOW | Task definition change |
| Ensure App Mesh controller is running | GREEN | Health verification |
| Use AWS-recommended Envoy image | GREEN | Image selection |
| Configure Envoy resource limits | GREEN | Resource configuration |

## Escalation Conditions

- Fix requires modifying Envoy proxy config in production
- Fix requires changing routing rules

## Data Sensitivity

- **Pod annotations**: Mesh and virtual node mapping — INTERNAL
- **ECS task definitions**: Container configuration — INTERNAL
- **Controller logs**: Injection and reconciliation events — INTERNAL

## Prohibited Actions

- NEVER suggest removing mTLS to fix connectivity
- NEVER enable sidecar injection on a production namespace without testing
- NEVER modify Envoy init container iptables rules manually

## Phase 3 — Rollback

1. Remove namespace injection annotation: `kubectl annotate namespace <ns> appmesh.k8s.aws/sidecarInjectorWebhook-`
2. For ECS: revert task definition to version without Envoy container
3. Restart affected pods/tasks to remove injected sidecars
4. Verify services are communicating without the mesh
5. Monitor application health after sidecar removal

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
