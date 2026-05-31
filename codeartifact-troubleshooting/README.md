# CodeArtifact Diagnostics Skill

Agent skill for investigating and troubleshooting AWS CodeArtifact problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for CodeArtifact when the console alone isn't enough — repository creation and upstream connection failures, authentication token issues, domain permission errors, npm/pip/Maven package resolution problems, package publishing failures, version conflicts, domain configuration, cross-account access, CI/CD integration, and build tool configuration.

### Activate When

- Repository creation fails
- Upstream connection not resolving packages
- Auth token expired or invalid
- Domain permissions blocking access
- npm install fails against CodeArtifact
- pip install fails against CodeArtifact PyPI
- Maven build fails to resolve CodeArtifact dependencies
- Package publishing fails or version conflict
- Domain configuration errors
- Cross-account access denied
- CI/CD pipeline cannot authenticate to CodeArtifact
- Build tool configuration issues

---

## Skill Structure

```
codeartifact-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-repository-creation.md
    ├── A2-upstream-connections.md
    ├── B1-auth-token.md
    ├── B2-domain-permissions.md
    ├── C1-npm-issues.md
    ├── C2-pip-pypi-issues.md
    ├── C3-maven-issues.md
    ├── D1-package-publishing.md
    ├── D2-version-conflicts.md
    ├── E1-domain-configuration.md
    ├── E2-cross-account.md
    ├── F1-cicd-integration.md
    ├── F2-build-tools.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (18 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Repository** | A1–A2 | Creation failures, upstream connections |
| **B — Auth** | B1–B2 | Auth token issues, domain permissions |
| **C — Package** | C1–C3 | npm issues, pip/PyPI issues, Maven issues |
| **D — Publishing** | D1–D2 | Package publishing, version conflicts |
| **E — Domain** | E1–E2 | Domain configuration, cross-account |
| **F — Integration** | F1–F2 | CI/CD integration, build tools |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## License

MIT-0
