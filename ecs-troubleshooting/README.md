# ECS Container Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon ECS problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for ECS services when the console alone isn't enough — stopped task analysis, container exit codes, OOM kills, image pull errors, networking issues, IAM permission failures, capacity provider scaling, deployment circuit breakers, load balancer health checks, and exec debugging.

### Activate When

- Tasks fail to start or keep restarting
- Services can't reach desired count
- Containers are OOM-killed (exit code 137)
- Images can't be pulled from ECR
- Load balancer health checks fail
- Service discovery doesn't work
- Tasks can't reach VPC resources or the internet
- IAM permissions are denied (execution role, task role)
- Capacity providers can't scale
- Deployments are stuck or circuit breaker rolls back
- ECS Exec into containers fails
- The user says something is wrong with ECS

---

## Skill Structure

```
ecs-troubleshooting/
├── SKILL.md                          # Main skill definition and investigation workflow
├── README.md                         # This file
└── references/
    ├── A1-essential-container-exit.md
    ├── A2-oom-kill.md
    ├── A3-runtime-crash.md
    ├── A4-health-check-failure.md
    ├── B1-ecr-auth.md
    ├── B2-network-image-pull.md
    ├── B3-image-not-found.md
    ├── C1-no-internet-fargate.md
    ├── C2-eni-limits.md
    ├── C3-security-group.md
    ├── C4-service-discovery.md
    ├── D1-execution-role.md
    ├── D2-task-role.md
    ├── D3-ecr-pull-permissions.md
    ├── D4-secrets-ssm.md
    ├── E1-stuck-deployment.md
    ├── E2-circuit-breaker-rollback.md
    ├── E3-capacity-provider.md
    ├── F1-cpu-throttling.md
    ├── F2-memory-pressure.md
    ├── F3-io-bottleneck.md
    ├── G1-health-check-failure.md
    ├── G2-target-registration.md
    ├── G3-deregistration-delay.md
    ├── H1-no-ec2-capacity.md
    ├── H2-fargate-capacity.md
    ├── H3-resource-reservation.md
    ├── I1-ecs-exec-failures.md
    ├── I2-log-driver-issues.md
    ├── J1-placement-constraints.md
    ├── J2-attribute-matching.md
    ├── Z1-general-troubleshooting.md
    ├── ecs-guardrails.md
    └── ecs-hallucination-patterns.yaml
```

---

## Runbook Library (33 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Task Failures** | A1–A4 | Essential container exit, OOM kill, runtime crash, health check failure |
| **B — Image Pull** | B1–B3 | ECR auth, network/VPC endpoint, image not found |
| **C — Networking** | C1–C4 | No internet (Fargate), ENI limits, security group, service discovery |
| **D — IAM/Permissions** | D1–D4 | Execution role, task role, ECR pull, Secrets Manager/SSM |
| **E — Deployment** | E1–E3 | Stuck deployment, circuit breaker rollback, capacity provider |
| **F — Performance** | F1–F3 | CPU throttling, memory pressure, I/O bottleneck |
| **G — Load Balancer** | G1–G3 | Health check failure, target registration, deregistration delay |
| **H — Capacity** | H1–H3 | No EC2 capacity, Fargate capacity, resource reservation |
| **I — Exec / Debug** | I1–I2 | ECS Exec failures, log driver issues |
| **J — Scheduling** | J1–J2 | Placement constraints, attribute matching |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Guardrails & Anti-Hallucination

### ECS Guardrails (`ecs-guardrails.md`)
12 rules covering: task role vs execution role, exit code 137 interpretation, Fargate internet access, awsvpc ENI limits, hard vs soft memory limits, service event retention, platform versions, container vs ELB health checks, circuit breaker behavior, bridge mode port conflicts, assignPublicIp requirements, and ECS Exec prerequisites.

### Hallucination Patterns (`ecs-hallucination-patterns.yaml`)
8 patterns that LLMs commonly get wrong about ECS, including:
- Confusing task role with execution role
- Assuming exit code 137 always means OOM
- Claiming Fargate has internet access by default
- Confusing hard memory limit with soft limit (memoryReservation)
- Treating container health checks and ELB health checks as the same

---

## Investigation Workflow

Each runbook follows a consistent phased structure:

### Phase 1 — Triage
Collect initial evidence using service events, stopped task reasons, container exit codes, and logs. Classify the failure domain.

### Phase 2 — Enrich / Remediate
Deep dive using task definition review, IAM policy analysis, VPC configuration, load balancer health, or ECS Exec.

### Phase 3 — Report
State root cause with evidence, severity, and recommended mitigations.

### Output Format
```yaml
root_cause: "<category> — <detail>"
evidence:
  - type: <source>
    content: "<specific finding>"
severity: CRITICAL | HIGH | MEDIUM
mitigation:
  immediate: "<action>"
  long_term: "<prevention>"
```

---

## Prerequisites

- AWS CLI or SDK access with ECS, CloudWatch Logs, EC2, ECR, ELB, and IAM permissions
- For container debugging: ECS Exec enabled with SSM Session Manager plugin
- For EC2 launch type: access to container instance details
- For networking: VPC, security group, and NACL permissions

---

## Usage Examples

### Tasks Keep Failing
```
ECS service my-api in cluster prod keeps restarting tasks. Check
stopped task reasons, exit codes, and container logs. The service
can't reach desired count of 3.
```

### Image Pull Failure
```
ECS tasks failing with CannotPullContainerError. The service is on
Fargate in a private subnet. Check ECR permissions, VPC endpoints,
and network connectivity.
```

### Deployment Stuck
```
Rolling deployment for service my-api has been stuck for 30 minutes.
New tasks start but fail health checks. Check the new task definition,
ALB health check configuration, and security groups.
```

### Performance Investigation
```
ECS service my-worker is slow. Tasks are on Fargate with 512 CPU
and 1024 memory. Check CPU throttling, memory pressure, and I/O
bottlenecks.
```

---

## License

MIT-0
