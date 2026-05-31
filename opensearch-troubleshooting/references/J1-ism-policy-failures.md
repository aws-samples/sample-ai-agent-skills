---
title: "J1 — OpenSearch ISM Policy Failures"
description: "Diagnose and resolve Index State Management policy execution failures"
status: active
severity: MEDIUM
triggers:
  - "ISM failed"
  - "ISM policy"
  - "index state management"
  - "policy execution"
  - "transition failed"
  - "ISM error"
owner: devops-agent
objective: "Identify and resolve ISM policy execution failures to restore automated index lifecycle"
context: "Index State Management (ISM) automates index lifecycle operations: rollover, warm migration, force merge, snapshot, read-only, shrink, close, and delete. ISM policies define states and transitions with conditions (age, size, doc count). ISM runs on a configurable schedule (default 5 minutes). Failed transitions leave the index in a failed state requiring manual intervention. The _plugins/_ism/explain API shows current state and errors."
---

## Phase 1 — Triage

MUST:
- Check ISM status for the index: `curl -XGET "https://<endpoint>/_plugins/_ism/explain/<index>?pretty"`
- Check ISM policy definition: `curl -XGET "https://<endpoint>/_plugins/_ism/policies/<policy-name>?pretty"`
- List all ISM policies: `curl -XGET "https://<endpoint>/_plugins/_ism/policies?pretty"`
- Check index state: `curl -XGET "https://<endpoint>/_cat/indices/<index>?v&h=index,health,status,store.size,docs.count"`
- Check cluster health: `curl -XGET "https://<endpoint>/_cluster/health?pretty"`

SHOULD:
- Check ISM execution history: `curl -XGET "https://<endpoint>/_plugins/_ism/explain/<index>?pretty" | grep -E "state|action|failed|info"`
- Check if UltraWarm is enabled (for warm transition): `aws opensearch describe-domain --domain-name <domain> --query 'DomainStatus.ClusterConfig.WarmEnabled'`
- Check disk space (for force merge, snapshot actions): `curl -XGET "https://<endpoint>/_cat/allocation?v"`

MAY:
- Check ISM template associations: `curl -XGET "https://<endpoint>/_index_template?pretty"`
- Review ISM job scheduler settings

## Phase 2 — Remediate

MUST:
- If transition failed: fix the underlying issue and retry: `curl -XPOST "https://<endpoint>/_plugins/_ism/retry/<index>" -H 'Content-Type: application/json' -d '{"state":"<current-state>"}'`
- If policy misconfigured: update policy: `curl -XPUT "https://<endpoint>/_plugins/_ism/policies/<policy-name>" -H 'Content-Type: application/json' -d '<updated-policy>'`
- If warm migration failed: check UltraWarm capacity and retry

SHOULD:
- Attach ISM policy to index if not attached: `curl -XPOST "https://<endpoint>/_plugins/_ism/add/<index>" -H 'Content-Type: application/json' -d '{"policy_id":"<policy-name>"}'`
- Use ISM templates to automatically attach policies to new indices
- Monitor ISM execution via CloudWatch or periodic explain checks

MAY:
- Remove ISM policy from index: `curl -XPOST "https://<endpoint>/_plugins/_ism/remove/<index>"`
- Change ISM policy on an index: `curl -XPOST "https://<endpoint>/_plugins/_ism/change_policy/<index>" -H 'Content-Type: application/json' -d '{"policy_id":"<new-policy>"}'`

## Common Issues

- symptoms: "ISM transition to warm state failed"
  diagnosis: "UltraWarm not enabled or warm storage full."
  resolution: "Enable UltraWarm on the domain. Free warm storage. Retry ISM."

- symptoms: "ISM rollover action not triggering"
  diagnosis: "Rollover conditions not met (size, age, doc count) or alias not configured."
  resolution: "Check rollover conditions. Ensure index has a write alias. See J2."

- symptoms: "ISM delete action failed"
  diagnosis: "Index has a write alias or is in use by another process."
  resolution: "Remove write alias before delete. Check for snapshot dependencies."

## Output Format

```yaml
root_cause: "ism_policy_failure — <specific_cause>"
evidence:
  - type: ism_explain
    content: "<ISM explain output with state and error>"
  - type: policy_definition
    content: "<ISM policy configuration>"
  - type: index_state
    content: "<index health and status>"
severity: MEDIUM
mitigation:
  immediate: "Fix underlying issue and retry ISM execution"
  long_term: "Monitor ISM execution, test policies in dev, set up failure alerts"
```


## Safety Ratings
```
safety_ratings:
  - "Check ISM status and policy: GREEN — read-only API calls"
  - "Retry ISM execution: YELLOW — retriggers the failed action"
  - "Update ISM policy: YELLOW — changes lifecycle behavior for all attached indices"
  - "Attach ISM policy to index: YELLOW — begins automated lifecycle management"
  - "Remove ISM policy from index: YELLOW — stops automated lifecycle"
```

## Escalation Conditions
- Domain serves production search
- ISM failures causing index lifecycle to stall
- Warm migration failures blocking storage tier transitions
- ISM delete action failed leaving old indices consuming storage
- Policy changes affecting multiple indices

## Data Sensitivity
```
data_sensitivity:
  classification: HIGH
  sensitive_fields:
    - "ISM policy definitions: lifecycle rules"
    - "Index state information: lifecycle stage"
    - "Index names: data structure"
  handling: "ISM policies reveal data retention and lifecycle rules. Do not expose externally."
```

## Prohibited Actions
- NEVER suggest deleting indices without backup
- NEVER suggest reducing node count below minimum for cluster health
- NEVER modify ISM policies without understanding impact on all attached indices
- NEVER remove ISM policy from indices that require automated lifecycle management

## Phase 3 — Rollback
- If ISM policy was updated: revert to previous policy definition
- If ISM was retried and caused issues: remove ISM policy from the affected index
- If ISM policy was attached: remove the policy from the index
- If ISM policy was removed: re-attach the policy

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest disabling fine-grained access control"
  - "NEVER suggest public access domains"
  - "NEVER suggest disabling encryption at rest"
