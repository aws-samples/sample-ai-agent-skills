---
title: "G1 — Observability Access Manager"
description: "Diagnose CloudWatch Observability Access Manager (OAM) setup issues"
status: active
severity: HIGH
triggers:
  - "OAM"
  - "observability access"
  - "cross-account sink"
  - "cross-account link"
  - "monitoring account"
  - "source account"
owner: devops-agent
objective: "Identify and fix OAM sink/link configuration issues for cross-account observability"
context: "CloudWatch Observability Access Manager (OAM) enables cross-account observability. The monitoring account creates a sink, and source accounts create links to that sink. Links specify which resource types to share: metrics, logs, traces, and Application Signals. Both sink and link must be configured correctly."
---

## Phase 1 — Triage

MUST:
- In monitoring account, check sink exists: `aws oam list-sinks`
- In monitoring account, check sink policy: `aws oam get-sink-policy --sink-identifier <sink-arn>`
- In source account, check link exists: `aws oam list-links`
- Verify link resource types include the needed types (AWS::CloudWatch::Metric, AWS::Logs::LogGroup, AWS::XRay::Trace)

SHOULD:
- Verify sink policy allows the source account(s)
- Check link status is active
- Verify IAM permissions for OAM operations in both accounts

MAY:
- Check CloudTrail for OAM API errors in both accounts
- Verify the sink and link are in the same region (OAM is regional)

## Phase 2 — Remediate

MUST:
- Create sink in monitoring account if missing: `aws oam create-sink --name <name>`
- Update sink policy to allow source accounts
- Create link in source account if missing: `aws oam create-link --sink-identifier <sink-arn> --resource-types <types> --label-template <template>`

SHOULD:
- Include all needed resource types in the link
- Use organization ID in sink policy for org-wide access
- Set up links in all regions where resources exist

MAY:
- Automate link creation via CloudFormation StackSets
- Document the OAM architecture for the operations team

## Common Issues

- symptoms: "Cannot see metrics from source account in monitoring account"
  diagnosis: "OAM link does not include AWS::CloudWatch::Metric in resource types."
  resolution: "Update the link to include AWS::CloudWatch::Metric."

- symptoms: "Link creation fails with access denied"
  diagnosis: "Sink policy does not allow the source account."
  resolution: "Update sink policy to include the source account ID or organization ID."

- symptoms: "OAM configured but data not visible"
  diagnosis: "Sink and link are in different regions. OAM is regional."
  resolution: "Create sink and link in the same region. Repeat for each region."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Create sink in monitoring account | GREEN | New resource creation |
| Update sink policy | YELLOW | Cross-account access policy change |
| Create link in source account | GREEN | Establishes cross-account sharing |
| Include resource types in link | GREEN | Additive configuration |

## Escalation Conditions

- Cross-account observability broken for production monitoring — escalate to operations team
- Sink policy changes affecting multiple source accounts — escalate to monitoring account administrator
- OAM link creation across organization — escalate to management account administrator
- Cross-account access policy changes — escalate to security team

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws oam list-sinks` | MEDIUM | Exposes sink configuration |
| `aws oam get-sink-policy` | HIGH | Exposes cross-account access policy with account IDs |
| `aws oam list-links` | MEDIUM | Exposes cross-account sharing configuration |

## Prohibited Actions

- **NEVER** suggest deleting log groups with retention data without confirming data is no longer needed
- **NEVER** suggest disabling alarms in production
- **NEVER** suggest removing metric filters that feed production alarms
- **NEVER** grant sink access to untrusted accounts

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Created sink | Delete: `aws oam delete-sink --identifier <sink-arn>` |
| Updated sink policy | Restore previous sink policy |
| Created link | Delete: `aws oam delete-link --identifier <link-arn>` |

## Output Format

```yaml
root_cause: "oam_setup — <specific_cause>"
evidence:
  - type: sink_config
    content: "<sink and sink policy>"
  - type: link_config
    content: "<link and resource types>"
severity: HIGH
mitigation:
  immediate: "Fix sink policy, create link, or add resource types"
  long_term: "Automate OAM setup via CloudFormation StackSets"
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
  - "NEVER suggest deleting alarms without review"
  - "NEVER reduce log retention below compliance requirements"
  - "NEVER disable detailed monitoring in production"
