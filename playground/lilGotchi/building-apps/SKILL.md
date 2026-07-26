---
name: gotchi
description: "Write MicroPython apps for a gotchiOS device. Use when user mentions gotchi, creating, or building an app."
metadata:
  openclaw:
    emoji: "🐾"
---

# gotchiOS — Building Apps (OpenGotchi)

Write MicroPython apps for a gotchiOS (ESP32-S3, 280x240 display, 4 buttons, touch, IMU, buzzer).

## Behavior rules

- Focus purely on writing correct app code. The playground handles running and installing the app itself — never ask the user for a device hash, secret, or any connection detail.
- Always include `app_logo_raw` — 1bpp 24x24 pixel icon as base64 (72 bytes raw). Generate one if the user doesn't provide it.
- File naming: `breakout.py` → "breakout", `space_invaders.py` → "space invaders". Prefix `z_` to sort to end.

## Writing apps — CRITICAL RULES

1. **Exit gesture REQUIRED in every app main loop:**
   ```python
   g = touch.gesture()
   if g == 'swipe_right':
       system.exit()
   ```

2. **Sleep REQUIRED in every loop:** `time.sleep_ms(33)` minimum. No sleep = watchdog reboot.

3. **Call `gc.collect()` periodically** in long-running apps.

4. **DO NOT USE:** `str()`, f-strings, `print()`, `open()`, `import random`, `import math`, `import json`, any standard library. They don't exist.

5. **Use `%` formatting:** `"%d" % n` not `str(n)`. `"%s %d" % (a, b)` not f-strings.

6. **Display:** 280x240, call `display.flush()` once per frame, `display.clear(color)` before redraw. `display.text(x, y, string, size, color)` — x,y first!

7. **Colors:** Always `display.color(r, g, b)`, never hardcode RGB565.

8. **Keep files under 15KB.** Larger works but loads from slower PSRAM.

## Quick API

```python
import display, touch, time, system, gc, buttons, buzzer, imu, battery, rtc

# Display
display.color(r,g,b)  display.clear(c)  display.text(x,y,s,sz,c)
display.rect_filled(x,y,w,h,c)  display.circle_filled(cx,cy,r,c)
display.line(x0,y0,x1,y1,c)  display.flush()

# Input
touch.gesture()  touch.pos()  touch.touching()
buttons.pressed(1-4)  buttons.any()

# Sensors
imu.accel()->(ax,ay,az)  imu.tilt()->(tx,ty)  imu.temperature()

# Audio
buzzer.tone(hz,ms)  buzzer.beep()  buzzer.click()

# System
system.exit()  system.launch(path)  system.readfile(p)  system.writefile(p,d)
rtc.now()->(y,m,d,h,min,s)  battery.percent()  time.ticks_ms()
```

## Full API Reference

### display (280x240 RGB565 landscape)

```python
import display
display.WIDTH                              # 280
display.HEIGHT                             # 240
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
display.flush()                            # push to screen
display.backlight(0-255)
```

Font: built-in 5x7 monospace. Size 0=6px wide, 1=12px, 2=18px, 3=24px per char.

### imu (QMI8658C)

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

Edge detection pattern:
```python
prev = [False] * 4
while True:
    for i in range(1, 5):
        cur = buttons.pressed(i)
        if cur and not prev[i-1]:
            pass  # button i just pressed
        prev[i-1] = cur
    time.sleep_ms(30)
```

### touch (CST816T)

```python
import touch
touch.touching()    # bool
touch.pos()         # (x, y) or None
touch.gesture()     # one-shot string, clears after read
```

Gestures: "none", "press", "release", "swipe_up", "swipe_down", "swipe_left", "swipe_right", "long_press"

### buzzer

```python
import buzzer
buzzer.tone(freq_hz, duration_ms)   # non-blocking
buzzer.beep()
buzzer.click()
buzzer.stop()
```

### battery

```python
import battery
battery.voltage()    # float volts
battery.percent()    # int 0-100
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
system.readfile("/littlefs/data/x.txt")     # read file (max 4KB) → str or None
system.writefile("/littlefs/data/x.txt", "data")  # write file → bool
```

