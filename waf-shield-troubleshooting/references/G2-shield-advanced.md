---
title: "G2 — Shield Advanced Configuration Issues"
description: "Diagnose Shield Advanced subscription, protection, and configuration problems"
status: active
severity: HIGH
triggers:
  - "Shield Advanced"
  - "Shield subscription"
  - "Shield protection"
  - "Shield configuration"
  - "advanced DDoS"
owner: devops-agent
objective: "Identify and fix Shield Advanced configuration issues to ensure comprehensive DDoS protection"
context: "Shield Advanced costs $3,000/month per organization with a 1-year commitment. It provides L7 DDoS protection, DDoS cost protection, 24/7 SRT access, advanced metrics, WAF fee waiver for protected resources, and proactive engagement. Each resource must be explicitly added as a protection. Health checks are required for cost protection and proactive engagement."
---

## Phase 1 — Triage

MUST:
- Check subscription status: `aws shield describe-subscription`
- List protected resources: `aws shield list-protections`
- Check protection details for a specific resource: `aws shield describe-protection --resource-arn <resource-arn>`
- Verify health checks are associated: `aws shield describe-protection --resource-arn <resource-arn> --query 'Protection.HealthCheckIds'`

SHOULD:
- Check if the DRT role is configured: `aws shield describe-drt-access`
- Verify proactive engagement status: `aws shield describe-emergency-contact-settings`
- Check if all critical resources have protections enabled

MAY:
- Review Shield Advanced metrics in CloudWatch
- Check the subscription auto-renew status
- Verify WAF fee waiver is applied to protected resources

## Phase 2 — Remediate

MUST:
- Add protections for unprotected resources: `aws shield create-protection --name <name> --resource-arn <resource-arn>`
- Associate Route 53 health checks with protections: `aws shield associate-health-check --protection-id <protection-id> --health-check-arn <health-check-arn>`
- Configure the DRT access role: `aws shield associate-drt-role --role-arn <role-arn>`

SHOULD:
- Set up emergency contacts: `aws shield associate-proactive-engagement-details --emergency-contact-list EmailAddress=<email>,PhoneNumber=<phone>,ContactNotes=<notes>`
- Enable proactive engagement: `aws shield enable-proactive-engagement`
- Create Route 53 health checks for all protected resources

MAY:
- Use AWS Firewall Manager for cross-account Shield Advanced management
- Set up CloudWatch dashboards for Shield Advanced metrics
- Implement automated protection for new resources via CloudFormation or Terraform

## Common Issues

- symptoms: "Shield Advanced subscription active but resource not protected"
  diagnosis: "Subscription enables the service but protections must be added per resource."
  resolution: "Create a protection for each resource using create-protection."

- symptoms: "Cost protection claim denied"
  diagnosis: "No Route 53 health check associated with the protected resource."
  resolution: "Create a health check monitoring the resource and associate it with the protection."

- symptoms: "Proactive engagement not working"
  diagnosis: "Emergency contacts not configured or proactive engagement not enabled."
  resolution: "Configure emergency contacts and enable proactive engagement."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Add protections | YELLOW | New protection; can be removed |
| Associate health checks | YELLOW | Configuration change; reversible |
| Configure DRT access role | YELLOW | IAM change; reversible |
| Enable proactive engagement | YELLOW | Configuration change; reversible |

## Escalation Conditions

- Shield Advanced configuration changes
- Production web ACL rule changes
- DRT access role modifications
- Proactive engagement enablement

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-subscription` | LOW | Subscription status |
| `list-protections` | LOW | Protected resource ARNs |
| `describe-drt-access` | MEDIUM | DRT role ARN |
| `describe-emergency-contact-settings` | HIGH | Contact email and phone |

## Prohibited Actions

- NEVER suggest removing all WAF rules to fix false positives
- NEVER suggest disabling Shield Advanced
- NEVER suggest setting rate limit to maximum to "disable" rate limiting
- NEVER remove Shield Advanced protections without explicit approval

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Protection addition | Remove via `delete-protection` |
| Health check association | Disassociate via `disassociate-health-check` |
| DRT role association | Disassociate via `disassociate-drt-role` |
| Proactive engagement | Disable via `disable-proactive-engagement` |

## Output Format

```yaml
root_cause: "shield_advanced — <specific_cause>"
evidence:
  - type: subscription
    content: "<subscription status and details>"
  - type: protections
    content: "<list of protected resources>"
  - type: health_checks
    content: "<associated health checks>"
severity: HIGH
mitigation:
  immediate: "Add missing protections and health checks"
  long_term: "Implement comprehensive Shield Advanced with proactive engagement"
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
