# Amazon Location Service Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon Location Service problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for Amazon Location Service — map rendering, place search, route calculation, geofencing, device tracking, and provider configuration.

### Activate When

- Map rendering issues
- Place search failures or inaccurate results
- Route calculation errors
- Geofence creation or event problems
- Tracker or device position issues
- Provider configuration problems
- Authentication (API key, Cognito) issues
- General Location Service errors

---

## Skill Structure

```
location-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-map-rendering.md, A2-map-styles.md
    ├── B1-place-search.md, B2-geocoding.md
    ├── C1-route-calculation.md, C2-route-optimization.md
    ├── D1-geofence-creation.md, D2-geofence-events.md
    ├── E1-tracker-setup.md, E2-device-positions.md
    ├── F1-provider-config.md, F2-provider-coverage.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Prerequisites

- AWS CLI v2 with `geo:*`, `events:*` (for geofence events), `iam:*`

---

## License

MIT-0
