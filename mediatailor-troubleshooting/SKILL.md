---
name: mediatailor-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Elemental MediaTailor
  problems by analyzing configuration creation, playback configuration,
  ad insertion, VAST responses, channel assembly, source locations,
  CDN configuration, manifest manipulation, logging, prefetch schedules,
  bumper/slate configuration, and following structured runbooks.
  Activate when: configuration creation errors, playback failures,
  ad insertion issues, VAST response errors, channel assembly problems,
  source location errors, CDN issues, manifest manipulation failures,
  log configuration problems, prefetch schedule issues, bumper/slate
  configuration errors, or the user says something is wrong with MediaTailor.
compatibility: >
  Requires AWS CLI or SDK access with mediatailor, cloudwatch, cloudfront,
  s3, iam, and logs permissions.
---

# AWS Elemental MediaTailor Diagnostics

## When to use

Any AWS Elemental MediaTailor investigation — configuration creation, playback configuration, ad insertion failures, VAST response errors, channel assembly, source locations, CDN configuration, manifest manipulation, log configuration, prefetch schedules, or bumper/slate configuration.

## Investigation workflow

### Step 1 — Collect and triage

```
aws mediatailor list-playback-configurations --query 'Items[*].{Name:Name,AdDecisionServerUrl:AdDecisionServerUrl,VideoContentSourceUrl:VideoContentSourceUrl}'
aws mediatailor list-channels --query 'Items[*].{Name:ChannelName,State:ChannelState,Tier:Tier}'
aws mediatailor list-source-locations --query 'Items[*].{Name:SourceLocationName,HttpConfiguration:HttpConfiguration}'
```

### Step 2 — Domain deep dive

```
aws mediatailor get-playback-configuration --Name <config-name>
aws mediatailor describe-channel --ChannelName <channel-name>
aws mediatailor describe-source-location --SourceLocationName <source-location-name>
```

### Step 3 — Detailed investigation

```
aws logs filter-log-events --log-group-name /aws/mediatailor/<config-name> --start-time <epoch-ms> --filter-pattern "ERROR"
aws mediatailor list-prefetch-schedules --PlaybackConfigurationName <config-name>
aws cloudwatch get-metric-statistics --namespace AWS/MediaTailor --metric-name AdDecisionServer.Errors --dimensions Name=Configuration,Value=<config-name> --start-time <start> --end-time <end> --period 300 --statistics Sum
```

Read `references/guardrails.md` before concluding on any MediaTailor issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `mediatailor list-playback-configurations` | List all playback configurations |
| `mediatailor get-playback-configuration` | Get configuration details |
| `mediatailor list-channels` | List channel assembly channels |
| `mediatailor describe-channel` | Get channel details and outputs |
| `mediatailor list-source-locations` | List source locations |
| `mediatailor describe-source-location` | Get source location details |
| `mediatailor list-prefetch-schedules` | List prefetch schedules |

## Gotchas: AWS Elemental MediaTailor

- MediaTailor has TWO modes: server-side ad insertion (SSAI) for live/VOD and channel assembly for linear channel creation. They use different configurations.
- Ad decision server (ADS) URLs must return valid VAST/VMAP XML. MediaTailor does NOT validate ADS URLs at configuration time — errors appear only during playback.
- CDN configuration requires proper segment and ad segment URL prefixes. Misconfigured CDN prefixes cause 403/404 errors during playback, not during configuration.
- Channel assembly requires source locations with VOD sources. Live sources use HLS/DASH manifests directly. Mixing them incorrectly causes channel playback failures.
- Prefetch schedules improve ad fill rates by pre-fetching ad decisions. They require proper time windows and do NOT guarantee 100% ad fill.
- Bumpers and slates are different: bumpers play before/after ad breaks, slates fill unfilled ad time. Both require valid MP4 URLs accessible to MediaTailor.
- Log configuration uses CloudWatch Logs. Logs are NOT enabled by default — you must configure log percentages (0-100%) per configuration.

## Anti-hallucination rules

1. Always cite specific configuration names, channel names, or API responses as evidence.
2. SSAI and channel assembly are separate features. Never conflate their configurations.
3. VAST/VMAP errors come from the ad decision server, not MediaTailor. Never blame MediaTailor for ADS response issues.
4. CDN prefix configuration affects playback URLs. Never assume default CDN behavior without checking configuration.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 12 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Configuration | A1-A2 | Creation errors, playback configuration |
| B — Ad Insertion | B1-B2 | Ad insertion failures, VAST response errors |
| C — Channel Assembly | C1-C2 | Channel assembly issues, source location errors |
| D — CDN & Manifest | D1-D2 | CDN configuration, manifest manipulation |
| E — Logging & Prefetch | E1-E2 | Log configuration, prefetch schedule issues |
| F — Bumper & Slate | F1 | Bumper/slate configuration |
| Z — Catch-All | Z1 | General troubleshooting |
