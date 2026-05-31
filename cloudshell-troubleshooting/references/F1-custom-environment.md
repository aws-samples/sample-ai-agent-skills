---
title: "F1 — Custom Environment"
description: "Diagnose CloudShell custom environment configuration issues"
status: active
severity: LOW
triggers: ["custom tools", "install software", "configure environment", "bashrc", "custom setup"]
owner: devops-agent
objective: "Resolve custom environment setup and configuration issues in CloudShell"
context: "Customization is limited to $HOME directory. System packages don't persist. Custom binaries must be in $HOME/bin. Shell profiles (.bashrc) persist."
---

## Phase 1 — Triage

MUST:
- Check current shell configuration: `cat $HOME/.bashrc` (inside CloudShell)
- Verify $HOME/bin exists and is in PATH: `echo $PATH` (inside CloudShell)
- Check installed custom tools: `ls -la $HOME/bin/` (inside CloudShell)

SHOULD:
- Verify Python user packages: `pip list --user` (inside CloudShell)
- Check Node.js global packages: `ls $HOME/.npm-global/lib/node_modules/` (inside CloudShell)
- Review environment variables: `env | sort` (inside CloudShell)

MAY:
- Check if .bash_profile conflicts with .bashrc: `cat $HOME/.bash_profile` (inside CloudShell)
- Verify custom tool versions: `$HOME/bin/<tool> --version`

## Phase 2 — Remediate

MUST:
- Create $HOME/bin for custom binaries: `mkdir -p $HOME/bin`
- Add $HOME/bin to PATH in .bashrc: `echo 'export PATH=$HOME/bin:$PATH' >> $HOME/.bashrc`
- Use pip install --user for Python packages: `pip install --user <package>`

SHOULD:
- Create a setup script for system packages: `cat > $HOME/setup.sh << 'EOF'\n#!/bin/bash\nsudo yum install -y <packages>\nEOF`
- Configure npm to install globally in $HOME: `npm config set prefix $HOME/.npm-global`

MAY:
- Set up a dotfiles git repository for environment portability
- Create aliases in .bashrc for frequently used commands
- Install language version managers (pyenv, nvm) in $HOME

## Safety Ratings

```
safety_ratings:
  - "cat / ls / echo: GREEN — read-only inspection"
  - "pip install --user: GREEN — installs in user directory"
  - "Modifying .bashrc: YELLOW — affects all future sessions"
  - "sudo yum install: YELLOW — system-level change, does not persist"
```

## Escalation Conditions

- Custom tool requires system libraries not available in CloudShell base image
- .bashrc corruption prevents CloudShell from starting
- Custom environment needs exceed CloudShell capabilities (need Cloud9 or EC2)
- Security policy prohibits custom tool installation

## Data Sensitivity

- `cat $HOME/.bashrc`: MEDIUM — may contain environment variables or paths
- `env | sort`: MEDIUM — reveals all environment variables
- `pip list / npm list`: LOW — package inventory
- Custom tool binaries: varies — depends on tool purpose

## Prohibited Actions

- NEVER store credentials or secrets in .bashrc or environment variables
- NEVER modify system files outside $HOME expecting persistence
- NEVER install untrusted binaries from unknown sources
- NEVER override AWS CLI with custom versions that may have compatibility issues

## Phase 3 — Rollback

1. If .bashrc was corrupted: `cp /etc/skel/.bashrc $HOME/.bashrc`
2. If custom tool causes issues: `rm $HOME/bin/<tool>` and remove PATH entry
3. If pip packages conflict: `pip uninstall --user <package>`
4. If npm global config was broken: `rm $HOME/.npmrc` and reconfigure
5. If environment variables were set incorrectly: remove from .bashrc and start new session

## Common Issues

- symptoms: "Custom tool works in current session but is gone after reopening CloudShell"
  diagnosis: "Tool was installed system-wide (e.g., sudo yum install) instead of in $HOME."
  resolution: "Download binary to $HOME/bin/ or use pip install --user. Add $HOME/bin to PATH in .bashrc."

## Output Format

```yaml
root_cause: "custom_environment — <specific_cause>"
evidence:
  - type: configuration_error
    content: "<error details>"
severity: LOW
mitigation:
  immediate: "Fix custom environment configuration"
  long_term: "Create reproducible setup script in $HOME for environment bootstrapping"
```

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Data Sensitivity

data_sensitivity:
  - command: "describe-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
