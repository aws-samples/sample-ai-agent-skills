---
title: "E1 — SSL/TLS Offload Issues"
description: "Diagnose and resolve CloudHSM SSL/TLS offload configuration and operation failures"
status: active
severity: CRITICAL
triggers:
  - "SSL offload failed"
  - "TLS handshake error"
  - "NGINX CloudHSM"
  - "Apache CloudHSM"
  - "OpenSSL engine"
owner: devops-agent
objective: "Restore SSL/TLS offload functionality using CloudHSM as the cryptographic backend"
context: "CloudHSM can be used for SSL/TLS offload with web servers like NGINX and Apache via the OpenSSL Dynamic Engine or PKCS#11 library. The private key remains on the HSM while the web server uses it for TLS handshakes. Issues arise from engine configuration, certificate-key mismatches, HSM connectivity during TLS operations, or library path problems."
---

## Phase 1 — Triage

MUST:
- Verify CloudHSM client is running: `sudo systemctl status cloudhsm-client`
- Check the OpenSSL engine is installed: `ls /opt/cloudhsm/lib/libcloudhsm_openssl_engine.so`
- Verify the web server configuration references the CloudHSM engine correctly
- Test the OpenSSL engine: `openssl engine -t cloudhsm`
- Check web server error logs: `sudo tail -50 /var/log/nginx/error.log` or `sudo tail -50 /var/log/httpd/error_log`

SHOULD:
- Verify the fake PEM key file exists and matches the HSM key: check the key file referenced in the web server config
- Confirm the certificate chain is complete and valid: `openssl verify -CAfile ca-chain.pem server.crt`
- Check if the CU credentials are set in the environment: verify `n3fips_password` environment variable
- Verify the HSM key handle matches the fake PEM file

MAY:
- Test TLS handshake manually: `openssl s_client -connect localhost:443`
- Check if the CloudHSM PKCS#11 library is accessible: `ls /opt/cloudhsm/lib/libcloudhsm_pkcs11.so`
- Review the web server's SSL configuration directives

## Phase 2 — Remediate

MUST:
- If engine not loaded: add the CloudHSM OpenSSL engine to the OpenSSL config or web server config
- If fake PEM missing: generate it using the key handle: `/opt/cloudhsm/bin/configure-ssl --key-handle <handle> --certificate <cert-file>`
- If CU credentials not set: export credentials: `export n3fips_password=<CU_username>:<CU_password>`
- If certificate-key mismatch: verify the certificate matches the HSM private key

SHOULD:
- Restart the web server after configuration changes: `sudo systemctl restart nginx` or `sudo systemctl restart httpd`
- Test TLS connectivity after remediation: `curl -v https://localhost`
- Verify the CloudHSM client daemon starts before the web server (systemd dependency)

MAY:
- Configure the web server to use PKCS#11 URI format for key reference
- Set up health checks for the CloudHSM-backed TLS endpoint
- Implement automatic failover if HSM connectivity is lost

## Common Issues

- symptoms: "NGINX fails to start with 'SSL_CTX_use_PrivateKey_file failed'"
  diagnosis: "Fake PEM file is missing, corrupted, or does not match the HSM key handle."
  resolution: "Regenerate the fake PEM file with the correct key handle and ensure it is readable by the web server process."

- symptoms: "TLS handshake fails with 'internal error' on client side"
  diagnosis: "CloudHSM client cannot reach the HSM during the TLS handshake."
  resolution: "Verify CloudHSM client daemon is running and HSM connectivity is healthy."

- symptoms: "OpenSSL engine test returns 'engine not found'"
  diagnosis: "CloudHSM OpenSSL engine library is not installed or not in the expected path."
  resolution: "Install or reinstall the CloudHSM client package and verify the engine library exists."

- symptoms: "Web server starts but TLS connections are slow"
  diagnosis: "HSM latency or single-HSM bottleneck for cryptographic operations."
  resolution: "Add more HSMs to the cluster for throughput. Check network latency between web server and HSM ENIs."

## Output Format

```yaml
root_cause: "SSL/TLS offload failure — <specific_cause>"
evidence:
  - type: web_server_logs
    content: "<relevant error log entries>"
  - type: engine_test
    content: "<openssl engine test output>"
severity: CRITICAL
mitigation:
  immediate: "Fix engine configuration and restore TLS offload"
  long_term: "Implement HSM health monitoring and automated web server recovery"
```

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (status/logs/verify): GREEN — read-only"
  - "Phase 2 web server restart: YELLOW — briefly interrupts TLS traffic"
  - "Phase 2 credential configuration: RED — involves HSM authentication secrets"

## Escalation Conditions

escalation_conditions:
  - "TLS offload failure affects production HTTPS traffic"
  - "HSM private key may need to be regenerated"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Certificate renewal is required alongside key operations"
  - "Issue involves compliance-critical TLS endpoints"

## Data Sensitivity

data_sensitivity:
  - command: "n3fips_password environment variable"
    sensitivity: CRITICAL
    contains: "HSM CU credentials — never log or display"
  - command: "web server SSL configuration"
    sensitivity: HIGH
    contains: "Certificate paths and key references"
  - command: "openssl s_client output"
    sensitivity: MEDIUM
    contains: "Certificate chain and TLS negotiation details"

## Prohibited Actions

prohibited_actions:
  - "NEVER log the n3fips_password or CU credentials in plaintext"
  - "NEVER use a software-based private key as fallback without explicit approval"
  - "NEVER disable TLS to work around HSM issues"
  - "NEVER store HSM credentials in web server configuration files"
  - "NEVER expose the fake PEM file publicly — it contains the key handle reference"

## Phase 3 — Rollback

- Before changing web server SSL configuration, back up the current config files
- If web server fails to start after changes, restore the previous configuration
- If CU credentials were changed, update all services using those credentials
- If the fake PEM file was regenerated, keep the previous version until the new one is verified
- Verify TLS traffic is flowing correctly after any configuration changes
