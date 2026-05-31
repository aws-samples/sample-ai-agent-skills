# Application Discovery Service Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Application Discovery Service problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS Application Discovery Service — agent deployment, agentless discovery, data collection, export, dependency mapping, and Athena integration.

### Activate When

- Agent deployment or health issues
- Agentless collector or vCenter problems
- Data collection failures
- Export or continuous export errors
- Dependency mapping issues
- Athena query failures on discovery data

---

## Skill Structure

```
appdiscovery-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-agent-deployment.md
    ├── A2-agent-health.md
    ├── B1-agentless-collector.md
    ├── B2-vcenter-integration.md
    ├── C1-data-collection.md
    ├── C2-network-data.md
    ├── D1-data-export.md
    ├── D2-continuous-export.md
    ├── E1-dependency-mapping.md
    ├── F1-athena-integration.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Prerequisites

- AWS CLI v2 with `discovery:*`, `s3:*`, `athena:*`
- For agent-based: Discovery Agent installed on target servers
- For agentless: Agentless Collector OVA deployed in VMware environment

---

## License

MIT-0
