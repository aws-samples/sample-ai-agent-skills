---
title: "B2 — CloudHSM CLI Errors"
description: "Diagnose and resolve CloudHSM CLI and management utility errors"
status: active
severity: MEDIUM
triggers:
  - "cloudhsm_mgmt_util error"
  - "key_mgmt_util error"
  - "CloudHSM CLI failed"
  - "CMU connection error"
owner: devops-agent
objective: "Resolve CloudHSM CLI tool errors and restore management utility functionality"
context: "CloudHSM provides several CLI tools: cloudhsm_mgmt_util (CMU) for cluster management, key_mgmt_util (KMU) for key operations, and the newer CloudHSM CLI. Errors can arise from authentication failures, incorrect configuration, version mismatches, or HSM state issues. The CLI tools require an active connection to at least one HSM in the cluster."
---

## Phase 1 — Triage

MUST:
- Check CloudHSM client daemon status: `sudo systemctl status cloudhsm-client`
- Verify CLI tool version: `/opt/cloudhsm/bin/cloudhsm_mgmt_util --version`
- Check client configuration: `cat /opt/cloudhsm/etc/cloudhsm_client.cfg`
- Verify HSM cluster is active: `aws cloudhsmv2 describe-clusters --filters clusterIds=<cluster-id> --query 'Clusters[0].State'`
- Check client daemon logs: `sudo journalctl -u cloudhsm-client --no-pager -n 50`

SHOULD:
- Verify the HSM user credentials are correct by attempting login in CMU: `loginHSM CU <username> <password>`
- Check if the client certificate and key are valid and accessible
- Verify the CloudHSM SDK package is properly installed: `rpm -qa | grep cloudhsm` or `dpkg -l | grep cloudhsm`

MAY:
- Check for multiple CloudHSM client versions installed simultaneously
- Review recent package updates that may have affected the client
- Test with the newer CloudHSM CLI if using legacy CMU/KMU

## Phase 2 — Remediate

MUST:
- If client daemon is not running: `sudo systemctl start cloudhsm-client`
- If configuration is incorrect: update `/opt/cloudhsm/etc/cloudhsm_client.cfg` with correct HSM IPs and cluster ID
- If authentication fails: verify CU/CO credentials; reset password if needed via CMU with CO account
- If SDK is corrupted: reinstall the CloudHSM client package

SHOULD:
- Restart the client daemon after any configuration changes: `sudo systemctl restart cloudhsm-client`
- Update to the latest CloudHSM client SDK version for bug fixes
- Verify CLI operations work after remediation by listing users: `listUsers` in CMU

MAY:
- Migrate from legacy CMU/KMU to the newer CloudHSM CLI
- Set up log rotation for CloudHSM client logs
- Create a test CU account for connectivity verification

## Common Issues

- symptoms: "CMU shows 'Connection to server failed' on launch"
  diagnosis: "CloudHSM client daemon is not running or cannot reach the HSM."
  resolution: "Start the client daemon and verify network connectivity to HSM ENI IPs."

- symptoms: "loginHSM returns 'RET_USER_LOGIN_FAILURE'"
  diagnosis: "Incorrect username or password, or the user account is locked."
  resolution: "Verify credentials. If locked, use a CO account to reset the CU password."

- symptoms: "KMU commands return 'Cfm3xxx' error codes"
  diagnosis: "Key operation failed due to permissions, key state, or HSM capacity."
  resolution: "Check the specific Cfm3 error code in AWS documentation. Common: Cfm3FindKey (key not found), Cfm3CreateObject (permission denied)."

- symptoms: "CLI tools crash with segmentation fault"
  diagnosis: "Corrupted installation or incompatible library versions."
  resolution: "Reinstall the CloudHSM client package: `sudo yum reinstall aws-cloudhsm-client`."

## Output Format

```yaml
root_cause: "CloudHSM CLI error — <specific_cause>"
evidence:
  - type: cli_error
    content: "<exact error message from CLI tool>"
  - type: client_logs
    content: "<relevant client daemon log entries>"
severity: MEDIUM
mitigation:
  immediate: "Fix configuration/credentials and restart client daemon"
  long_term: "Implement automated client health checks and version management"
```

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (status/version/config checks): GREEN — read-only"
  - "Phase 2 client daemon restart: YELLOW — briefly interrupts HSM operations"
  - "Phase 2 password reset: RED — changes authentication credentials"

## Escalation Conditions

escalation_conditions:
  - "CO (Crypto Officer) account is locked and no other CO exists"
  - "CLI errors indicate HSM firmware issues"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Multiple CLI tools fail simultaneously across different client instances"
  - "Issue involves potential key material access concerns"

## Data Sensitivity

data_sensitivity:
  - command: "cat cloudhsm_client.cfg"
    sensitivity: HIGH
    contains: "HSM IP addresses and certificate paths"
  - command: "listUsers"
    sensitivity: HIGH
    contains: "HSM user accounts and roles"
  - command: "client daemon logs"
    sensitivity: MEDIUM
    contains: "Connection details and error messages"

## Prohibited Actions

prohibited_actions:
  - "NEVER log or display HSM user passwords in plaintext"
  - "NEVER delete CO accounts without ensuring another CO exists"
  - "NEVER run CLI tools with verbose/debug logging in production without redacting output"
  - "NEVER share CMU/KMU session output containing key handles or user lists"

## Phase 3 — Rollback

- Before changing client configuration, back up the current config: `cp /opt/cloudhsm/etc/cloudhsm_client.cfg /opt/cloudhsm/etc/cloudhsm_client.cfg.bak`
- If client daemon restart causes application disruption, coordinate with application teams
- If password reset was performed, update all applications using the affected credentials
- If SDK reinstallation fails, restore from the previous package version
- Verify all dependent applications can authenticate after credential changes
