# AWS CloudHSM Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any AWS CloudHSM issue.

## Guardrail 1: AWS Cannot Recover Keys or Reset HSM Users
CloudHSM is single-tenant. AWS has NO access to your HSM contents. If all crypto officer (CO) credentials are lost, AWS cannot reset them. If keys are deleted, AWS cannot recover them. Backups are the only recovery mechanism. Never suggest contacting AWS Support for key or user recovery.

## Guardrail 2: HSM Users Are Not IAM Users
HSM users (PRECO, CO, CU) are managed INSIDE the HSM via cloudhsm-cli, not via IAM. IAM controls the management plane (create/delete clusters, describe operations). HSM users control the data plane (key operations, crypto operations). Do not confuse the two authentication systems.

## Guardrail 3: Client Connectivity Requires Specific Ports
CloudHSM client communicates with HSMs on ports 2223-2225. Security groups must allow inbound on these ports from client instances to HSM ENIs. The cluster security group is auto-created but may need modification. Port 443 is NOT used for HSM data plane operations.

## Guardrail 4: Single-HSM Clusters Have No Failover
A cluster with one HSM has no high availability. If that HSM fails, all operations stop until a new HSM is provisioned (can take ~20 minutes). Production clusters should have 2+ HSMs in different AZs. The client SDK automatically fails over between HSMs in a cluster.

## Guardrail 5: Keys Sync Automatically Across Cluster HSMs
When a key is created on one HSM, it automatically synchronizes to all other HSMs in the cluster. This happens within seconds. Do not manually copy keys between HSMs. If sync fails, check HSM status and cluster health. New HSMs added to a cluster receive all existing keys.

## Guardrail 6: Cluster Must Be Initialized Before Use
A new cluster must be initialized: (1) Get the CSR from the cluster, (2) Sign it with your CA, (3) Upload the signed certificate and CA chain, (4) Initialize the cluster. Only then can you create HSM users and keys. Skipping initialization leaves the cluster in UNINITIALIZED state.

## Guardrail 7: Backups Contain All Users and Keys
CloudHSM backups are complete snapshots of HSM state including all users, keys, and configuration. Restoring a backup creates a new cluster with identical contents. Backups are encrypted and stored in AWS-managed S3. Automatic daily backups are enabled by default.

## Guardrail 8: PKCS#11, JCE, and OpenSSL Use Different Libraries
Each integration (PKCS#11, JCE, OpenSSL) uses a different library from the CloudHSM SDK. They share the same client daemon but have separate configuration. Version mismatches between components cause failures. Always install matching versions of all SDK components.

## Guardrail 9: Client Configuration Must Point to Cluster
The CloudHSM client configuration file must contain the cluster's HSM IP addresses or the cluster ID for auto-discovery. Incorrect configuration causes connection failures. After adding/removing HSMs, update client configuration or use cluster ID for automatic discovery.

## Guardrail 10: CU Owns Keys, CO Manages Users
Crypto Users (CU) create and own keys. Only the key owner (or shared users) can use a key. Crypto Officers (CO) manage users but cannot perform crypto operations. A CO cannot use keys created by a CU. Key sharing between CUs must be explicitly configured.

## Guardrail 11: VPC Peering and Transit Gateway Require Route Configuration
CloudHSM ENIs are in the cluster's VPC. Clients in other VPCs need VPC peering or Transit Gateway with proper route tables. Security groups must reference the peered VPC CIDR. DNS resolution must work across VPCs for cluster discovery.

## Guardrail 12: CloudHSM SDK Version Must Match HSM Firmware
The CloudHSM client SDK version must be compatible with the HSM firmware version. Major version mismatches cause connection failures or unexpected behavior. Check compatibility matrix before upgrading SDK or when HSM firmware is updated by AWS.
