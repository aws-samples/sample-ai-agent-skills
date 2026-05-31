---
title: "C2 — MWAA Web Server Access Issues"
description: "Diagnose web server accessibility problems in MWAA"
status: active
severity: MEDIUM
triggers:
  - "web server inaccessible"
  - "cannot access Airflow UI"
  - "login failed"
  - "web server timeout"
  - "private web server"
owner: devops-agent
objective: "Identify why the MWAA web server is inaccessible and restore access"
context: "MWAA supports PUBLIC_ONLY and PRIVATE_ONLY web server access modes. Public mode creates an internet-facing endpoint. Private mode requires VPC endpoint access. Access is controlled by IAM policies."
---

## Phase 1 — Triage

MUST:
- Check web server access mode: `aws mwaa get-environment --name <env-name> --query 'Environment.WebserverAccessMode'`
- Check environment status: `aws mwaa get-environment --name <env-name> --query 'Environment.Status'`
- Check web server URL: `aws mwaa get-environment --name <env-name> --query 'Environment.WebserverUrl'`
- Verify IAM permissions for web login: `aws iam simulate-principal-policy --policy-source-arn <user-arn> --action-names airflow:CreateWebLoginToken --resource-arns <env-arn>`
- Test web server health: `aws mwaa create-web-login-token --name <env-name>`

SHOULD:
- If PRIVATE_ONLY, check VPC endpoint exists: `aws ec2 describe-vpc-endpoints --filters Name=service-name,Values=com.amazonaws.<region>.airflow.api Name=vpc-id,Values=<vpc-id>`
- Verify VPC endpoint security groups allow inbound HTTPS from client
- Check if web server logs show errors: `aws logs filter-log-events --log-group-name airflow-<env-name>-WebServer --filter-pattern "ERROR" --start-time <epoch-ms>`

MAY:
- Check browser console for CORS or certificate errors
- Verify DNS resolution of the web server URL

## Phase 2 — Remediate

MUST:
- Grant `airflow:CreateWebLoginToken` permission to users who need web access
- For PRIVATE_ONLY, create VPC endpoint for `com.amazonaws.<region>.airflow.api`
- Ensure VPC endpoint security groups allow HTTPS from client networks

SHOULD:
- Use IAM policies to control which environments users can access
- Configure VPN or Direct Connect for private web server access from on-premises

MAY:
- Switch to PUBLIC_ONLY if private access is not required (requires environment update)
- Set up a bastion host or proxy for private web server access

## Common Issues

- symptoms: "Cannot access Airflow UI — connection timeout"
  diagnosis: "Web server is PRIVATE_ONLY and no VPC endpoint or network path exists."
  resolution: "Create VPC endpoint for airflow.api or switch to PUBLIC_ONLY mode."

- symptoms: "Access denied when trying to log in"
  diagnosis: "IAM user/role lacks airflow:CreateWebLoginToken permission."
  resolution: "Grant the required IAM permission for the MWAA environment."

- symptoms: "Web server returns 503 error"
  diagnosis: "Environment is unhealthy or in UPDATING status."
  resolution: "Check environment status and wait for AVAILABLE. Check WebServer logs."

## Output Format

```yaml
root_cause: "web_server_access_issue — <specific_cause>"
evidence:
  - type: access_mode
    content: "<PUBLIC_ONLY or PRIVATE_ONLY>"
  - type: iam_permissions
    content: "<user permissions for web login>"
  - type: vpc_endpoint
    content: "<VPC endpoint status if private mode>"
severity: MEDIUM
mitigation:
  immediate: "Fix IAM permissions or VPC endpoint configuration"
  long_term: "Document access procedures for both public and private modes"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: MEDIUM — web server provides access to DAG configurations and logs
- Service Disruption Risk: LOW — web server issues do not affect DAG execution
- Compliance Impact: MEDIUM — web server access controls affect audit requirements

## Escalation Conditions
- Web server completely inaccessible for all users
- Authentication/authorization failures after IAM policy changes
- Web server returns 5xx errors persistently
- Web server access token generation fails
- Public web server endpoint exposed without proper access controls

## Data Sensitivity
- Web server login tokens — RESTRICTED
- Airflow UI displayed configurations — CONFIDENTIAL
- User access patterns — INTERNAL
- Web server access logs — INTERNAL
- Never share web server tokens or CLI login URLs

## Prohibited Actions
- Do NOT configure public web server access without network-level restrictions
- Do NOT share web server login tokens across users
- Do NOT disable IAM authentication for the web server
- Do NOT expose the web server endpoint in public documentation
- Do NOT grant Admin role in Airflow UI without proper authorization
- Do NOT bypass web server access controls for troubleshooting

## Phase 3 — Rollback
- Revoke compromised web server tokens immediately
- Restore IAM policies to re-enable web server access
- Revert web server access mode changes (public/private)
- Re-create web server login tokens if authentication is broken
- Verify user access levels are correct after any IAM changes
- Document access issues and resolution for security audit trail

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
  - "NEVER suggest disabling webserver authentication"
  - "NEVER suggest public network access mode for production"
  - "NEVER suggest overly broad Airflow execution role"
