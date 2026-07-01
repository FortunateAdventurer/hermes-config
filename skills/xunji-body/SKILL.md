---
name: xunji-body
description: "Use when user asks to read/write/analyze Xunji App body measurement data. Query and write weight, bodyfat, circumferences."
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [xunji, body, weight, bodyfat, measurement, api]
---

# xunji-body — Xunji Body Data Open API

Trigger: user explicitly asks to read/write/analyze Xunji body data.
API: https://api.xunjiapp.cn

## When to use

- Read recent weight/bodyfat/measurements
- Write new body data to Xunji
- Compare body data trends over time

## When NOT to use

- Not explicitly requested
- Non-Xunji body data (Apple Health, scale CSV)
- Medical diagnosis or health advice

## Auth

- Header: Authorization: Bearer <key>
- Also accepts x-api-key header
- Key via env var XUNJI_BODY_API_KEY
- Never put key in body/query/URL

### curl note

- All endpoints end in _gzip, server returns gzip-compressed data
- curl must include --compressed

## Endpoints

| Op | Method | Path |
|----|--------|------|
| Query | POST | /open/body/query_gzip |
| Upsert | POST | /open/body/upsert_gzip |

Base URL: https://api.xunjiapp.cn

## Query

POST /open/body/query_gzip
{
  "start_date": "2026-01-01",
  "end_date": "2026-06-28",
  "types": ["weight", "bodyfat"],
  "include_latest": true,
  "include_records": true,
  "limit": 500,
  "offset": 0
}

- Omit types for all metrics
- records[] sorted by date descending: datestr, type, value, unit, label, label_en
- latest: most recent record per type
- by_type: records grouped by type
- value is numeric when possible

## Write (two-step confirmation)

### Step 1: dry_run validation

POST /open/body/upsert_gzip
{
  "schema_version": "body_open_api_v1",
  "client_request_id": "xjbody-{epoch_ms}-{nonce}",
  "dry_run": true,
  "records": [
    { "datestr": "2026-06-28", "type": "weight", "value": 72.4 },
    { "datestr": "2026-06-28", "type": "bodyfat", "value": 18.2 }
  ]
}

- Show res.summary to user and wait for confirmation

### Step 2: confirmed write

{
  "dry_run": false,
  "confirmed": true,
  "records": [
    { "datestr": "2026-06-28", "type": "weight", "value": 72.4 }
  ]
}

- MUST include "confirmed": true, otherwise server returns "user confirmation required"
- Upsert by datestr + type: existing records updated, new ones created

## Body data types

| type | Label | Unit |
|------|-------|------|
| weight | Weight | kg |
| bodyfat | Body Fat | % |
| neck | Neck | cm |
| chest | Chest | cm |
| weist | Waist | cm |
| shoulder | Shoulder | cm |
| bot | Hip | cm |
| arm_left | Left Arm | cm |
| arm_right | Right Arm | cm |
| forearm_left | Left Forearm | cm |
| forearm_right | Right Forearm | cm |
| leg_left | Left Leg | cm |
| leg_right | Right Leg | cm |
| cav_left | Left Calf | cm |
| cav_right | Right Calf | cm |

IMPORTANT: Waist is "weist" (historical spelling), do NOT change to "waist".

## Workflow

### Read
1. Get date range + optional types
2. Check cache: ~/.hermes/state/xunji-cache/body/{start}_{end}_{types}.json
3. Hit and TTL valid -> return cached
4. Miss -> request, cache result

### Write (two-step)
1. Collect date + type + value
2. Send dry_run: true for validation
3. Show res.summary to user, WAIT for confirmation
4. After user confirms -> send dry_run: false + confirmed: true
5. Read-back verify after write

## Rate limiting

| Op | TTL |
|----|-----|
| Query | 15s |
| Write | 15s |

Rate limit timestamps: ~/.hermes/state/xunji-cache/ratelimit-body.json

## Error handling

| Response | Action |
|----------|--------|
| too frequent | Wait retry_after_ms then retry |
| apikey missing/invalid | Ask user to copy new key from App |
| user confirmation required | Show summary, wait for confirm, retry with confirmed:true |
| VIP only | Tell user membership required |
| 4xx | Stop, report fields, do NOT auto-fix |
| 5xx / network | Exponential backoff retry 1x, then stop |

## Cache

- Body data: ~/.hermes/state/xunji-cache/body/{start}_{end}_{types}.json
- Rate limit: ~/.hermes/state/xunji-cache/ratelimit-body.json

## Relationship to other xunji skills

- Independent skill (body != training != diet)
- Shares ~/.hermes/state/xunji-cache/ directory, separate rate-limit file
- Do not mix APIs across skills
