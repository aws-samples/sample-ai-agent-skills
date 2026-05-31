# D2 — Migration Hub Analysis Configuration Issues

## Symptoms
- Analysis configuration errors
- Source code analysis failures
- Database analysis issues

## Triage Commands

```bash
aws migrationhub-strategy get-portfolio-preferences
aws migrationhub-strategy list-analyzable-servers --max-results 20
aws migrationhub-strategy list-application-components --max-results 20
```

## Common Causes

1. **Source code repo inaccessible** — Credentials or network issues
2. **Unsupported language/framework** — Limited language support for analysis
3. **Database connection failed** — Can't connect to analyze database schema
4. **Collector not configured** — Analysis requires Strategy collector
5. **Permissions insufficient** — Collector needs read access to source code/DB

## Resolution

- Configure source code analysis with valid repository credentials
- Check supported languages and frameworks in documentation
- For database analysis, ensure collector can connect to database
- Verify collector has appropriate network access and credentials
- Review analysis results per component: `aws migrationhub-strategy get-application-component-details --application-component-id <id>`


## Safety Ratings

| Aspect | Rating | Notes |
|--------|--------|-------|
| Data Risk | Medium | Operations may affect service configurations and data access patterns |
| Service Impact | High | Incorrect changes can disrupt active workloads and dependent services |
| Recovery Complexity | Medium | Most actions are reversible but may require coordinated rollback procedures |
| Blast Radius | Medium | Impact typically scoped to the affected resource and its direct dependents |

## Escalation Conditions

- **Escalate immediately** if the issue involves potential data loss or corruption
- **Escalate immediately** if a production workload is impacted and cannot be restored within the agreed SLA
- **Escalate immediately** if the root cause is unclear after initial diagnostic steps are exhausted
- **Escalate to service team** if the issue appears to be a platform-level bug or undocumented behavior
- **Escalate to security team** if unauthorized access, policy violations, or credential exposure is suspected

## Data Sensitivity

- Verify data classification before performing any read or export operations
- Ensure all troubleshooting outputs (logs, configurations, snapshots) are handled according to the customer's data handling policy
- Mask or redact personally identifiable information (PII) in all shared diagnostics
- Do not copy production data to non-production environments without explicit authorization
- Retain troubleshooting artifacts only for the duration required and dispose of them securely

## Prohibited Actions

- **NEVER** delete migration tasks or application groupings that are actively tracking in-progress migrations
- **NEVER** change the home region setting without understanding the impact on all integrated migration tools
- **NEVER** remove discovery agent data or imported server records without confirming migration planning is complete
- **NEVER** modify Refactor Spaces routing configurations on production environments without a rollback plan
- **NEVER** override strategy recommendations without documenting the business justification

## Phase 3 � Rollback

1. **Identify rollback trigger**: Determine if the applied change has caused degradation, unexpected behavior, or failed validation checks
2. **Preserve current state**: Capture logs, configuration snapshots, and resource states before initiating rollback
3. **Execute rollback steps**: Revert configuration changes in reverse order of application, verifying each step
4. **Validate service recovery**: Confirm that the service has returned to its pre-change operational state using health checks and functional tests
5. **Document rollback**: Record the rollback trigger, steps taken, time to recovery, and any residual issues for post-incident review
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
