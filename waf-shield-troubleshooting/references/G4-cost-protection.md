---
title: "G4 — Shield Advanced Cost Protection"
description: "Diagnose DDoS cost protection configuration and claim issues"
status: active
severity: HIGH
triggers:
  - "cost protection"
  - "DDoS cost"
  - "scaling charges"
  - "DDoS credit"
  - "Shield billing"
owner: devops-agent
objective: "Ensure DDoS cost protection is properly configured and understand the claim process"
context: "Shield Advanced DDoS cost protection provides credits for scaling charges incurred during DDoS attacks. This covers charges for EC2, ALB, CloudFront, Global Accelerator, and Route 53 that spike during an attack. Cost protection requires Shield Advanced subscription, the resource must have an active protection, AND a Route 53 health check must be associated with the protection. Without the health check, claims are denied."
---

## Phase 1 — Triage

MUST:
- Verify Shield Advanced subscription: `aws shield describe-subscription`
- Check the resource has a protection: `aws shield describe-protection --resource-arn <resource-arn>`
- Verify a health check is associated: `aws shield describe-protection --resource-arn <resource-arn> --query 'Protection.HealthCheckIds'`
- Check the health check status: `aws route53 get-health-check-status --health-check-id <health-check-id>`

SHOULD:
- Verify the health check monitors the actual protected resource
- Check if the attack was detected by Shield: `aws shield list-attacks --start-time FromInclusive=<start>,ToExclusive=<end> --resource-arns <resource-arn>`
- Review the attack timeline and correlate with billing spikes

MAY:
- Check AWS Cost Explorer for the billing period during the attack
- Review the Shield Advanced subscription terms for cost protection eligibility

## Phase 2 — Remediate

MUST:
- Associate a Route 53 health check with each protected resource
- Create health checks that accurately monitor resource availability: `aws route53 create-health-check --caller-reference <ref> --health-check-config Type=HTTPS,FullyQualifiedDomainName=<domain>,Port=443,ResourcePath=/health`
- Associate the health check: `aws shield associate-health-check --protection-id <protection-id> --health-check-arn arn:aws:route53:::healthcheck/<health-check-id>`

SHOULD:
- Configure health checks before an attack occurs (retroactive association does not cover past attacks)
- Set appropriate health check thresholds to accurately reflect resource health
- Document the cost protection claim process for your finance team

MAY:
- Set up CloudWatch alarms on health check status for early warning
- Create a cost protection claim checklist for incident response
- Use AWS Support to file cost protection claims after verified attacks

## Common Issues

- symptoms: "Cost protection claim denied"
  diagnosis: "No Route 53 health check was associated with the protection at the time of the attack."
  resolution: "Associate health checks with all protected resources proactively."

- symptoms: "Health check shows healthy during attack"
  diagnosis: "Health check endpoint is not representative of the protected resource's actual health."
  resolution: "Configure the health check to monitor the actual application endpoint affected by the attack."

- symptoms: "Scaling charges not covered by cost protection"
  diagnosis: "The resource type is not eligible, or the charges are not directly related to DDoS-induced scaling."
  resolution: "Cost protection covers EC2, ALB, CloudFront, Global Accelerator, and Route 53 scaling charges only."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Create Route 53 health check | YELLOW | New resource; can be deleted |
| Associate health check with protection | YELLOW | Configuration change; reversible |

## Escalation Conditions

- Shield Advanced configuration changes
- Health check changes for protected resources
- Cost protection claim filing

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-subscription` | LOW | Subscription status |
| `describe-protection` | LOW | Protection and health check details |
| `list-attacks` | MEDIUM | Attack details |
| `get-health-check-status` | LOW | Health check status |

## Prohibited Actions

- NEVER suggest removing all WAF rules to fix false positives
- NEVER suggest disabling Shield Advanced
- NEVER suggest setting rate limit to maximum to "disable" rate limiting
- NEVER remove health checks from protected resources

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Health check creation | Delete via `delete-health-check` |
| Health check association | Disassociate via `disassociate-health-check` |

## Output Format

```yaml
root_cause: "cost_protection — <specific_cause>"
evidence:
  - type: protection
    content: "<protection status and health check>"
  - type: attack
    content: "<attack detection details>"
  - type: health_check
    content: "<health check status during attack>"
severity: HIGH
mitigation:
  immediate: "Associate health checks with all protected resources"
  long_term: "Implement comprehensive cost protection with proactive health monitoring"
```

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
  - "NEVER suggest disabling WAF to fix access issues"
  - "NEVER suggest removing all WAF rules"
  - "NEVER suggest allowing all IPs to bypass rate limiting"
