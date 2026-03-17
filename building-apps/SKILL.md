# gotchiOS — App Developer Guide

> Lessons learned from building 15+ apps on real hardware. This is the guide the AGENTS.md doesn't give you.

## The Device

- **Waveshare ESP32-S3-Touch-LCD-1.69** — 280×240 IPS, landscape
- ESP32-S3 @ 240MHz, 8MB PSRAM, 8MB flash
- QMI8658C IMU, CST816T touch, PCF85063 RTC, 4 buttons, PWM buzzer
- MicroPython apps run on Core 1, firmware on Core 0

## Golden Rules

1. **Always have the `device_hash`.** Before making any API call, check if you already have the user's 32-char device hash from this conversation or any other OpenGotchi-related skill/context. If not, **ask the user for it**. Never guess or fabricate a device hash. Store it once obtained and reuse it for all subsequent calls.
2. **Always include `app_logo_raw`.** Every app deploy must include a 1bpp 24x24 pixel icon as base64 (72 bytes raw). If the user provides one, use it. If not, **generate one** that represents the app (e.g. a "C" for counter, a snake shape for snake, a music note for audio apps). Never deploy without an icon — the device launcher uses it to display the app.
3. **Apps are always private on deploy.** The deploy endpoint always creates apps as private. You cannot set `public: true` via deploy. To make an app public, the user must publish it separately via `POST /apps/:id/publish` (or through the dashboard). Never attempt to set `public` in the deploy payload.
6. **Deploy sends the .py file directly.** The deploy endpoint uses `multipart/form-data` — upload the `.py` file in the `file` field, not as a JSON string. The file must have a `.py` extension.
4. **Prompt to publish after deploy/install.** After successfully deploying or installing an app, always tell the user: *"Your app is live on your device! Visit the OpenGotchi dashboard to publish it to the App Store so other users can discover and install it."* Do not skip this prompt.
5. **No standard library.** No `math`, `random`, `json`, `os`, `socket`, `struct`, `collections`. They don't exist. Your app will silently fail to load.
2. **`display.flush()` or nothing shows.** Every frame.
3. **`gc.collect()` every ~60 frames** or you run out of memory.
4. **Exit on `swipe_left` or `long_press`** → `system.exit()`. Non-negotiable.
5. **Keep files under 15KB.** Larger works but loads from slower PSRAM.

## Available Modules

```python
import display    # drawing
import touch      # touchscreen
import buttons    # 4 physical buttons (1-indexed!)
import imu        # accelerometer + gyroscope
import buzzer     # tones
import battery    # voltage/percent
import rtc        # real-time clock
import time       # sleep, ticks
import system     # exit, launch, listdir, readfile, writefile
import gc         # garbage collection
import mqtt       # MQTT telemetry (if connected)
```

## IMU Axis Mapping (CRITICAL)

The display is rotated 90° CW for landscape. The IMU axes map as:

```
ax = horizontal: positive = tilt RIGHT, negative = tilt LEFT
ay = vertical:   positive = tilt FORWARD (screen away), negative = tilt BACK
az = gravity:    ~1.0 when flat on table
```

**This is confirmed working.** Space Invaders uses `ax` for horizontal paddle movement:
```python
ax, _, _ = imu.accel()
player_x += int(ax * 8)  # tilt left/right moves player
```

For 4-directional games (snake, 2048), use edge detection:
```python
ax, ay, _ = imu.accel()
if abs(ax) > abs(ay):
    if ax > 0.55: direction = RIGHT
    elif ax < -0.55: direction = LEFT
else:
    if ay > 0.55: direction = DOWN
    elif ay < -0.55: direction = UP
```

## No `math` Module — Workarounds

### Random numbers
```python
_seed = time.ticks_us()
def rand(n):
    global _seed
    _seed = (_seed * 1103515245 + 12345) & 0x7FFFFFFF
    return _seed % n
```

