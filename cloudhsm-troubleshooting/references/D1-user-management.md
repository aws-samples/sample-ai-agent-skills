---
title: "D1 — User Management"
description: "Diagnose and resolve CloudHSM user management and authentication issues"
status: active
severity: HIGH
triggers:
  - "user login failed"
  - "createUser error"
  - "user locked"
  - "HSM user management"
  - "PRECO user"
owner: devops-agent
objective: "Resolve HSM user management issues and restore proper authentication and authorization"
context: "CloudHSM has three user types: PRECO (Precrypto Officer — initial admin), CO (Crypto Officer — manages users), and CU (Crypto User — performs crypto operations). User management is done through cloudhsm_mgmt_util (CMU) or the CloudHSM CLI. Common issues include locked accounts, forgotten passwords, PRECO not being changed, and user synchronization across HSMs."
---

## Phase 1 — Triage

MUST:
- List all HSM users: `listUsers` (in CMU)
- Check if the PRECO user still exists (should be changed to CO on first use): look for user type PRECO in listUsers output
- Verify the user can log in: `loginHSM <user-type> <username> <password>` (in CMU)
- Check cluster state: `aws cloudhsmv2 describe-clusters --filters clusterIds=<cluster-id> --query 'Clusters[0].State'`
- Verify the CMU is connected to the cluster: `info` (in CMU)

SHOULD:
- Check if the user exists on all HSMs in the cluster (user sync)
- Verify the CO account used for user management is not the only CO
- Check CloudHSM client logs for authentication errors: `sudo journalctl -u cloudhsm-client --no-pager -n 50`

MAY:
- Review the number of users against HSM limits
- Check if any recent user management operations were performed via CloudTrail
- Verify user roles match the intended access pattern (CO vs CU)

## Phase 2 — Remediate

MUST:
- If PRECO still exists: change PRECO password to activate as CO: `changePswd PRECO admin <new-password>` (in CMU as PRECO)
- If CU password forgotten: reset via CO account: `changePswd CU <username> <new-password>` (in CMU as CO)
- If user needs to be created: `createUser <user-type> <username> <password>` (in CMU as CO)
- If user is locked: CO can reset the password to unlock the account

SHOULD:
- Create at least 2 CO accounts for redundancy
- Verify user creation is synchronized across all HSMs: `listUsers` on each HSM
- Update all applications with new credentials after password changes

MAY:
- Implement a password rotation schedule for HSM users
- Document all HSM user accounts and their purposes (without passwords)
- Set up alerts for failed login attempts via CloudHSM client log monitoring

## Common Issues

- symptoms: "loginHSM returns RET_USER_LOGIN_FAILURE"
  diagnosis: "Incorrect password, wrong user type specified, or account locked."
  resolution: "Verify username and user type (CO/CU). If password forgotten, reset via CO account."

- symptoms: "PRECO user still exists after cluster initialization"
  diagnosis: "The initial PRECO password was never changed to activate the first CO."
  resolution: "Log in as PRECO with the default password and change it: `changePswd PRECO admin <new-password>`."

- symptoms: "createUser fails with USER_ALREADY_EXISTS"
  diagnosis: "A user with that name already exists on the HSM."
  resolution: "Use `listUsers` to verify. Choose a different username or delete the existing user if appropriate."

- symptoms: "User exists on some HSMs but not others"
  diagnosis: "User synchronization failed, possibly due to an HSM being offline during creation."
  resolution: "The cluster will auto-sync when all HSMs are healthy. If not, delete and recreate the user."

## Output Format

```yaml
root_cause: "User management issue — <specific_cause>"
evidence:
  - type: user_list
    content: "<listUsers output>"
  - type: login_error
    content: "<login error message>"
severity: HIGH
mitigation:
  immediate: "Reset password or create required user accounts"
  long_term: "Implement user management procedures with CO redundancy and password rotation"
```

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (listUsers/info): GREEN — read-only"
  - "Phase 2 password changes: RED — changes authentication credentials"
  - "Phase 2 user creation/deletion: RED — modifies HSM access control"

## Escalation Conditions

escalation_conditions:
  - "All CO accounts are locked or passwords forgotten (cluster lockout risk)"
  - "PRECO default password may have been compromised"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "User management changes affect production cryptographic operations"
  - "Unauthorized user accounts are discovered on the HSM"

## Data Sensitivity

data_sensitivity:
  - command: "listUsers"
    sensitivity: HIGH
    contains: "HSM user accounts, types, and IDs"
  - command: "loginHSM"
    sensitivity: CRITICAL
    contains: "Authentication credentials — never log passwords"
  - command: "changePswd"
    sensitivity: CRITICAL
    contains: "Password change operation — never log new passwords"

## Prohibited Actions

prohibited_actions:
  - "NEVER log or display HSM user passwords in any output"
  - "NEVER delete the last CO account on the cluster"
  - "NEVER share CO credentials across teams without proper access controls"
  - "NEVER leave the PRECO default password unchanged in production"
  - "NEVER create CU accounts with weak or default passwords"

## Phase 3 — Rollback

- Before deleting users, document their username, type, and key ownership
- If a password change breaks applications, coordinate with application teams to update credentials
- If a user was accidentally deleted, recreate with the same username (keys owned by the deleted user are lost)
- If PRECO password change fails, the cluster may need to be restored from backup
- Verify all applications can authenticate after any user management changes
