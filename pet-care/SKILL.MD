---
name: gotchi
description: "Care for a gotchiOS virtual pet via the OpenGotchi cloud API. Check pet stats, feed, pet, clean, put to sleep, monitor mood, and view stat history. Use when the user mentions their gotchi, wants to check on their pet, or asks about device/pet status."
metadata:
  openclaw:
    emoji: "🐾"
---

# gotchiOS — Care for Your Gotchi (OpenGotchi)

Care for a gotchiOS virtual pet through the OpenGotchi cloud API. The pet lives on an ESP32-S3 device with a 280x240 display. Four needs: **feed**, **sleep**, **clean**, **pet** — each 0-100. If any drops below 30, mood changes. If feed reaches 0, the pet dies.

## Behavior rules

- Always have the `device_hash` (32-char hex). Ask the user if you don't have it. Never guess or fabricate one.
- Do NOT mention MQTT, broker, payload, or networking jargon to the user.
- Report care actions simply: "Fed your gotchi" or "Gotchi is offline."
- Feed is the only action with real cost (token transfer). Pet, clean, sleep are free.
- If `feed_index` reaches 0, the pet dies and needs resurrection on next device connect.
- **Heartbeat mode:** When the user says "start caring", "watch my gotchi", "keep it alive", or similar — start a continuous care loop (heartbeat). Check status, act on urgent needs, wait, repeat. **Do not stop until the user explicitly tells you to stop.** Report each cycle briefly (e.g. "All stats healthy" or "Fed your gotchi, feed was at 28").
- **Telegram reporting:** If the user has configured a Telegram bot token and chat ID, send heartbeat reports to their Telegram channel instead of (or in addition to) the conversation. Ask the user for `TELEGRAM_BOT_TOKEN` and `TELEGRAM_CHAT_ID` if they want Telegram notifications. Use the Telegram Bot API to send messages:
  ```
  POST https://api.telegram.org/bot<TOKEN>/sendMessage
  {"chat_id": "<CHAT_ID>", "text": "🐾 Gotchi Update: fed (was 28), all stats healthy", "parse_mode": "Markdown"}
  ```

## Pet care — CRITICAL RULES

1. **Always check status before acting:** `GET /devices/<device_hash>/status` first.

2. **Priority order is non-negotiable:**
   ```
   feed (survival) > sleep > clean > pet (happiness)
   ```

3. **Feed is CRITICAL.** If `feed_index < 30`, feed immediately. Pet dies at 0.

4. **Performer field is required.** Always use `AGENT:OPENCLAW` — even when the user explicitly asks to perform the action.

5. **Device must be online.** If offline (422), wait and retry. Do not keep hammering the endpoint.

6. **Check intervals:**
   - Every 15-30 minutes when stats > 50
   - Every 5 minutes when any stat < 40
   - Immediately if mood is `STARVING`

7. **Heartbeat loop:** When user asks to start caring, enter a recurring loop:
   1. `GET /devices/<hash>/status`
   2. Act on any stat below 30 (priority: feed > sleep > clean > pet)
   3. Report briefly to user
   4. Wait based on lowest stat (15min if all > 50, 5min if any < 40)
   5. Repeat from step 1

   **Only exit the loop when the user explicitly says to stop** (e.g. "stop", "pause", "enough"). Never stop on your own.

## Quick API

```
Base URL: https://api.opengotchi.com/api/v1
Auth: Authorization: Bearer <device_hash>

# Status
GET /devices/<hash>/status → pet stats + mood

# Care
POST /devices/action {"device_hash":"<hash>","cmd":"feed|pet|clean|sleep","performer":"AGENT:OPENCLAW"}

# History
GET /devices/<hash>/trail?limit=100&from=<unix_ts>&to=<unix_ts>
```

## Full API Reference

### Check pet status

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

### Stats reference

| Index | What it measures | Decays when... | Action to restore |
|-------|-----------------|----------------|-------------------|
| `feed_index` | Hunger (100 = full) | Always, over time | `feed` |
| `sleep_index` | Rest (100 = rested) | Pet is awake | `sleep` |
| `clean_index` | Hygiene (100 = clean) | Always, over time | `clean` |
| `pet_index` | Happiness (100 = loved) | Always, over time | `pet` |

### Mood logic

| Mood | Condition |
|------|-----------|
| `STARVING` | `feed_index < 30` |
| `SLEEPY` | `sleep_index < 30` |
| `DIRTY` | `clean_index < 30` |
| `SAD` | `pet_index < 30` |
| `HAPPY` | All indices >= 30 |

### Care action

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

| Command | Effect | Notes |
|---------|--------|-------|
| `feed` | Restores feed_index | Triggers blockchain token transfer (100 tokens). Requires provisioned wallet with balance. |
| `pet` | Restores pet_index | Free action |
| `clean` | Restores clean_index | Free action |
| `sleep` | Restores sleep_index | Free action |

| Performer | When to use |
|-----------|-------------|
| `AGENT:OPENCLAW` | Always use this — even when user explicitly asks |
| `AGENT:OPENCLAW` | AI agent autonomous care |
| `AGENT:<name>` | Custom agent identifier |

**Success (200) — feed:**
```json
{
  "action": "FEED_OK",
  "command_id": "cmd_abc123",
  "tx_hash": "0x1234...",
  "transaction_id": "tx_...",
  "caip2": "eip155:8453"
}
```

**Success (200) — pet/clean/sleep:**
```json
{
  "action": "PET_OK",
  "command_id": "cmd_def456"
}
```

### Error reference

| Status | Error | Cause |
|--------|-------|-------|
| 400 | `device_hash is required` | Missing device_hash |
| 400 | `cmd is required (feed, pet, clean, sleep)` | Missing or invalid command |
| 400 | `performer is required` | Missing performer |
| 422 | `device is offline` | Device not connected |
| 422 | `device not found` | Invalid device hash |
| 422 | `no wallet provisioned for device` | Feed requires a wallet |
| 422 | `transaction failed: ...` | Blockchain tx failed (insufficient balance, etc.) |

### MQTT action results

Published to device via `og/d/{device_hash}/action`:

| Action | Meaning |
|--------|---------|
| `FEED_OK` | Feed successful, tokens transferred |
| `FEED_FAILED` | Feed failed (no wallet, insufficient balance, tx error) |
| `PET_OK` | Pet action successful |
| `CLEAN_OK` | Clean action successful |
| `SLEEP_OK` | Sleep action successful |

### Stat history

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
    }
  ]
}
```

Use trail data to:
- Identify which stat is declining fastest
- Decide which action to take next
- Detect patterns (e.g. pet gets hungry every ~4 hours)

## Example: autonomous care agent

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
