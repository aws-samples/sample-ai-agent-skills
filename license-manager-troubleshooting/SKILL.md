---
name: license-manager-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS License Manager
  problems by analyzing license configurations, rule violations, grants,
  host resource groups, dedicated hosts, Linux subscriptions, user
  subscriptions, cross-account sharing, organization settings, automated
  discovery, and following structured runbooks.
  Activate when: license configuration errors, rule violations, grant
  failures, host resource group issues, dedicated host management,
  Linux subscription discovery, user subscription issues, cross-account
  sharing, organization settings, automated discovery, or the user says
  something is wrong with License Manager.
compatibility: >
  Requires AWS CLI or SDK access with license-manager, iam, ec2,
  organizations, ram, cloudtrail, and cloudwatch permissions.
---

# AWS License Manager Diagnostics

## When to use

Any AWS License Manager investigation — license configurations, rule enforcement, grants, host resource groups, dedicated hosts, Linux subscriptions, user subscriptions, cross-account sharing, or automated discovery.

## Investigation workflow

### Step 1 — Collect and triage

```
aws license-manager list-license-configurations --query 'LicenseConfigurations[*].{Arn:LicenseConfigurationArn,Name:Name,Status:Status,Count:ConsumedLicenses}'
aws license-manager list-received-licenses --query 'Licenses[*].{Arn:LicenseArn,Name:LicenseName,Status:Status,Issuer:Issuer}'
aws license-manager list-received-grants --query 'Grants[*].{Arn:GrantArn,Name:GrantName,Status:Status}'
```

### Step 2 — Domain deep dive

```
aws license-manager get-license-configuration --license-configuration-arn <arn>
aws license-manager list-usage-for-license-configuration --license-configuration-arn <arn>
aws license-manager list-associations-for-license-configuration --license-configuration-arn <arn>
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=license-manager.amazonaws.com --max-results 20
aws license-manager list-resource-inventory --filters '[{"Name":"Platform","Values":["LINUX"],"Condition":"EQUALS"}]'
aws ec2 describe-hosts --query 'Hosts[*].{Id:HostId,State:State,Type:InstanceType,AutoPlacement:AutoPlacement}'
```

Read `references/guardrails.md` before concluding on any License Manager issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `license-manager list-license-configurations` | List license configs |
| `license-manager get-license-configuration` | Get config details |
| `license-manager list-received-licenses` | List received licenses |
| `license-manager list-received-grants` | List grants |
| `license-manager list-usage-for-license-configuration` | Check license usage |
| `license-manager list-resource-inventory` | Inventory resources |
| `ec2 describe-hosts` | Check dedicated hosts |

## Gotchas: AWS License Manager

- License configurations track license consumption but do NOT enforce limits by default. Enforcement requires enabling hard limits which can block resource launches.
- Grants are the mechanism for sharing licenses across accounts. A grant must be ACCEPTED by the recipient before it takes effect.
- Host resource groups manage dedicated host allocation. Auto-placement must be enabled for automatic instance placement.
- Linux subscription discovery requires the Systems Manager inventory plugin. It discovers RHEL, SUSE, and Ubuntu Pro subscriptions.
- User-based subscriptions (e.g., Microsoft Visual Studio) require Active Directory integration and specific VPC configuration.
- Cross-account license sharing uses AWS RAM (Resource Access Manager) under the hood. Organization sharing must be enabled.
- Automated discovery uses Systems Manager inventory to find software installations. It requires SSM Agent on managed instances.

## Anti-hallucination rules

1. Always cite specific license configuration ARNs, grant ARNs, or API responses as evidence.
2. License configurations and received licenses are different concepts. Configurations are self-managed; received licenses come from ISVs.
3. Grants must be accepted before they take effect. Never assume a created grant is active.
4. Hard limits block launches; soft limits only alert. Never conflate enforcement modes.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 12 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — License Config | A1-A2 | Configuration errors, rule violations |
| B — Grants | B1-B2 | Grant creation failures, grant acceptance |
| C — Dedicated Hosts | C1-C2 | Host resource groups, dedicated host management |
| D — Subscriptions | D1-D2 | Linux subscription discovery, user subscriptions |
| E — Cross-Account | E1-E2 | Cross-account sharing, organization settings |
| F — Discovery | F1 | Automated discovery |
| Z — Catch-All | Z1 | General troubleshooting |
