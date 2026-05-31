# Global Accelerator Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Global Accelerator problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for Global Accelerator when the console alone isn't enough — accelerator configuration analysis, endpoint health debugging, routing policy issues, performance optimization, DNS resolution problems, custom routing configuration, DDoS protection, and cross-account endpoint management.

### Activate When

- Accelerator creation failures
- Static anycast IP issues
- Accelerator configuration problems
- Endpoint health check failures
- Endpoint group misconfiguration
- Traffic dial not working as expected
- Client affinity issues
- Flow log analysis needed
- High latency through Global Accelerator
- Throughput degradation
- DNS resolution failures for accelerator
- Custom routing port mapping issues
- DDoS protection concerns
- Cross-account endpoint problems
- ALB/NLB/EC2/EIP endpoint integration issues
- Custom routing accelerator configuration

---

## Skill Structure

```
globalaccelerator-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-creation-failures.md
    ├── A2-static-ip-issues.md
    ├── A3-configuration.md
    ├── B1-endpoint-health.md
    ├── B2-endpoint-groups.md
    ├── B3-traffic-dial.md
    ├── C1-client-affinity.md
    ├── C2-flow-logs.md
    ├── D1-latency.md
    ├── D2-throughput.md
    ├── E1-dns-resolution.md
    ├── E2-custom-routing.md
    ├── F1-ddos-protection.md
    ├── F2-cross-account-endpoints.md
    ├── G1-alb-nlb-ec2-eip-endpoints.md
    ├── G2-custom-routing-accelerator.md
    ├── Z1-general-troubleshooting.md
    ├── globalaccelerator-guardrails.md
    └── globalaccelerator-hallucination-patterns.yaml
```

---

## Runbook Library (22 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Accelerator** | A1–A3 | Creation failures, static IP issues, configuration |
| **B — Endpoints** | B1–B3 | Endpoint health, endpoint groups, traffic dial |
| **C — Routing** | C1–C2 | Client affinity, flow logs |
| **D — Performance** | D1–D2 | Latency, throughput |
| **E — DNS** | E1–E2 | DNS resolution, custom routing |
| **F — Security** | F1–F2 | DDoS protection, cross-account endpoints |
| **G — Integration** | G1–G2 | ALB/NLB/EC2/EIP endpoints, custom routing accelerator |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Guardrails Summary

12 guardrails in `references/globalaccelerator-guardrails.md` covering static anycast IPs, endpoint health check independence, traffic dial scope, client affinity scope, custom routing limitations, GA vs CDN distinction, DDoS protection, endpoint types and limits, cross-account requirements, flow log configuration, listener port ranges, and accelerator deployment time.

---

## Investigation Workflow

1. **Triage** — Collect accelerator config, check CloudWatch metrics for flow counts and processed bytes
2. **Deep Dive** — Examine listeners, endpoint groups, endpoint health, traffic dial settings
3. **Detailed** — Inspect custom routing mappings, flow logs, CloudTrail events

---

## Prerequisites

- AWS CLI v2 configured with appropriate credentials
- Permissions: `globalaccelerator:*`, `ec2:Describe*`, `elasticloadbalancing:Describe*`, `cloudwatch:GetMetricStatistics`, `cloudtrail:LookupEvents`, `s3:GetObject` (for flow logs)
- Flow logs enabled (recommended)
- CloudWatch metrics enabled

---

## Usage Examples

```
# List all accelerators
aws globalaccelerator list-accelerators

# Describe accelerator
aws globalaccelerator describe-accelerator --accelerator-arn arn:aws:globalaccelerator::123456789012:accelerator/abcd-1234

# Check endpoint health
aws globalaccelerator describe-endpoint-group --endpoint-group-arn <endpoint-group-arn>

# Check flow counts
aws cloudwatch get-metric-statistics --namespace AWS/GlobalAccelerator \
  --metric-name NewFlowCount --dimensions Name=Accelerator,Value=<accelerator-arn> \
  --start-time 2024-01-01T00:00:00Z --end-time 2024-01-02T00:00:00Z --period 3600 --statistics Sum
```

---

## License

MIT-0
