---
name: gotchi
description: "Write MicroPython apps for a gotchiOS device. Use when user mentions gotchi, creating, or building an app."
metadata:
  openclaw:
    emoji: "🐾"
---

# gotchiOS — Building Apps (OpenGotchi)

Write MicroPython apps for a gotchi (Waveshare ESP32-S3-Touch-AMOLED-1.8 — SH8601 QSPI AMOLED **448x368**, capacitive touch, IMU, RTC, ES8311 speaker + mic, WiFi, 16MB flash + PSRAM).

## Behavior rules

- Focus purely on writing correct app code. The playground handles running and installing the app itself — never ask the user for a device hash, secret, or any connection detail.
- Always include `app_logo_raw` — 1bpp 24x24 pixel icon as base64 (72 bytes raw). Generate one if the user doesn't provide it.
- File naming: `breakout.py` → "breakout", `space_invaders.py` → "space invaders". Prefix `z_` to sort to end.

## Device spec

| | gotchi |
|---|---|
| Display | SH8601 QSPI AMOLED, **448 x 368** landscape (368x448 panel, rotated 90°) |
| Touch | FT3168 / FT6146 capacitive (I2C 0x38) |
| Buttons | **Button 1 only** (BOOT). Buttons 2-4 are NOT wired |
| Buzzer | No piezo — `buzzer.*` is a shim over the speaker |
| Audio | ES8311 codec: speaker + microphone, 16-bit mono PCM |
| IMU | QMI8658 (accel + gyro + temp) |
| RTC | PCF85063 |
| Power | AXP2101 PMIC — battery voltage, percent, charging state |
| Network | WiFi + MQTT + HTTP client + on-device HTTP server |

## Writing apps — CRITICAL RULES

1. **Exit gesture REQUIRED in every app main loop:**
   ```python
   g = touch.gesture()
   if g == 'swipe_left':
       system.exit()
   ```

2. **Sleep REQUIRED in every loop:** `time.sleep_ms(33)` minimum. No sleep = watchdog reboot.

3. **Call `gc.collect()` periodically** in long-running apps.

4. **DO NOT USE:** `str()`, f-strings, `print()`, `open()`, `import random`, `import math`, `import json`, any standard library. They don't exist.

5. **Use `%` formatting:** `"%d" % n` not `str(n)`. `"%s %d" % (a, b)` not f-strings.

6. **Display:** 448x368, call `display.flush()` once per frame, `display.clear(color)` before redraw. `display.text(x, y, string, size, color)` — x,y first!

7. **Colors:** Always `display.color(r, g, b)`, never hardcode RGB565. AMOLED blacks are true black — `display.color(0,0,0)` costs no power, so dark UIs are cheap.

8. **Never hardcode 448/368.** Derive from `display.WIDTH` / `display.HEIGHT` so the app also runs on smaller gotchiOS panels.

9. **Only Button 1 exists.** Design input around touch + gestures + IMU. Use buttons 2-4 only as an optional extra, never as the only way to do something.

10. **Keep files under 15KB.** Larger works but loads from slower PSRAM.

## Quick API

```python
import display, touch, time, system, gc, buttons, buzzer, imu, battery, rtc
import audio, http, wifi, mqtt          # new on gotchi

# Display
display.color(r,g,b)  display.clear(c)  display.text(x,y,s,sz,c)
display.rect_filled(x,y,w,h,c)  display.circle_filled(cx,cy,r,c)
display.line(x0,y0,x1,y1,c)  display.flush()

# Input
touch.gesture()  touch.pos()  touch.touching()
buttons.pressed(1)  buttons.any()

# Sensors
imu.accel()->(ax,ay,az)  imu.tilt()->(tx,ty)  imu.temperature()

# Audio
buzzer.tone(hz,ms)  buzzer.beep()  buzzer.click()
audio.tone(hz,ms,vol)  audio.play(pcm,rate)  audio.record(ms,rate)

# Network
http.get(url)  http.request(m,url,hdrs,body)  http.server.register(m,path,fn)

# System
system.exit()  system.launch(path)  system.readfile(p)  system.writefile(p,d)
rtc.now()->(y,m,d,h,min,s)  battery.percent()  battery.charging()  time.ticks_ms()
```

