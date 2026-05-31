---
title: "Z1 — General Cloud Map Troubleshooting (Catch-All)"
description: "Fallback SOP for Cloud Map issues that do not match any specific runbook"
status: active
severity: MEDIUM
triggers:
  - ".*"
owner: devops-agent
objective: "Systematically investigate an unknown Cloud Map issue, classify the failure domain, and match to an existing SOP or escalate"
context: "This SOP is invoked when symptoms don't match any of the specific runbooks."
---

## Phase 1 — Triage

MUST:
- List all namespaces: `aws servicediscovery list-namespaces`
- Check namespace details: `aws servicediscovery get-namespace --id <namespace-id>`
- List services: `aws servicediscovery list-services --filters Name=NAMESPACE_ID,Values=<namespace-id>`
- List instances: `aws servicediscovery list-instances --service-id <service-id>`
- Check instance health: `aws servicediscovery get-instances-health-status --service-id <service-id>`
- Check CloudTrail: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=servicediscovery.amazonaws.com --max-results 20`

SHOULD:
- Test discovery: `aws servicediscovery discover-instances --namespace-name <namespace> --service-name <service>`
- Check DNS records for DNS namespaces
- Verify VPC associations for private namespaces

## Phase 2 — Classify

Based on triage results, classify into a failure domain:
- Namespace creation/configuration → Namespace (A1-A2)
- Service registration/health checks → Service (B1-B2)
- Instance registration/deregistration → Instance (C1-C2)
- DNS resolution/TTL → DNS (D1-D2)
- ECS/EKS integration → Integration (E1-E2)
- IAM/VPC issues → Security (F1-F2)

If classified: switch to the specific SOP immediately.
If unclassified: continue to Phase 3.

## Phase 3 — Deep Investigation

MUST:
- Check all namespace, service, and instance configurations
- Review CloudTrail for recent changes
- Verify IAM permissions and VPC settings
- Test both DNS and API discovery

## Phase 4 — Report

```yaml
root_cause: "<identified_cause OR unclassified>"
failure_domain: "<namespace|service|instance|dns|integration|security|unknown>"
investigation_path: "namespaces → services → instances → <domain_classification>"
evidence:
  - type: namespace_config
    content: "<namespace details>"
  - type: discovery_test
    content: "<discovery results>"
  - type: cloudtrail
    content: "<relevant events>"
severity: MEDIUM
mitigation:
  immediate: "<specific action if root cause found, or escalate>"
  long_term: "Implement monitoring for the identified failure pattern"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| List all namespaces, services, instances | GREEN |
| Check instance health | GREEN |
| Check CloudTrail events | GREEN |
| Test discovery | GREEN |
| Check DNS records | GREEN |

## Escalation Conditions

- Issue cannot be classified into any specific failure domain
- Fix requires modifying production namespace or service configurations

## Data Sensitivity

- **MEDIUM**: general troubleshooting may expose service discovery topology and infrastructure details
- Investigation may reveal endpoint addresses, health status, and network configurations

## Prohibited Actions

- NEVER suggest deleting namespaces with active service registrations

## Phase 3 — Rollback

1. If any configuration was changed during investigation, restore previous settings
2. If permissions were modified, revert to previous IAM policies
3. Document all changes made during investigation for audit purposes
4. Verify rollback by confirming all Cloud Map resources match pre-investigation state

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
