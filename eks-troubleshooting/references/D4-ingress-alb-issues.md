---
title: "D4 — Ingress / ALB Issues"
description: "Diagnose and resolve AWS Load Balancer Controller and Ingress resource issues"
status: active
severity: HIGH
triggers:
  - "Ingress"
  - "ALB"
  - "NLB"
  - "load balancer"
  - "aws-load-balancer-controller"
  - "target group"
owner: devops-agent
objective: "Identify why the Ingress/ALB is not working and restore traffic routing"
context: "The AWS Load Balancer Controller (formerly ALB Ingress Controller) manages ALBs and NLBs for Kubernetes Ingress and Service resources. It requires specific IAM permissions, subnet tags, and correct annotations. Common issues: missing subnet tags, IAM permission errors, target group health, and annotation misconfiguration."
---

## Phase 1 — Triage

MUST:
- Check Ingress resource status: `kubectl get ingress -n <namespace> -o wide`
- Describe the Ingress for events: `kubectl describe ingress <name> -n <namespace>`
- Check AWS Load Balancer Controller pods: `kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller`
- Check controller logs: `kubectl logs -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller --tail=100`
- Verify subnet tags: `aws ec2 describe-subnets --filters "Name=tag-key,Values=kubernetes.io/role/elb" --query 'Subnets[*].{Id:SubnetId,AZ:AvailabilityZone,Tags:Tags}'`

SHOULD:
- Check the controller's IAM role/service account: `kubectl get sa aws-load-balancer-controller -n kube-system -o yaml`
- Verify the IRSA annotation on the service account
- Check if the ALB was created: `aws elbv2 describe-load-balancers --query 'LoadBalancers[?contains(LoadBalancerName, \`k8s\`)]'`
- Check target group health: `aws elbv2 describe-target-health --target-group-arn <tg-arn>`

MAY:
- Check IngressClass resource: `kubectl get ingressclass`
- Verify the Ingress annotations match the controller (alb vs nginx)
- Check for conflicting Ingress resources on the same host/path

## Phase 2 — Remediate

MUST:
- If no ALB created: check subnet tags — public subnets need `kubernetes.io/role/elb: 1`, private need `kubernetes.io/role/internal-elb: 1`
- If IAM permission error: fix the controller's IAM role policy (needs elasticloadbalancing:*, ec2:Describe*, etc.)
- If target group unhealthy: check pod readiness, security groups, and health check path
- If controller not running: check deployment and IRSA configuration

SHOULD:
- Ensure the IngressClass is set correctly: `spec.ingressClassName: alb`
- Add required annotations: `alb.ingress.kubernetes.io/scheme: internet-facing` or `internal`
- Verify security groups allow traffic from ALB to node ports
- Tag subnets with `kubernetes.io/cluster/<cluster-name>: owned` or `shared`

MAY:
- Use `alb.ingress.kubernetes.io/target-type: ip` for direct pod targeting (requires VPC CNI)
- Configure WAF integration via annotations
- Set up SSL/TLS with ACM certificate ARN annotation

## Common Issues

- symptoms: "Ingress created but no ALB appears, no events on the Ingress"
  diagnosis: "AWS Load Balancer Controller is not running or not watching this Ingress."
  resolution: "Check controller pods. Verify ingressClassName is 'alb'. Check controller logs."

- symptoms: "ALB created but returns 502 Bad Gateway"
  diagnosis: "Target group has no healthy targets. Pods are not reachable from the ALB."
  resolution: "Check target group health. Verify security groups allow ALB → node traffic on target port."

- symptoms: "Controller logs show 'AccessDenied' errors"
  diagnosis: "Controller's IAM role lacks required permissions."
  resolution: "Update the IAM policy with the full AWS Load Balancer Controller policy."

- symptoms: "ALB not created, controller logs show 'no subnets found'"
  diagnosis: "Subnets missing required tags for ALB discovery."
  resolution: "Tag public subnets with `kubernetes.io/role/elb: 1` and private with `kubernetes.io/role/internal-elb: 1`."

## Output Format

```yaml
root_cause: "Ingress/ALB issue — <specific_cause>"
evidence:
  - type: ingress_events
    content: "<Ingress describe events>"
  - type: controller_logs
    content: "<relevant controller log entries>"
severity: HIGH
mitigation:
  immediate: "Fix subnet tags, IAM permissions, or target group configuration"
  long_term: "Automate subnet tagging in IaC, implement ALB health monitoring"
```

## Safety Ratings
- GREEN: read-only (`kubectl get ingress`, `kubectl describe ingress`, `kubectl logs`, `aws elbv2 describe-load-balancers`, `aws elbv2 describe-target-health`)
- YELLOW: state-changing recoverable (`kubectl apply ingress`, `kubectl annotate`, `aws ec2 create-tags` for subnet tagging, update IAM policy for LB controller)
- RED: destructive/irreversible (`kubectl delete ingress` in production, removing subnet tags causing ALB deletion, deleting AWS Load Balancer Controller)

## Escalation Conditions
- "Remediation requires editing aws-auth ConfigMap in production"
- "Fix involves modifying RBAC ClusterRoleBindings"
- "Change affects node group scaling in production"
- "Fix involves modifying network policies or security groups affecting production traffic"
- "Remediation requires modifying the AWS Load Balancer Controller IAM role in production"

## Rollback
- Pre-change: "Back up aws-auth ConfigMap before editing"
- Pre-change: "Save current Ingress resource and annotations before modification"
- Pre-change: "Save current RBAC bindings before modification"
- Verification: "Verify ALB health and target group status after change"
- Revert: "Restore Ingress resource from backup if traffic routing breaks"

## Data Sensitivity
- HIGH: "aws-auth ConfigMap reveals all IAM-to-RBAC mappings"
- HIGH: "Service account annotations reveal IAM role ARNs (IRSA)"
- MEDIUM: "Ingress annotations reveal ALB configuration, ACM certificate ARNs, and WAF settings"
- LOW: "Pod status and events"

## Prohibited Actions
- NEVER suggest adding `system:masters` group for troubleshooting access
- NEVER suggest disabling Pod Security Standards/Admission
- NEVER suggest running containers as root to fix permission issues
- NEVER suggest `kubectl delete namespace` in production without confirmation
- NEVER suggest modifying kube-system resources without backup

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Data Sensitivity

data_sensitivity:
  - command: "describe-cluster"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "kubectl get configmap aws-auth"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "kubectl get pods"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest system:masters group for troubleshooting"
  - "NEVER suggest disabling RBAC or Pod Security Standards"
  - "NEVER suggest running pods as root to fix permissions"

## Phase 3 � Rollback

- Document the current state before making any changes
- If remediation worsens the issue, revert to the documented pre-change state
- Remove any temporary resources created during investigation
- Verify the system returns to its previous functional state after rollback
