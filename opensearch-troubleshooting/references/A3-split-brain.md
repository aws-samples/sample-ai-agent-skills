---
title: "A3 — OpenSearch Split Brain"
description: "Diagnose and resolve split brain conditions where multiple nodes believe they are the master"
status: active
severity: CRITICAL
triggers:
  - "split brain"
  - "two masters"
  - "cluster split"
  - "master election"
  - "MasterReachableFromNode"
owner: devops-agent
objective: "Identify split brain condition, restore single master authority, and prevent recurrence"
context: "Split brain occurs when network partitions cause multiple nodes to elect themselves as master, leading to divergent cluster states and potential data inconsistency. In managed OpenSearch Service, split brain is rare due to AWS infrastructure, but can occur with misconfigured dedicated master nodes or even-numbered master-eligible nodes. Using 3 or 5 dedicated master nodes with proper quorum settings prevents split brain."
---

## Phase 1 — Triage

MUST:
- Check master node: `curl -XGET "https://<endpoint>/_cat/master?v"`
- Check all nodes and roles: `curl -XGET "https://<endpoint>/_cat/nodes?v&h=name,node.role,master"`
- Check cluster health: `curl -XGET "https://<endpoint>/_cluster/health?pretty"`
- Check dedicated master configuration: `aws opensearch describe-domain --domain-name <domain> --query 'DomainStatus.ClusterConfig.{DedicatedMasterEnabled:DedicatedMasterEnabled,DedicatedMasterCount:DedicatedMasterCount,DedicatedMasterType:DedicatedMasterType}'`
- Check CloudWatch MasterReachableFromNode: `aws cloudwatch get-metric-statistics --namespace AWS/ES --metric-name MasterReachableFromNode --dimensions Name=DomainName,Value=<domain> Name=ClientId,Value=<account-id> --start-time <start> --end-time <end> --period 60 --statistics Minimum`

SHOULD:
- Check pending cluster tasks: `curl -XGET "https://<endpoint>/_cat/pending_tasks?v"`
- Check cluster state metadata: `curl -XGET "https://<endpoint>/_cluster/state/master_node,nodes?pretty"`
- Review CloudTrail for recent domain changes: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=es.amazonaws.com --max-results 20`

MAY:
- Check node connectivity: `curl -XGET "https://<endpoint>/_nodes/stats/transport?pretty"`
- Review domain events in the AWS console for master node issues

## Phase 2 — Remediate

MUST:
- Enable dedicated master nodes if not enabled: `aws opensearch update-domain-config --domain-name <domain> --cluster-config DedicatedMasterEnabled=true,DedicatedMasterCount=3,DedicatedMasterType=m6g.large.search`
- Ensure odd number of dedicated masters (3 or 5)
- If active split brain: contact AWS Support — managed service may require intervention

SHOULD:
- Use 3 dedicated master nodes for clusters with up to 200 nodes
- Use 5 dedicated master nodes for very large clusters
- Enable multi-AZ deployment for master node redundancy

MAY:
- Review and adjust cluster.routing.allocation settings after resolution
- Verify data consistency across indices after split brain resolution

## Common Issues

- symptoms: "MasterReachableFromNode drops to 0"
  diagnosis: "Data nodes cannot reach the master node. Possible master node overload or network issue."
  resolution: "Check dedicated master node health. Scale master node type if under resource pressure."

- symptoms: "No dedicated master nodes configured on production cluster"
  diagnosis: "Data nodes are master-eligible, causing instability under load."
  resolution: "Enable 3 dedicated master nodes. Use m6g.large.search or larger."

- symptoms: "Even number of master-eligible nodes"
  diagnosis: "Even numbers risk tie in master election, leading to split brain."
  resolution: "Always use odd number (3 or 5) of dedicated master nodes."

## Output Format

```yaml
root_cause: "split_brain — <specific_cause>"
evidence:
  - type: master_node
    content: "<_cat/master output>"
  - type: node_roles
    content: "<_cat/nodes showing master-eligible nodes>"
  - type: master_config
    content: "<dedicated master configuration>"
severity: CRITICAL
mitigation:
  immediate: "Restore single master authority, contact AWS Support if needed"
  long_term: "Deploy 3 or 5 dedicated master nodes with appropriate instance types"
```


## Safety Ratings
```
safety_ratings:
  - "Check master node and cluster state: GREEN — read-only API calls"
  - "Check dedicated master configuration: GREEN — read-only API call"
  - "Enable dedicated master nodes: YELLOW — triggers blue/green deployment"
  - "Contact AWS Support: GREEN — escalation, no state change"
```

## Escalation Conditions
- Domain serves production search
- Active split brain condition (CRITICAL — contact AWS Support immediately)
- No dedicated master nodes on production cluster
- MasterReachableFromNode dropping to 0
- Fix requires blue/green deployment to add dedicated masters

## Data Sensitivity
```
data_sensitivity:
  classification: HIGH
  sensitive_fields:
    - "Cluster state: master node identity and topology"
    - "Node roles: infrastructure configuration"
  handling: "Split brain may cause data inconsistency. Do not expose cluster state details externally."
```

## Prohibited Actions
- NEVER suggest deleting indices without backup
- NEVER suggest reducing node count below minimum for cluster health
- NEVER use an even number of dedicated master nodes
- NEVER attempt to manually resolve split brain without AWS Support guidance

## Phase 3 — Rollback
- If dedicated masters were enabled: cannot easily disable — plan capacity accordingly
- If master instance type was upgraded: can be downgraded via domain config update
- After split brain resolution: verify data consistency across all indices

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
