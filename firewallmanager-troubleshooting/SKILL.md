---
name: firewallmanager-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Firewall Manager
  problems by analyzing policy creation, policy compliance, auto-remediation,
  WAF policy issues, rule group management, Shield Advanced policy, DDoS
  protection, security group policies, audit policies, Network Firewall
  policy deployment, organization admin accounts, member compliance, DNS
  Firewall policies, domain lists, and following structured runbooks.
  Activate when: policy creation failures, non-compliant resources, auto-remediation
  not working, WAF rule group errors, Shield Advanced policy issues, DDoS
  protection gaps, security group policy violations, audit policy findings,
  Network Firewall deployment failures, admin account configuration issues,
  member account compliance problems, DNS Firewall policy errors, domain list
  issues, or the user says something is wrong with Firewall Manager without
  naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with FMS, WAFv2, Shield, EC2, Network Firewall,
  Route 53 Resolver, Organizations, IAM, CloudTrail, and CloudWatch permissions.
  Firewall Manager requires AWS Organizations with all features enabled.
---

# AWS Firewall Manager Diagnostics

## When to use

Any AWS Firewall Manager investigation where the console alone is insufficient — policy creation failures, compliance issues, auto-remediation problems, WAF rule group management, Shield Advanced protection, security group policies, Network Firewall deployment, organization-wide policy enforcement, DNS Firewall configuration, or cross-account policy compliance.

## Investigation workflow

### Step 1 — Collect and triage

```
aws fms list-policies --query 'PolicyList[*].{Id:PolicyId,Name:PolicyName,Type:SecurityServiceType,RemediationEnabled:RemediationEnabled}'
version: "1.0.0"
last_updated: "2025-04-12"
aws fms get-admin-account
aws fms get-compliance-detail --policy-id <policy-id> --member-account <account-id>
aws organizations describe-organization --query 'Organization.{Id:Id,MasterAccountId:MasterAccountId,FeatureSet:FeatureSet}'
```

### Step 2 — Domain deep dive

```
aws fms get-policy --policy-id <policy-id>
aws fms list-compliance-status --policy-id <policy-id> --query 'PolicyComplianceStatusList[*].{Account:MemberAccount,Status:ComplianceStatus}'
aws fms get-violation-details --policy-id <policy-id> --member-account <account-id> --resource-id <resource-id> --resource-type <type>
aws fms list-member-accounts
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=fms.amazonaws.com --max-results 20
aws wafv2 list-rule-groups --scope REGIONAL
aws shield describe-subscription
aws network-firewall list-firewall-policies
```

Read `references/firewallmanager-guardrails.md` before concluding on any Firewall Manager issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `fms list-policies` | List all Firewall Manager policies |
| `fms get-policy` | Get detailed policy configuration |
| `fms get-compliance-detail` | Check resource compliance for a policy |
| `fms list-compliance-status` | Check compliance across member accounts |
| `fms get-violation-details` | Get specific violation details |
| `fms get-admin-account` | Verify Firewall Manager administrator |
| `fms list-member-accounts` | List organization member accounts |
| `wafv2 list-rule-groups` | List WAF rule groups |
| `shield describe-subscription` | Check Shield Advanced subscription |
| `network-firewall list-firewall-policies` | List Network Firewall policies |

## Gotchas: AWS Firewall Manager

- Firewall Manager requires AWS Organizations with ALL features enabled. It will not work with consolidated billing only. The management account or a delegated administrator must configure FMS.
- FMS administrator account is NOT automatically the management account. You must explicitly designate an FMS administrator account. Only one FMS administrator account is allowed per organization.
- Auto-remediation does NOT fix all violations instantly. Some resources require manual intervention. Auto-remediation creates or modifies resources but may not delete non-compliant configurations. Check remediation status separately.
- WAF policies in FMS create WAF web ACLs in member accounts. FMS-managed web ACLs have a specific naming convention. Manual changes to FMS-managed web ACLs are overwritten by FMS.
- Shield Advanced policies require an active Shield Advanced subscription. The subscription is per account and costs $3,000/month. FMS can enable Shield Advanced on member accounts but the subscription cost applies per account.
- Security group policies can be AUDIT or USAGE type. Audit policies report non-compliant security groups. Usage policies can auto-remediate by removing unused or redundant security groups. These are fundamentally different policy types.
- Network Firewall policies deploy firewall endpoints in member account VPCs. This creates ENIs and incurs costs. Deployment can take several minutes per VPC.
- DNS Firewall policies use domain lists to filter DNS queries. FMS distributes DNS Firewall rule groups to member accounts. Domain lists can be AWS-managed or custom.
- Policy scope uses resource tags, accounts, and OUs for targeting. Incorrect scope configuration is the most common cause of policies not applying to expected resources.

## Anti-hallucination rules

1. Always cite specific policy IDs, compliance status, or API responses as evidence.
2. FMS requires Organizations with ALL features. Never assume consolidated billing is sufficient.
3. Only ONE FMS administrator account per organization. Never suggest multiple admins.
4. Auto-remediation has limitations. Never claim it fixes all violations automatically.
5. Shield Advanced costs $3,000/month per account. Never omit cost implications.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 20 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Policy | A1-A3 | Policy creation, policy compliance, auto-remediation |
| B — WAF | B1-B2 | WAF policy issues, rule group management |
| C — Shield | C1-C2 | Shield Advanced policy, DDoS protection |
| D — Security Groups | D1-D2 | Security group policy, audit policy |
| E — Network Firewall | E1-E2 | Network Firewall policy, deployment |
| F — Organization | F1-F2 | Admin account, member compliance |
| G — DNS Firewall | G1-G2 | DNS Firewall policy, domain lists |
| Z — Catch-All | Z1 | General troubleshooting |
