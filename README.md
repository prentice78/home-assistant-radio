# Home Assistant Config — Knob-Mat Media Controller

## What this is

Config files and instructions for the **"Media Controller: Knob-Mat"** HA automation. A physical D1 Mini with a rotary encoder and dedicated push button controls a Google Home Mini via Home Assistant and Music Assistant.

All media playback is routed through **Music Assistant** (`media_player.office_mini_2`), which handles both direct stream URLs and YouTube Music search strings. The `button_double` action cycles sequentially through a list of media sources stored in a plain text file, looping back after the last one.

## Files

| File | Destination | Purpose |
|---|---|---|
| `media_sources.txt` | `/config/media_sources.txt` | Line-separated sources — stream URLs or YT Music search strings |
| `knob_mat_sensor.yaml` | snippet for `configuration.yaml` | `command_line` sensor that reads the file into a HA attribute |
| `automation_knob_mat.yaml` | paste into automation editor | Full automation YAML |
| `knob.yaml` | ESPHome config for D1 Mini | ESPHome firmware config for the knob-mat device |

---

## Controls

### Dial (rotary encoder)
| Gesture | Action |
|---|---|
| Rotate | Volume up/down |
| Single click | Next track |
| Double click | Previous track |
| Long press | Set volume to 40% |

### Dedicated button
| Gesture | Action |
|---|---|
| Single click | Play/pause |
| Double click | Advance to next media source |
| Long press | Stop |

---

## Media Sources

Edit `/config/media_sources.txt` — one source per line. Two formats are supported:

- **Direct stream URL**: `https://streaming.live365.com/a25002`
- **YouTube Music search string**: `60s & 70s Summer Road Trip`

The automation auto-detects the type: URLs are passed directly to MA, search strings use `media_type: playlist` with `radio_mode: true` for continuous shuffled playback.

The sensor re-reads the file every 30 seconds — no restart needed after edits.

---

## Secrets & API Keys

`knob.yaml` uses ESPHome's `!secret` system — values are pulled from `secrets.yaml` in your ESPHome config directory (not committed to this repo).

You need the following keys defined in that file:

```yaml
wifi_ssid: "YourNetworkName"
wifi_password: "YourWifiPassword"
ota_pass: "your_ota_password"
api_encryption_key: "your_32_byte_base64_key"
```

**To generate a new `api_encryption_key`:**
In the ESPHome dashboard, create any new device and copy the generated key, or run:
```
python3 -c "import secrets, base64; print(base64.b64encode(secrets.token_bytes(32)).decode())"
```

The `ota_pass` can be any string you choose — it protects over-the-air firmware updates.

---

## Dependencies

- **Music Assistant** running and connected to HA (entity: `media_player.office_mini_2`)
- **YouTube Music provider** enabled in MA with PO token server (`ytmusic-po-token` container on port 4416)
- **`input_number.knob_mat_stream_index`** helper (see setup below)
- **`sensor.knob_mat_sources`** command_line sensor (see setup below)

> **Note:** `knob.yaml` references `media_player.office_mini` (the direct Cast entity) for volume control and the LED state sensor. This is intentional — volume and LED don't need to go through MA.

---

## Setup Steps

### 1. Copy `media_sources.txt` to `/config/media_sources.txt`

### 2. Add the sensor to `configuration.yaml`

Add the `command_line:` block from `knob_mat_sensor.yaml`. Requires a **full HA restart**.

### 3. Create the `input_number` helper

Settings → Helpers → Create Helper → Number:

| Field | Value |
|---|---|
| Name | `Knob Mat Stream Index` |
| Min | `0` |
| Max | `999` |
| Step | `1` |
| Mode | Input field |

Entity ID: `input_number.knob_mat_stream_index`

### 4. Import the automation

Settings → Automations → three-dot menu → **Edit in YAML** → paste `automation_knob_mat.yaml` → Save.

### 5. Flash the D1 Mini

Open ESPHome, load `knob.yaml`, and flash to the device.

### 6. Restart HA

Required for the `command_line` sensor to load.

---

## Verification

After restart, in **Developer Tools → States**:
- `sensor.knob_mat_sources` → state = number of sources, attributes include `sources: [...]`
- `input_number.knob_mat_stream_index` → state = `0`

Test by firing in **Developer Tools → Events**:
```
Event type:  esphome.media_controller_action
Event data:  {"action": "dedicated_double_click"}
```

---

## Architecture

```
/config/media_sources.txt
        ↓ (re-read every 30s)
sensor.knob_mat_sources
  .state          = number of sources
  .attributes.sources = ["url1", "search string", ...]
        ↓
input_number.knob_mat_stream_index  (persisted across restarts)
        ↓
button_double sequence:
  1. read sources[idx]
  2. play via music_assistant.play_media on media_player.office_mini_2
     - http URLs → direct stream, no media_type
     - search strings → media_type: playlist, radio_mode: true
  3. advance idx = (idx + 1) % len(sources)
```
