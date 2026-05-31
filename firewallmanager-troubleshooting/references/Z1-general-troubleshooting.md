---
title: "Z1 — General Firewall Manager Troubleshooting"
description: "Catch-all systematic investigation for FMS issues not covered by specific runbooks"
status: active
severity: MEDIUM
triggers:
  - "Firewall Manager issue"
  - "FMS error"
  - "FMS problem"
  - "policy not working"
  - "general troubleshooting"
owner: devops-agent
objective: "Systematically investigate FMS issues using a structured diagnostic approach"
context: "This runbook provides a general-purpose investigation framework for Firewall Manager issues that don't clearly match a specific runbook. It covers admin account verification, policy listing, compliance checks, and systematic elimination of common causes. Use this as a starting point when the issue category is unclear, then pivot to a specific runbook once the problem area is identified."
---

## Phase 1 — Triage

MUST:
- Verify FMS admin account: `aws fms get-admin-account`
- List all policies: `aws fms list-policies --query 'PolicyList[*].{Id:PolicyId,Name:PolicyName,Type:SecurityServiceType}'`
- Check Organizations status: `aws organizations describe-organization --query 'Organization.FeatureSet'`
- Check overall compliance: `aws fms list-compliance-status --policy-id <policy-id>`

SHOULD:
- Check CloudTrail for recent FMS events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=fms.amazonaws.com --max-results 20`
- List member accounts: `aws fms list-member-accounts`
- Verify FMS service-linked role: `aws iam get-role --role-name AWSServiceRoleForFMS`
- Check for recent policy changes

MAY:
- Check AWS Health Dashboard for FMS service issues
- Review application and security logs
- Check for SCPs affecting FMS operations

## Phase 2 — Remediate

MUST:
- Based on triage findings, pivot to the appropriate specific runbook:
  - Policy issues → A1 (creation), A2 (compliance), A3 (auto-remediation)
  - WAF issues → B1 (WAF policy), B2 (rule groups)
  - Shield issues → C1 (Shield policy), C2 (DDoS protection)
  - Security group issues → D1 (SG policy), D2 (audit policy)
  - Network Firewall → E1 (NF policy), E2 (deployment)
  - Organization → F1 (admin account), F2 (member compliance)
  - DNS Firewall → G1 (DNS policy), G2 (domain lists)
- Document findings and root cause

SHOULD:
- Check for multiple concurrent issues
- Review recent changes that may have caused the issue
- Verify the fix resolves the issue without introducing new problems

MAY:
- Set up comprehensive monitoring to prevent recurrence
- Create a post-incident report for significant issues

## Common Issues

- symptoms: "FMS operations failing with access denied"
  diagnosis: "Not using the FMS admin account or missing permissions."
  resolution: "Verify FMS admin account. Check IAM permissions."

- symptoms: "Policies not applying to resources"
  diagnosis: "Policy scope misconfiguration or propagation delay."
  resolution: "Check policy scope (tags, accounts, OUs). Wait for propagation."

- symptoms: "Compliance status not updating"
  diagnosis: "Evaluation cycle not completed or service issue."
  resolution: "Wait for evaluation cycle. Check AWS Health Dashboard."

## Output Format

```yaml
root_cause: "general — <specific_cause>"
evidence:
  - type: admin_account
    content: "<FMS admin account status>"
  - type: policies
    content: "<policy list and status>"
  - type: compliance
    content: "<overall compliance status>"
severity: MEDIUM
general_analysis:
  admin_account: "configured | not configured"
  organizations: "all features | consolidated billing only"
  policies: "active | issues found"
  specific_runbook: "<recommended runbook ID>"
mitigation:
  immediate: "Follow the identified specific runbook"
  long_term: "Implement comprehensive FMS monitoring and alerting"
```

---

## Safety Ratings

| Aspect | Rating |
|--------|--------|
| Service Impact | CRITICAL � Firewall Manager enforces security policies across the entire organization |
| Data Risk | HIGH � Policy changes can expose resources or block legitimate traffic |
| Recovery Complexity | HIGH � Organization-wide policy rollbacks require coordinated multi-account changes |
| Blast Radius | CRITICAL � Policy changes propagate to all member accounts in scope |

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

## Phase 3 � Rollback

- If policy change blocked legitimate traffic: revert policy configuration and verify traffic flow restoration
- If auto-remediation caused issues: disable auto-remediation, manually restore affected resources, then fix policy
- If WAF rule group update broke applications: revert to previous rule group version and clear WAF logs
- If security group policy caused compliance failures: restore previous policy scope and re-evaluate compliance
- If Network Firewall deployment failed: revert to previous firewall policy and verify VPC traffic inspection
- Document all rollback actions with policy ID, affected accounts, and compliance status before/after
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
