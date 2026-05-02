# CAT PAT

A physical-digital interactive installation combining an Arduino Nano 33 IoT, two e-ink displays, a light sensor, and a browser-based game — all centered around one cat named Rodeo.

---

## The Story

Rodeo was rescued a year ago by the Cat Allies Project. The vet estimated he was around one years old at the time, meaning he spent his first year entirely alone on the streets of New York City.
He loves his comfortable home now, but the street never fully leaves him. Sometimes tiny, rat-sized objects move, and he slips back into that feral place.
When that happens, all it takes is a small rub on his head. Then his pupils will go from round to a thin almond.

---

## What It Does

The installation has two parts that run simultaneously:

**Physical hardware** — A cat head enclosure houses two e-ink displays acting as Rodeo's eyes, and an LDR (light sensor) hidden in the snout. When a player covers the sensor with their hand, the Arduino detects the change in light and commands both eyes to refresh — Rodeo's pupils contract from wide circles to thin slits, the way a real cat's pupils narrow when calm.

**Browser game** — A split-screen game running in Chrome. The left panel shows the human's perspective (petting the cat). The right panel shows the mouse's perspective — hiding under a car, eating cheese, while cats lurk in the shadows. Covering the sensor pets the cat in both the physical and digital world simultaneously.

---

## Game Objective

You play as both a human and a mouse at once.

- The mouse is always eating cheese under a car
- Street cats appear in three stages — far, mid, close — getting more dangerous each step
- Cover the light sensor to pet the cat and scare it off
- The closer the cat, the faster the mouse eats when you pet
- Pet too long and the cat bites you
- Don't pet in time and the cat charges the mouse

Eat all the cheese before getting bitten or caught.

---

## Hardware

| Component | Quantity | Notes |
|---|---|---|
| Arduino Nano 33 IoT | 1 | Main microcontroller |
| Waveshare 1.54" e-ink display (GxEPD2_154_D67) | 2 | 200×200px, black and white |
| LDR photoresistor module | 1 | Analog input on A0 |
| USB cable | 1 | Powers Arduino, carries serial to browser |

### Wiring

| Signal | Arduino Pin | Notes |
|---|---|---|
| LDR VCC | 3.3V | |
| LDR GND | GND | |
| LDR AO | A0 | Analog read |
| Both displays VCC | 3.3V | Shared |
| Both displays GND | GND | Shared |
| Both displays DIN | D11 (MOSI) | Shared SPI data |
| Both displays CLK | D13 (SCK) | Shared SPI clock |
| Both displays DC | D9 | Shared data/command |
| Both displays RST | D8 | Shared reset |
| Display 1 CS | D10 | Must be set HIGH before init |
| Display 1 BUSY | D7 | |
| Display 2 CS | D6 | |
| Display 2 BUSY | D5 | |

### Key wiring note

Only `display2` is declared in code. Display 1 receives identical SPI data via the shared pins. **Both CS pins must be set HIGH before `display2.init()` is called** — if D10 floats during init, display 1's state corrupts and it will not refresh.

```cpp
pinMode(10, OUTPUT); digitalWrite(10, HIGH);  // display1 CS — must come first
pinMode(6,  OUTPUT); digitalWrite(6,  HIGH);  // display2 CS
display2.init(115200);                         // init after both CS are stable
```

---

## Software

### Arduino

**Library required:** GxEPD2 by ZinggJM (install via Arduino Library Manager)

The sketch:
- Reads LDR on A0 every 500ms (16-sample average to filter noise)
- Sends `LDR:xxx` to browser via serial every 500ms
- ADC ≤ 200 = bright (hand not covering) → pupil contracts
- ADC > 200 = dark (hand covering) → pupil expands
- Enforces 4000ms minimum gap between e-ink refreshes
- Listens for `LIGHT:1`, `LIGHT:0`, `RESET` commands from browser

### Browser Game

**Requirements:** Chrome or Edge (Web Serial API required — Firefox not supported)

All assets are embedded in a single HTML file — no server, no install, no dependencies.

**To run:**
1. Upload `cat_eye_with_ldr.ino` to the Arduino
2. Open `cat_pat_game.html` in Chrome
3. Click **▶ START GAME**
4. When prompted, select the Arduino's serial port
5. The serial connection persists across game restarts — you will only be asked once per browser session

### Serial Protocol

| Direction | Message | Meaning |
|---|---|---|
| Arduino → Browser | `LDR:512` | Raw ADC value, sent every 500ms |
| Browser → Arduino | `LIGHT:1` | Light detected |
| Browser → Arduino | `LIGHT:0` | Light not detected |
| Browser → Arduino | `RESET` | Game ended |

---

## File Structure

```
cat-pat/
├── cat_eye_with_ldr.ino     # Arduino sketch with e-ink bitmaps
├── cat_pat_game.html        # Browser game (fully self-contained)
└── README.md
```

---

## Audio

The game uses Web Audio API with all audio embedded as base64. No external files needed.

| Sound | Trigger |
|---|---|
| Background loop | Plays throughout game |
| Cat approach sfx (×2) | Random, plays when cat appears at stage 1 |
| Intense sfx | Loops during cat stage 2 and 3 |
| Heavy breathing | Loops during cat stage 3 (closest) |
| Cute music | Plays only while petting — all horror sounds pause |
| Cat purring | Loops while sensor is covered |
| Mouse eating sfx | Always playing, volume rises with eating speed |
| Cat annoyed meow | On game over / bite |
| Button press | Every UI button click |
| Game completion | On win — all other sounds pause |

---

## Energy

The e-ink displays consume power **only during a refresh** (roughly 3 seconds), then drop to effectively zero current draw while holding their image. Between refreshes — which are rate-limited to one every 4 seconds minimum — both displays draw less than 0.01 mA combined.

| Component | Average draw |
|---|---|
| Arduino Nano 33 IoT | ~100 mW |
| 2× e-ink displays (averaged) | ~15 mW |
| LDR module | ~2 mW |
| **Total (hardware only)** | **~117 mW** |

The LDR sensor is entirely passive — it requires no drive voltage and generates its signal from ambient room light. The player's hand is the switch.

---

## Credits

- Rodeo — the cat
- Cat Allies Project — for rescuing him
- GxEPD2 library — ZinggJM
- All game assets — generated by gemini + online sources