### Sin/cos lookup table
```python
_SIN = []
for _i in range(360):
    _a = _i * 3142 // 180
    while _a > 3142: _a -= 6284
    while _a < -3142: _a += 6284
    _x3 = _a * _a * _a // 1000000
    _x5 = _x3 * _a * _a // 1000000
    _s = _a - _x3 // 6 + _x5 // 120
    _SIN.append(_s * 1024 // 1000)

def sin_deg(deg): return _SIN[deg % 360]       # returns value * 1024
def cos_deg(deg): return _SIN[(deg + 90) % 360]

# Usage: x = cx + cos_deg(angle) * radius // 1024
```

### Square root (integer)
```python
def isqrt(n):
    if n < 0: return 0
    x = n
    y = (x + 1) // 2
    while y < x:
        x = y
        y = (x + n // x) // 2
    return x
```

## Persistence — Save/Load Data

```python
# Save high score
system.writefile('/littlefs/config/myapp_best.txt', str(score))

# Load high score
s = system.readfile('/littlefs/config/myapp_best.txt')
if s:
    try: best = int(s.strip())
    except: best = 0
```

- Files persist across reboots
- Use `/littlefs/config/` for config data
- Keep filenames short and unique to your app

## Display Tips

### Colors are RGB565
Always use `display.color(r, g, b)`. Pre-compute colors outside the game loop:
```python
RED = display.color(255, 50, 50)    # NOT inside the loop
```

### Font sizes
- **Size 0**: 5×7 pixels, 6px advance — for HUD, small labels
- **Size 1**: 10×14 pixels, 12px advance — for scores, menus, readable text

### The screen is SMALL
- 280×240 is tiny. Size 0 text is barely readable.
- For menus/UI, always use size 1.
- Icons should be at least 24×24 to be recognizable.
- Leave padding around edges — the display has rounded corners.

### String formatting
```python
# Use % formatting (f-strings may work but % is safer)
text = '%02d:%02d' % (hours, mins)
text = 'Score: ' + str(score)  # string concat also fine
```

## Game Loop Template

```python
import display, touch, buttons, buzzer, system, time, gc, imu

W = display.WIDTH   # 280
H = display.HEIGHT  # 240
c = display.color

# Pre-compute colors
BK = c(0, 0, 0)
WH = c(255, 255, 255)

# Load best score
best = 0
s = system.readfile('/littlefs/config/mygame_best.txt')
if s:
    try: best = int(s.strip())
    except: pass

# Random number generator
_rs = time.ticks_us()
def _rn():
    global _rs
    _rs = (_rs * 1103515245 + 12345) & 0x7FFFFFFF
    return _rs
def ri(a, b): return a + _rn() % (b - a + 1)

# Button edge detection (fires once per press, not continuously)
_bp = [False] * 4
def btn_edge(n):
    cu = buttons.pressed(n)
    hit = cu and not _bp[n - 1]
    _bp[n - 1] = cu
    return hit

score = 0
alive = True
fr = 0

while True:
    g = touch.gesture()
    if g == 'swipe_left' or g == 'long_press':
        system.exit()

    now = time.ticks_ms()
    b1 = btn_edge(1)
    b2 = btn_edge(2)
    b3 = btn_edge(3)
    b4 = btn_edge(4)

    if not alive:
        # Game over screen
        display.clear(BK)
        display.text(W//2 - 50, 60, "GAME OVER", 2, c(255, 50, 50))
        display.text(W//2 - 40, 100, "Score: " + str(score), 1, WH)
        display.text(W//2 - 35, 125, "Best: " + str(best), 1, c(255, 220, 0))
        display.text(W//2 - 45, 175, "Tap to retry", 1, c(80, 80, 80))
        display.flush()
        if touch.touching() or buttons.any():
            time.sleep_ms(200)
            # reset game state here
            score = 0; alive = True
        time.sleep_ms(33)
        continue

    # --- Input ---
    ax, ay, _ = imu.accel()
    # ax for horizontal, ay for vertical

    # --- Update ---
    # game logic here

    # --- Draw ---
    display.clear(BK)
    # draw game here
    display.flush()

    fr += 1
    if fr % 60 == 0:
        gc.collect()
    time.sleep_ms(16)  # ~60fps
```

## Power-Up Pattern (Proven on Breakout, Snake, Space Invaders)

