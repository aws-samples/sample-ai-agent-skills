# AWS Nitro Enclaves Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Nitro Enclaves issue.

## Guardrail 1: Enclave Support Must Be Enabled at Launch
The parent EC2 instance must have `EnclaveOptions.Enabled: true` set at launch time. This setting cannot be changed on a running instance. If enclave support is not enabled, the instance must be terminated and relaunched with the option enabled.

## Guardrail 2: Enclaves Have No Network Access
Enclaves are completely isolated from the network. They cannot make HTTP calls, DNS lookups, or any network communication. The ONLY communication channel is vsock (virtual socket) between the enclave and its parent instance. All external communication must be proxied through the parent.

## Guardrail 3: Vsock Is the Only Communication Channel
Vsock provides a socket-based communication channel between the parent instance and the enclave. There is no shared memory, shared filesystem, or network path. Applications must implement a vsock-based protocol for all data exchange between parent and enclave.

## Guardrail 4: PCR Values Are Deterministic and Image-Specific
Platform Configuration Register (PCR) values are cryptographic measurements of the enclave image. They are deterministic — the same image always produces the same PCR values. Changing the image (even slightly) changes the PCR values. KMS policies referencing PCR values must be updated when the enclave image changes.

## Guardrail 5: Resource Allocation Is Shared with Parent
Enclaves use CPU and memory from the parent instance. The parent must retain at least 2 vCPUs and sufficient memory for its own operation. Over-allocating resources to the enclave can crash the parent instance or prevent the enclave from starting.

## Guardrail 6: Enclaves Have No Persistent Storage
Enclaves have no disk access. All data in the enclave is in-memory and lost when the enclave terminates. Applications requiring persistence must send data to the parent via vsock for storage. Sensitive data should be encrypted before leaving the enclave.

## Guardrail 7: Only Nitro-Based Instances Support Enclaves
Not all EC2 instance types support Nitro Enclaves. The instance must be a Nitro-based type (most 5th generation and newer). Check instance type support: aws ec2 describe-instance-types --filters Name=enclave-support,Values=true.

## Guardrail 8: KMS Attestation Requires Specific Key Policy
For KMS integration with attestation, the KMS key policy must include a condition that checks the enclave's attestation document. The condition uses `kms:RecipientAttestation:PCR0` (and optionally other PCRs). Without this condition, KMS cannot verify the enclave identity.

## Guardrail 9: Docker Image Must Be Self-Contained
Enclave images are built from Docker images. Since enclaves have no network access, the Docker image must contain all dependencies, libraries, and data needed at runtime. Any attempt to download packages or fetch data at startup will fail silently inside the enclave.

## Guardrail 10: ACM Integration Requires Helper Service
ACM for Nitro Enclaves requires the `acm-for-nitro-enclaves` helper service running on the parent instance. This service fetches certificates from ACM and delivers them to the enclave via vsock. The enclave itself cannot access ACM directly.

## Guardrail 11: Enclave Console Output Is the Primary Debug Tool
Since enclaves have no network or SSH access, the `nitro-cli console` command is the primary debugging tool. It shows stdout/stderr from the enclave application. Applications should log diagnostic information to stdout for troubleshooting.

## Guardrail 12: Multiple Enclaves Per Instance Are Not Supported
Each parent instance can run only ONE enclave at a time. Attempting to start a second enclave while one is running will fail. Terminate the existing enclave before starting a new one.
