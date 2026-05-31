---
title: "C1 — Campaign Deployment Issues"
description: "Diagnose campaign creation and deployment failures"
status: active
severity: HIGH
triggers: ["campaign creation failed", "campaign not active", "CreateCampaign error"]
owner: devops-agent
objective: "Identify and resolve campaign deployment issues"
context: "Campaigns provide real-time inference endpoints. Require ACTIVE solution version. Billed per TPS while active. MinProvisionedTPS determines capacity."
---

## Phase 1 — Triage

MUST:
- Check campaign: `aws personalize describe-campaign --campaign-arn <arn>`
- Verify solution version is ACTIVE
- Review campaign status and failureReason
- Check MinProvisionedTPS setting

SHOULD:
- Verify IAM permissions for CreateCampaign
- Check for campaign quota limits
- Test with GetRecommendations after deployment

MAY:
- Monitor campaign utilization metrics
- Check CloudTrail for campaign events

## Phase 2 — Remediate

MUST:
- Ensure solution version is ACTIVE before campaign creation
- Set appropriate MinProvisionedTPS
- Wait for campaign ACTIVE status

SHOULD:
- Start with MinProvisionedTPS=1 for testing
- Monitor and scale TPS based on traffic

MAY:
- Implement campaign lifecycle automation

## Common Issues

- symptoms: "Campaign stuck in CREATE PENDING"
  diagnosis: "Solution version not yet ACTIVE."
  resolution: "Wait for solution version to reach ACTIVE status."

- symptoms: "Campaign creation fails"
  diagnosis: "Solution version failed or quota exceeded."
  resolution: "Check solution version status. Request quota increase if needed."

## Output Format

```yaml
root_cause: "campaign_deployment — <specific_cause>"
evidence:
  - type: campaign_status
    content: "<campaign details>"
severity: HIGH
mitigation:
  immediate: "Fix solution version or campaign configuration"
  long_term: "Implement campaign deployment automation"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: MEDIUM — campaigns serve personalized recommendations
- Service Disruption Risk: HIGH — deployment failures affect recommendation availability
- Compliance Impact: MEDIUM — campaign deployment affects user experience and data usage

## Escalation Conditions
- Campaign deployment fails with provisioning errors
- Campaign becomes unavailable after deployment
- Campaign latency exceeds acceptable thresholds
- Campaign costs exceed budget due to provisioned throughput
- Campaign serves incorrect or biased recommendations

## Data Sensitivity
- Campaign configurations — INTERNAL
- Recommendation outputs — CONFIDENTIAL
- User interaction patterns — RESTRICTED
- Throughput metrics — INTERNAL
- Never expose individual user recommendations

## Prohibited Actions
- Do NOT deploy campaigns without model quality validation
- Do NOT set excessive provisioned throughput without cost approval
- Do NOT deploy campaigns serving biased recommendations
- Do NOT expose campaign endpoints without authentication
- Do NOT ignore campaign health degradation
- Do NOT deploy campaigns without monitoring and alerting

## Phase 3 — Rollback
- Roll back to previous campaign version
- Restore previous solution version in campaign
- Revert throughput configuration changes
- Verify recommendation quality after rollback
- Restore monitoring configurations
- Document deployment issues and resolution steps

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
