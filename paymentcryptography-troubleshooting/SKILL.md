---
name: paymentcryptography-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Payment
  Cryptography problems by analyzing key creation, import/export,
  encryption/decryption, PIN operations, MAC verification, key
  aliases, access control, audit logging, compliance, and key
  rotation using structured runbooks.
  Activate when: key creation failures, key import errors,
  encryption/decryption failures, PIN operation issues, MAC
  verification errors, key alias problems, access control issues,
  audit logging problems, compliance configuration errors, key
  rotation failures, or the user says something is wrong with
  Payment Cryptography.
compatibility: >
  Requires AWS CLI or SDK access with payment-cryptography,
  payment-cryptography-data, iam, cloudtrail, and cloudwatch
  permissions.
---

# AWS Payment Cryptography Diagnostics

## When to use

Any AWS Payment Cryptography investigation — key creation, key import/export, encryption/decryption operations, PIN operations, MAC verification, key aliases, access control, audit logging, compliance configuration, or key rotation.

## Investigation workflow

### Step 1 — Collect and triage

```
aws payment-cryptography list-keys --query 'Keys[*].{KeyArn:KeyArn,KeyState:KeyAttributes.KeyClass,Enabled:Enabled}'
aws payment-cryptography list-aliases --query 'Aliases[*].{AliasName:AliasName,KeyArn:KeyArn}'
aws payment-cryptography get-key --key-identifier <key-arn>
```

### Step 2 — Domain deep dive

```
aws payment-cryptography get-parameters-for-import --key-material-type TR31_KEY_BLOCK --wrapping-key-algorithm RSA_2048
aws payment-cryptography get-parameters-for-export --key-material-type TR31_KEY_BLOCK --signing-key-algorithm RSA_2048
aws payment-cryptography-data encrypt-data --key-identifier <key-arn> --plain-text <hex-data> --encryption-attributes '{"Symmetric":{"Mode":"CBC"}}'
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=payment-cryptography.amazonaws.com --max-results 20
aws payment-cryptography list-tags-for-resource --resource-arn <key-arn>
aws payment-cryptography get-alias --alias-name <alias>
```

Read `references/guardrails.md` before concluding on any Payment Cryptography issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `payment-cryptography list-keys` | List payment keys |
| `payment-cryptography get-key` | Get key details |
| `payment-cryptography list-aliases` | List key aliases |
| `payment-cryptography-data encrypt-data` | Encrypt payment data |
| `payment-cryptography-data decrypt-data` | Decrypt payment data |
| `payment-cryptography-data generate-pin-data` | Generate PIN data |
| `payment-cryptography-data verify-mac` | Verify MAC |

## Gotchas: AWS Payment Cryptography

- Payment Cryptography uses TWO API endpoints: control plane (payment-cryptography) for key management and data plane (payment-cryptography-data) for cryptographic operations. IAM permissions are separate.
- Keys have specific key classes (SYMMETRIC_KEY, ASYMMETRIC_KEY_PAIR, PRIVATE_KEY, PUBLIC_KEY) and usage attributes. A key created for encryption cannot be used for MAC operations.
- Key import uses TR-31 key block format with wrapping keys. The wrapping key must be established first via get-parameters-for-import.
- PIN operations require specific key types: PIN Encryption Key (PEK) for PIN blocks, PIN Verification Key (PVK) for PIN verification.
- MAC operations require MAC keys (HMAC or CMAC). Using the wrong key type returns cryptographic errors, not permission errors.
- Key aliases provide friendly names but are NOT the same as key ARNs. Some operations require ARNs, not aliases.
- Key deletion has a configurable waiting period (7-30 days). Keys scheduled for deletion can be cancelled during this window.

## Anti-hallucination rules

1. Always cite specific key ARNs, alias names, or API responses as evidence.
2. Control plane and data plane are separate services with separate IAM permissions. Never conflate them.
3. Key usage attributes are enforced. Never claim a key can be used for operations outside its designated usage.
4. PIN and MAC operations require specific key types. Never suggest using generic encryption keys.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 14 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Key Management | A1-A2 | Key creation failures, key import errors |
| B — Cryptographic Ops | B1-B2 | Encryption failures, decryption errors |
| C — Payment Ops | C1-C2 | PIN operations, MAC verification |
| D — Key Lifecycle | D1-D2 | Key alias issues, key export errors |
| E — Security | E1-E2 | Access control, audit logging |
| F — Compliance | F1-F2 | Compliance configuration, key rotation |
| G — Integration | G1 | Integration issues |
| Z — Catch-All | Z1 | General troubleshooting |
