---
title: "Z1 — General ECR Troubleshooting (Catch-All)"
description: "Fallback SOP for ECR issues that do not match any specific runbook"
status: active
severity: MEDIUM
triggers:
  - ".*"
owner: devops-agent
objective: "Systematically investigate an unknown ECR issue, classify the failure domain, and match to an existing SOP or escalate"
context: "This SOP is invoked when symptoms don't match any of the specific runbooks. It provides a broad, methodical investigation that narrows the failure domain step by step."
---

## Phase 1 — Triage

MUST:
- List all repositories: `aws ecr describe-repositories`
- Check registry configuration: `aws ecr describe-registry`
- If a specific repository is involved, get details: `aws ecr describe-repositories --repository-names <repo>`
- Check recent ECR events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=ecr.amazonaws.com --max-results 10`
- Verify authentication status: `aws ecr get-authorization-token --query 'authorizationData[0].expiresAt'`

SHOULD:
- Check scanning configuration: `aws ecr get-registry-scanning-configuration`
- Check replication configuration: `aws ecr describe-registry --query 'replicationConfiguration'`
- List pull-through cache rules: `aws ecr describe-pull-through-cache-rules`
- Verify IAM permissions for the calling principal

MAY:
- Check AWS Health Dashboard for ECR service events
- Check service quotas: `aws service-quotas list-service-quotas --service-code ecr`
- Review ECR Public if applicable: `aws ecr-public describe-repositories --region us-east-1`

## Phase 2 — Classify

Based on triage results, classify into a failure domain:
- Docker push errors → Push/Pull (A1)
- Docker pull errors → Push/Pull (A2)
- Authentication failures → Push/Pull (A3)
- Scanning issues → Images (B1)
- Lifecycle policy problems → Images (B2)
- Tag immutability conflicts → Images (B3)
- Repository creation failures → Repository (C1)
- Cross-account access denied → Repository (C2)
- Cross-region replication issues → Replication (D1)
- Cross-account replication issues → Replication (D2)
- Repository policy errors → Security (E1)
- Encryption/KMS issues → Security (E2)
- Pull throughput/rate limits → Performance (F1)
- Pull-through cache issues → Performance (F2)
- ECR Public problems → Public (G1)
- Gallery visibility issues → Public (G2)

If classified: switch to the specific SOP immediately.
If unclassified: continue to Phase 3.

## Phase 3 — Deep Investigation

MUST:
- Check all repository configurations systematically
- Review CloudTrail for recent ECR configuration changes
- Verify network connectivity to ECR endpoints
- Check VPC endpoint configuration if applicable

SHOULD:
- Check AWS Health Dashboard for ECR service events
- Compare with a known-good repository configuration
- Test basic operations: list, describe, pull, push

MAY:
- Check for regional service outages
- Review Docker daemon logs for client-side errors
- Verify DNS resolution for ECR endpoints

## Phase 4 — Report

MUST:
- State the investigation path taken
- State root cause if identified, or "unclassified" with best hypothesis
- List all evidence collected
- Recommend next steps

## Output Format

```yaml
root_cause: "<identified_cause OR unclassified>"
failure_domain: "<push_pull|images|repository|replication|security|performance|public|unknown>"
investigation_path: "registry status → repository config → auth → <domain_classification>"
evidence:
  - type: registry_config
    content: "<registry configuration summary>"
  - type: cloudtrail
    content: "<relevant events>"
severity: MEDIUM
mitigation:
  immediate: "<specific action if root cause found, or escalate>"
  long_term: "Implement monitoring for the identified failure pattern"
```

---

## Safety Ratings

| Aspect | Rating |
|--------|--------|
| Service Impact | HIGH � ECR hosts container images critical to deployments |
| Data Risk | MEDIUM � Images may contain application code and secrets |
| Recovery Complexity | MEDIUM � Deleted images can be rebuilt but cause deployment delays |
| Blast Radius | HIGH � Image issues affect all services pulling from the repository |

## Escalation Conditions

- Image pull failures affecting production deployments or auto-scaling events
- Repository policy changes locking out CI/CD pipelines from pushing images
- Cross-account replication failures causing stale images in secondary regions
- Image scanning revealing critical CVEs in production-running images
- Lifecycle policy unexpectedly deleting images referenced by running tasks
- Encryption key issues preventing image push or pull operations

## Data Sensitivity

- Container images may embed application secrets, API keys, or credentials
- Repository policies may reveal cross-account trust relationships
- Image tags and digests are deployment identifiers � treat as operational metadata
- Scan results may expose vulnerability details that should not be shared broadly
- ECR auth tokens grant broad repository access � treat as short-lived secrets

## PROHIBITED Actions

- NEVER suggest deleting images that are referenced by running tasks or services
- NEVER modify repository policies without verifying CI/CD pipeline access requirements
- NEVER disable tag immutability on repositories enforcing deployment traceability
- NEVER delete a repository without confirming all dependent services have been migrated
- NEVER share or log ECR authentication tokens in plain text
- NEVER disable image scanning on repositories subject to compliance requirements

## Phase 3 � Rollback

- If image was accidentally deleted: rebuild and push from CI/CD pipeline or pull from replication target
- If repository policy change blocks access: revert to previous policy JSON from version control
- If lifecycle policy deleted needed images: restore from cross-region replica or rebuild from source
- If tag immutability was disabled: re-enable and audit for any overwritten tags
- If encryption configuration was changed: verify KMS key access and re-push affected images
- Document all rollback actions with image digests and tags for audit trail
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
