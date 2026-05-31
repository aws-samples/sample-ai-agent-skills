---
name: nitro-enclaves-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Nitro Enclaves
  problems by analyzing enclave creation, image builds, startup errors,
  vsock communication, KMS integration, attestation, resource allocation,
  ACM integration, cryptographic attestation, and parent instance issues
  following structured runbooks.
  Activate when: enclave creation failures, image build errors, enclave
  start problems, vsock communication issues, KMS integration failures,
  attestation errors, memory or CPU allocation issues, ACM integration
  problems, or the user says something is wrong with Nitro Enclaves.
compatibility: >
  Requires AWS CLI or SDK access with ec2, kms, acm, nitro-cli on the
  parent instance, cloudwatch, and cloudtrail permissions. Parent instance
  must be a Nitro-based instance type with enclave support enabled.
---

# AWS Nitro Enclaves Diagnostics

## When to use

Any AWS Nitro Enclaves investigation — enclave creation, image building, startup errors, vsock communication, KMS integration, attestation, memory/CPU allocation, ACM integration, cryptographic attestation, or parent instance configuration.

## Investigation workflow

### Step 1 — Collect and triage

```
nitro-cli describe-enclaves
nitro-cli describe-eif --eif-path <path-to-eif>
aws ec2 describe-instances --instance-ids <id> --query 'Reservations[*].Instances[*].{Type:InstanceType,EnclaveOptions:EnclaveOptions}'
```

### Step 2 — Domain deep dive

```
nitro-cli console --enclave-id <enclave-id>
aws kms describe-key --key-id <key-id> --query '{KeyState:KeyMetadata.KeyState,KeyPolicy:KeyMetadata.Description}'
dmesg | grep -i nitro
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=kms.amazonaws.com --max-results 20
cat /var/log/nitro_enclaves/nitro_enclaves.log
aws ec2 describe-instance-types --instance-types <type> --query 'InstanceTypes[*].{vCPUs:VCpuInfo.DefaultVCpus,Memory:MemoryInfo.SizeInMiB}'
```

Read `references/guardrails.md` before concluding on any Nitro Enclaves issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `nitro-cli run-enclave` | Start an enclave |
| `nitro-cli describe-enclaves` | List running enclaves |
| `nitro-cli console` | View enclave console output |
| `nitro-cli describe-eif` | Inspect enclave image file |
| `nitro-cli terminate-enclave` | Stop an enclave |
| `nitro-cli build-enclave` | Build enclave image from Docker |
| `ec2 describe-instances` | Check parent instance config |
| `kms describe-key` | Check KMS key configuration |

## Gotchas: AWS Nitro Enclaves

- Enclaves run on Nitro-based instances ONLY. The instance must have `EnclaveOptions.Enabled: true` set at launch time. This cannot be changed after launch — the instance must be terminated and relaunched.
- Enclaves share CPU and memory with the parent instance. Allocating too much to the enclave starves the parent. At least 2 vCPUs and some memory must remain for the parent.
- Vsock is the ONLY communication channel between parent and enclave. There is no network, disk, or shared memory access. All data must flow through vsock.
- KMS attestation-based policies use PCR values (Platform Configuration Registers) from the enclave image. Changing the enclave image changes PCR values, breaking KMS policies that reference specific PCRs.
- Enclave images (EIF files) are built from Docker images. The Docker image must be self-contained — no external dependencies can be fetched at runtime since enclaves have no network access.
- ACM for Nitro Enclaves allows enclaves to use TLS certificates without exposing private keys to the parent instance. It requires the acm-for-nitro-enclaves helper running on the parent.
- Enclaves have no persistent storage. All state is lost when the enclave terminates. Applications must handle state persistence through vsock communication with the parent.

## Anti-hallucination rules

1. Always cite specific enclave IDs, EIF paths, instance IDs, or CLI output as evidence.
2. Enclaves have NO network access. Never suggest network-based solutions for enclave communication.
3. Vsock is the ONLY communication channel. Never suggest shared memory, files, or network sockets between parent and enclave.
4. PCR values are deterministic from the image. Never suggest PCR values can be set manually.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 12 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Enclave Lifecycle | A1-A2 | Creation failures, image build |
| B — Startup & Communication | B1-B2 | Start errors, vsock communication |
| C — KMS & Attestation | C1-C2 | KMS integration errors, attestation failures |
| D — Resource Allocation | D1-D2 | Memory allocation, CPU allocation |
| E — Integrations | E1-E2 | ACM integration, cryptographic attestation |
| F — Parent Instance | F1 | Parent instance issues |
| Z — Catch-All | Z1 | General troubleshooting |