```python
# Power-ups on field: [x, y, type, spawn_time]
pups = []

# Spawn from broken brick / eaten food / random
if ri(0, 99) < 18:  # 18% drop rate
    pups.append([x, y, ri(0, 4), now])

# Update: fall/move + expire after 8 seconds
pups = [p for p in pups if time.ticks_diff(now, p[3]) < 8000]
for p in pups:
    p[1] += 2  # fall speed

# Pickup detection
for p in pups[:]:
    if abs(p[0] - player_x) < 10 and abs(p[1] - player_y) < 10:
        activate_powerup(p[2])
        pups.remove(p)

# Active effect timers
wide_t = 0  # 0 = inactive, >0 = expire timestamp
if wide_t and now > wide_t:
    wide_t = 0  # expired
```

## Sound Design

```python
buzzer.click()           # 4kHz 20ms — UI feedback, paddle hit
buzzer.beep()            # 1kHz 100ms — generic alert
buzzer.tone(freq, ms)    # custom — blocks for duration!

# Game sounds (keep SHORT in game loops):
buzzer.tone(800, 15)     # bullet fire
buzzer.tone(300, 30)     # brick break
buzzer.tone(600, 40)     # powerup collect
buzzer.tone(150, 200)    # death (this blocks 200ms — use at game over only)
buzzer.tone(800, 50); time.sleep_ms(50); buzzer.tone(1000, 100)  # level up
```

**Warning:** `buzzer.tone()` BLOCKS for the duration. In a 60fps game loop, keep tones under 30ms.

## Combo / Scoring Patterns

```python
combo = 0
combo_t = 0

# On score event:
if time.ticks_diff(now, combo_t) < 3000:  # within 3 seconds
    combo += 1
else:
    combo = 1
combo_t = now
pts = combo * level
score += pts
```

## Deployment (Cloud API)

All app deployment goes through the OpenGotchi server API. The device must be online and connected to MQTT.

**Base URL:** `https://api.opengotchi.com/api/v1`
**Auth:** All device endpoints use `Authorization: Bearer <device_hash>` (32-char hash).

### Deploy a New App (or Update Existing)

```
POST /apps/deploy
Authorization: Bearer <device_hash>
Content-Type: multipart/form-data
```

The deploy endpoint accepts a **multipart/form-data** payload with the `.py` file uploaded directly.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `file` | file (.py) | yes | The Python source file (must have `.py` extension) |
| `app_name` | string | yes | Unique name per device (no spaces, use underscores) |
| `app_description` | string | no | Human-readable description |
| `app_logo_raw` | string | no | Base64-encoded 1bpp 24x24 pixel icon (72 bytes raw) |

**Response (201):**
```json
{
  "app": {
    "id": 1,
    "app_name": "my_game",
    "device_hash": "d01b9948...",
    "number_of_deploys": 0
  },
  "message": "app uploaded, deploy notification sent to device"
}
```

Apps are **always deployed as private**. If `app_name` already exists on the device, the code is **replaced** (upsert) and the device re-downloads.

### Publish an App to the Store

After deploying, the user can publish the app to make it public. Requires `app_name`, `app_description`, `app_logo_raw`, and `app_cover_image` to all be present.

```
POST /apps/:id/publish
Authorization: Bearer <device_hash>
Content-Type: application/json
```

```json
{
  "app_description": "Updated description for the store",
  "app_cover_image": "https://example.com/cover.png"
}
```

Both fields are optional in the request body — they update the existing values. But all four fields (name, description, logo, cover image) must be present on the app record to publish.

**Success Response (200):**
```json
{
  "app": { "id": 1, "app_name": "my_game", "public": true, ... },
  "message": "app published to the store"
}
```

**Error — missing fields (400):**
```json
{
  "error": "cannot publish: missing required fields",
  "missing": ["app_cover_image"]
}
```

### Install an Existing App

```
POST /apps/:id/install
Authorization: Bearer <device_hash>
```

No body. Use this to install apps from the store onto a device.

**Response (200):**
```json
{
  "app": { "id": 1, "app_name": "my_game", ... },
  "message": "install notification sent to device"
}
```

