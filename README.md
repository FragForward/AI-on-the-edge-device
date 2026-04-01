<h1 align="center">AI on the Edge Device - FragForward Fork</h1>

<p align="center">
  <img src="images/icon/watermeter.svg" width="150px">
</p>

<p align="center">
  <b>Custom firmware for <a href="https://github.com/jomjol/AI-on-the-edge-device">jomjol/AI-on-the-edge-device</a> with additional features and bug fixes.</b>
</p>

<p align="center">
  <a href="https://github.com/jomjol/AI-on-the-edge-device">Original Project</a> |
  <a href="https://jomjol.github.io/AI-on-the-edge-device-docs/">Documentation</a> |
  <a href="https://github.com/jomjol/AI-on-the-edge-device/releases/">Original Releases</a>
</p>

---

## What's different in this fork?

### 1. Individual Digit & Analog ROI values via MQTT and JSON

The original firmware only publishes the aggregated total value. This fork exposes **every individual ROI value**.

**JSON output** now includes `dig` and `ana` objects:
```json
{
  "value": "1391.5875",
  "raw": "01391.5875",
  "pre": "1391.5875",
  "error": "no error",
  "rate": "0.000000",
  "timestamp": "2026-03-29T22:17:23+0200",
  "dig": {
    "dig0": 0.0,
    "dig1": 1.0,
    "dig2": 3.0,
    "dig3": 9.0,
    "dig4": 1.9
  },
  "ana": {
    "ana1": 5.6,
    "ana2": 8.8,
    "ana3": 7.8,
    "ana4": 5.8
  }
}
```

**Separate MQTT topics** for each ROI:
```
watermeter/main/dig0  --> 0.0
watermeter/main/dig1  --> 1.0
watermeter/main/ana1  --> 5.6
watermeter/main/ana3  --> 7.8
...
```

**Home Assistant Discovery** automatically registers sensors for all individual ROIs.

---

### 2. Bug Fix: MaxRateValue ignores AnalogToDigitTransitionStart

**Known issue since 2022** (Issue [#712](https://github.com/jomjol/AI-on-the-edge-device/issues/712), [#2903](https://github.com/jomjol/AI-on-the-edge-device/issues/2903), [#659](https://github.com/jomjol/AI-on-the-edge-device/issues/659)) - never fixed upstream.

**Problem:** Legitimate single-digit rollovers (+1.0) are rejected as "Rate too high" even when the analog pointer confirms the transition. This causes the meter to freeze for extended periods.

**Example from real log data:**
```
raw=1392.595  ana3=9.5  pre=1391.593
--> REJECTED: "Rate too high" (diff=1.002 > MaxRateValue=0.3)
--> But ana3=9.5 >= AnalogToDigitTransitionStart=9.2, confirming a valid rollover!
```

**Fix:** When `MaxRateValue` is exceeded, the firmware now checks if:
- The difference is between 0.8 and 1.2 (single-digit rollover)
- The analog pointer value >= `AnalogToDigitTransitionStart`

If both conditions are met, the value is accepted instead of rejected.

---

### 3. Meter Image as Home Assistant Entity

The meter image (`alg_roi.jpg`) is published as an **HA MQTT image entity** after each digitization round. Automatically visible as `image.watermeter_meter_image` in Home Assistant - no manual configuration needed.

---

### 4. Set PreValue from Home Assistant

A **number entity** (`number.watermeter_set_prevalue`) is registered via MQTT Discovery. This allows setting the PreValue directly from the HA UI instead of using the web interface.

Uses the existing `ctrl/set_prevalue` MQTT handler on the ESP.

---

### 5. Additional Bug Fixes

- **Fixed:** "Neg. Rate" error message was missing the actual read value (used undefined variable `zwvalue` - always showed empty string)
- **Fixed:** `checkDigitConsistency()` could crash with `log10(0)` when input value was zero or negative

---

## Installation

This fork is a **drop-in replacement** for the original firmware. Flash it the same way:

1. Download `firmware.bin` from the [Releases](https://github.com/FragForward/AI-on-the-edge-device/releases) page
2. Upload via the OTA update page on your ESP (`http://<IP>/ota`)
3. No configuration changes needed - all existing settings are preserved

For first-time setup, see the [original documentation](https://jomjol.github.io/AI-on-the-edge-device-docs/).

---

## Changed Files

Only 4 files were modified (186 lines added, 13 removed):

| File | Changes |
|------|---------|
| `code/components/jomjol_flowcontroll/ClassFlowPostProcessing.cpp` | JSON extension, rollover fix, bug fixes |
| `code/components/jomjol_flowcontroll/ClassFlowMQTT.cpp` | Individual ROI MQTT topics, image URL publishing |
| `code/components/jomjol_mqtt/server_mqtt.cpp` | HA Discovery for ROIs, image entity, prevalue number entity |
| `README.md` | This file |

---

## Credits

This fork is based on the excellent work of [jomjol](https://github.com/jomjol) and all [contributors](https://github.com/jomjol/AI-on-the-edge-device#our-contributors-%EF%B8%8F) of the original project.