## Full API Reference

### display (448x368 RGB565 landscape)

```python
import display
display.WIDTH                              # 448
display.HEIGHT                             # 368
display.color(r, g, b)                     # RGB888 → RGB565 int
display.clear(color)                       # fill screen
display.pixel(x, y, color)
display.rect(x, y, w, h, color)           # outline
display.rect_filled(x, y, w, h, color)    # filled
display.circle(cx, cy, r, color)          # outline
display.circle_filled(cx, cy, r, color)   # filled
display.line(x0, y0, x1, y1, color)
display.text(x, y, string, size, color)   # x, y first, then string
display.bitmap(x, y, w, h, data, color)   # 1-bit bitmap
display.sprite(x, y, w, h, data, transparent_color)  # RGB565 sprite
display.gray4(x, y, w, h, data, fg, bg)   # 4bpp grayscale blend fg→bg
display.qr(x, y, text, scale, fg, bg)     # → QR side length in modules (0 = encode error)
display.flush()                            # push to screen
display.backlight(0-255)                   # AMOLED brightness (SH8601 WRDISBV)
```

Font: built-in 5x7 monospace. Char advance is `6 * (size + 1)` px — size 0=6px wide, 1=12px, 2=18px, 3=24px per char.

There is **no `display.flush_rect`** — flush the whole frame.

Centering helper:
```python
def cw(size): return 6 * (size + 1)
def cx(t, size): return max(0, (display.WIDTH - len(t) * cw(size)) // 2)
display.text(cx("HELLO", 2), 40, "HELLO", 2, WHITE)
```

### imu (QMI8658)

```python
import imu
ax, ay, az = imu.accel()       # g (float), X=right Y=down Z=out
gx, gy, gz = imu.gyro()        # degrees/sec (float)
tx, ty = imu.tilt()             # degrees (float)
temp = imu.temperature()        # °C (float)
```

IMU axis mapping (display rotated 90° CW for landscape):

```
ax = horizontal: positive = tilt RIGHT, negative = tilt LEFT
ay = vertical:   positive = tilt FORWARD (screen away), negative = tilt BACK
az = gravity:    ~1.0 when flat on table
```

Horizontal movement (Space Invaders):
```python
ax, _, _ = imu.accel()
player_x += int(ax * 8)  # tilt left/right moves player
```

4-directional (snake, 2048):
```python
ax, ay, _ = imu.accel()
if abs(ax) > abs(ay):
    if ax > 0.55: direction = RIGHT
    elif ax < -0.55: direction = LEFT
else:
    if ay > 0.55: direction = DOWN
    elif ay < -0.55: direction = UP
```

### buttons

```python
import buttons
buttons.pressed(n)    # n=1-4, returns bool
buttons.any()         # any button pressed?
buttons.which()       # list of pressed button numbers
```

**Only Button 1 (BOOT) is physically wired on this device.** `buttons.pressed(2..4)` always returns `False`. Treat button input as a single action key and put everything else on touch.

Edge detection pattern:
```python
prev = False
while True:
    cur = buttons.pressed(1)
    if cur and not prev:
        pass  # button 1 just pressed
    prev = cur
    time.sleep_ms(30)
```

### touch (FT3168 / FT6146)

```python
import touch
touch.touching()    # bool
touch.pos()         # (x, y) or None — already in landscape coords
touch.gesture()     # one-shot string, clears after read
```

Gestures: "none", "press", "release", "swipe_up", "swipe_down", "swipe_left", "swipe_right", "long_press"

Touch coords come out of the panel in portrait space and are remapped by the driver, so `touch.pos()` matches `display` coordinates directly.

### buzzer

```python
import buzzer
buzzer.tone(freq_hz, duration_ms)
buzzer.beep()
buzzer.click()
buzzer.stop()
```

