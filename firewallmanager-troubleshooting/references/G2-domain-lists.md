---
title: "G2 â€” Domain List Issues"
description: "Diagnose DNS Firewall domain list configuration problems"
status: active
severity: MEDIUM
triggers:
  - "domain list error"
  - "domain not blocked"
  - "domain list import failed"
  - "managed domain list"
owner: devops-agent
objective: "Identify and resolve DNS Firewall domain list issues"
context: "DNS Firewall uses domain lists to define which domains to allow, block, or alert on. Domain lists can be AWS-managed (e.g., malware domains, botnet C&C) or custom. Custom domain lists support bulk import from S3. Domain list updates propagate to all associated rule groups."
---

## Phase 1 â€” Triage

MUST:
- List domain lists: `aws route53resolver list-firewall-domain-lists`
- Get domain list details: `aws route53resolver get-firewall-domain-list --firewall-domain-list-id <id>`
- Check domain list contents: `aws route53resolver list-firewall-domains --firewall-domain-list-id <id>`
- Verify domain list is referenced in rule groups: `aws route53resolver list-firewall-rules --firewall-rule-group-id <id>`

SHOULD:
- Check domain list status (COMPLETE, IMPORTING, etc.)
- Verify domain format (must be valid domain names)
- Check for wildcard domain entries (*.example.com)
- Verify AWS-managed domain list availability

MAY:
- Check S3 source for bulk import domain lists
- Review domain list update history

## Phase 2 â€” Remediate

MUST:
- Ensure domain list status is COMPLETE
- Verify domains are in correct format
- Check that domain list is associated with active rule groups

SHOULD:
- Use wildcard entries for broad domain blocking (*.malicious.com)
- Test domain blocking with nslookup/dig before deploying
- Keep domain lists updated with latest threat intelligence

MAY:
- Automate domain list updates from threat feeds
- Create separate domain lists for different categories

## Common Issues

- symptoms: "Domain list import failed"
  diagnosis: "S3 file format incorrect or permissions issue."
  resolution: "Verify S3 file has one domain per line. Check S3 bucket permissions."

- symptoms: "AWS-managed domain list not available"
  diagnosis: "Managed domain lists may not be available in all regions."
  resolution: "Check regional availability. Use custom domain lists as alternative."

## Output Format

```yaml
root_cause: "domain_list â€” <specific_cause>"
evidence:
  - type: domain_list_status
    content: "<domain list status and contents>"
  - type: rule_association
    content: "<rule group associations>"
severity: MEDIUM
mitigation:
  immediate: "Fix domain list configuration"
  long_term: "Automate domain list management and threat feed integration"
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