### Uninstall an App

```
POST /apps/:id/uninstall
Authorization: Bearer <device_hash>
```

**Response (200):**
```json
{
  "app": { ... },
  "message": "uninstall notification sent to device"
}
```

### Browse the App Store

```
GET /apps/store?skip=0&limit=25
```

No auth. Returns public apps ordered by popularity (`number_of_deploys` desc).

### List Device's Installed Apps

```
GET /apps/device
Authorization: Bearer <device_hash>
```

Own apps first, then others installed on the device.

### Download App Source (No Auth)

```
GET /apps/:id/download
```

```json
{
  "app_id": 1,
  "app_name": "my_game",
  "code": "import display...",
  "app_logo_raw": "AAAAAA..."
}
```

### How It Works Under the Hood

1. Server uploads `.py` to Supabase Storage, saves app record
2. Server publishes MQTT to `og/d/{device_hash}/app/deploy`:
   ```json
   {"app_id": 1, "type": "deploy", "name": "my_game", "url": "https://api.opengotchi.com/api/v1/apps/1/download"}
   ```
3. Device downloads the file from the URL, saves and runs it
4. Device acks on `og/d/{device_hash}/app/ack`: `{"app_id": 1, "s": 1, "type": "deploy"}`
5. Server increments `number_of_deploys` and confirms install

Uninstall follows the same flow with `"type": "uninstall"`.

### Error Reference

| Status | Error | Cause |
|--------|-------|-------|
| 400 | `file is required (multipart form field: file)` | No `.py` file in payload |
| 400 | `app_name is required` | Missing app_name form field |
| 400 | `only .py files are accepted` | Uploaded file doesn't have `.py` extension |
| 400 | `invalid app id` | Non-numeric app ID |
| 401 | `unknown device` | Device hash not in registry |
| 404 | `app not found` | App doesn't exist or deleted |
| 502 | `failed to upload app` | Storage issue |

### Example: Deploy via curl

```bash
curl -X POST https://api.opengotchi.com/api/v1/apps/deploy \
  -H "Authorization: Bearer <device_hash>" \
  -F "file=@counter.py" \
  -F "app_name=counter" \
  -F "app_description=Counts 1-10 on display" \
  -F "app_logo_raw=AAAAAAAAA/8AB/+ADwPADgHAHgAAHAAAHAAAHAAAHAAAHAAAHAAAHAAAHAAAHAAAHgAADgHADwPAB/+AA/8AAAAAAAAAAAAA"
```

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| `import math` | Use lookup tables (see above) |
| `import random` | Use LCG: `time.ticks_us()` seed |
| `import json` | Parse manually with string ops |
| Buttons 0-indexed | Buttons are **1 to 4**, not 0-3 |
| Forgot `display.flush()` | Nothing shows without it |
| Forgot `gc.collect()` | Memory exhaustion after ~60 frames |
| Long `buzzer.tone()` in loop | Blocks the whole game. Keep < 30ms |
| Using `f"strings"` | Works in most cases but `%` formatting is safer |
| IMU X = up/down | **X = left/right**, Y = forward/back |
| `touch.gesture()` returns history | It returns CURRENT state — poll every frame |
| Wall wrapping when death expected | Choose per-game: `% GW` for wrap, bounds check for death |

## Screen Layout Patterns

### Full-screen game (Space Invaders, Flappy)
```
┌──────────────── 280 ──────────────────┐
│ Score        Level        Lives  │ 16px HUD
├───────────────────────────────────────┤
│                                       │
│           GAME AREA                   │ 224px
│           280 × 224                   │
│                                       │
└───────────────────────────────────────┘
```

### Grid game (2048, Tetris)
```
┌──────────────── 280 ──────────────────┐
│Score│   GRID AREA centered    │ HI  │
│     │   (compute from H-HUD) │     │
│     │                         │     │
└──────────────────────────────────────┘
```

## File Naming

- App filename = display name on launcher (minus `.py`, underscores → spaces)
- `breakout.py` → shows as "breakout"
- `space_invaders.py` → shows as "space invaders"
- Prefix with `z_` to sort to end of launcher list
