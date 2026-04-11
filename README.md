# Home Assistant Config — Knob-Mat Media Controller

## What this is

Config files and instructions for the **"Media Controller: Knob-Mat"** HA automation. The `button_double` action cycles sequentially through a list of media stream URLs stored in a plain text file, looping back after the last one.

## Files

| File | Destination on HA | Purpose |
|---|---|---|
| `media_sources.txt` | `/config/media_sources.txt` | Line-separated stream URLs — edit this to add/swap stations |
| `knob_mat_sensor.yaml` | snippet for `configuration.yaml` | `command_line` sensor that reads the file into a HA attribute |
| `automation_knob_mat.yaml` | paste into automation editor | Full updated automation YAML |

---

## Setup steps

### 1. Copy `media_sources.txt` to `/config/media_sources.txt`
Edit lines 2 and 3 to replace the placeholder URLs with real streams when ready.

### 2. Add the sensor to `configuration.yaml`

Open `/config/configuration.yaml` on your HA instance and add the `command_line:` block from `knob_mat_sensor.yaml`.

- Use the **new-style** block (top-level `command_line:` key) for HA 2022.6+
- Use the **old-style** block (under `sensor:`) if you already have a `sensor:` list

**Requires a full HA restart** — `command_line` sensors don't support partial reload.

### 3. Create the `input_number` helper

Settings → Helpers → Create Helper → Number:

| Field | Value |
|---|---|
| Name | `Knob Mat Stream Index` |
| Min | `0` |
| Max | `999` |
| Step | `1` |
| Mode | Input field |

Entity ID will be `input_number.knob_mat_stream_index`. No restart needed.

### 4. Update the automation

Settings → Automations → "Media Controller: Knob-Mat" → three-dot menu → **Edit in YAML** → paste the full contents of `automation_knob_mat.yaml` → Save.

No restart needed (just an automation reload).

### 5. Restart HA

Required for the `command_line` sensor to load (step 2).

---

## Verification

After restart, in **Developer Tools → States**:
- `sensor.knob_mat_sources` → state = `3`, attributes include `sources: [...]`
- `input_number.knob_mat_stream_index` → state = `0`

Then in **Developer Tools → Events**, fire:
```
Event type:  esphome.media_controller_action
Event data:  {"action": "dedicated_double_click"}
```

Each fire should play the current stream and advance the index: `0 → 1 → 2 → 0`.

---

## Adding / changing streams

Just edit `/config/media_sources.txt` — one URL per line. The sensor re-reads the file every 30 seconds, no restart needed. Blank lines are ignored.

If you shorten the list and the saved index goes out of range, reset it manually:  
Settings → Helpers → Knob Mat Stream Index → set value to `0`.

---

## Architecture

```
/config/media_sources.txt
        ↓ (re-read every 30s)
sensor.knob_mat_sources
  .state          = number of URLs
  .attributes.sources = ["url1", "url2", ...]
        ↓
input_number.knob_mat_stream_index  (persisted across restarts)
        ↓
button_double sequence:
  1. read sources[idx]
  2. play it on media_player.office_mini
  3. advance idx = (idx + 1) % len(sources)
```
