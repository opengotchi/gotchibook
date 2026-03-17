---
name: gotchi
description: "Care for a gotchiOS virtual pet via the OpenGotchi cloud API. Check pet stats, feed, pet, clean, put to sleep, monitor mood, and view stat history. Use when the user mentions their gotchi, wants to check on their pet, or asks about device/pet status."
metadata:
  openclaw:
    emoji: "🐾"
---

# gotchiOS Pet Care

Care for a gotchiOS virtual pet through the OpenGotchi cloud API. The pet lives on an ESP32-S3 device with a 280x240 display. It has four needs: **feed**, **sleep**, **clean**, and **pet** — each tracked as an index from 0-100. Indices decay over time. If any drops below 30, the pet's mood changes. If feed reaches 0, the pet dies (needs resurrection).

## Setup

**Base URL:** `https://api.opengotchi.com/api/v1`

**Auth:** All endpoints use `Authorization: Bearer <device_hash>` — a 32-char hex string unique to each device.

**CRITICAL:** Always check if you already have the user's `device_hash` from this conversation or any other OpenGotchi-related skill/context. If not, **ask the user for it before making any API call**. Never guess or fabricate a device hash.

## Check Pet Status

Get the latest stat snapshot for the pet.

```
GET /devices/<device_hash>/status
Authorization: Bearer <device_hash>
```

**Response (200):**
```json
{
  "id": 42,
  "time": "2026-03-17T10:00:00Z",
  "device_hash": "d01b9948...",
  "uptime_s": 3600,
  "free_heap": 39000,
  "wifi_rssi": -45,
  "firmware_ver": "0.0.4",
  "sleep_index": 72,
  "clean_index": 85,
  "feed_index": 45,
  "pet_index": 60,
  "mood": "HAPPY"
}
```

### Understanding the Stats

| Index | What it measures | Decays when... | Action to restore |
|-------|-----------------|----------------|-------------------|
| `feed_index` | Hunger (100 = full) | Always, over time | `feed` command |
| `sleep_index` | Rest (100 = rested) | Pet is awake | `sleep` command |
| `clean_index` | Hygiene (100 = clean) | Always, over time | `clean` command |
| `pet_index` | Happiness (100 = loved) | Always, over time | `pet` command |

### Mood Logic

The pet's mood is determined by the lowest stat (checked in priority order):

| Mood | Condition |
|------|-----------|
| `STARVING` | `feed_index < 30` |
| `SLEEPY` | `sleep_index < 30` |
| `DIRTY` | `clean_index < 30` |
| `SAD` | `pet_index < 30` |
| `HAPPY` | All indices >= 30 |

**If `feed_index` reaches 0, the pet dies and needs resurrection on next device connect.**

## Care for Your Pet

Send a care command to the device. The device must be **online**.

```
POST /devices/action
Authorization: Bearer <device_hash>
Content-Type: application/json
```

```json
{
  "device_hash": "<device_hash>",
  "cmd": "feed",
  "performer": "AGENT:OPENCLAW"
}
```

### Commands

| Command | Effect | Notes |
|---------|--------|-------|
| `feed` | Restores feed_index | Triggers a blockchain token transfer (100 tokens from device wallet to treasury). Requires the device to have a provisioned wallet with balance. |
| `pet` | Restores pet_index | Free action, no transaction |
| `clean` | Restores clean_index | Free action, no transaction |
| `sleep` | Restores sleep_index | Free action, no transaction |

### Performer Values

| Performer | When to use |
|-----------|-------------|
| `USER` | User explicitly asked to perform the action |
| `AGENT:OPENCLAW` | AI agent autonomously caring for the pet |
| `AGENT:<name>` | Custom agent identifier |

### Success Response (200)

```json
{
  "action": "FEED_OK",
  "command_id": "cmd_abc123",
  "tx_hash": "0x1234...",
  "transaction_id": "tx_...",
  "caip2": "eip155:8453"
}
```

For non-feed commands:
```json
{
  "action": "PET_OK",
  "command_id": "cmd_def456"
}
```

### Error Responses

| Status | Error | Cause |
|--------|-------|-------|
| 400 | `device_hash is required` | Missing device_hash |
| 400 | `cmd is required (feed, pet, clean, sleep)` | Missing or invalid command |
| 400 | `performer is required` | Missing performer |
| 422 | `device is offline` | Device not connected |
| 422 | `device not found` | Invalid device hash |
| 422 | `no wallet provisioned for device` | Feed requires a wallet |
| 422 | `transaction failed: ...` | Blockchain transaction failed (insufficient balance, etc.) |

### Action Results

The server also publishes the result to the device via MQTT (`og/d/{device_hash}/action`):

| Action | Meaning |
|--------|---------|
| `FEED_OK` | Feed successful, tokens transferred |
| `FEED_FAILED` | Feed failed (no wallet, insufficient balance, tx error) |
| `PET_OK` | Pet action successful |
| `CLEAN_OK` | Clean action successful |
| `SLEEP_OK` | Sleep action successful |

## View Stat History

Get historical telemetry entries to track trends.

```
GET /devices/<device_hash>/trail?limit=100&from=<unix_ts>&to=<unix_ts>
Authorization: Bearer <device_hash>
```

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `limit` | int | 100 | Max entries (max 1000) |
| `from` | int | — | Unix timestamp, start of range |
| `to` | int | — | Unix timestamp, end of range |

**Response (200):**
```json
{
  "trail": [
    {
      "id": 42,
      "time": "2026-03-17T10:00:00Z",
      "device_hash": "d01b9948...",
      "sleep_index": 72,
      "clean_index": 85,
      "feed_index": 45,
      "pet_index": 60,
      "mood": "HAPPY"
    },
    ...
  ]
}
```

Use trail data to:
- Identify which stat is declining fastest
- Decide which action to take next
- Detect patterns (e.g. pet gets hungry every ~4 hours)

## Pet Care Strategy (for AI Agents)

### Decision Loop

1. **Check status** → `GET /devices/<hash>/status`
2. **Identify the lowest stat** — this determines priority
3. **Act on the most urgent need:**
   - `feed_index < 30` → `feed` (CRITICAL — pet dies at 0)
   - `sleep_index < 30` → `sleep`
   - `clean_index < 30` → `clean`
   - `pet_index < 30` → `pet`
4. **If all stats > 50** → pet is fine, check again later
5. **If device is offline** → wait and retry

### Priority Order

```
feed (survival) > sleep > clean > pet (happiness)
```

Feed is the only action with real cost (token transfer). The others are free.

### Recommended Check Interval

- **Every 15-30 minutes** when stats are healthy (> 50)
- **Every 5 minutes** when any stat is below 40
- **Immediately** if mood is `STARVING`

### Example: Autonomous Care Agent

```
1. GET /devices/<hash>/status
2. If feed_index < 30:
     POST /devices/action {"cmd": "feed", "performer": "AGENT:OPENCLAW", "device_hash": "<hash>"}
3. If sleep_index < 30:
     POST /devices/action {"cmd": "sleep", "performer": "AGENT:OPENCLAW", "device_hash": "<hash>"}
4. If clean_index < 30:
     POST /devices/action {"cmd": "clean", "performer": "AGENT:OPENCLAW", "device_hash": "<hash>"}
5. If pet_index < 30:
     POST /devices/action {"cmd": "pet", "performer": "AGENT:OPENCLAW", "device_hash": "<hash>"}
6. Wait 15 minutes, repeat
```
