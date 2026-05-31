---
title: "D2 — Wisdom Search Failures"
description: "Diagnose Amazon Connect Wisdom search and recommendation failures"
status: active
severity: MEDIUM
triggers: ["search not working", "no recommendations", "Wisdom search empty", "agent assist not showing"]
owner: devops-agent
objective: "Resolve Wisdom search and agent recommendation issues"
context: "Wisdom provides real-time search and recommendations to agents during calls. Search depends on indexed content in knowledge bases. Recommendations use Contact Lens transcription to suggest relevant articles."
---

## Phase 1 — Triage

MUST:
- Test search directly: `aws wisdom search-content --knowledge-base-id <kb-id> --search-expression '{"filters":[{"field":"NAME","operator":"EQUALS","value":"<search-term>"}]}'`
- Verify knowledge base is ACTIVE: `aws wisdom get-knowledge-base --knowledge-base-id <kb-id>`
- Check if content exists and is indexed: `aws wisdom list-contents --knowledge-base-id <kb-id>`

SHOULD:
- Verify the Wisdom assistant is configured for the Connect instance
- Check if Contact Lens is enabled (recommendations depend on transcription)
- Verify the agent's CCP has the Wisdom panel enabled

MAY:
- Test with different search terms to isolate the issue
- Check IAM permissions for the agent's role to access Wisdom

## Phase 2 — Remediate

MUST:
- Ensure knowledge base has indexed content
- Verify the Wisdom assistant is associated with the Connect instance
- Ensure Contact Lens is enabled for real-time recommendations

SHOULD:
- Improve content quality and metadata for better search relevance
- Verify the agent desktop configuration includes the Wisdom panel
- Test search with various query terms to validate content coverage

MAY:
- Add more content to the knowledge base to improve coverage
- Configure content metadata and tags for improved search ranking
- Set up search analytics to track query patterns and gaps

## Safety Ratings

```
safety_ratings:
  - "search-content: GREEN — read-only search operation"
  - "get-knowledge-base: GREEN — read-only details"
  - "list-contents: GREEN — read-only content listing"
  - "update assistant association: YELLOW — modifies agent experience"
```

## Escalation Conditions

- Search returns no results for content that is confirmed indexed
- Recommendations not appearing despite Contact Lens transcription working
- Search latency is unacceptable for real-time agent assist
- Search relevance is poor despite well-structured content

## Data Sensitivity

- `aws wisdom search-content`: MEDIUM — reveals knowledge base content
- Search queries: LOW — contain agent search terms
- Search results: MEDIUM — contain knowledge base article content
- Recommendation data: MEDIUM — reveals what content is surfaced to agents

## Prohibited Actions

- NEVER assume Wisdom answers customers directly — it provides recommendations to agents
- NEVER delete knowledge base content to troubleshoot search — test with new content instead
- NEVER disable Contact Lens to troubleshoot Wisdom — they are complementary
- NEVER share knowledge base content outside authorized systems

## Phase 3 — Rollback

1. Restore previous assistant association if changes broke recommendations
2. Re-index content if indexing was interrupted
3. Restore agent desktop configuration if Wisdom panel was removed
4. Verify search and recommendations work for a test agent after changes
5. Document the search issue and resolution

## Common Issues

- symptoms: "Wisdom panel shows 'No recommendations' during active call"
  diagnosis: "Contact Lens real-time transcription not enabled — Wisdom needs transcript to generate recommendations."
  resolution: "Enable real-time Contact Lens analytics in the contact flow."

## Output Format

```yaml
root_cause: "wisdom_search — <specific_cause>"
evidence:
  - type: search_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix search configuration"
  long_term: "Implement content management and search quality monitoring"
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
  - "NEVER suggest disabling call recording without compliance review"
  - "NEVER suggest removing authentication"
  - "NEVER expose customer PII in logs"
