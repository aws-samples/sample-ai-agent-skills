---
title: "C3 — NFS/SMB Location Issues"
description: "Diagnose NFS and SMB on-premises location configuration problems"
status: active
severity: HIGH
triggers:
  - "NFS mount failed"
  - "SMB connection failed"
  - "cannot connect to NFS server"
  - "SMB authentication error"
owner: devops-agent
objective: "Identify and resolve NFS or SMB location configuration issues for on-premises storage"
context: "NFS and SMB locations represent on-premises or self-managed storage accessed through a DataSync agent. The agent must be able to mount the NFS export or connect to the SMB share. Common issues include NFS export permissions, SMB credentials, NFS version mismatches, and network connectivity between agent and storage."
---

## Phase 1 — Triage

MUST:
- Check location configuration: `aws datasync describe-location-nfs --location-arn <location-arn>` or `aws datasync describe-location-smb --location-arn <location-arn>`
- Verify agent is ONLINE: `aws datasync describe-agent --agent-arn <agent-arn>`
- Test network connectivity from agent to NFS/SMB server (port 2049 for NFS, port 445 for SMB)
- Check NFS export permissions or SMB share permissions on the storage server

SHOULD:
- Verify NFS version compatibility (NFS3, NFS4_0, NFS4_1, or AUTOMATIC)
- Check SMB credentials are valid and have read/write access
- Verify the subdirectory/export path exists on the server

MAY:
- Test mount manually from the agent VM: `mount -t nfs <server>:<export> /mnt/test`
- Check firewall rules between agent and storage server

## Phase 2 — Remediate

MUST:
- Ensure NFS server exports allow the agent's IP address
- Verify SMB credentials have appropriate permissions
- Use correct NFS version matching server configuration

SHOULD:
- Create NFS location: `aws datasync create-location-nfs --server-hostname <server-ip> --on-prem-config AgentArns=<agent-arn> --subdirectory /export/path/ --mount-options Version=NFS4_0`
- Create SMB location: `aws datasync create-location-smb --server-hostname <server-ip> --on-prem-config AgentArns=<agent-arn> --subdirectory /share/ --user <user> --password <password> --domain <domain>`

MAY:
- Configure multiple agents for high availability
- Use specific NFS version instead of AUTOMATIC for reliability

## Common Issues

- symptoms: "NFS mount failed during task execution"
  diagnosis: "NFS server does not allow agent IP or NFS version mismatch."
  resolution: "Add agent IP to NFS exports. Specify correct NFS version in location config."

- symptoms: "SMB authentication failed"
  diagnosis: "Invalid credentials or domain configuration."
  resolution: "Verify username, password, and domain. Ensure user has read/write access to share."

- symptoms: "Location validation fails"
  diagnosis: "Agent cannot reach storage server or subdirectory does not exist."
  resolution: "Verify network connectivity. Check that the export/share path exists."

## Output Format

```yaml
root_cause: "nfs_smb_location_issue — <specific_cause>"
evidence:
  - type: location_config
    content: "<NFS/SMB location configuration>"
  - type: agent_connectivity
    content: "<agent to storage server connectivity>"
  - type: mount_permissions
    content: "<NFS export or SMB share permissions>"
severity: HIGH
mitigation:
  immediate: "Fix mount permissions and network connectivity"
  long_term: "Document storage server access requirements for DataSync agents"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | HIGH — On-premises storage may contain sensitive production data |
| Blast Radius | MEDIUM — NFS export or SMB share changes affect all clients |
| Rollback Complexity | LOW — Revert export/share permissions; recreate location |
| Compliance Scope | HIGH — On-premises data access must comply with data governance policies |

## Escalation Conditions

- Escalate to storage team if NFS export or SMB share changes are needed
- Escalate to security if SMB credentials need to be rotated
- Escalate to network team if firewall rules between agent and storage need modification

## Data Sensitivity

- **SMB credentials** (username, password, domain) are authentication secrets — store securely
- **NFS export permissions** control data access — changes affect all NFS clients
- **On-premises storage contents** may be highly sensitive

## PROHIBITED Actions

- Do NOT store SMB passwords in plain text or version control
- Do NOT modify NFS exports without coordinating with all NFS clients
- Do NOT open firewall rules broader than required (specific agent IP to storage server)
- Do NOT use root/admin credentials for DataSync SMB access — use a dedicated service account

## Phase 3 — Rollback

- Revert NFS export changes if they were modified during troubleshooting
- Revert SMB share permissions if they were changed
- Delete the misconfigured location and recreate with correct settings
- Restore firewall rules to previous state

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
