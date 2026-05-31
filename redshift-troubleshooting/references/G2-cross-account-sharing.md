---
title: "G2 — Cross-Account Data Sharing"
description: "Diagnose and resolve cross-account Redshift data sharing issues"
status: active
severity: MEDIUM
triggers:
  - "cross-account"
  - "cross account sharing"
  - "account authorization"
  - "datashare authorization"
owner: devops-agent
objective: "Fix cross-account data sharing authorization and access"
context: "Cross-account data sharing requires explicit authorization from the producer account and acceptance from the consumer account. Both accounts must be in the same region. The producer authorizes the consumer account, and the consumer associates the datashare with their cluster or Serverless namespace."
---

## Phase 1 — Triage

MUST:
- Check datashare authorization on producer: `aws redshift describe-data-shares-for-producer --producer-arn <producer-namespace-arn>`
- Check datashare status on consumer: `aws redshift describe-data-shares-for-consumer --consumer-arn <consumer-namespace-arn>`
- Verify both clusters are in the same region
- Check authorization status: `aws redshift describe-data-share-associations --data-share-arn <share-arn>`

SHOULD:
- Verify producer authorized the correct consumer account ID
- Check if consumer has accepted/associated the datashare
- Verify both clusters use RA3 or Serverless

MAY:
- Check CloudTrail in both accounts for authorization events
- Verify no SCP blocks Redshift data sharing operations
- Check for AWS Organizations integration

## Phase 2 — Remediate

MUST:
- If not authorized: producer must authorize: `aws redshift authorize-data-share --data-share-arn <share-arn> --consumer-identifier <consumer-account-id>`
- If not associated: consumer must associate: `aws redshift associate-data-share-consumer --data-share-arn <share-arn> --consumer-arn <consumer-namespace-arn>`
- If wrong account ID: deauthorize and reauthorize with correct account
- If different regions: data sharing requires same region — use cross-region snapshot copy instead

SHOULD:
- Document cross-account sharing relationships
- Use AWS Organizations for simplified authorization
- Set up monitoring for datashare status in both accounts

MAY:
- Implement tagging for cross-account datashares
- Create runbooks for cross-account datashare lifecycle
- Consider AWS RAM for managing cross-account access

## Common Issues

- symptoms: "Consumer account cannot see the datashare"
  diagnosis: "Producer has not authorized the consumer account."
  resolution: "Producer runs: aws redshift authorize-data-share with consumer account ID."

- symptoms: "Authorization granted but consumer still cannot query"
  diagnosis: "Consumer has not associated the datashare with their cluster/namespace."
  resolution: "Consumer runs: aws redshift associate-data-share-consumer."

## Output Format

```yaml
root_cause: "Cross-account sharing — <specific_cause>"
evidence:
  - type: datashare_authorization
    content: "<authorization and association status>"
severity: MEDIUM
mitigation:
  immediate: "Complete authorization and association on both sides"
  long_term: "Document cross-account relationships, use Organizations for simplified management"
```


## Safety Ratings
```
safety_ratings:
  - "Describe datashare authorization: GREEN — read-only API call"
  - "Check datashare status on consumer: GREEN — read-only API call"
  - "Authorize consumer account: YELLOW — grants cross-account data access"
  - "Associate datashare with consumer: YELLOW — enables data access from consumer account"
  - "Deauthorize consumer account: YELLOW — revokes cross-account access, breaks consumer queries"
```

## Escalation Conditions
- Cluster serves production analytics
- Cross-account authorization requires coordination between account owners
- SCP restrictions blocking data sharing operations
- Wrong account authorized requiring deauthorization
- Both accounts must be in the same region

## Data Sensitivity
```
data_sensitivity:
  classification: HIGH
  sensitive_fields:
    - "Account IDs: AWS account identifiers for both producer and consumer"
    - "Namespace ARNs: cluster/namespace identifiers"
    - "Datashare authorization status: cross-account access configuration"
  handling: "Cross-account sharing exposes account relationships. Use specific account IDs, never wildcards."
```

## Prohibited Actions
- NEVER suggest deleting a cluster without snapshot
- NEVER suggest VACUUM FULL on production during peak hours
- NEVER authorize cross-account access without explicit approval from both account owners
- NEVER deauthorize a consumer account without confirming no active queries depend on the shared data
- NEVER share data cross-account without understanding data governance and compliance requirements

## Phase 3 — Rollback
- If consumer was authorized: deauthorize the consumer account on the producer side
- If datashare was associated on consumer: disassociate the datashare from the consumer cluster
- If wrong account was authorized: deauthorize the wrong account and authorize the correct one
- If cross-account sharing was fully set up: deauthorize on producer and disassociate on consumer to fully revoke

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest making clusters publicly accessible"
  - "NEVER suggest disabling audit logging"
  - "NEVER suggest disabling encryption"
