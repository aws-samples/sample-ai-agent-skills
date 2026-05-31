---
title: "A1 â€” Policy Creation Failures"
description: "Diagnose why an AWS Firewall Manager policy creation is failing"
status: active
severity: HIGH
triggers:
  - "policy creation failed"
  - "cannot create FMS policy"
  - "PutPolicy error"
  - "Firewall Manager policy error"
owner: devops-agent
objective: "Identify why FMS policy creation is failing and resolve the issue"
context: "FMS policy creation can fail due to missing Organizations setup, incorrect admin account, invalid policy configuration, missing service-linked role, resource type not supported, or IAM permission issues. The FMS administrator account must be properly designated."
---

## Phase 1 â€” Triage

MUST:
- Verify FMS admin account is set: `aws fms get-admin-account`
- Verify Organizations is enabled with all features: `aws organizations describe-organization --query 'Organization.FeatureSet'`
- Check IAM permissions for FMS: `aws iam simulate-principal-policy --policy-source-arn <caller-arn> --action-names fms:PutPolicy`
- Check if the FMS service-linked role exists: `aws iam get-role --role-name AWSServiceRoleForFMS`

SHOULD:
- Verify the policy configuration is valid for the security service type
- Check CloudTrail for the creation failure event: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=PutPolicy --max-results 10`
- Verify resource tags in policy scope are correct
- Check if the target accounts/OUs exist in the organization

MAY:
- Verify the security service (WAF, Shield, Network Firewall) is available in the target region
- Check FMS policy quotas: `aws service-quotas get-service-quota --service-code fms --quota-code L-0B28E140`

## Phase 2 â€” Remediate

MUST:
- Ensure Organizations has all features enabled
- Designate an FMS admin account if not set: `aws fms associate-admin-account --admin-account <account-id>`
- Create the FMS service-linked role if missing
- Fix policy configuration based on the security service type

SHOULD:
- Use correct resource type for the policy (e.g., AWS::ElasticLoadBalancingV2::LoadBalancer for WAF)
- Verify policy scope includes the intended accounts and resource tags
- Test with a single account before applying organization-wide

MAY:
- Request quota increase if at policy limit
- Enable auto-remediation after verifying policy configuration

## Common Issues

- symptoms: "AccessDeniedException when creating policy"
  diagnosis: "Caller is not the FMS administrator or lacks permissions."
  resolution: "Verify FMS admin account. Ensure caller has fms:PutPolicy permission."

- symptoms: "Organizations not enabled error"
  diagnosis: "AWS Organizations is not enabled or only has consolidated billing."
  resolution: "Enable Organizations with all features. Designate FMS admin account."

- symptoms: "Invalid policy configuration"
  diagnosis: "Policy security service configuration is malformed."
  resolution: "Verify the SecurityServicePolicyData matches the policy type (WAF, Shield, SG, etc.)."

## Output Format

```yaml
root_cause: "policy_creation_failure â€” <specific_cause>"
evidence:
  - type: admin_account
    content: "<FMS admin account status>"
  - type: organizations
    content: "<Organizations feature set>"
  - type: iam_permissions
    content: "<IAM policy evaluation>"
severity: HIGH
mitigation:
  immediate: "Fix the specific creation blocker"
  long_term: "Automate FMS policy deployment with IaC and validate prerequisites"
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
