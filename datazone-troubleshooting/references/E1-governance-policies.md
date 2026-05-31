# E1 — DataZone Governance Policy Issues

## Symptoms
- Policy not enforcing as expected
- Metadata policy errors
- Access policy conflicts

## Triage Commands

```bash
aws datazone get-domain --identifier <domain-id>
aws datazone search --domain-identifier <domain-id> --search-scope ASSET --search-text "*" --max-results 10
```

## Common Causes
1. **Policy not applied** — Policy created but not associated with assets
2. **Policy conflict** — Multiple policies with conflicting rules
3. **Metadata requirements** — Required metadata fields not enforced
4. **Glossary enforcement** — Glossary term requirements not configured
5. **Quality gates** — Data quality thresholds not set

## Resolution
- Review and apply governance policies to appropriate assets/projects
- Resolve policy conflicts by reviewing priority and scope
- Configure required metadata fields in asset type definitions
- Set glossary term requirements for asset publishing
- Configure data quality gates with appropriate thresholds


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | HIGH — Governance policies control data access and quality requirements |
| Blast Radius | HIGH — Policy changes affect all assets and projects in scope |
| Rollback Complexity | MEDIUM — Policy changes may need to be manually reverted |
| Compliance Scope | CRITICAL — Governance policies are the foundation of data compliance |

## Escalation Conditions

- Escalate to compliance if governance policies are not enforcing required controls
- Escalate to data governance team if policy conflicts create access gaps
- Escalate to management if policy changes affect multiple teams

## Data Sensitivity

- **Policy configurations** define governance rules — treat as security-critical
- **Policy enforcement results** reveal compliance status

## PROHIBITED Actions

- Do NOT disable governance policies without compliance team approval
- Do NOT create conflicting policies without understanding priority resolution
- Do NOT weaken data quality gates for convenience

## Phase 3 — Rollback

- Restore previous governance policy configuration
- Re-apply policies to affected assets and projects
- Verify policy enforcement is working correctly after rollback
- Document the policy change and rollback for audit trail

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