There is no piezo buzzer on this device — these calls are routed to the ES8311 speaker at the current `audio.volume()`. They're **blocking** for `duration_ms` (unlike the PWM buzzer on older boards), so keep tones short (≤50ms) inside a frame loop. For anything richer, use `audio` directly.

### audio (ES8311 codec — speaker + mic)

PCM is always **16-bit signed, mono, little-endian**. Default rate 16000 Hz.

```python
import audio
audio.init()                  # bool — brought up lazily, rarely needed explicitly
audio.ready()                 # bool

# --- speaker ---
audio.play(pcm, rate=16000)   # write PCM bytes → bytes written
audio.tone(freq, ms, vol=70)  # sine beep, BLOCKING → bool
audio.volume(pct)             # 0..100 → bool

# --- microphone (one-shot) ---
audio.record(ms, rate=16000)  # → bytes of PCM, or None

# --- microphone (streaming) ---
audio.mic_open(rate=16000)    # → bool
audio.mic_read(nbytes)        # → bytes (blocking), or None
audio.mic_read_into(buf)      # bytes read into a bytearray, 0 on error
audio.mic_gain(db)            # analog gain, e.g. 30.0 → bool
audio.mic_close()

# --- continuous background recorder (mic → PSRAM ring) ---
audio.rec_start(rate=16000, ring_secs=20, ulaw=False)  # → bool
audio.rec_running()                       # → bool
audio.rec_read_into(buf, timeout_ms=2000) # bytes popped from the ring
audio.rec_available()                     # bytes waiting
audio.rec_dropped()                       # bytes lost (consumer too slow)
audio.rec_level()                         # recent peak 0..32767 — for live VU meters
audio.rec_stop()
```

Recorder notes:
- The ES8311 clock is fixed at 16 kHz and the recorder decimates 2:1 in software, so **the ring is always 8 kHz** regardless of the `rate` you pass. Size `ring_secs` against 8 kHz.
- `ulaw=True` stores G.711 mu-law (1 byte/sample instead of 2) — half the byte rate, telephony speech quality. 8 kHz mu-law = 8 KB/s.
- The recorder runs on its own task, so `rec_read_into` never blocks the UI. Poll `rec_level()` each frame to animate a waveform.

Never call `audio.record()` or `audio.tone()` on the frame path — they block. Use the recorder ring plus `rec_read_into` for anything longer than a beep.

### http (client)

```python
import http

# One-shot request. Response body capped at 64KB; anything past that is dropped.
status, body = http.request(method, url, headers=None, body=b"", timeout_ms=15000)
status, body = http.get(url, headers=None, timeout_ms=15000)

# Streaming GET — callback(bytes) per TCP chunk, no whole-body buffer.
# Return False from the callback to abort mid-stream.
status = http.stream(url, callback, headers=None, timeout_ms=60000)

# PUT whose body is the concatenation of parts. Each part is either
# bytes/str (sent verbatim) or ("mic", nbytes) to pull straight from an
# already-open audio mic without touching Python memory.
status = http.put_stream(url, parts, headers=None, timeout_ms=60000)
http.put_close()
```

HTTPS works out of the box (ESP-IDF cert bundle). `status` is `0` on transport failure. Client buffers live on the system heap, not the app's small GC heap. There is **no `json` module** — build request bodies with `%` formatting and pull fields out of the response bytes with `find()`/slicing.

Background uploader — a C task drains a FIFO so the UI never blocks on the network:

```python
http.upload_start(base_url, headers={}, queue=4)       # → bool
http.upload_queue(filename, prefix_bytes, mic_bytes)   # → True queued / False full
http.upload_poll()                                     # → (in_flight, done_ok, done_fail)
http.upload_drain(timeout_ms=120000)                   # → True if queue went idle
http.upload_stop()                                     # drop queue + connection
```

`upload_queue` sends `prefix_bytes` (e.g. a WAV header) followed by `mic_bytes` drained directly from the recorder ring — so audio uploads *while* recording continues.

### http.server (on-device HTTP server)

Also importable as the top-level `http_server` module; `http.server` is the same object.

