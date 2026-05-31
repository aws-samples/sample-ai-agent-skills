---
title: "Z1 — General Troubleshooting"
description: "General GameLift troubleshooting for issues not covered by specific runbooks"
status: active
severity: MEDIUM
triggers: ["GameLift error", "GameLift issue", "GameLift not working", "GameLift problem"]
owner: devops-agent
objective: "Diagnose and resolve general GameLift issues using systematic investigation"
context: "Catch-all runbook for GameLift issues. Start here when the specific problem category is unclear."
---

## Phase 1 — Triage

MUST:
- Identify the symptom category: fleet, session, build, server, matchmaking, queue, or Spot
- Check fleet status: `aws gamelift describe-fleet-attributes --query 'FleetAttributes[*].{Id:FleetId,Name:Name,Status:Status,Type:FleetType}'`
- Check recent events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=gamelift.amazonaws.com --max-results 20`

SHOULD:
- Check game session queues: `aws gamelift describe-game-session-queues`
- Verify IAM permissions for GameLift operations
- Check AWS Health Dashboard for GameLift service issues

MAY:
- Review CloudWatch metrics for anomalies
- Compare behavior across different fleets or regions

## Phase 2 — Remediate

MUST:
- Route to specific runbook once symptom category is identified:
  - Fleet creation → A1-fleet-creation-failures
  - Scaling → A2-fleet-scaling-issues
  - Session placement → B1-game-session-placement
  - Matchmaking → B2-matchmaking-failures
  - Build upload → C1-build-upload-errors
  - Script deployment → C2-script-deployment
  - Server crashes → D1-server-process-crashes
  - Health checks → D2-health-check-failures
  - Realtime Servers → E1-realtime-servers
  - FlexMatch → E2-flexmatch-configuration
  - Queues → F1-queue-configuration
  - Multi-region → F2-multi-region-fleets
  - Spot interruptions → G1-spot-fleet-interruptions

SHOULD:
- Document the issue for pattern recognition
- Check if issue is reproducible

MAY:
- Open AWS Support case if issue persists after runbook remediation
- Check AWS forums and re:Post for similar reported issues

## Safety Ratings

```
safety_ratings:
  - "describe-fleet-attributes / describe-game-session-queues: GREEN — read-only enumeration"
  - "cloudtrail lookup-events: GREEN — read-only event history"
  - "describe-* APIs: GREEN — read-only inspection"
  - "delete-fleet: RED — terminates all instances and game sessions"
```

## Escalation Conditions

- Issue persists after following specific runbook remediation steps
- Multiple fleets or regions affected simultaneously
- Issue involves AWS service degradation or outage
- Player-facing impact requiring immediate attention

## Data Sensitivity

- `aws gamelift describe-fleet-attributes`: LOW — fleet metadata
- `aws gamelift describe-game-session-queues`: LOW — queue configuration
- `aws cloudtrail lookup-events`: MEDIUM — user activity records
- Game session data: MEDIUM — may contain player information

## Prohibited Actions

- NEVER delete fleets with active game sessions without player notification
- NEVER escalate to AWS Support without first following relevant runbook
- NEVER assume the issue is GameLift-specific without checking IAM and network
- NEVER share game session data containing player information externally

## Phase 3 — Rollback

1. If any remediation step made the issue worse: revert the specific change
2. If fleet was deleted: create new fleet with same configuration
3. If queue was modified: restore original queue configuration
4. If matchmaking was changed: restore original matchmaking configuration
5. Document all changes made during troubleshooting

## Common Issues

- symptoms: "GameLift API calls return AccessDeniedException"
  diagnosis: "IAM user/role lacks gamelift:* permissions."
  resolution: "Attach AmazonGameLiftFullAccess managed policy or add specific gamelift:* permissions to the IAM role."

## Output Format

```yaml
root_cause: "general — <specific_cause>"
evidence:
  - type: general_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Apply specific runbook remediation"
  long_term: "Document issue pattern and implement preventive measures"
```

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
