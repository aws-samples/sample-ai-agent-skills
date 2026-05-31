# AppSync Diagnostics Skill

Agent skill for investigating and troubleshooting AWS AppSync problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS AppSync GraphQL APIs and merged APIs when the console alone isn't enough — resolver errors, schema deployment failures, authorization issues, subscription connection problems, caching misconfigurations, query latency, N+1 data fetching, conflict detection, and merged API conflicts.

### Activate When

- Schema deployment or validation errors
- VTL mapping template errors or unexpected null responses
- JavaScript resolver runtime errors
- Pipeline resolver function chain failures
- DynamoDB, Lambda, HTTP, or RDS data source errors
- API key expired or authorization denied
- Cognito user pool token validation failures
- IAM or OIDC authorization errors
- WebSocket subscription connection failures
- Subscription messages not delivered or filtered incorrectly
- Cache not working or returning stale data
- High query latency or slow resolvers
- N+1 resolver invocation problems
- Merged API schema conflicts

---

## Skill Structure

```
appsync-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-api-creation-failures.md
    ├── A2-schema-errors.md
    ├── A3-deployment-issues.md
    ├── B1-vtl-errors.md
    ├── B2-javascript-resolver-errors.md
    ├── B3-pipeline-resolver-failures.md
    ├── B4-resolver-timeout.md
    ├── C1-dynamodb-data-source-errors.md
    ├── C2-lambda-data-source-errors.md
    ├── C3-http-data-source-errors.md
    ├── C4-rds-aurora-data-source-errors.md
    ├── D1-api-key-issues.md
    ├── D2-cognito-auth.md
    ├── D3-iam-oidc-auth.md
    ├── E1-websocket-connection-failures.md
    ├── E2-subscription-filtering.md
    ├── F1-cache-configuration.md
    ├── F2-cache-invalidation.md
    ├── G1-query-latency.md
    ├── G2-n-plus-1-resolver-issues.md
    ├── Z1-general-troubleshooting.md
    ├── appsync-guardrails.md
    └── appsync-hallucination-patterns.yaml
```

---

## Runbook Library (26 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — API** | A1–A3 | API creation failures, schema errors, deployment issues |
| **B — Resolvers** | B1–B4 | VTL errors, JavaScript resolver errors, pipeline failures, timeout |
| **C — Data Sources** | C1–C4 | DynamoDB, Lambda, HTTP, RDS/Aurora data source errors |
| **D — Auth** | D1–D3 | API key issues, Cognito auth, IAM/OIDC auth |
| **E — Subscriptions** | E1–E2 | WebSocket connection failures, subscription filtering |
| **F — Caching** | F1–F2 | Cache configuration, cache invalidation |
| **G — Performance** | G1–G2 | Query latency, N+1 resolver issues |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## License

MIT-0
