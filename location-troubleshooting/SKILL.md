---
name: location-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon Location Service
  problems by analyzing maps, places, routes, geofences, trackers,
  device positions, map provider configuration (HERE, Esri, GrabMaps),
  and following structured runbooks. Activate when: map rendering
  issues, place search failures, route calculation errors, geofence
  creation problems, tracker issues, device position errors, provider
  configuration problems, or the user says something is wrong with
  Location Service without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with Location Service, IAM, CloudWatch,
  CloudTrail, and optionally EventBridge and Lambda permissions for
  geofence events.
---

# Amazon Location Service Diagnostics

## When to use

Any Amazon Location Service investigation where the console alone is insufficient — map rendering, place search, route calculation, geofencing, device tracking, or provider configuration.

## Investigation workflow

### Step 1 — Collect and triage

```
aws location list-maps
aws location list-place-indexes
aws location list-route-calculators
aws location list-geofence-collections
aws location list-trackers
```

### Step 2 — Domain deep dive

```
aws location describe-map --map-name <name>
aws location describe-place-index --index-name <name>
aws location describe-route-calculator --calculator-name <name>
aws location describe-geofence-collection --collection-name <name>
aws location describe-tracker --tracker-name <name>
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=geo.amazonaws.com --max-results 20
aws location search-place-index-for-text --index-name <name> --text "Seattle"
aws location calculate-route --calculator-name <name> --departure-position '[-122.33,47.60]' --destination-position '[-122.20,47.61]'
```

Read `references/guardrails.md` before concluding on any Location Service issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `location list-maps` | List map resources |
| `location describe-map` | Get map configuration |
| `location search-place-index-for-text` | Search places |
| `location calculate-route` | Calculate routes |
| `location list-geofences` | List geofences |
| `location get-device-position` | Get device position |
| `location batch-update-device-position` | Update device positions |

## Gotchas: Amazon Location Service

- Map providers (Esri, HERE, GrabMaps, OpenData) have different capabilities and coverage areas.
- GrabMaps only covers Southeast Asia. Using it outside that region returns no results.
- Place search results vary by provider. Different providers have different POI databases.
- Route calculation supports car, truck, and walking modes. Not all modes available with all providers.
- Geofences use GeoJSON polygon format. Maximum 1000 vertices per geofence.
- Trackers store device positions. Position updates trigger geofence ENTER/EXIT events via EventBridge.
- API keys and Cognito are the authentication methods. API keys are simpler but less secure.
- Map tiles are served via HTTPS. CORS configuration needed for web applications.

## Anti-hallucination rules

1. Always cite specific resource names, provider details, or API responses as evidence.
2. Provider coverage varies. Never assume global coverage for all providers.
3. GrabMaps is Southeast Asia only. Never assume global availability.
4. Geofences use GeoJSON. Never assume other geometry formats.
5. Route modes vary by provider. Never assume all modes available.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 14 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Maps | A1-A2 | Map rendering, map styles |
| B — Places | B1-B2 | Place search, geocoding |
| C — Routes | C1-C2 | Route calculation, route optimization |
| D — Geofences | D1-D2 | Geofence creation, geofence events |
| E — Trackers | E1-E2 | Tracker setup, device positions |
| F — Providers | F1-F2 | Provider configuration, provider coverage |
| Z — Catch-All | Z1 | General troubleshooting |
