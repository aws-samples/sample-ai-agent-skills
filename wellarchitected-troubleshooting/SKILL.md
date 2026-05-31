---
name: wellarchitected-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Well-Architected Tool
  problems by analyzing workload reviews, lens selection, improvement plans,
  milestones, custom lenses, sharing, and following structured runbooks.
  Activate when: workload review issues, lens configuration problems,
  improvement plan errors, milestone failures, custom lens issues, sharing
  problems, or the user says something is wrong with Well-Architected Tool.
compatibility: >
  Requires AWS CLI with wellarchitected, IAM, and Organizations permissions.
---

# AWS Well-Architected Tool Diagnostics

## When to use

Any AWS Well-Architected Tool investigation — workload reviews, lens selection, improvement plans, milestones, custom lenses, or sharing.

## Investigation workflow

### Step 1 — Collect and triage

```
aws wellarchitected list-workloads
aws wellarchitected list-lenses --lens-type AWS_OFFICIAL
aws wellarchitected list-lenses --lens-type CUSTOM_SELF
aws wellarchitected list-share-invitations
```

### Step 2 — Domain deep dive

```
aws wellarchitected get-workload --workload-id <id>
aws wellarchitected list-answers --workload-id <id> --lens-alias wellarchitected
aws wellarchitected list-milestones --workload-id <id>
aws wellarchitected get-lens-review --workload-id <id> --lens-alias wellarchitected
```

### Step 3 — Detailed investigation

```
aws wellarchitected list-lens-review-improvements --workload-id <id> --lens-alias wellarchitected
aws wellarchitected get-milestone --workload-id <id> --milestone-number <num>
aws wellarchitected list-workload-shares --workload-id <id>
```

Read `references/guardrails.md` before concluding on any Well-Architected Tool issue.

## Gotchas: AWS Well-Architected Tool

- Workload reviews are manual assessments, not automated scans.
- Lenses define the question sets. Different lenses for different domains.
- Milestones snapshot the current state. They cannot be modified after creation.
- Custom lenses must follow specific JSON schema.
- Sharing requires acceptance by the recipient.
- Improvement plans are generated from review answers.

## Anti-hallucination rules

1. Always cite workload IDs, lens aliases, or API responses as evidence.
2. Reviews are manual. Never suggest automated compliance scanning.
3. Milestones are immutable snapshots. Never suggest editing milestones.
4. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 12 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Workloads | A1-A2 | Workload reviews, answer management |
| B — Lenses | B1-B2 | Lens selection, lens configuration |
| C — Improvement | C1-C2 | Improvement plans, risk tracking |
| D — Milestones | D1-D2 | Milestone creation, milestone comparison |
| E — Custom | E1-E2 | Custom lens creation, lens publishing |
| F — Sharing | F1-F2 | Workload sharing, cross-account access |