```python
import http

def handler(req):
    # req = {"method": str, "path": str, "query": str, "body": bytes}
    return (200, b"<h1>hi</h1>", "text/html")
    # or (status, body) — or just status for an empty body

http.server.register("GET", "/", handler)     # method: GET|POST|PUT|DELETE|PATCH
http.server.unregister("GET", "/")
http.server.routes()                          # [(method, path), ...]
http.server.start(8080)
http.server.stop()
http.server.is_running()                      # bool
http.server.port()                            # int, 0 when stopped
```

Handlers run on the MicroPython task, one request at a time, and block the app while running — keep them short. **Always `http.server.stop()` before `system.exit()`.**

### battery

```python
import battery
battery.voltage()    # float volts (via AXP2101)
battery.percent()    # int 0-100
battery.charging()   # bool — USB charge current flowing into the cell
```

### rtc (PCF85063)

```python
import rtc
year, mon, day, hrs, mins, secs = rtc.now()
rtc.set(year, mon, day, hrs, mins, secs)
```

### time

```python
import time
time.sleep(seconds)          # float
time.sleep_ms(ms)            # int
time.ticks_ms()              # monotonic ms
time.ticks_us()              # monotonic us
time.ticks_diff(a, b)        # a - b with wrap handling
```

### system

```python
import system
system.exit()                               # back to launcher
system.launch("/littlefs/apps/x.py")        # switch to another app
system.listdir("/littlefs/apps")            # list app filenames
system.readfile("/littlefs/data/x.txt")     # read text file (max 4KB) → str or None
system.readbytes("/littlefs/data/x.bin")    # read binary (max 256KB) → bytes or None
system.writefile("/littlefs/data/x.txt", "data")  # write file → bool
```

`readbytes` at 256KB covers RGB565 sprite sheets up to 256x256 — big enough to stream animation frames off flash.

### wifi

```python
import wifi
wifi.status()          # "connected" | "connecting" | "disconnected" | "ap_portal" | "off"
wifi.ip()              # "192.168.1.42" or None
wifi.send("msg")       # send to WebSocket clients → bool
wifi.recv()            # receive WebSocket message → str or None
wifi.notify_count()    # unread notification count
wifi.notifications()   # list of {"title":..., "body":..., "time":...}
wifi.forget()          # forget the saved network
wifi.clear_creds()     # erase WiFi credentials, reboot into AP mode
```

Always check `wifi.status() == "connected"` before any `http.*` call, and draw a fallback state when it isn't.

### mqtt

```python
import mqtt
mqtt.connected()       # bool
mqtt.hash()            # str — device hash
mqtt.secret()          # str | None — provisioned device secret
mqtt.log_count()       # int
mqtt.log_read()        # str — oldest unread log line
mqtt.send_telemetry(feed, sleep, clean, pet)   # → bool
```

Resolve device credentials at runtime via `mqtt.hash()` / `mqtt.secret()` — never hardcode them in an app.

### gc

```python
import gc
gc.collect()    # run garbage collector
```

## No `math` module — workarounds

Random numbers:
```python
_seed = time.ticks_us()
def rand(n):
    global _seed
    _seed = (_seed * 1103515245 + 12345) & 0x7FFFFFFF
    return _seed % n
```

Sin/cos lookup table:
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

Square root (integer):
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

## Persistence

```python
# Save high score
system.writefile('/littlefs/config/myapp_best.txt', "%d" % score)

# Load high score
s = system.readfile('/littlefs/config/myapp_best.txt')
if s:
    try: best = int(s.strip())
    except: best = 0
```

Files persist across reboots. Use `/littlefs/config/` for config data. Keep filenames short and unique.

## Game patterns

Power-ups (proven on Breakout, Snake, Space Invaders):
```python
pups = []  # [x, y, type, spawn_time]

if ri(0, 99) < 18:  # 18% drop rate
    pups.append([x, y, ri(0, 4), now])

pups = [p for p in pups if time.ticks_diff(now, p[3]) < 8000]
for p in pups:
    p[1] += 2

for p in pups[:]:
    if abs(p[0] - player_x) < 10 and abs(p[1] - player_y) < 10:
        activate_powerup(p[2])
        pups.remove(p)

wide_t = 0
if wide_t and now > wide_t:
    wide_t = 0
```

