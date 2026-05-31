---
title: "G1 — CloudHSM SDK Issues"
description: "Diagnose and resolve CloudHSM SDK installation, compatibility, and runtime issues"
status: active
severity: MEDIUM
triggers:
  - "SDK installation failed"
  - "SDK version mismatch"
  - "PKCS#11 error"
  - "SDK compatibility"
  - "client SDK upgrade"
owner: devops-agent
objective: "Resolve CloudHSM SDK installation and runtime issues across all supported interfaces"
context: "CloudHSM provides multiple SDK interfaces: PKCS#11, OpenSSL Dynamic Engine, JCE provider, and the CloudHSM CLI. SDK versions must be compatible with the HSM firmware version. SDK 5 (Client SDK 5) is the latest major version with improved features. Upgrading from SDK 3 to SDK 5 requires careful migration. Common issues include installation failures, version conflicts, library path problems, and API compatibility."
---

## Phase 1 — Triage

MUST:
- Check installed SDK packages: `rpm -qa | grep cloudhsm` or `dpkg -l | grep cloudhsm`
- Verify SDK version: `/opt/cloudhsm/bin/configure --version` or check package version
- Check the CloudHSM client daemon status: `sudo systemctl status cloudhsm-client`
- Verify library files exist: `ls -la /opt/cloudhsm/lib/`
- Check system logs for SDK errors: `sudo journalctl -u cloudhsm-client --no-pager -n 50`

SHOULD:
- Verify OS compatibility with the installed SDK version
- Check if multiple SDK versions are installed simultaneously
- Verify the SDK configuration: `cat /opt/cloudhsm/etc/cloudhsm_client.cfg`

MAY:
- Check the AWS CloudHSM release notes for known issues with the installed version
- Verify the PKCS#11 library is loadable: `pkcs11-tool --module /opt/cloudhsm/lib/libcloudhsm_pkcs11.so -L`
- Review application-specific SDK integration logs

## Phase 2 — Remediate

MUST:
- If SDK not installed: install the appropriate package: `sudo yum install aws-cloudhsm-client aws-cloudhsm-pkcs11 aws-cloudhsm-jce` (for SDK 3) or `sudo yum install aws-cloudhsm-cli aws-cloudhsm-pkcs11 aws-cloudhsm-jce` (for SDK 5)
- If version mismatch: upgrade to the compatible version: `sudo yum update aws-cloudhsm-client`
- If libraries missing: reinstall the SDK package: `sudo yum reinstall aws-cloudhsm-client`
- If configuration is corrupted: reconfigure: `/opt/cloudhsm/bin/configure -a <hsm-ip>`

SHOULD:
- Restart the client daemon after SDK changes: `sudo systemctl restart cloudhsm-client`
- Verify all SDK interfaces work after update (PKCS#11, JCE, OpenSSL engine)
- Test application connectivity after SDK changes

MAY:
- Plan migration from SDK 3 to SDK 5 for improved features and support
- Set up automated SDK updates via configuration management (Ansible, Chef, etc.)
- Document the SDK version requirements for each application

## Common Issues

- symptoms: "Package installation fails with dependency errors"
  diagnosis: "Missing OS dependencies or conflicting package versions."
  resolution: "Install dependencies first: `sudo yum install -y libjson-c-dev` or equivalent. Remove conflicting packages."

- symptoms: "PKCS#11 library returns CKR_GENERAL_ERROR on initialization"
  diagnosis: "CloudHSM client daemon not running or configuration invalid."
  resolution: "Start the client daemon and verify configuration points to active HSMs."

- symptoms: "SDK 5 migration breaks existing application"
  diagnosis: "API changes between SDK 3 and SDK 5 — some functions renamed or behavior changed."
  resolution: "Review the SDK 5 migration guide. Update application code for API changes."

- symptoms: "configure command fails with 'cluster not found'"
  diagnosis: "Incorrect cluster ID or region in the configuration."
  resolution: "Verify cluster ID and region: `aws cloudhsmv2 describe-clusters` and reconfigure."

## Output Format

```yaml
root_cause: "SDK issue — <specific_cause>"
evidence:
  - type: sdk_version
    content: "<installed SDK package versions>"
  - type: error_details
    content: "<specific SDK error messages>"
severity: MEDIUM
mitigation:
  immediate: "Install, update, or reconfigure the SDK"
  long_term: "Standardize SDK versions across fleet and automate updates"
```

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (version/status/list): GREEN — read-only"
  - "Phase 2 SDK installation/update: YELLOW — may require application restart"
  - "Phase 2 SDK migration (v3 to v5): RED — may break existing applications"

## Escalation Conditions

escalation_conditions:
  - "SDK update breaks production applications"
  - "SDK version is end-of-life with known security vulnerabilities"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Multiple SDK interfaces fail simultaneously"
  - "SDK migration requires application code changes in production"

## Data Sensitivity

data_sensitivity:
  - command: "cloudhsm_client.cfg"
    sensitivity: HIGH
    contains: "HSM IP addresses and cluster configuration"
  - command: "SDK library files"
    sensitivity: MEDIUM
    contains: "Cryptographic library binaries"
  - command: "Application integration logs"
    sensitivity: MEDIUM
    contains: "SDK operation details and error messages"

## Prohibited Actions

prohibited_actions:
  - "NEVER downgrade SDK in production without testing in a staging environment"
  - "NEVER install SDK packages from unofficial sources"
  - "NEVER modify SDK library files directly"
  - "NEVER run SDK migration on all instances simultaneously — use rolling updates"
  - "NEVER skip application testing after SDK version changes"

## Phase 3 — Rollback

- Before SDK updates, document the current version: `rpm -qa | grep cloudhsm > sdk-versions-backup.txt`
- If SDK update breaks applications, downgrade to the previous version: `sudo yum downgrade aws-cloudhsm-client-<previous-version>`
- If configuration was changed, restore from backup: `cp /opt/cloudhsm/etc/cloudhsm_client.cfg.bak /opt/cloudhsm/etc/cloudhsm_client.cfg`
- If SDK migration (v3 to v5) fails, reinstall SDK 3 packages
- Verify all applications resume normal operation after rollback
