# AWS Network Firewall Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Network Firewall problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS Network Firewall when the console alone isn't enough — firewall deployment and endpoint management, stateless and stateful rule configuration, rule evaluation order, route table configuration for traffic inspection, asymmetric routing prevention, inspection VPC design, alert and flow logging, throughput and latency optimization, TLS inspection and certificate management, and integration with Firewall Manager and Transit Gateway.

### Activate When

- Firewall creation or update failures
- Firewall deployment issues
- Firewall endpoint not receiving traffic
- Stateless rules not matching expected traffic
- Stateful rules not detecting threats
- Rule evaluation order causing unexpected behavior
- Route tables not directing traffic through firewall
- Asymmetric routing breaking stateful inspection
- Inspection VPC design issues
- Alert logs not showing expected entries
- Flow logs missing or incomplete
- Throughput degradation through firewall
- Latency increase after enabling firewall
- TLS inspection not working
- Certificate management errors for TLS inspection
- Firewall Manager policy deployment issues
- Transit Gateway integration problems
- General Network Firewall errors without clear symptoms

---

## Skill Structure

```
networkfirewall-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-creation-failures.md
    ├── A2-deployment.md
    ├── A3-endpoint-issues.md
    ├── B1-stateless-rules.md
    ├── B2-stateful-rules.md
    ├── B3-rule-order.md
    ├── C1-route-table-config.md
    ├── C2-asymmetric-routing.md
    ├── C3-inspection-vpc.md
    ├── D1-alert-logs.md
    ├── D2-flow-logs.md
    ├── E1-throughput.md
    ├── E2-latency.md
    ├── F1-tls-inspection.md
    ├── F2-certificate-management.md
    ├── G1-firewall-manager.md
    ├── G2-transit-gateway.md
    ├── Z1-general-troubleshooting.md
    ├── networkfirewall-guardrails.md
    └── networkfirewall-hallucination-patterns.yaml
```

---

## Runbook Library (24 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Firewall** | A1–A3 | Creation failures, deployment, endpoint issues |
| **B — Rules** | B1–B3 | Stateless rules, stateful rules, rule order |
| **C — Routing** | C1–C3 | Route table config, asymmetric routing, inspection VPC |
| **D — Logging** | D1–D2 | Alert logs, flow logs |
| **E — Performance** | E1–E2 | Throughput, latency |
| **F — TLS** | F1–F2 | TLS inspection, certificate management |
| **G — Integration** | G1–G2 | Firewall Manager, Transit Gateway |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Guardrails Summary

12 guardrails in `references/networkfirewall-guardrails.md` covering routing requirement for traffic inspection, stateless-before-stateful evaluation order, stateful rule order modes, asymmetric routing impact, dedicated firewall subnets, TLS inspection requirements, logging configuration, firewall endpoint placement, rule group capacity, default actions, cross-AZ considerations, and firewall policy structure.

---

## Investigation Workflow

1. **Triage** — Check firewall status, policy, rule groups, routing, logging
2. **Deep Dive** — Examine specific rules, route tables, endpoint configuration
3. **Detailed** — Review alert/flow logs, CloudTrail events, performance metrics

---

## Prerequisites

- AWS CLI v2 configured with appropriate credentials
- Permissions: `network-firewall:Describe*`, `network-firewall:List*`, `ec2:DescribeRouteTables`, `ec2:DescribeVpcEndpoints`, `logs:FilterLogEvents`, `cloudtrail:LookupEvents`, `cloudwatch:GetMetricStatistics`
- Access to VPC route table configuration
- CloudWatch Logs or S3 access for log analysis

---

## Usage Examples

```
# List firewalls
aws network-firewall list-firewalls

# Check firewall details
aws network-firewall describe-firewall --firewall-name my-firewall

# Check firewall policy
aws network-firewall describe-firewall-policy --firewall-policy-name my-policy

# Check rule group
aws network-firewall describe-rule-group --rule-group-name my-rules --type STATEFUL

# Check logging
aws network-firewall describe-logging-configuration --firewall-arn <arn>

# Check route tables
aws ec2 describe-route-tables --filters Name=vpc-id,Values=<vpc-id>
```

---

## License

MIT-0
