---
name: appdiscovery-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Application Discovery
  Service problems by analyzing agent deployment, agentless discovery, data
  collection, export, dependency mapping, and Athena integration using
  structured runbooks.
  Activate when: agent deployment failures, agentless collector issues, data
  collection problems, export failures, dependency mapping errors, Athena
  query issues, or the user says something is wrong with Application
  Discovery Service without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with Application Discovery Service,
  S3, Athena, IAM, CloudWatch, CloudTrail permissions.
---

# Application Discovery Service Diagnostics

## When to use

Any AWS Application Discovery Service investigation — agent deployment, agentless discovery, data collection, export, dependency mapping, or Athena integration.

## Investigation workflow

### Step 1 — Collect and triage

```
aws discovery describe-agents --max-results 20
aws discovery describe-continuous-exports --max-results 10
aws discovery list-configurations --configuration-type SERVER --max-results 20
```

### Step 2 — Domain deep dive

```
aws discovery describe-agents --filters name=agentIds,values=<agent-id>,condition=EQUALS
aws discovery describe-export-tasks --max-results 10
aws discovery get-discovery-summary
```

### Step 3 — Detailed investigation

```
aws discovery start-data-collection-by-agent-ids --agent-ids <id1> <id2>
aws discovery describe-tags --filters name=configurationId,values=<server-id>
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=discovery.amazonaws.com --max-results 10
```

Read `references/guardrails.md` before concluding on any Application Discovery Service issue.

## Gotchas: Application Discovery Service

- Two discovery methods: agent-based (installed on servers) and agentless (VMware vCenter).
- Agents collect detailed data including network connections for dependency mapping.
- Agentless collector discovers VMware VMs via vCenter API — no agent installation needed.
- Data is stored in Migration Hub home region.
- Continuous export to S3 enables Athena queries on discovery data.
- Agent requires outbound HTTPS to discovery service endpoint.

## Anti-hallucination rules

1. Agent-based and agentless discovery collect different data — agents provide more detail.
2. Discovery data is stored in the Migration Hub home region, not the agent's region.
3. Continuous export must be enabled separately — it's not automatic.
4. Athena integration requires continuous export to be active.
5. Agentless collector requires VMware vCenter access — it doesn't work with other hypervisors.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 12 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Agent | A1-A2 | Agent deployment, agent health |
| B — Agentless | B1-B2 | Agentless collector, vCenter |
| C — Collection | C1-C2 | Data collection, network data |
| D — Export | D1-D2 | Data export, continuous export |
| E — Dependency | E1 | Dependency mapping |
| F — Athena | F1 | Athena integration |
| Z — Catch-All | Z1 | General troubleshooting |
