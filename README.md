# DPS Control – Web Serial controller for RD Tech (Riden) DPS & DPH power supplies

`dps-control.html` is a browser app for controlling a DPS-series programmable power module from a PC over its serial port: live readout, chart, logging, ramp/I-V sweep, sequences, auto cutoff, memory groups M0–M9, raw Modbus console and a simulator.

It needs only desktop **Chrome or Edge** (Web Serial) and talks **Modbus RTU** directly to the supply. Nothing to install, no vendor software.

| File | Purpose |
|---|---|
| `dps-control.html` | The app |
| `devices.json` | Supported devices: model register, limits, scaling, default baud rate and address. Edit it to add or tune devices. The HTML also contains a built-in copy, so it works on its own. |

---

## Compatible devices

The supply must be the **communication version** (sold with a USB or Bluetooth module, often labelled "USB/BT" or "CNC communication version"). Plain DPS units without the serial header cannot be controlled.

| Model | Type | Range | MODEL reg (0x0B) | I-SET / S-OCP | IOUT | Status |
|---|---|---|---|---|---|---|
| **DPS5015** | Buck | 0–50 V, 0–15 A, 750 W | 5015 | ×100 (0.01 A) | ×100 | ✅ auto-detected |
| **DPS5020** | Buck | 0–50 V, 0–20 A, 1000 W | 5020 | ×100 | ×100 | ✅ auto-detected |
| **DPS3005** | Buck | 0–30 V, 0–5 A, 160 W | 3005 | ×1000 (0.001 A) | ×100 | ✅ auto-detected |
| **DPS5005** | Buck | 0–50 V, 0–5 A, 250 W | 5005 | ×1000 | ×100 | ✅ auto-detected |
| **DPS8005** | Buck | 0–80 V, 0–5.1 A, 408 W | 8005 | ×1000 | ×100 | ✅ auto-detected |
| **DPH5005** | Buck/Boost | 0–50 V, 0–5 A, 250 W | **5205** | ×1000 | ×100 | ✅ auto-detected |
| DPH3205 | Buck/Boost | 0–32 V, 0–5 A, 160 W | 3205 (expected) | ×1000 (expected) | ×100 | ⚠️ in model list, not confirmed by any source |

The list above is what ships in `devices.json`. In **⚙ Settings → Device** you can leave **Auto-detect** (the device is identified from register 0x0B) or pick a device, which presets its baud rate and address and applies its limits. For a model number that is not in the list, the app guesses limits from the number (e.g. 5015 → 50 V / 15 A) and marks it "unverified device".

> **Tested status:** the logic was tested against the built-in simulator, not on physical hardware. Register addresses and scaling come from the sources listed at the end. If a reading is off by ×10, choose the model manually and report it.

### Not compatible

| Device | Why |
|---|---|
| Riden **RD6006 / RD6012 / RD6018 / RD6024 / RD6030** and Joy-IT **JT-RD60xx / PS1440** | Also Modbus, but a completely different register map |
| DPS units running **OpenDPS** firmware | OpenDPS uses its own serial protocol |
| DPS units **without** the communication board, DP-series (e.g. DP30V5A, DP50V5A), DPS3012 | No serial interface |
| DPS3003 | No communication version found in any source |
| DPM8600, XY-series (XY6020L, XY-SK…) | Different protocols / register maps |

---

## Wiring

| DPS serial header | USB-UART adapter (3.3 V TTL) |
|---|---|
| TX | RX |
| RX | TX |
| GND | GND |
| VCC | **not connected** |

- The header on the DPS is a small 4-pin 1.25 mm (GH / Molex Pico-style) connector.
- Use a **3.3 V** logic adapter (CP2102, CH340, FT232 set to 3.3 V). Never feed the VCC pin of the DPS.
- The official RD USB or Bluetooth module plugs straight in and shows up as a COM port.

## Serial settings

| Setting | Value |
|---|---|
| Protocol | Modbus RTU, function codes 0x03 (read), 0x06 (write one), 0x10 (write many) |
| Frame | 8 data bits, no parity, 1 stop bit |
| Baud | **9600** default. Change it by holding **▲ (V)** while powering the unit on; the same menu sets the slave address and Bluetooth PIN |
| Slave address | 1 (factory setting in practice; the protocol document does not state it) |
| Max registers per request | 32 |
| Inter-frame gap | ≥ 3.5 character times (≈ 4 ms at 9600) |

---

## devices.json

```json
{
  "schema": 1,
  "defaults": { "baud": 9600, "address": 1, "bauds": [2400, 4800, 9600, 19200], "pollMs": 500, "timeoutMs": 300 },
  "devices": [
    {
      "id": "DPS5015", "model": 5015, "type": "Buck", "verified": true,
      "vMax": 50, "iMax": 15, "pMax": 750,
      "scale": { "v": 100, "iSet": 100, "iOut": 100, "opp": 10 },
      "baud": 9600, "address": 1,
      "appLimits": { "ovp": 52, "ocp": 15.5, "opp": 760 }
    }
  ]
}
```

| Field | Meaning |
|---|---|
| `id` | Name shown in the dropdown (must be unique) |
| `model` | Value of register 0x0B, used for auto-detection |
| `vMax`, `iMax`, `pMax` | Setpoint limits enforced by the app |
| `scale` | Register multipliers: value = register ÷ scale. `iSet` covers I-SET and S-OCP, `iOut` covers IOUT, `opp` covers S-OPP |
| `baud`, `address` | Preset into Settings when the device is picked (optional, falls back to `defaults`) |
| `appLimits` | Highest OVP / OCP / OPP the app lets you write. These are app-side guards, not device specifications |
| `verified`, `notes` | Shown in the UI; `false` marks the device as unverified |

