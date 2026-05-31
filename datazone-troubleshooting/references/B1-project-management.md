# B1 — DataZone Project Management Issues

## Symptoms
- Project creation fails
- Can't add members to project
- Project not visible in portal

## Triage Commands

```bash
aws datazone list-projects --domain-identifier <domain-id> --max-results 20
aws datazone get-project --domain-identifier <domain-id> --identifier <project-id>
aws datazone list-project-memberships --domain-identifier <domain-id> --project-identifier <project-id>
```

## Common Causes
1. **Domain not active** — Domain must be AVAILABLE
2. **IAM permissions** — Missing datazone:CreateProject
3. **Member not in domain** — User must have domain access first
4. **Project limit** — Service quota on projects per domain
5. **Name conflict** — Project name already exists in domain

## Resolution
- Verify domain is AVAILABLE
- Create project: `aws datazone create-project --domain-identifier <domain-id> --name <name>`
- Add members: `aws datazone create-project-membership --domain-identifier <domain-id> --project-identifier <project-id> --member identifier=<user-id>,type=IAM_USER --designation PROJECT_CONTRIBUTOR`
- Check service quotas for project limits


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | MEDIUM — Projects contain data assets and member access configurations |
| Blast Radius | LOW — Project changes affect only project members |
| Rollback Complexity | LOW — Members can be added/removed; projects can be recreated |
| Compliance Scope | MEDIUM — Project membership controls data access |

## Escalation Conditions

- Escalate to domain admin if project creation fails due to quota limits
- Escalate to security if unauthorized users are added to projects

## Data Sensitivity

- **Project membership** defines who can access data assets — access-control sensitive
- **Project configurations** reveal data governance structure

## PROHIBITED Actions

- Do NOT add members to projects without verifying their authorization level
- Do NOT delete projects containing active subscriptions without notifying consumers
- Do NOT create projects without proper naming conventions and ownership

## Phase 3 — Rollback

- Remove incorrectly added members: `aws datazone delete-project-membership --domain-identifier <id> --project-identifier <id> --member identifier=<user>,type=IAM_USER`
- Restore project configuration from documentation
- Recreate deleted projects if needed

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
