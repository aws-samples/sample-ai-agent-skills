---
name: control-tower-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Control Tower
  problems by analyzing landing zone state, organizational unit
  configuration, account provisioning, guardrail compliance, and
  following structured runbooks. Activate when: landing zone setup
  or update fails, account provisioning via Account Factory fails,
  guardrails (controls) fail to enable or show non-compliant,
  drift is detected on OUs or accounts, Service Catalog provisioned
  products are stuck, SSO/IAM Identity Center access issues,
  CloudTrail or Config aggregation is broken, nested OU issues,
  account enrollment or import fails, customizations pipeline
  errors, or the user says something is wrong with Control Tower
  without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with Control Tower, Organizations,
  Service Catalog, CloudFormation, Config, CloudTrail, IAM Identity
  Center (SSO), and STS permissions in the management account.
---

# Control Tower Diagnostics

## When to use

Any Control Tower investigation where the console alone is insufficient — landing zone failures, account provisioning errors, guardrail compliance violations, drift detection, SCP conflicts, SSO access issues, or customization pipeline failures.

## Investigation workflow

### Step 1 — Collect and triage

```
# Get landing zone status
aws controltower list-landing-zones
aws controltower get-landing-zone --landing-zone-identifier <arn>

# Check for drift
aws controltower list-landing-zone-operations --filter statuses=FAILED

# List enabled controls (guardrails)
aws controltower list-enabled-controls --target-identifier <ou-arn>

# Check account factory provisioned products
aws servicecatalog search-provisioned-products --access-level-filter Key=Account,Value=self

# Check Organizations state
aws organizations describe-organization
aws organizations list-roots
aws organizations list-organizational-units-for-parent --parent-id <root-id>
```

Triage returns:
- Landing zone version and status
- Failed operations and drift indicators
- Enabled controls and compliance state
- Account provisioning status
- Organizational structure

If landing zone status is not ACTIVE, that IS the root cause domain. Don't chase downstream symptoms.

### Step 2 — Domain deep dive (only if needed)

```
# Control (guardrail) details
aws controltower get-enabled-control --enabled-control-identifier <arn>
aws controltower list-enabled-controls --target-identifier <ou-arn>

# CloudFormation stacks (Control Tower uses StackSets)
aws cloudformation list-stack-sets --status ACTIVE
aws cloudformation describe-stack-set --stack-set-name AWSControlTowerBP-*
aws cloudformation list-stack-instances --stack-set-name <name> --filters Name=DETAILED_STATUS,Values=FAILED

# Config compliance
aws configservice describe-compliance-by-config-rule --compliance-types NON_COMPLIANT

# SCPs
aws organizations list-policies-for-target --target-id <ou-id> --filter SERVICE_CONTROL_POLICY
aws organizations describe-policy --policy-id <id>

# SSO / IAM Identity Center
aws sso-admin list-instances
aws sso-admin list-permission-sets --instance-arn <arn>
```

Read `references/control-tower-guardrails.md` before concluding on any Control Tower issue.

### Step 3 — Detailed path (low-confidence cases only)

```
# CloudTrail for API-level events
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateManagedAccount

# CloudFormation stack events for failed provisioning
aws cloudformation describe-stack-events --stack-name <stack>

# Service Catalog provisioned product details
aws servicecatalog describe-provisioned-product --id <id>
aws servicecatalog describe-record --id <record-id>
```

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `get-landing-zone` | Landing zone version, status, drift state |
| `list-landing-zone-operations` | Recent LZ operations and failures |
| `list-enabled-controls` | Controls (guardrails) enabled on an OU |
| `get-enabled-control` | Specific control status and compliance |
| `list-organizational-units-for-parent` | OU hierarchy |
| `search-provisioned-products` | Account Factory provisioned products |
| `describe-provisioned-product` | Specific account provisioning status |
| `list-stack-sets` / `list-stack-instances` | StackSet deployment status |
| `describe-compliance-by-config-rule` | Config rule compliance |
| `list-policies-for-target` | SCPs applied to OU/account |
| `list-instances` (sso-admin) | IAM Identity Center configuration |
| `lookup-events` (cloudtrail) | API-level audit trail |

## Gotchas: Control Tower

These are the mistakes commonly made during Control Tower troubleshooting.

