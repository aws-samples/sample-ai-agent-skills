---
title: "G1 — DDoS Detection Issues"
description: "Diagnose DDoS attack detection and response with Shield"
status: active
severity: CRITICAL
triggers:
  - "DDoS"
  - "DDoS attack"
  - "DDoS detection"
  - "traffic spike"
  - "volumetric attack"
owner: devops-agent
objective: "Identify DDoS attacks and verify Shield detection and mitigation is working"
context: "Shield Standard automatically protects all AWS resources against common L3/L4 DDoS attacks. Shield Advanced adds L7 protection, advanced detection, and real-time metrics. DDoS attacks can be volumetric (L3/L4), protocol-based, or application-layer (L7). Shield Advanced provides attack visibility through the Shield console and API. Detection relies on traffic baselines and anomaly detection."
---

## Phase 1 — Triage

MUST:
- Check for active or recent attacks: `aws shield list-attacks --start-time FromInclusive=<start>,ToExclusive=<end>`
- Get attack details if an attack is detected: `aws shield describe-attack --attack-id <attack-id>`
- Check Shield subscription status: `aws shield describe-subscription`
- Check CloudWatch metrics for traffic anomalies on the protected resource

SHOULD:
- Review the attack vectors and mitigation actions in the attack details
- Check if the resource has Shield Advanced protection: `aws shield describe-protection --resource-arn <resource-arn>`
- Verify health checks are configured for proactive engagement: `aws shield describe-protection --resource-arn <resource-arn> --query 'Protection.HealthCheckIds'`

MAY:
- Check Route 53 health check status for the protected resource
- Review CloudWatch metrics for the specific resource (ALB, CloudFront, EIP)
- Check VPC Flow Logs for network-level traffic patterns

## Phase 2 — Remediate

MUST:
- For active L3/L4 attacks: Shield Standard mitigates automatically. Verify mitigation is active in attack details.
- For L7 attacks: ensure WAF rate-based rules are in place. Add emergency rate-based rules if needed.
- If Shield Advanced: engage the SRT if the attack is severe (see G3-srt-engagement.md)

SHOULD:
- Add WAF rate-based rules to limit request rates from attacking IPs
- Enable WAF logging to capture attack traffic details
- Scale resources (ALB, CloudFront) to absorb traffic while mitigation takes effect

MAY:
- Implement geographic blocking if the attack originates from specific regions
- Add IP reputation managed rule groups to block known bad IPs
- Use AWS Global Accelerator for additional DDoS protection

## Common Issues

- symptoms: "Application slow but no DDoS attack detected by Shield"
  diagnosis: "Attack is below Shield detection thresholds, or it's an L7 attack and Shield Standard is in use (L3/L4 only)."
  resolution: "Check WAF metrics for L7 attack patterns. Consider Shield Advanced for L7 detection."

- symptoms: "Shield Advanced shows attack but application is still impacted"
  diagnosis: "L7 attack bypassing Shield's L3/L4 mitigation. WAF rules not configured for the attack pattern."
  resolution: "Add WAF rate-based rules. Engage SRT for assistance. Review WAF logs for attack patterns."

- symptoms: "Frequent false positive DDoS detections"
  diagnosis: "Legitimate traffic spikes (flash sales, marketing campaigns) trigger anomaly detection."
  resolution: "Configure Route 53 health checks for health-based detection. Inform SRT of expected traffic patterns."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Add WAF rate-based rules | YELLOW | New rule; can be removed |
| Scale resources | YELLOW | Capacity change; reversible |
| Engage SRT | GREEN | Advisory engagement; no direct change |
| Add geographic blocking | YELLOW | New rule; can be removed |

## Escalation Conditions

- Active DDoS attack in progress
- Production web ACL rule changes
- Shield Advanced configuration changes
- SRT engagement required

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `list-attacks` | MEDIUM | Attack details and target resources |
| `describe-attack` | MEDIUM | Attack vectors and mitigation details |
| `describe-subscription` | LOW | Subscription status |
| CloudWatch traffic metrics | LOW | Aggregate traffic data |

## Prohibited Actions

- NEVER suggest removing all WAF rules to fix false positives
- NEVER suggest disabling Shield Advanced
- NEVER suggest setting rate limit to maximum to "disable" rate limiting
- NEVER remove DDoS protections during an active attack

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Rate-based rule addition | Remove rule via `update-web-acl` |
| Resource scaling | Scale down after attack subsides |
| Geographic blocking | Remove geo rule via `update-web-acl` |

## Output Format

```yaml
root_cause: "ddos_detection — <specific_cause>"
evidence:
  - type: attack_details
    content: "<attack vectors, duration, mitigation>"
  - type: shield_subscription
    content: "<Standard or Advanced>"
  - type: traffic_metrics
    content: "<CloudWatch traffic data>"
severity: CRITICAL
mitigation:
  immediate: "Verify Shield mitigation is active. Add WAF rate-based rules for L7."
  long_term: "Enable Shield Advanced with health checks and proactive engagement"
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