Combo scoring:
```python
combo = 0
combo_t = 0

if time.ticks_diff(now, combo_t) < 3000:
    combo += 1
else:
    combo = 1
combo_t = now
pts = combo * level
score += pts
```

Sound design:
```python
buzzer.click()           # UI feedback, paddle hit
buzzer.beep()            # generic alert
buzzer.tone(800, 15)     # bullet fire
buzzer.tone(300, 30)     # brick break
buzzer.tone(600, 40)     # powerup collect
buzzer.tone(150, 200)    # death — long, prefer audio.tone off the frame path
audio.volume(60)         # set once at startup
```

## Screen layouts

Full-screen game (Space Invaders, Flappy):
```
+---------------- 448 ------------------+
| Score        Level        Lives  | 24px HUD
+---------------------------------------+
|                                       |
|           GAME AREA                   | 344px
|           448 x 344                   |
|                                       |
+---------------------------------------+
```

Grid game (2048, Tetris):
```
+---------------- 448 ------------------+
|Score|   GRID AREA centered    | HI  |
|     |   (compute from H-HUD)  |     |
|     |                         |     |
+---------------------------------------+
```

The panel is nearly 60% wider and 50% taller than the older 280x240 gotchiOS screen — bump font sizes up one step (body text at size 1, headings at 2-3) rather than leaving small text floating in empty space.

## Common mistakes

| Mistake | Fix |
|---------|-----|
| `import math` / `random` / `json` | Use workarounds above; no `json` even for HTTP bodies |
| Using `str()`, f-strings, `print()` | Use `"%d" % n` formatting |
| Hardcoding 280x240 coordinates | This device is **448x368** — derive from `display.WIDTH`/`HEIGHT` |
| Using buttons 2-4 | Only **Button 1** is wired — build around touch |
| `display.flush_rect(...)` | Doesn't exist — `display.flush()` the whole frame |
| Forgot `display.flush()` | Nothing shows without it |
| Forgot `gc.collect()` | Memory exhaustion after ~60 frames |
| No `time.sleep_ms()` in loop | Watchdog reboot — minimum 33ms |
| Exit on `swipe_right` | Exit gesture is `swipe_left` |
| `audio.record()` in the frame loop | Blocking — use `audio.rec_start` + `rec_read_into` |
| Left `http.server` running on exit | Call `http.server.stop()` before `system.exit()` |
| IMU X = up/down | **X = left/right**, Y = forward/back |

## Template

```python
import display, touch, time, system, gc

W = display.WIDTH
H = display.HEIGHT

BLACK = display.color(0, 0, 0)
WHITE = display.color(255, 255, 255)

while True:
    g = touch.gesture()
    if g == 'swipe_left':
        system.exit()
    display.clear(BLACK)
    display.text(20, H // 2, "Hello", 3, WHITE)
    display.flush()
    time.sleep_ms(100)
    gc.collect()
```

## Example: game loop

