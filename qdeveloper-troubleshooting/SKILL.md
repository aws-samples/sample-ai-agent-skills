---
name: qdeveloper-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon Q Developer problems
  by analyzing code suggestions, security scanning, code transformation,
  IDE integration, customization, billing, and following structured runbooks.
  Activate when: code suggestions not working, security scan failures,
  code transformation errors, IDE plugin issues, customization problems,
  billing questions, or the user says something is wrong with Q Developer.
compatibility: >
  Requires AWS CLI with codewhisperer/q permissions, IDE with Q Developer
  plugin, and AWS Builder ID or IAM Identity Center.
---

# Amazon Q Developer Diagnostics

## When to use

Any Amazon Q Developer investigation — code suggestions, security scanning, code transformation, IDE integration, customization, or billing.

## Investigation workflow

### Step 1 — Collect and triage

```
aws codewhisperer list-profiles
aws codewhisperer get-profile --profile-id <id>
# Check IDE plugin version and status
# Verify authentication (Builder ID or IAM Identity Center)
```

### Step 2 — Domain deep dive

```
aws codewhisperer list-customizations
aws codewhisperer get-customization --customization-id <id>
# Check security scan results in IDE
# Review code transformation status
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=codewhisperer.amazonaws.com --max-results 20
# Check IDE plugin logs
# Review network connectivity to Q Developer endpoints
```

Read `references/guardrails.md` before concluding on any Q Developer issue.

## Gotchas: Amazon Q Developer

- Q Developer requires authentication via AWS Builder ID or IAM Identity Center.
- Code suggestions depend on IDE context. Insufficient context produces poor suggestions.
- Security scanning has file size and language limitations.
- Code transformation supports specific language/framework migrations.
- Customization requires training data and takes time to build.
- Free tier has monthly limits on suggestions and scans.

## Anti-hallucination rules

1. Always cite specific error messages, IDE logs, or API responses as evidence.
2. Q Developer requires authentication. Never suggest it works without sign-in.
3. Code suggestions are context-dependent. Quality varies by context.
4. Security scanning has language limitations. Not all languages supported.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 12 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Suggestions | A1-A2 | Code suggestions, suggestion quality |
| B — Security | B1-B2 | Security scanning, scan results |
| C — Transform | C1-C2 | Code transformation, migration issues |
| D — IDE | D1-D2 | IDE integration, plugin issues |
| E — Custom | E1-E2 | Customization, training data |
| F — Billing | F1-F2 | Usage tracking, tier management |
