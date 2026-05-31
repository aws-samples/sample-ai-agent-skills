---
title: "C2 â€” DDoS Protection Issues"
description: "Diagnose DDoS protection gaps and Shield Advanced effectiveness"
status: active
severity: CRITICAL
triggers:
  - "DDoS attack"
  - "DDoS protection not working"
  - "Shield not detecting attack"
  - "DDoS mitigation"
owner: devops-agent
objective: "Identify DDoS protection gaps and ensure Shield Advanced is properly configured"
context: "Shield Advanced provides enhanced DDoS detection and mitigation for protected resources. Effectiveness depends on proper resource protection, health check configuration, and proactive engagement setup. Shield Standard provides basic protection automatically but has limited visibility and no DRT access."
---

## Phase 1 â€” Triage

MUST:
- Check Shield subscription: `aws shield describe-subscription`
- List current protections: `aws shield list-protections`
- Check for recent attacks: `aws shield list-attacks --start-time <start> --end-time <end>`
- Describe specific attack: `aws shield describe-attack --attack-id <attack-id>`

SHOULD:
- Check health check associations: `aws shield describe-protection --protection-id <id>`
- Verify proactive engagement status: `aws shield describe-emergency-contact-settings`
- Check DRT access: `aws shield describe-drt-access`
- Review CloudWatch DDoS metrics

MAY:
- Check Shield Advanced cost protection claims
- Review application-layer attack patterns

## Phase 2 â€” Remediate

MUST:
- Ensure all internet-facing resources are protected by Shield Advanced
- Associate Route 53 health checks with protected resources
- Configure emergency contacts for DRT engagement

SHOULD:
- Enable proactive engagement for automatic DRT response
- Grant DRT access to WAF and Shield APIs
- Configure application-layer automatic mitigation

MAY:
- Create runbooks for DDoS incident response
- Set up CloudWatch alarms for DDoS detection metrics

## Common Issues

- symptoms: "Shield not detecting application-layer attacks"
  diagnosis: "Health checks not configured or application-layer mitigation not enabled."
  resolution: "Associate health checks with protected resources. Enable application-layer automatic mitigation."

- symptoms: "DRT not responding during attack"
  diagnosis: "Proactive engagement not enabled or emergency contacts not configured."
  resolution: "Enable proactive engagement. Configure emergency contacts."

## Output Format

```yaml
root_cause: "ddos_protection â€” <specific_cause>"
evidence:
  - type: protections
    content: "<protected resources list>"
  - type: attack_details
    content: "<attack detection and mitigation details>"
severity: CRITICAL
mitigation:
  immediate: "Ensure all resources are protected and health checks configured"
  long_term: "Enable proactive engagement and application-layer mitigation"
```

---

## Safety Ratings

| Aspect | Rating |
|--------|--------|
| Service Impact | CRITICAL — Firewall Manager enforces security policies across the entire organization |
| Data Risk | HIGH — Policy changes can expose resources or block legitimate traffic |
| Recovery Complexity | HIGH — Organization-wide policy rollbacks require coordinated multi-account changes |
| Blast Radius | CRITICAL — Policy changes propagate to all member accounts in scope |

## Escalation Conditions

- Security policy non-compliance detected across multiple member accounts
- Auto-remediation actions causing unintended resource modifications
- WAF rule group changes blocking legitimate production traffic
- Shield Advanced policy gaps leaving resources unprotected during active DDoS
- Network Firewall policy deployment failures leaving VPCs without inspection
- DNS Firewall policy blocking legitimate domain resolutions

## Data Sensitivity

- Security policies reveal organizational security posture and architecture
- Member account compliance data exposes security gaps across the organization
- WAF rule configurations may contain IP allowlists and rate limiting thresholds
- Admin account credentials grant organization-wide security policy control
- Audit policy results reveal security group configurations across all accounts

## PROHIBITED Actions

- NEVER suggest removing the Firewall Manager administrator account without a replacement
- NEVER disable auto-remediation on policies protecting against active threats
- NEVER modify WAF rule groups without testing impact on legitimate traffic patterns
- NEVER delete Shield Advanced policies during an active DDoS event
- NEVER change security group audit policies without notifying affected account owners
- NEVER deploy Network Firewall policy changes to all accounts simultaneously without staged rollout

## Phase 3 — Rollback

- If policy change blocked legitimate traffic: revert policy configuration and verify traffic flow restoration
- If auto-remediation caused issues: disable auto-remediation, manually restore affected resources, then fix policy
- If WAF rule group update broke applications: revert to previous rule group version and clear WAF logs
- If security group policy caused compliance failures: restore previous policy scope and re-evaluate compliance
- If Network Firewall deployment failed: revert to previous firewall policy and verify VPC traffic inspection
- Document all rollback actions with policy ID, affected accounts, and compliance status before/after
## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN â€” read-only"
  - "Phase 2 configuration changes: YELLOW â€” state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED â€” destructive or irreversible"

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
