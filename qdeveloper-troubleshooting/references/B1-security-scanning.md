---
title: "B1 — Security Scanning Failures"
description: "Diagnose security scan execution failures"
status: active
severity: MEDIUM
triggers:
  - "security scan failed"
  - "scan not starting"
  - "scan error"
owner: devops-agent
objective: "Resolve security scanning failures"
context: "Security scanning analyzes code for vulnerabilities. Supports specific languages."
---

## Phase 1 — Triage

MUST:
- Verify file language is supported for scanning
- Check authentication status
- Verify scan limits not exceeded
- Check IDE plugin logs for scan errors

SHOULD:
- Check file size limits
- Verify project structure is valid
- Check for scan timeout issues

## Phase 2 — Remediate

MUST:
- Use supported language files
- Fix authentication
- Wait for limit reset if exceeded

SHOULD:
- Scan smaller file sets if timing out
- Update plugin for latest scan support

## Common Issues

- symptoms: "Scan completes with 0 findings on known vulnerable code"
  diagnosis: "Language not supported or vulnerability pattern not detected."
  resolution: "Verify language support. Use additional security tools."

## Output Format

```yaml
root_cause: "security_scan — <specific_cause>"
evidence:
  - type: scan_status
    content: "<scan execution details>"
severity: MEDIUM
mitigation:
  immediate: "Fix scan execution"
  long_term: "Integrate multiple security tools"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: MEDIUM — security scanning analyzes source code
- Service Disruption Risk: LOW — scanning issues do not affect development
- Compliance Impact: HIGH — security scanning is a compliance requirement

## Escalation Conditions
- Security scanning completely fails
- Scanning misses known vulnerabilities
- Scanning produces excessive false positives
- Scanning results not being delivered
- Scanning causes IDE performance issues

## Data Sensitivity
- Scan results — CONFIDENTIAL
- Vulnerability details — CONFIDENTIAL
- Source code analyzed — CONFIDENTIAL
- Scan configurations — INTERNAL
- Never expose vulnerability details externally

## Prohibited Actions
- Do NOT disable security scanning on production code
- Do NOT ignore critical vulnerability findings
- Do NOT share scan results with unauthorized parties
- Do NOT bypass security scanning requirements
- Do NOT dismiss findings without review
- Do NOT deploy code with unresolved critical findings

## Phase 3 — Rollback
- Restore previous scanning configurations
- Re-enable disabled scanning features
- Re-run scans after configuration changes
- Verify scanning functionality after changes
- Document scanning issues and findings
- Update scanning procedures

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Data Sensitivity

data_sensitivity:
  - command: "describe-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
