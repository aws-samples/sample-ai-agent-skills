# ELB Diagnostics Skill

Agent skill for investigating and troubleshooting Elastic Load Balancing problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for ALB, NLB, and CLB when the console alone isn't enough — 5xx errors, health check failures, target registration, SSL/TLS certificate issues, sticky sessions, cross-zone load balancing, WAF integration, slow targets, connection draining, and path-based routing.

### Activate When

- 502, 503, or 504 errors from the load balancer
- Health checks failing for targets
- Targets not registering or deregistering slowly
- SSL/TLS certificate errors or negotiation failures
- Sticky sessions not working
- Cross-zone load balancing issues
- WAF blocking legitimate traffic
- Slow response times through the load balancer
- Connection draining taking too long
- Path-based or host-based routing not working
- NLB source IP preservation issues
- WebSocket connection failures through ALB

---

## Skill Structure

```
elb-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-failing-health-checks.md
    ├── A2-grace-period.md
    ├── A3-unhealthy-targets.md
    ├── A4-health-check-config.md
    ├── B1-502-errors.md
    ├── B2-503-errors.md
    ├── B3-504-errors.md
    ├── B4-4xx-errors.md
    ├── C1-certificate-errors.md
    ├── C2-tls-negotiation.md
    ├── C3-backend-ssl.md
    ├── D1-path-based-routing.md
    ├── D2-host-based-routing.md
    ├── D3-fixed-response.md
    ├── E1-source-ip-preservation.md
    ├── E2-cross-zone.md
    ├── E3-tcp-health-checks.md
    ├── F1-slow-targets.md
    ├── F2-connection-draining.md
    ├── F3-idle-timeout.md
    ├── G1-target-registration.md
    ├── G2-target-deregistration.md
    ├── H1-waf-integration.md
    ├── H2-access-logs.md
    ├── Z1-general-troubleshooting.md
    ├── elb-guardrails.md
    └── elb-hallucination-patterns.yaml
```

---

## Runbook Library (30 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Health Checks** | A1–A4 | Failing health checks, grace period, unhealthy targets, config |
| **B — Errors** | B1–B4 | 502, 503, 504, 4xx errors |
| **C — SSL/TLS** | C1–C3 | Certificate errors, TLS negotiation, backend SSL |
| **D — Routing** | D1–D3 | Path-based, host-based, fixed response |
| **E — NLB-Specific** | E1–E3 | Source IP preservation, cross-zone, TCP health checks |
| **F — Performance** | F1–F3 | Slow targets, connection draining, idle timeout |
| **G — Target Groups** | G1–G2 | Registration, deregistration |
| **H — Integration** | H1–H2 | WAF, access logs |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## License

MIT-0