Only `id`, `model`, `vMax` and `iMax` are required; everything else has defaults.

**Where the list comes from:** the app first tries `devices.json` next to the HTML. That works when the page is served over http (e.g. `python -m http.server`, then open http://localhost:8000/dps-control.html). Chrome does not let a page opened by double-click (file://) read neighbouring files, so in that case click **Load devices.json…** in Settings once; the list is remembered until you press **Reset**. If neither is available, the built-in copy is used. Settings shows which source is active.

---

## Quick start

1. Open `dps-control.html` in desktop Chrome or Edge (double-click works, file:// is fine).
2. In **⚙ Settings** pick your **Device** (or leave Auto-detect), check **Baud** and **Addr** match the supply, click **Connect** and pick the COM port. Next time the same USB adapter reconnects without the picker.
3. The model is read from register 0x0B. With Auto-detect the matching device from the list is used; if you picked a device and the supply reports a different model, you get a warning.
4. **Esc** turns the output off immediately and stops every running tool.
5. The setpoint fields follow the supply automatically (front panel, presets, tools). An orange field holds a value not yet sent; **⟳ Sync from supply** reloads them.
6. The browser tab title shows live V / A, readouts grey out as *stale* if the supply stops answering, and the screen is kept awake while a tool, cutoff or recording is running.

No hardware at hand? Click **Simulator**. It emulates the device chosen in Settings (Auto-detect = DPS5015) with an adjustable resistive load, realistic output settling and ~1 % dropped frames.

---

## Register map (verified)

✅ = the protocol document and rdserialtool agree · ⚠️ = sources differ, see notes

| Reg | Name | R/W | Meaning / scaling | Check |
|---|---|---|---|---|
| 0x00 | U-SET | R/W | Voltage setpoint, V ×100 | ✅ |
| 0x01 | I-SET | R/W | Current setpoint, A ×100 (15/20 A models) or ×1000 (5 A models) | ✅ |
| 0x02 | UOUT | R | Output voltage, V ×100 | ✅ |
| 0x03 | IOUT | R | Output current, A ×100 | ⚠️ note 1 |
| 0x04 | POWER | R | Output power; ×100, or ×10 above 99.99 W | ⚠️ note 2 |
| 0x05 | UIN | R | Input voltage, V ×100 | ✅ |
| 0x06 | LOCK | R/W | Key lock: 0 off, 1 on | ✅ |
| 0x07 | PROTECT | R | 0 OK, 1 OVP, 2 OCP, 3 OPP | ✅ |
| 0x08 | CV/CC | R | 0 CV, 1 CC | ✅ |
| 0x09 | ONOFF | R/W | Output: 0 off, 1 on | ✅ |
| 0x0A | B-LED | R/W | Backlight 0 (dark) – 5 (bright) | ✅ |
| 0x0B | MODEL | R | Model number, e.g. 5015 (DPH5005 → 5205) | ✅ |
| 0x0C | VERSION | R | Firmware, e.g. 17 = 1.7 | ✅ |
| 0x23 | EXTRACT_M | R/W | Write 0–9 to load data group M0–M9 | ✅ |

### Data groups M0–M9

Base address = **0x50 + 0x10 × n** (M0 = 0x50, M1 = 0x60 … M9 = 0xE0). M0 is the active / power-on group, M1 and M2 are the quick-recall groups, M3–M9 are storage.

| Offset | Name | Scaling | Check |
|---|---|---|---|
| +0 | U-SET | V ×100 | ✅ |
| +1 | I-SET | same as 0x01 | ✅ |
| +2 | S-OVP | V ×100 | ✅ |
| +3 | S-OCP | same as 0x01 | ✅ |
| +4 | S-OPP | W ×10 | ✅ |
| +5 | B-LED | 0–5 | ✅ |
| +6 | M-PRE | flag | ⚠️ note 3 |
| +7 | S-INI | output on at power-up, 0/1 | ✅ |

### Notes on the ⚠️ items

1. **IOUT on 5 A models.** The protocol document gives IOUT 2 decimals while I-SET has 3, and rdserialtool (written on a DPS5005) also divides IOUT by 100. The app uses ×100 for every model. On a DPS5015/5020 there is no ambiguity (both ×100). If your 5 A unit shows current 10× too high, report it.
2. **POWER (0x04).** The decimal point moves with the range, so the app ignores this register and computes P = U × I instead.
3. **Group offset +6.** The protocol document calls it M-PRE, rdserialtool calls it "maintain output". Its exact effect is unclear, so the app reads it and writes the same value back unchanged.

Other corrections from the document: CV/CC lists "CV" for both values (1 is CC); S-INI is listed with 2 decimals but is a 0/1 flag.

---

## Safety

- Set **OVP / OCP / OPP** before connecting a load. They live in group M0 and are enforced by the supply itself, independently of the PC.
- The auto-cutoff and sequences run in the browser tab. If the PC sleeps, the tab is closed or the USB cable is pulled, the supply keeps its last setting. Rely on the hardware protection limits, not on the software.
- Keep the setpoint ≤ input voltage − ~1.5 V on buck (DPS) models.

---

## Sources

- DPS3005 CNC Communication Protocol V1.2 (RD Tech): https://wpsodp.ee.ntust.edu.tw/chapter_02/DPS3005%20CNC%20Communication%20%20Protocol%20V1.2.pdf
- DPS5020 communication protocol, English translation V1.2: https://redmine.laas.fr/attachments/download/4753/DPS5020communication%20protocol%20-english%20translation%20V1.2.pdf
- Riden RD60xx Modbus protocol (to show it is a different map): https://www.joy-it.net/files/files/Produkte/JT-PS1440-C-Pro/MODBUS_Protocol_RD60xx_PSxx.pdf
