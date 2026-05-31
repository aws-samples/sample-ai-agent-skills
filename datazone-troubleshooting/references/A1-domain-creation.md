# A1 — DataZone Domain Creation Issues

## Symptoms
- CreateDomain API fails
- Domain stuck in CREATING state
- Domain portal not accessible

## Triage Commands

```bash
aws datazone list-domains --max-results 20
aws datazone get-domain --identifier <domain-id>
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateDomain --max-results 5
```

## Common Causes
1. **IAM permissions** — Missing datazone:CreateDomain permission
2. **Service role** — DataZone service role not configured
3. **KMS key** — Invalid or inaccessible encryption key
4. **Region not supported** — DataZone not available in all regions
5. **Domain name conflict** — Name already in use

## Resolution
- Verify IAM permissions include datazone:CreateDomain
- Configure DataZone service role with appropriate trust policy
- Verify KMS key exists and policy allows DataZone service
- Check DataZone region availability
- Domain creation takes a few minutes — wait for AVAILABLE status


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | HIGH — Domain creation involves KMS keys and service roles with broad access |
| Blast Radius | MEDIUM — Domain is the top-level container for all data governance |
| Rollback Complexity | MEDIUM — Domain deletion removes all projects, assets, and subscriptions |
| Compliance Scope | HIGH — Domain configuration affects data governance posture |

## Escalation Conditions

- Escalate to security if KMS key configuration is incorrect or inaccessible
- Escalate to management if domain creation fails repeatedly blocking data governance rollout
- Escalate to AWS Support if domain gets stuck in CREATING state

## Data Sensitivity

- **KMS key** used for domain encryption protects all data within the domain
- **Service role** has broad access to Glue, Redshift, Athena — audit permissions
- **Domain configuration** defines the governance boundary

## PROHIBITED Actions

- Do NOT create domains without proper KMS key configuration
- Do NOT use overly permissive service roles for DataZone
- Do NOT delete domains without understanding the impact on all contained resources
- Do NOT create domains in unsupported regions

## Phase 3 — Rollback

- If domain creation failed, check and fix prerequisites (KMS key, service role, region)
- Delete the failed domain if it's in an error state: `aws datazone delete-domain --identifier <id>`
- Recreate with corrected configuration
- Document the failure for future reference

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
