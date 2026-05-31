# AWS Payment Cryptography Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Payment Cryptography issue.

## Guardrail 1: Two Separate API Endpoints
Payment Cryptography has two API endpoints: control plane (payment-cryptography) for key management and data plane (payment-cryptography-data) for cryptographic operations. IAM permissions are separate for each. Do not confuse control plane and data plane permissions.

## Guardrail 2: Key Usage Attributes Are Enforced
Keys have specific usage attributes (encrypt/decrypt, generate/verify, wrap/unwrap). A key created for encryption cannot be used for MAC operations. Key usage is enforced at the service level — mismatched usage returns cryptographic errors, not permission errors.

## Guardrail 3: Key Import Requires Wrapping Key Exchange
Key import uses TR-31 key block format. Before importing, you must establish a wrapping key via get-parameters-for-import. The wrapping key is temporary and expires. Do not assume direct key material import is possible.

## Guardrail 4: PIN Operations Require Specific Key Types
PIN operations require specific key types: PIN Encryption Key (PEK) for PIN block encryption, PIN Verification Key (PVK) for PIN verification. Using incorrect key types causes cryptographic failures. Always verify key type before PIN operations.

## Guardrail 5: MAC Operations Require MAC Keys
MAC generation and verification require keys with MAC usage attributes (HMAC or CMAC). Encryption keys cannot be used for MAC operations. Verify key usage attributes match the intended operation.

## Guardrail 6: Key Deletion Has a Waiting Period
Key deletion has a configurable waiting period (7-30 days, default 7). During this period, the key is disabled but can be recovered by cancelling deletion. After the waiting period, deletion is permanent and irreversible.

## Guardrail 7: Aliases Are Not Key ARNs
Key aliases provide friendly names but are distinct from key ARNs. Some API operations accept aliases, others require full ARNs. Always check API documentation for accepted identifier types. Aliases can be reassigned to different keys.

## Guardrail 8: Key Export Requires Export Permissions
Key export uses TR-31 key block format with wrapping keys. Not all keys are exportable — the Exportable attribute must be set at creation time and cannot be changed later. Non-exportable keys cannot be exported regardless of IAM permissions.

## Guardrail 9: Cryptographic Errors vs Permission Errors
Cryptographic operation failures can be caused by wrong key type, wrong algorithm, or wrong parameters — not just permission issues. Always check key attributes and operation parameters before investigating IAM permissions.

## Guardrail 10: PCI DSS Compliance Requirements
Payment Cryptography is designed for PCI DSS compliance. Key management operations must follow PCI DSS requirements including dual control, split knowledge, and key rotation. Do not suggest shortcuts that violate PCI DSS.

## Guardrail 11: Regional Key Isolation
Keys are regional resources. A key created in one region cannot be used in another. Cross-region operations require key export and re-import. Verify the correct region before troubleshooting key-not-found errors.

## Guardrail 12: Data Plane Operations Are Latency-Sensitive
Payment cryptographic operations (encrypt, decrypt, PIN, MAC) are latency-sensitive. Throttling or high latency may indicate quota limits, not service failures. Check service quotas and request rates before assuming a bug.
