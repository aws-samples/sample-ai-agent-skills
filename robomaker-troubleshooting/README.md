# AWS RoboMaker Diagnostics Skill

Agent skill for investigating and troubleshooting AWS RoboMaker problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS RoboMaker — simulation jobs, robot applications, fleet management, ROS integration, world generation, and deployment pipelines.

### Activate When

- Simulation job failures or timeouts
- Robot application build or runtime errors
- Fleet management or robot registration issues
- ROS connectivity or topic problems
- World generation failures
- Greengrass deployment issues
- General RoboMaker errors without clear symptoms

---

## Skill Structure

```
robomaker-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-simulation-job-failures.md
    ├── A2-simulation-configuration.md
    ├── A3-batch-simulation.md
    ├── B1-robot-app-build.md
    ├── B2-robot-app-runtime.md
    ├── C1-fleet-management.md
    ├── C2-robot-registration.md
    ├── D1-ros-integration.md
    ├── D2-ros-topic-connectivity.md
    ├── E1-world-generation.md
    ├── E2-world-templates.md
    ├── F1-greengrass-deployment.md
    ├── F2-deployment-updates.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Prerequisites

- AWS CLI v2 with RoboMaker, S3, IAM, ECR, IoT, Greengrass permissions
- ROS development environment for application debugging

---

## License

MIT-0
