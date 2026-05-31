# AWS Snow Family Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Snow Family issue.

## Guardrail 1: Snowball Edge Client vs AWS CLI Are Different Tools
The Snowball Edge client runs locally against the device endpoint (https://<device-ip>:8243). AWS CLI snowball commands run against the AWS API. Device-level operations (describe-device, start-service) require the Snowball Edge client, not the AWS CLI.

## Guardrail 2: Manifest and Unlock Code Are Job-Specific
Each job has a unique manifest file and unlock code. They are generated when the device ships and expire when the job completes. You cannot use one job's manifest to unlock another job's device. Always verify you have the correct manifest for the specific job.

## Guardrail 3: S3 Adapter Is Not Amazon S3
The S3 adapter on Snow devices supports a subset of S3 API operations. Features like S3 versioning, lifecycle policies, cross-region replication, S3 Select, and bucket policies are NOT supported. Do not assume full S3 API compatibility.

## Guardrail 4: EC2 on Snow Uses Device-Specific Instance Types
EC2 instances on Snowball Edge use sbe1 instance types (sbe1.small, sbe1.medium, sbe1.large, sbe1.xlarge, sbe1.2xlarge, sbe1.4xlarge). Standard EC2 instance types (t3, m5, etc.) are not available. AMIs must be specifically created for Snow devices.

## Guardrail 5: Cluster Operations Are Immutable After Creation
Snowball Edge clusters require a minimum of 5 devices. Once a cluster is created, you cannot add or remove devices. If a device in a cluster fails, the cluster continues with reduced capacity but the device cannot be replaced without creating a new cluster.

## Guardrail 6: Device Network Configuration Is Local Only
Network configuration on Snow devices (IP address, subnet, gateway) is set locally using the Snowball Edge client or LCD display. These settings are not managed through the AWS console or API. Incorrect network settings require physical access to the device.

## Guardrail 7: OTA Updates Require Internet Connectivity
Over-the-air (OTA) updates for Snowball Edge require the device to have internet connectivity to reach AWS endpoints. Devices in air-gapped or disconnected environments cannot receive OTA updates. Plan updates before disconnecting devices.

## Guardrail 8: Data Transfer Speed Depends on Network Interface
Data transfer speed is limited by the network interface used. RJ45 (1 Gbps), SFP+ (10 Gbps), and QSFP28 (25/40/50/100 Gbps) have different maximum throughputs. The bottleneck is often the network interface, not the device storage.

## Guardrail 9: Snowcone Has Different Capabilities Than Snowball Edge
Snowcone has less storage (14 TB), fewer compute resources, and supports fewer services than Snowball Edge. Snowcone supports DataSync agent for data transfer but has limited EC2 compute capability. Do not assume Snowball Edge features are available on Snowcone.

## Guardrail 10: Device Return Shipping Has Strict Timelines
Snow devices must be returned within the job's allotted time (typically 10-15 days for Snowball, 30 days for Snowball Edge). Late returns incur daily charges. Devices not returned may be reported as lost. Always track return shipping deadlines.

## Guardrail 11: NFS File Interface Has Size and Name Limitations
The NFS file interface on Snow devices has file size limits, path length limits, and character restrictions in file names. Files exceeding these limits will fail to transfer. Verify file characteristics before starting large transfers.

## Guardrail 12: Data on Device Is Encrypted and Tamper-Protected
All data on Snow devices is encrypted with KMS keys. If the device is tampered with or the encryption key is deleted, data is irrecoverable. The device performs a secure erase after data import to S3. Never attempt to physically access device storage.
