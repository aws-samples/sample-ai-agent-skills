# Trusted Advisor Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Trusted Advisor problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS Trusted Advisor when the console alone isn't enough — check failures, stale results, refresh issues, cost optimization recommendations, security findings, performance checks, priority recommendations, EventBridge integration, organizational view, support plan requirements, and API access.

### Activate When

- Trusted Advisor check showing unexpected results
- Check results appear stale or outdated
- Cannot refresh checks
- Cost optimization recommendations unclear
- Security check findings need investigation
- Performance recommendations need validation
- Priority recommendations not appearing
- Risk level configuration issues
- EventBridge not receiving TA events
- Organizational view not working
- Cannot access Trusted Advisor API
- Support plan limitations blocking features

---

## Skill Structure

```
trustedadvisor-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-check-failures.md
    ├── A2-stale-results.md
    ├── A3-refresh-issues.md
    ├── B1-cost-optimization.md
    ├── B2-security.md
    ├── B3-performance.md
    ├── C1-priority-recommendations.md
    ├── C2-risk-levels.md
    ├── D1-eventbridge.md
    ├── D2-organizational-view.md
    ├── E1-support-plan.md
    ├── E2-api-access.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (18 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Checks** | A1–A3 | Check failures, stale results, refresh issues |
| **B — Categories** | B1–B3 | Cost optimization, security, performance |
| **C — Priority** | C1–C2 | Priority recommendations, risk levels |
| **D — Integration** | D1–D2 | EventBridge, organizational view |
| **E — Access** | E1–E2 | Support plan requirements, API access |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## License

MIT-0