### wifi

```python
import wifi
wifi.status()          # "connected" | "connecting" | "disconnected" | "ap_portal" | "off"
wifi.ip()              # "192.168.1.42" or None
wifi.send("msg")       # send to WebSocket clients → bool
wifi.recv()            # receive WebSocket message → str or None
wifi.notify_count()    # unread notification count
wifi.notifications()   # list of {"title":..., "body":..., "time":...}
wifi.clear_creds()     # erase WiFi credentials, reboot into AP mode
```

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
buzzer.tone(150, 200)    # death
buzzer.stop()            # stop any playing tone
```

## Screen layouts

Full-screen game (Space Invaders, Flappy):
```
+---------------- 280 ------------------+
| Score        Level        Lives  | 16px HUD
+---------------------------------------+
|                                       |
|           GAME AREA                   | 224px
|           280 x 224                   |
|                                       |
+---------------------------------------+
```

Grid game (2048, Tetris):
```
+---------------- 280 ------------------+
|Score|   GRID AREA centered    | HI  |
|     |   (compute from H-HUD) |     |
|     |                         |     |
+---------------------------------------+
```

## Common mistakes

| Mistake | Fix |
|---------|-----|
| `import math` / `random` / `json` | Use workarounds above |
| Using `str()`, f-strings, `print()` | Use `"%d" % n` formatting |
| Buttons 0-indexed | Buttons are **1 to 4**, not 0-3 |
| Forgot `display.flush()` | Nothing shows without it |
| Forgot `gc.collect()` | Memory exhaustion after ~60 frames |
| No `time.sleep_ms()` in loop | Watchdog reboot — minimum 33ms |
| Exit on `swipe_left` | Exit gesture is `swipe_right` |
| IMU X = up/down | **X = left/right**, Y = forward/back |

## Template

```python
import display, touch, time, system, gc

BLACK = display.color(0, 0, 0)
WHITE = display.color(255, 255, 255)

while True:
    g = touch.gesture()
    if g == 'swipe_right':
        system.exit()
    display.clear(BLACK)
    display.text(10, 100, "Hello", 2, WHITE)
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
    if g == 'swipe_right':
        system.exit()

    now = time.ticks_ms()
    b1 = btn_edge(1)
    b2 = btn_edge(2)
    b3 = btn_edge(3)
    b4 = btn_edge(4)

    if not alive:
        display.clear(BK)
        display.text(W//2 - 50, 60, "GAME OVER", 2, c(255, 50, 50))
        display.text(W//2 - 40, 100, "Score: %d" % score, 1, WH)
        display.text(W//2 - 35, 125, "Best: %d" % best, 1, c(255, 220, 0))
        display.text(W//2 - 45, 175, "Tap to retry", 1, c(80, 80, 80))
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
    # draw game here
    display.flush()

    fr += 1
    if fr % 60 == 0:
        gc.collect()
    time.sleep_ms(33)
```

## Example: button counter

```python
import display, buttons, touch, time, system

BLACK = display.color(0, 0, 0)
WHITE = display.color(255, 255, 255)
GREEN = display.color(0, 255, 0)
GRAY = display.color(120, 120, 120)

count = 0
prev = [False, False, False, False]

def draw():
    display.clear(BLACK)
    display.text(90, 20, "Counter", 2, WHITE)
    s = "%d" % count
    x = 140 - len(s) * 12
    display.text(x, 100, s, 4, GREEN)
    display.text(20, 220, "B1:+1  B2:-1  B3:reset", 1, GRAY)
    display.flush()

draw()

while True:
    g = touch.gesture()
    if g == 'swipe_right':
        system.exit()
    for i in range(1, 4):
        cur = buttons.pressed(i)
        if cur and not prev[i - 1]:
            if i == 1:
                count += 1
            elif i == 2:
                count -= 1
            elif i == 3:
                count = 0
            draw()
        prev[i - 1] = cur
    time.sleep_ms(30)
```
