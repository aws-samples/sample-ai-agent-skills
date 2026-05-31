---
title: "B1 — Client Connectivity"
description: "Diagnose and resolve CloudHSM client connectivity issues"
status: active
severity: CRITICAL
triggers:
  - "client connection failed"
  - "unable to connect to HSM"
  - "connection timeout"
  - "cloudhsm client error"
owner: devops-agent
objective: "Restore connectivity between CloudHSM client instances and the HSM cluster"
context: "CloudHSM clients communicate with HSMs over TCP ports 2223-2225 via ENIs in the VPC. Connectivity failures can stem from security group misconfiguration, NACL rules, route table issues, client SDK misconfiguration, or HSM unavailability. The client SDK must be configured with the cluster's IP addresses and proper TLS certificates."
---

## Phase 1 — Triage

MUST:
- Verify HSM cluster and HSM states: `aws cloudhsmv2 describe-clusters --filters clusterIds=<cluster-id> --query 'Clusters[0].{State:State,Hsms:Hsms[*].{ID:HsmId,State:State,IP:EniIp}}'`
- Check client instance security group allows outbound to HSM ports: `aws ec2 describe-security-groups --group-ids <client-sg-id>`
- Check HSM security group allows inbound from client: `aws ec2 describe-security-groups --group-ids <hsm-sg-id>`
- Test network connectivity from client to HSM IP on port 2223: `nc -zv <hsm-eni-ip> 2223` (from client instance)
- Verify the CloudHSM client configuration file: `cat /opt/cloudhsm/etc/cloudhsm_client.cfg`

SHOULD:
- Check client SDK version: `/opt/cloudhsm/bin/cloudhsm_client --version` or `pip show cloudhsm`
- Verify the client instance is in the same VPC as the HSM cluster or has VPC peering configured
- Check route tables for the client subnet: `aws ec2 describe-route-tables --filters Name=association.subnet-id,Values=<client-subnet-id>`
- Verify NACLs are not blocking traffic: `aws ec2 describe-network-acls --filters Name=association.subnet-id,Values=<client-subnet-id>`

MAY:
- Check VPC flow logs for denied traffic between client and HSM ENIs
- Verify DNS resolution if using DNS names instead of IPs
- Check if the client instance has the correct IAM role for CloudHSM API calls

## Phase 2 — Remediate

MUST:
- If security group blocks traffic: add inbound rule to HSM SG for TCP 2223-2225 from client SG
- If client config has wrong IPs: update `/opt/cloudhsm/etc/cloudhsm_client.cfg` with correct HSM ENI IPs
- If client SDK is outdated: update to the latest version: `sudo yum install -y aws-cloudhsm-client` or `sudo apt-get install -y aws-cloudhsm-client`
- If HSM is not ACTIVE: check HSM provisioning status and reprovision if needed

SHOULD:
- Restart the CloudHSM client daemon after configuration changes: `sudo systemctl restart cloudhsm-client`
- Verify connectivity after remediation: `/opt/cloudhsm/bin/cloudhsm_mgmt_util /opt/cloudhsm/etc/cloudhsm_mgmt_util.cfg`
- Update client configuration to include all HSM IPs for failover

MAY:
- Configure the client for automatic HSM IP discovery using the cluster ID
- Set up monitoring for client-HSM connectivity
- Document the network path between client and HSM for future reference

## Common Issues

- symptoms: "Connection timeout when connecting to HSM"
  diagnosis: "Security group or NACL blocking TCP 2223-2225 between client and HSM ENI."
  resolution: "Update security group to allow inbound TCP 2223-2225 from the client security group."

- symptoms: "Client connects but authentication fails"
  diagnosis: "Client TLS certificate mismatch or HSM user credentials incorrect."
  resolution: "Re-download the cluster CSR and issue a new client certificate, or verify HSM user credentials."

- symptoms: "Client loses connection intermittently"
  diagnosis: "HSM failover or network instability. Single-HSM cluster has no redundancy."
  resolution: "Add a second HSM in a different AZ. Verify client SDK is configured with all HSM IPs."

- symptoms: "Client works from one subnet but not another"
  diagnosis: "Route table or NACL differences between subnets."
  resolution: "Compare route tables and NACLs between working and non-working subnets."

## Output Format

```yaml
root_cause: "Client connectivity failure — <specific_cause>"
evidence:
  - type: network_test
    content: "<nc/telnet connectivity test results>"
  - type: security_groups
    content: "<relevant security group rules>"
severity: CRITICAL
mitigation:
  immediate: "Fix security group/NACL rules or update client configuration"
  long_term: "Implement network monitoring and multi-AZ HSM deployment"
```

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 security group modifications: YELLOW — state-changing but recoverable"
  - "Phase 2 client SDK update: YELLOW — may require application restart"

## Escalation Conditions

escalation_conditions:
  - "Connectivity loss affects production cryptographic operations"
  - "Security group changes could impact other services sharing the same SG"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Issue involves VPC peering or Transit Gateway configuration"
  - "Client certificate reissuance is required"

## Data Sensitivity

data_sensitivity:
  - command: "cat cloudhsm_client.cfg"
    sensitivity: HIGH
    contains: "HSM IP addresses and TLS certificate paths"
  - command: "describe-security-groups"
    sensitivity: MEDIUM
    contains: "Network access rules and IP ranges"
  - command: "VPC flow logs"
    sensitivity: HIGH
    contains: "Network traffic metadata including source/destination IPs"

## Prohibited Actions

prohibited_actions:
  - "NEVER open HSM ports (2223-2225) to 0.0.0.0/0"
  - "NEVER share or log HSM user credentials"
  - "NEVER modify HSM ENIs directly through EC2 APIs"
  - "NEVER disable TLS for client-HSM communication"

## Phase 3 — Rollback

- Before modifying security groups, document current rules: `aws ec2 describe-security-groups --group-ids <sg-id> > sg-backup.json`
- If security group changes break other services, revert to the documented rules
- If client configuration changes cause issues, restore from backup: `cp /opt/cloudhsm/etc/cloudhsm_client.cfg.bak /opt/cloudhsm/etc/cloudhsm_client.cfg`
- If client SDK update causes incompatibility, downgrade to the previous version
- Verify all dependent applications can connect after any changes