- Control Tower manages resources via CloudFormation StackSets. Manual changes to these stacks cause DRIFT. Never modify CT-managed stacks directly.
- Landing zone updates are NOT backward compatible. Always check the release notes before updating. Some updates require re-registering OUs.
- Account Factory uses Service Catalog under the hood. Provisioned product failures are often CloudFormation failures in the target account.
- Guardrails are now called "controls." There are three types: preventive (SCPs), detective (Config rules), and proactive (CloudFormation hooks). They behave very differently.
- Preventive controls (SCPs) block actions at the API level. They cannot be overridden by IAM policies. They apply to all principals in the account EXCEPT the management account.
- Detective controls (Config rules) only DETECT non-compliance. They do NOT prevent the action. Remediation is separate.
- The management account is NOT governed by SCPs. Never assume SCPs protect the management account.
- Control Tower creates a CloudTrail organization trail. Creating additional trails may cause duplicate log delivery and increased costs.
- The Audit account and Log Archive account are special. Moving or modifying them causes drift.
- Nested OUs are supported but controls are NOT inherited by child OUs. You must enable controls on each OU individually.
- Account email addresses must be globally unique across ALL AWS accounts. Reusing emails causes provisioning failures.
- Control Tower uses specific IAM roles (AWSControlTowerExecution, AWSControlTowerAdmin, etc.). Modifying or deleting these causes drift and operational failures.

### Control types comparison

| Type | Mechanism | Behavior | Enforcement |
|------|-----------|----------|-------------|
| Preventive | SCP | Blocks API calls | Proactive — prevents action |
| Detective | Config Rule | Evaluates compliance | Reactive — detects after action |
| Proactive | CF Hook | Validates CF templates | Proactive — blocks non-compliant CF deploys |

### Landing zone states

| State | Meaning |
|-------|---------|
| ACTIVE | Landing zone is operational |
| PROCESSING | Operation in progress (setup/update) |
| FAILED | Setup or update failed |

### Account provisioning states

| State | Meaning |
|-------|---------|
| AVAILABLE | Account provisioned successfully |
| UNDER_CHANGE | Provisioning or update in progress |
| PLAN_IN_PROGRESS | Change plan being calculated |
| ERROR | Provisioning failed |
| TAINTED | Provisioned but with errors |

## Gotchas: Drift

- Drift occurs when CT-managed resources are modified outside of Control Tower.
- Common drift causes: manual SCP changes, OU moves, role modifications, stack changes.
- Drift must be resolved before you can update the landing zone or enable new controls.
- Re-registering an OU resets its CT-managed resources to the expected state.
- Account-level drift (moved account) requires moving the account back or re-registering.

## Anti-hallucination rules

1. Always cite specific API responses, CloudFormation events, or Config compliance results as evidence.
2. SCPs do NOT apply to the management account. Never claim SCPs protect the management account.
3. Controls are NOT inherited by child OUs. Never claim enabling a control on a parent OU protects child OUs.
4. Detective controls detect but do NOT prevent. Never claim a detective control blocks an action.
5. Manual changes to CT-managed resources cause drift. Never recommend directly editing CT-managed CloudFormation stacks or SCPs.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 34 runbooks

Runbooks are organized by failure domain. Use the appropriate runbook based on the symptom category.

| Category | IDs | Covers |
|----------|-----|--------|
| A — Landing Zone | A1-A4 | Setup failure, update failure, version mismatch, region configuration |
| B — Account Provisioning | B1-B4 | Account Factory failure, email conflicts, enrollment failure, account import |
| C — Controls (Guardrails) | C1-C4 | Enable failure, non-compliant resources, SCP conflicts, proactive control issues |
| D — Drift | D1-D4 | Landing zone drift, OU drift, account drift, SCP drift |
| E — Organizational Units | E1-E3 | OU registration failure, nested OU issues, OU re-registration |
| F — IAM Identity Center (SSO) | F1-F3 | SSO configuration issues, permission set failures, access problems |
| G — Logging & Monitoring | G1-G3 | CloudTrail issues, Config aggregation, log archive problems |
| H — Customizations | H1-H3 | CfCT pipeline failures, lifecycle event issues, custom SCP conflicts |
| I — StackSet Operations | I1-I3 | StackSet failures, stack instance drift, deployment timeouts |
| Z — Catch-All | Z1 | General troubleshooting |
