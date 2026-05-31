# Amazon CodeGuru Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon CodeGuru problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for Amazon CodeGuru — reviewer recommendations, profiler, security detector, repository association, CI/CD integration, and suppression rules.

### Activate When

- Reviewer not generating recommendations
- Profiler agent not reporting data
- Security findings unexpected or missing
- Repository association failures
- CI/CD integration errors
- Suppression rules not working

---

## Prerequisites

- AWS CLI v2 with codeguru-reviewer, codeguru-profiler, S3, IAM permissions
- Repository access (CodeCommit, GitHub, Bitbucket)
- Application runtime for profiler agent

---

## License

MIT-0