```python
import display, touch, buttons, buzzer, system, time, gc, imu

W = display.WIDTH
H = display.HEIGHT
c = display.color

BK = c(0, 0, 0)
WH = c(255, 255, 255)

def cw(size): return 6 * (size + 1)
def cx(t, size): return max(0, (W - len(t) * cw(size)) // 2)

best = 0
s = system.readfile('/littlefs/config/mygame_best.txt')
if s:
    try: best = int(s.strip())
    except: pass

_rs = time.ticks_us()
def _rn():
    global _rs
    _rs = (_rs * 1103515245 + 12345) & 0x7FFFFFFF
    return _rs
def ri(a, b): return a + _rn() % (b - a + 1)

_bp = False
def btn_edge():
    global _bp
    cu = buttons.pressed(1)
    hit = cu and not _bp
    _bp = cu
    return hit

score = 0
alive = True
fr = 0

while True:
    g = touch.gesture()
    if g == 'swipe_left':
        system.exit()

    now = time.ticks_ms()
    fire = btn_edge() or g == 'press'

    if not alive:
        display.clear(BK)
        t = "GAME OVER"
        display.text(cx(t, 3), H // 2 - 80, t, 3, c(255, 50, 50))
        t = "Score: %d" % score
        display.text(cx(t, 2), H // 2 - 10, t, 2, WH)
        t = "Best: %d" % best
        display.text(cx(t, 2), H // 2 + 25, t, 2, c(255, 220, 0))
        t = "Tap to retry"
        display.text(cx(t, 1), H - 60, t, 1, c(80, 80, 80))
        display.flush()
        if touch.touching() or buttons.any():
            time.sleep_ms(200)
            score = 0; alive = True
        time.sleep_ms(33)
        continue

    ax, ay, _ = imu.accel()

    # --- Update ---
    # game logic here

    # --- Draw ---
    display.clear(BK)
    display.text(10, 8, "Score %d" % score, 1, WH)
    # draw game here
    display.flush()

    fr += 1
    if fr % 60 == 0:
        gc.collect()
    time.sleep_ms(33)
```

## Example: touch counter

Button 1 is the only physical key, so the counter is driven by taps and swipes.

```python
import display, touch, buttons, buzzer, time, system

W = display.WIDTH
H = display.HEIGHT

BLACK = display.color(0, 0, 0)
WHITE = display.color(255, 255, 255)
GREEN = display.color(0, 255, 0)
GRAY = display.color(120, 120, 120)

def cw(size): return 6 * (size + 1)
def cx(t, size): return max(0, (W - len(t) * cw(size)) // 2)

count = 0
prev = False

def draw():
    display.clear(BLACK)
    t = "Counter"
    display.text(cx(t, 2), 24, t, 2, WHITE)
    s = "%d" % count
    display.text(cx(s, 5), H // 2 - 40, s, 5, GREEN)
    t = "tap +1   swipe_down -1   swipe_up reset"
    display.text(cx(t, 0), H - 40, t, 0, GRAY)
    display.flush()

draw()

while True:
    g = touch.gesture()
    if g == 'swipe_left':
        system.exit()

    changed = False
    if g == 'press':
        count += 1; changed = True
    elif g == 'swipe_down':
        count -= 1; changed = True
    elif g == 'swipe_up':
        count = 0; changed = True

    cur = buttons.pressed(1)
    if cur and not prev:
        count += 1; changed = True
    prev = cur

    if changed:
        buzzer.click()
        draw()
    time.sleep_ms(30)
```

## Example: HTTP server app

```python
import display, touch, system, http, wifi, time, gc

W = display.WIDTH
H = display.HEIGHT
c = display.color

BG  = c(0, 0, 0)
FG  = c(240, 240, 240)
DIM = c(150, 150, 150)
GN  = c(120, 200, 130)
RD  = c(232, 75, 74)

PORT = 8080
hits = 0

def page_home(req):
    global hits
    hits += 1
    return (200, b"<h1>Hello from gotchi</h1>", "text/html")

def page_info(req):
    global hits
    hits += 1
    body = '{"device":"gotchi","hits":%d,"path":"%s"}' % (hits, req["path"])
    return (200, body.encode(), "application/json")

http.server.register("GET", "/", page_home)
http.server.register("GET", "/api/info", page_info)
http.server.start(PORT)

while True:
    g = touch.gesture()
    if g == 'swipe_left' or g == 'long_press':
        http.server.stop()
        system.exit()

    ip = wifi.ip()
    display.clear(BG)
    display.text(12, 12, "HTTP SERVER", 2, FG)
    if ip:
        display.text(12, 70, "http://%s:%d/" % (ip, PORT), 1, GN)
    else:
        display.text(12, 70, "No WiFi", 1, RD)
    display.text(12, 120, "GET  /            HTML", 1, DIM)
    display.text(12, 145, "GET  /api/info    JSON", 1, DIM)
    display.text(12, H - 50, "Hits: %d" % hits, 2, FG)
    display.flush()

    time.sleep_ms(100)
    gc.collect()
```
