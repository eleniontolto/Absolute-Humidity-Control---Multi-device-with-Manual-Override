# Smart Humidity Climate Control (Multi-Device with Manual Override)

A battle-tested Home Assistant blueprint for humidity-based device automation using **Magnus-Tetens thermodynamic normalization** — the same physics as absolute humidity calculations, expressed as an intuitive percentage offset rather than grams per cubic meter.

[![Import Blueprint](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https://raw.githubusercontent.com/eleniontolto/Absolute-Humidity-Control---Multi-device-with-Manual-Override/refs/heads/main/blueprints/automation/eleniontolto/multidevice_humidity_control.yaml)

---

## What It Does

Humidity control for Home Assistant that actually behaves the way you need. Most blueprints handle the basics — turn on a single device when humidity rises, turn off when it drops — but fall apart the moment a human gets involved or something unexpected happens. This one is built to handle the real world. At its core it turns devices on when humidity rises and off when it drops — but it goes further:

- Uses temperature-corrected humidity math (Magnus-Tetens) to eliminate false triggers from room-to-room temperature differences — configured in familiar percentage (%) rather than absolute water mass units
- Detects manual overrides and respects them, no tug-of-war
- Protects equipment with a configurable safety watchdog and dedicated rest period
- Controls multiple devices as a unified group with shared lockout logic
- Gracefully handles sensor failures with configurable fallback modes

---

## Features

- **Thermodynamic Normalization** — Magnus-Tetens formula converts reference humidity to an equivalent value at the primary room's temperature, eliminating false triggers from room-to-room temperature differences
- **Multi-Device Support** — Control fans, switches, lights, input booleans, or helper groups; all selected devices act as a unified team with shared lockout logic
- **Manual Override Detection** — Detects who changed a device via HA's context system; any manual touch on any device pauses the entire group for the full override cooldown — no automation tug-of-war
- **Two Independent Timers** — Safety Rest Period and Manual Override Cooldown are fully separate; tune each for your specific equipment and workflow needs
- **Safety Max Run Watchdog** — Force-stops devices after a configurable maximum run time to protect equipment from indefinite operation caused by sensor failure or open windows
- **Two-Track Max Run Timer** — Automation-activated devices and manually-activated devices use separate max run calculations, so the watchdog always fires at the right time regardless of how devices were turned on
- **Smart Safety Rest Loop** — During a safety rest, if you manually turn a device on it stays on for the manual override duration then turns back off; the rest period extends if needed to ensure neither timer cuts the other short
- **Direction-Aware Hysteresis** — Separate rising and falling thresholds prevent rapid cycling; devices turn on at one level and stay on until humidity drops to a lower level
- **Per-Device Recovery** — After any lockout expires, each device is individually checked against the target state, so a manually-toggled device in a group is correctly reclaimed even if others were unaffected
- **Dynamic Fail-Safe** — Falls back to a configurable Static Baseline slider or Raw RH% comparison if reference sensors go offline
- **Automatic Unit Conversion** — Handles mixed °C/°F sensor hardware transparently
- **Data Integrity** — Filters unknown/unavailable sensor states to prevent erratic behavior

---

## Requirements

- Home Assistant 2023.4 or later recommended
- A primary humidity sensor (required)
- Primary temperature sensor, reference humidity sensor, and reference temperature sensor (all optional, but all four are needed for full thermodynamic normalization)

---

## Installation

### One-Click Import
Click the import badge at the top of this page, or paste the raw file URL into **Settings → Automations → Blueprints → Import Blueprint**.

### Manual
1. Download `multidevice_humidity_control.yaml`
2. Place it in your Home Assistant `config/blueprints/automation/` directory
3. Restart Home Assistant or reload blueprints
4. Go to **Settings → Automations → Blueprints** and find *Smart Humidity Climate Control (Multi-Device with Manual Override)*

---

## Configuration

| Parameter | Description | Default |
|---|---|---|
| Primary Humidity Sensor | Humidity sensor in the controlled room | Required |
| Primary Temperature Sensor | Co-located temperature sensor for physics math | Optional |
| Reference Humidity Sensor | Humidity sensor in a dry reference area | Optional |
| Reference Temperature Sensor | Co-located with reference humidity sensor | Optional |
| Static Baseline Humidity | Fallback baseline when reference sensors unavailable | 50% |
| Sensor Failure Behavior | Static Baseline or Raw RH% when physics math can't run | Static |
| Control Device(s) | One or more devices to control | Required |
| Rising Threshold | How far above baseline before devices turn ON | 8% |
| Falling Threshold | How close to baseline before devices turn OFF | 3% |
| Safety Max Run Time | Maximum run time before force-stop | 30 min |
| Safety Rest Period | Pause after Safety Max Run before resuming control | 5 min |
| Manual Override Cooldown | Pause after any manual device touch | 30 min |

### Sensor Modes

**Full Thermodynamic Mode** (all four sensors configured): Reference humidity is normalized to the primary room's temperature using the Magnus-Tetens equation before comparison. Recommended for homes where the primary and reference sensors are at different temperatures.

**Raw RH% Mode** (primary and reference humidity sensors required): Compares raw relative humidity values directly. Only reliable when both sensor locations stay at near-identical temperatures.

**Static Baseline Mode** (no reference sensors required): Compares primary humidity against a fixed slider value. Simple and predictable; good for bathrooms or laundry rooms where you just want to trigger above a set level.

---

## How the Timers Work

### Safety Max Run
Devices will be force-stopped after this many minutes of continuous operation **when turned on by the automation**. If you manually turn devices on, the watchdog is suspended during the override window. After the override expires, the max run clock starts fresh from zero — giving the watchdog a clean slate.

**Note:** Due to HA's minute-tick polling, effective run time is approximately `max_run + 1` minutes per cycle. Set your value with this in mind.

### Safety Rest Period
After a Safety Max Run force-stop, the automation pauses for this many minutes before resuming normal control. Gives equipment time to cool down or rest. **This only activates after a max run shutoff** — not after humidity-triggered shutoffs or any other scenario.

If you manually turn a device on during a safety rest, it will stay on for the Manual Override Cooldown duration then turn back off. If that override window would outlast the rest period, the rest end time is extended to match, ensuring neither timer cuts the other short.

### Manual Override Cooldown
After any manual device touch, the automation pauses for this many minutes before reclaiming control. During this window no automatic ON or OFF actions will occur and the Safety Max Run watchdog is suspended.

---

## Known Limitations

- **Effective run time:** Due to minute-tick timing, devices run for approximately `max_run + 1` minutes per cycle rather than exactly `max_run`.
- **Sensor response lag:** Temperature and reference humidity sensor changes may take up to 60 seconds to affect behavior. Primary humidity sensor changes are always instant.
- **Sensor accuracy:** Consumer humidity sensors carry ±2–3% tolerance. Two sensors measuring the same air can read 4–6% apart. Avoid thresholds tighter than your sensors' combined accuracy.
- **Manual override detection:** Relies on HA's context system. Some voice assistants and third-party integrations may not correctly identify manual changes, bypassing the lockout.
- **HA restart during Safety Rest:** The rest timer is lost on restart. Devices may re-activate before the full rest period has elapsed.
- **Device flicker:** Devices may briefly turn off then immediately back on during transitions out of manual override periods. This is intentional — the automation clears stale user contexts before reclaiming devices to ensure timer logic stays accurate.
- **Helper Groups:** Manual override is only detected when the group's master toggle changes state. This means switching individual entities within the group may be missed, depending on the "All Entities" setting used when creating the Helper Group.

---

## Helper Groups vs. Direct Device Selection

You can select devices directly in the blueprint, or create a Helper Group first and select the group entity.

**Direct selection:** Any manual touch on any device triggers the group-wide lockout. Simple but sensitive — an accidental or unrelated change to any device in the list will pause the entire automation.

**Helper Group:** Lockout only triggers when the group's master toggle changes state. Use this if you want more control over what counts as a "manual override."

If using a Helper Group, the **"All Entities"** setting matters: with it **OFF** (default), the group turns on when any member turns on but only turns off when all members are off. With it **ON**, the group turns on only when all members are on but turns off when any single member turns off. The latter can cause partial manual-ON states to go undetected, so **"All Entities" OFF is recommended** with this blueprint.

---

## Credits

Based on [Switch a fan based on absolute humidity difference between two humid/temp sensors](https://community.home-assistant.io/t/switch-a-fan-based-on-absolute-humidity-difference-between-two-humid-temp-sensors/161999) by W6Es3QEa. Substantially rewritten with multi-device support, manual override intelligence, safety watchdog, independent timer architecture, per-device drift detection, and modern Home Assistant standards.

---

## Feedback & Contributions

Bug reports and suggestions welcome via [GitHub Issues](../../issues). If this blueprint is useful to you, consider sharing it on the [Home Assistant Community Blueprint Exchange](https://community.home-assistant.io/c/blueprints-exchange/53).
