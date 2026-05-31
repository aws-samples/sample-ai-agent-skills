# AMP Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon Managed Service for Prometheus (AMP) problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AMP workspaces when the AWS Console alone isn't enough — workspace lifecycle, remote write ingestion, SigV4 authentication, query performance, PromQL errors, alerting/ruler configuration, ADOT and Prometheus collector issues, Grafana integration, VPC endpoint access, and IAM permissions.

### Activate When

- Workspace creation or deletion fails
- Remote write returns errors (4xx, 5xx)
- SigV4 authentication failures on ingestion or query
- Ingestion rate limiting (HTTP 429)
- High cardinality causing throttling or cost spikes
- Query timeouts or slow PromQL queries
- PromQL syntax or evaluation errors
- Rule evaluation failures in the ruler
- SNS alert integration not working
- Alert manager configuration errors
- ADOT Collector not sending metrics to AMP
- Prometheus server remote write misconfigured
- EKS ADOT add-on issues
- Grafana cannot connect to AMP data source
- VPC endpoint connectivity problems
- IAM permissions insufficient for AMP operations
- Third-party tool integration failures
- The user says something is wrong with AMP without naming specific symptoms

---

## Skill Structure

```
amp-troubleshooting/
├── SKILL.md                                  # Main skill definition and investigation workflow
├── README.md                                 # This file
└── references/
    ├── amp-guardrails.md                     # 12 anti-misdiagnosis guardrails
    ├── amp-hallucination-patterns.yaml       # 8 common LLM hallucination patterns
    ├── A1-workspace-creation-failures.md
    ├── A2-workspace-configuration.md
    ├── A3-workspace-deletion.md
    ├── B1-remote-write-failures.md
    ├── B2-sigv4-auth-errors.md
    ├── B3-rate-limiting.md
    ├── B4-high-cardinality.md
    ├── C1-query-timeouts.md
    ├── C2-promql-errors.md
    ├── C3-query-performance.md
    ├── D1-rule-evaluation-failures.md
    ├── D2-sns-integration.md
    ├── D3-alert-manager-config.md
    ├── E1-adot-collector-issues.md
    ├── E2-prometheus-remote-write.md
    ├── E3-eks-addon.md
    ├── F1-iam-sigv4-configuration.md
    ├── F2-vpc-endpoint-access.md
    ├── G1-grafana-data-source.md
    ├── G2-third-party-tools.md
    └── Z1-general-troubleshooting.md
```

---

## Runbook Library (26 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Workspace** | A1–A3 | Workspace creation failures, configuration issues, deletion/recovery |
| **B — Ingestion** | B1–B4 | Remote write failures, SigV4 auth errors, rate limiting, high cardinality |
| **C — Query** | C1–C3 | Query timeouts, PromQL errors, query performance |
| **D — Alerting** | D1–D3 | Rule evaluation failures, SNS integration, alert manager config |
| **E — Collectors** | E1–E3 | ADOT collector issues, Prometheus server remote write, EKS add-on |
| **F — Security** | F1–F2 | IAM/SigV4 configuration, VPC endpoint access |
| **G — Integration** | G1–G2 | Grafana data source, third-party tools |
| **Z — Catch-All** | Z1 | General AMP troubleshooting |

---

## Guardrails & Anti-Hallucination

The skill includes two guardrail files that prevent common misdiagnosis:

### AMP Guardrails (`amp-guardrails.md`)
12 rules covering: SigV4 required (no basic auth), ingestion rate limits, active series limits, 150-day retention (not configurable), no Prometheus UI (query via API or Grafana), ruler/alerting requires SNS, workspace region-bound, remote write endpoint format, query endpoint format, ADOT vs Prometheus collector differences, label cardinality impact, workspace deletion permanent.

### Hallucination Patterns (`amp-hallucination-patterns.yaml`)
8 patterns that LLMs commonly get wrong about AMP, including:
- Self-hosted Prometheus features assumed (TSDB, WAL, config reload)
- Basic auth for remote write (only SigV4 works)
- Configurable retention (fixed at 150 days)
- Prometheus UI available (no UI — use Grafana or query API)
- Unlimited series (workspace has active series limits)
- AlertManager routing directly (must use SNS integration)
- Local storage access (AMP is remote storage only)
- Prometheus operator full compatibility (partial — remote write only)

---

## Investigation Workflow

Each runbook follows a consistent phased structure:

### Phase 1 — Triage
Collect initial evidence using AWS CLI (`list-workspaces`, `describe-workspace`, CloudWatch metrics), curl with SigV4 signing, and collector logs. Classify the failure domain.

### Phase 2 — Remediate
Deep dive into the specific domain using targeted commands, CloudWatch metrics, CloudTrail events, or collector configuration review. Apply the fix.

### Output Format
Every runbook produces structured YAML output:
```yaml
root_cause: "<category> — <detail>"
evidence:
  - type: <source>
    content: "<specific finding>"
severity: CRITICAL | HIGH | MEDIUM
mitigation:
  immediate: "<action>"
  long_term: "<prevention>"
```

---

## Prerequisites

- AWS CLI with `amp`, `cloudwatch`, `cloudtrail`, `iam`, and `ec2` permissions
- curl with `--aws-sigv4` support (curl 7.75+) for endpoint testing
- promtool (optional, for local rule validation)
- kubectl (for EKS-based collector troubleshooting)
- Grafana (optional, for query and visualization testing)

---

## Usage Examples

### Remote Write Failures
```
My ADOT collector on EKS is failing to send metrics to AMP workspace
"ws-12345" in us-east-1. I see 403 errors in the collector logs.
Check SigV4 configuration, IAM permissions, and endpoint URL.
```

### High Cardinality
```
My AMP workspace is hitting active series limits. CloudWatch shows
ActiveSeries at 9.8M. Help me identify high-cardinality labels and
reduce the series count.
```

### Query Performance
```
PromQL queries against my AMP workspace are timing out. Simple queries
like rate(http_requests_total[5m]) take over 30 seconds. Investigate
query performance and suggest optimizations.
```

### Alerting Not Working
```
I configured alerting rules in my AMP workspace but I'm not receiving
SNS notifications. Check the ruler configuration, alert manager
definition, and SNS topic setup.
```

### General Triage
```
Something is wrong with my AMP workspace but I'm not sure what.
Metrics seem to be missing and queries return empty results. Run a
general investigation and follow whichever runbook matches the symptoms.
```

---

## License

MIT-0
