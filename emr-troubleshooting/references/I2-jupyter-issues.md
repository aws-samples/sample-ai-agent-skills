---
title: "I2 — Jupyter Notebook Issues"
description: "Diagnose Jupyter notebook issues on EMR"
status: active
severity: MEDIUM
triggers:
  - "Jupyter not working"
  - "notebook connection failed"
  - "kernel error"
  - "JupyterHub issue"
owner: devops-agent
objective: "Identify and resolve Jupyter notebook issues on EMR"
context: "EMR supports JupyterHub and Jupyter Enterprise Gateway for notebook access. Issues include connectivity (port 9443), kernel failures, library installation, and authentication. JupyterHub runs on the master node."
---

## Phase 1 — Triage

MUST:
- Verify JupyterHub is installed: `aws emr describe-cluster --cluster-id <id> --query 'Cluster.Applications'`
- Check JupyterHub service status on master node
- Verify security group allows port 9443 (JupyterHub)
- Check JupyterHub logs on master node

SHOULD:
- Verify kernel status and availability
- Check for library installation errors
- Verify user authentication configuration
- Check master node resource utilization

MAY:
- Review Jupyter kernel gateway logs
- Check for browser compatibility issues

## Phase 2 — Remediate

MUST:
- Ensure JupyterHub application is included in cluster creation
- Open port 9443 in the master security group
- Verify JupyterHub service is running

SHOULD:
- Configure PAM or LDAP authentication for JupyterHub
- Install required libraries via bootstrap actions or conda
- Monitor master node resources

MAY:
- Use EMR Studio instead of direct JupyterHub access
- Create custom Jupyter kernels for specific workloads

## Common Issues

- symptoms: "Cannot connect to JupyterHub"
  diagnosis: "Port 9443 blocked or JupyterHub not installed."
  resolution: "Open port 9443 in master SG. Verify JupyterHub is in cluster applications."

- symptoms: "Kernel dies during execution"
  diagnosis: "Insufficient memory on master node or kernel OOM."
  resolution: "Use larger master instance type. Reduce notebook memory usage."

## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Open port 9443 in master security group | YELLOW | Opens network port — restrict source IPs |
| Verify JupyterHub service is running | GREEN | Verification — read-only |
| Configure PAM or LDAP authentication | YELLOW | Changes authentication — test thoroughly |
| Install libraries via bootstrap or conda | GREEN | Software installation — non-destructive |
| Use larger master instance type | RED | Requires cluster recreation — master is immutable |

## Escalation Conditions

- Cluster runs production Spark jobs
- Fix requires cluster termination and recreation
- Jupyter access failure blocks data science team productivity
- Master node resource exhaustion affects cluster stability

## Data Sensitivity

- **Classification: HIGH**
- Jupyter notebooks may contain sensitive data analysis and query results
- JupyterHub authentication configuration reveals access control patterns
- Master node logs may contain user activity and data access details
- Kernel configurations expose data processing environments

## Prohibited Actions

- **NEVER** suggest terminating a cluster with running steps without saving state
- **NEVER** suggest disabling encryption on a cluster or its storage
- **NEVER** suggest disabling JupyterHub authentication for convenience
- **NEVER** suggest opening port 9443 to 0.0.0.0/0 without IP restriction

## Phase 3 — Rollback

| State-Changing Action | Rollback Step |
|---|---|
| Opened port 9443 in security group | Remove the security group rule if no longer needed |
| Configured PAM/LDAP authentication | Revert to previous authentication method |
| Installed libraries via conda | Uninstall libraries if they cause conflicts |

## Output Format

```yaml
root_cause: "jupyter — <specific_cause>"
evidence:
  - type: jupyter_status
    content: "<JupyterHub service status>"
  - type: connectivity
    content: "<port and security group configuration>"
severity: MEDIUM
mitigation:
  immediate: "Fix Jupyter connectivity or kernel issues"
  long_term: "Consider EMR Studio for managed notebook experience"
```

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
  - "NEVER suggest disabling encryption"
  - "NEVER suggest public subnet placement for clusters"
  - "NEVER suggest removing Kerberos authentication"
