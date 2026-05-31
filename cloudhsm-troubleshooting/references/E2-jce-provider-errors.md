---
title: "E2 — JCE Provider Errors"
description: "Diagnose and resolve CloudHSM Java Cryptography Extension (JCE) provider issues"
status: active
severity: HIGH
triggers:
  - "JCE provider error"
  - "CaviumProvider"
  - "CloudHsmProvider"
  - "Java crypto failed"
  - "PKCS11 Java"
owner: devops-agent
objective: "Resolve JCE provider configuration and runtime errors for Java applications using CloudHSM"
context: "CloudHSM provides a JCE provider (CaviumProvider or the newer CloudHsmProvider) that allows Java applications to use HSM-backed cryptographic operations. The provider integrates with the Java Security framework. Issues arise from classpath configuration, provider registration, credential management, library compatibility, or JVM version mismatches."
---

## Phase 1 — Triage

MUST:
- Verify CloudHSM client is running: `sudo systemctl status cloudhsm-client`
- Check the JCE provider JAR is installed: `ls /opt/cloudhsm/java/`
- Verify the provider is registered in java.security: `grep -i cavium $JAVA_HOME/conf/security/java.security` or `grep -i cloudhsm $JAVA_HOME/conf/security/java.security`
- Check the Java application logs for specific provider errors
- Verify the CloudHSM Java library is on the classpath

SHOULD:
- Check Java version compatibility: `java -version` (CloudHSM SDK 5 requires Java 8+)
- Verify HSM login credentials are configured: check for LoginManager or system property configuration
- Test provider availability: write a small test that calls `Security.getProvider("CloudHSM")` or `Security.getProvider("Cavium")`

MAY:
- Check if multiple JCE providers are conflicting in the provider order
- Verify the native library path includes CloudHSM libraries: check `java.library.path`
- Review the CloudHSM SDK version compatibility matrix

## Phase 2 — Remediate

MUST:
- If provider JAR missing: install the CloudHSM Java library: `sudo yum install aws-cloudhsm-jce` or copy the JAR to the classpath
- If provider not registered: add to java.security: `security.provider.N=com.amazonaws.cloudhsm.jce.provider.CloudHsmProvider`
- If login fails: verify credentials via LoginManager: `LoginManager.getInstance().login("<partition>", "<cu_user>", "<cu_password>")`
- If native library not found: set `java.library.path` to include `/opt/cloudhsm/lib`

SHOULD:
- Restart the Java application after provider configuration changes
- Verify the provider is loaded: `Security.getProviders()` should list CloudHSM/Cavium
- Test a basic cryptographic operation (e.g., AES key generation) through the provider

MAY:
- Migrate from CaviumProvider (SDK 3) to CloudHsmProvider (SDK 5) for newer features
- Configure provider priority in java.security for optimal performance
- Set up JMX monitoring for CloudHSM provider health

## Common Issues

- symptoms: "NoSuchProviderException: no such provider: CloudHSM"
  diagnosis: "JCE provider not registered in java.security or JAR not on classpath."
  resolution: "Add the provider to java.security and ensure the CloudHSM JAR is on the application classpath."

- symptoms: "UnsatisfiedLinkError: libcloudhsm_jce.so not found"
  diagnosis: "Native CloudHSM library not in the Java library path."
  resolution: "Add `/opt/cloudhsm/lib` to java.library.path: `-Djava.library.path=/opt/cloudhsm/lib`."

- symptoms: "LoginException during HSM authentication from Java"
  diagnosis: "Incorrect CU credentials or CloudHSM client daemon not running."
  resolution: "Verify CU credentials and ensure the CloudHSM client daemon is active."

- symptoms: "InvalidKeyException when using HSM-generated keys"
  diagnosis: "JCE unlimited strength jurisdiction policy not installed, or key type not supported."
  resolution: "For Java 8: install JCE Unlimited Strength policy files. For Java 9+: unlimited is default."

## Output Format

```yaml
root_cause: "JCE provider error — <specific_cause>"
evidence:
  - type: java_error
    content: "<Java exception stack trace>"
  - type: provider_config
    content: "<java.security provider configuration>"
severity: HIGH
mitigation:
  immediate: "Fix provider registration, classpath, or credential configuration"
  long_term: "Standardize JCE provider deployment with configuration management"
```

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (check files/config/version): GREEN — read-only"
  - "Phase 2 java.security modification: YELLOW — affects JVM security configuration"
  - "Phase 2 application restart: YELLOW — briefly interrupts service"

## Escalation Conditions

escalation_conditions:
  - "JCE provider failure affects production cryptographic operations"
  - "Java security configuration changes could affect other providers"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "SDK version upgrade is required and may introduce breaking changes"
  - "Issue involves FIPS mode configuration for compliance"

## Data Sensitivity

data_sensitivity:
  - command: "LoginManager credentials"
    sensitivity: CRITICAL
    contains: "HSM CU username and password — never log"
  - command: "java.security file"
    sensitivity: HIGH
    contains: "JVM security provider configuration"
  - command: "Application logs"
    sensitivity: MEDIUM
    contains: "Stack traces that may reveal key operations"

## Prohibited Actions

prohibited_actions:
  - "NEVER hardcode HSM credentials in Java source code"
  - "NEVER disable JCE provider verification to bypass errors"
  - "NEVER downgrade Java security settings to work around provider issues"
  - "NEVER log cryptographic key material or HSM credentials in application logs"
  - "NEVER use the Cavium provider in FIPS mode without proper validation"

## Phase 3 — Rollback

- Before modifying java.security, back up the file: `cp $JAVA_HOME/conf/security/java.security $JAVA_HOME/conf/security/java.security.bak`
- If provider changes break the application, restore the previous java.security file
- If SDK upgrade causes issues, downgrade to the previous version
- If classpath changes affect other libraries, revert to the previous classpath configuration
- Verify all cryptographic operations work correctly after any changes
