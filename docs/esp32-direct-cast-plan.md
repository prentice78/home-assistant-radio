# Decoupling the Knob-Mat Radio from Home Assistant

## Context

The current `home-assistant-radio` project (D1 Mini + rotary encoder + button → ESPHome → HA automation → Music Assistant → Google Nest Mini) is functional but heavily dependent on Home Assistant. The user wants to explore reducing or eliminating that dependency while keeping the same physical controls and the Nest Mini as the output device.

This plan tracks research and the eventual implementation strategy. **Phase 1 focuses on one blocker: can the microcontroller change the Nest Mini's volume directly, without any server?** Other actions (play/pause, source switching, media load) will be scoped separately.

---

## Phase 1 Research: Direct ESP-to-Cast Volume Control

**Verdict: Yes, feasible today. No server required.**

### Why it works
- Google Cast's `RECEIVER` namespace (`urn:x-cast:com.google.cast.receiver`) accepts unauthenticated `SET_VOLUME` messages over TLS on port 8009. This is the same path `pychromecast` uses and has not been broken by Google. Volume is at the receiver level — no media-session establishment needed.
- ESP32 has the RAM, flash, and hardware TLS to maintain a persistent connection. `WiFiClientSecure.setInsecure()` accepts the Cast cert (Cast does not validate the client).
- mDNS discovery (`_googlecast._tcp`) is supported via Arduino `ESPmDNS`, or the Nest Mini IP can be hardcoded with a DHCP reservation (more reliable for a fixed install).

### Existing libraries / prior art
| Project | Notes |
|---|---|
| [ArduCastControl](https://github.com/andrasbiro/ArduCastControl) | Most mature. ESP8266-tested, ESP32-portable. Uses nanopb. Has `setVolume`/`setMute`. |
| [CastVolumeKnob](https://hackaday.io/project/163525-castvolumeknob) ([repo](https://github.com/WebmasterTD/CastVolumeKnob)) | Clever template-buffer trick that skips protobuf entirely (~200 LOC). Proven on Chromecast Audio; unconfirmed on Nest Mini but protocol is identical. |
| [ESPCaster](https://github.com/amitn/ESPCaster) | ESP32-S3 proof-of-concept, less mature. |
| [GHLocalApi](https://github.com/rithvikvibhu/GHLocalApi) port 8443 `/setup/` | **Dead end** for standalone use — requires a `cast-local-authorization-token` that can only be refreshed via Google's cloud Homegraph (OAuth). |

### Alternatives ruled out
- **UPnP / DLNA**: Nest Mini does not expose a RenderingControl service.
- **Bluetooth AVRCP**: Hijacks the audio path; only works while BT-paired.
- **Matter**: Not a Casting receiver target yet (mid-2026).

### Critical hardware gotcha
The user's current Knob-Mat hardware is a **D1 Mini (ESP8266)**. ESP8266 can technically do BearSSL to port 8009 but RAM pressure makes a long-running TLS session + encoder + button code unreliable. **An ESP32 board is recommended** — either an ESP32 dev board or an ESP32 "D1 Mini" form-factor clone (LOLIN D1 Mini ESP32) that preserves the existing pinout/footprint.

### Estimated effort for volume-only prototype
1–2 evenings:
1. Fork ArduCastControl (or transcribe CastVolumeKnob's template-buffer approach into C++).
2. Swap to ESP32 libs (`WiFiClientSecure`, `ESPmDNS`).
3. Hardcode Mini IP via DHCP reservation.
4. Wire rotary-encoder events to `setVolume(float 0.0..1.0)`, debounced.
5. Keep one TLS session alive (respond to Cast PING with PONG every 5s); reconnect on disconnect.

### Other gotchas
- `setInsecure()` skips cert validation — fine on trusted LAN, worth flagging.
- TLS handshake is ~1–2s and burns ~30 KB heap; reuse the session, don't reconnect per encoder tick.
- Music Assistant connects via the same CASTV2 path; both clients can coexist but rapid duplicate SET_VOLUME can race — debounce.
- Google has tightened the :8443 API historically but not the CASTV2 wire format. Keep firmware OTA-updatable as insurance.

---

## Phase 2 Research: Direct ESP-to-Cast Transport Control (Play/Pause/Next/Prev/Stop)

**Verdict (assuming ESP32 hardware): all transport commands feasible. Loading a new media URL is the only action that should stay on MA.**

### Per-action results
| Action | Direct from ESP32? | Mechanism |
|---|---|---|
| Volume / mute | ✅ | RECEIVER namespace, no session needed |
| Play / Pause | ✅ | MEDIA namespace `PLAY` / `PAUSE` with `mediaSessionId` |
| Next / Previous (FF/RW) | ✅ | MEDIA namespace `QUEUE_UPDATE` with `jump: ±1` |
| Stop | ✅ | MEDIA `STOP` (unloads media; does not kill the receiver app) |
| Load new stream / change source | ❌ practically | Needs `LAUNCH` + `LOAD` against the right receiver app; will fight MA's auto-relaunch |

### Why second-sender transport control works
- All transport commands target the existing `mediaSessionId`, not a new session. Any Cast client on the LAN can `GET_STATUS` on the receiver, discover `applications[0].sessionId` + `transportId`, connect to that transport, fetch `mediaSessionId` from `MEDIA_STATUS`, and issue commands. This is exactly how the Google Home phone app's "now playing" remote works for Spotify, YouTube Music, MA-cast sessions, etc.
- **No auth tokens, no pairing.** LAN reach is the only gate. SenderId is arbitrary.
- The frequently-cited "pychromecast can't take over" issues ([#804](https://github.com/home-assistant-libs/pychromecast/issues/804), [#1007](https://github.com/home-assistant-libs/pychromecast/issues/1007)) are about `LOAD` (replacing the stream), **not** transport commands. The distinction matters.

### Protocol sequence (~5 round-trips, then steady state)
1. TLS:8009 → `CONNECT` on `urn:x-cast:com.google.cast.tp.connection` to `receiver-0`
2. `GET_STATUS` on RECEIVER → parse `applications[0].{sessionId, transportId, appId}`
3. Second `CONNECT` to `transportId` (the app's virtual channel)
4. `GET_STATUS` on MEDIA → parse `status[0].mediaSessionId`
5. Send `{type:"PAUSE", mediaSessionId:N, requestId:K}` on MEDIA namespace

After initial connect, refresh `mediaSessionId` on each pushed `MEDIA_STATUS` (track changes).

### Music Assistant compatibility
- MA uses its **custom MA Cast Receiver app** (not the Default Media Receiver `CC1AD845`). [MA docs](https://www.music-assistant.io/player-support/google-cast/).
- The custom app still exposes `urn:x-cast:com.google.cast.media`, so transport control works without hardcoding an appId.
- **`LOAD` is the catch:** MA's custom receiver expects MA's manifest schema, not raw stream URLs. And if you `LAUNCH CC1AD845` to swap apps, MA will relaunch itself ("power on" reclaim).

### Library status
- [ArduCastControl](https://github.com/andrasbiro/ArduCastControl) already implements the full handshake and exposes `play()`, `pause()`, `next()`, `prev()`, `seek()`, `stop()`. Not just a volume library.
- Effort delta vs volume-only: **+1 evening.** You're wiring button events to existing library methods, plus handling the `MEDIA_STATUS` push to keep `mediaSessionId` fresh (the lib does this).

### Architectural implication
**Hybrid is the right answer.** Direct ESP32 → Cast for *all real-time control* (volume, play/pause, FF/RW, stop). Keep MA (no HA needed) as the one server-side dependency, used only when the button-double-click event needs to load a new stream/source. That's a single HTTP/WS call to MA's API, infrequent, latency-tolerant.

This means HA can be **fully removed** from the radio's runtime path; only MA remains, and only for source switching.

### Sources
- [pychromecast media.py](https://github.com/home-assistant-libs/pychromecast/blob/master/pychromecast/controllers/media.py) — `play/pause/stop/seek` + `queue_next/queue_prev`
- [pychromecast socket_client.py](https://github.com/home-assistant-libs/pychromecast/blob/master/pychromecast/socket_client.py) — handshake
- [ArduCastControl.cpp](https://github.com/andrasbiro/ArduCastControl/blob/main/ArduCastControl.cpp) — reference C++ implementation
- [MA Cast docs](https://www.music-assistant.io/player-support/google-cast/)
- ["takeover" issues](https://github.com/home-assistant-libs/pychromecast/issues/804) — context for what *doesn't* work (LOAD only)

---

## Phase 3 Research: Can ESP32 Trigger a YouTube Playlist Directly?

**Verdict: Not practical without a small helper. But the helper can be tiny (~80 lines) — MA still goes away.**

### Why direct YouTube triggering is hard
- The YouTube Cast receiver app (`233637DE`) **does not** accept a standard `LOAD` on the MEDIA namespace. It's driven by YouTube's **Lounge API** over HTTPS to `youtube.com`.
- Full handshake = 3 TLS endpoints (Cast TLS:8009 → `getMdxSessionStatus` → `youtube.com` pairing → `loungeToken` → `bind` for `SID`/`gsessionid` → `setPlaylist`).
- Two TLS cert chains in flash, a long-poll `/bc/bind` to keep playback queued reliably, and **OAuth Bearer auth** for any account-bound content (anonymous works only for public videos).
- YT Music has no separate first-class Cast receiver app — Google reuses the YouTube receiver. Personalized YTM content requires account auth.
- **Zero hits on GitHub/Hackaday** for ESP32 implementing Lounge `setPlaylist`. Nobody has ported this.
- PO Tokens / botguard (2024+) are handled receiver-side, so they're not the ESP32's problem — but YT free-tier ads will still play.

### The handoff itself is fine
Once `setPlaylist` lands, the Nest Mini fetches and plays on its own — the sender can disconnect both TLS connections and playback continues until the queue ends. So the "handoff and disconnect" semantic the user asked about is real; it's just the *initial handshake* that's heavy.

### Direct-URL sources are still possible from ESP32
For SomaFM and other raw stream URLs in `media_sources.txt`, the ESP32 *can* do `LAUNCH CC1AD845` (Default Media Receiver) + `LOAD <url>` on its own. No server needed for those.

### Recommended architecture: hybrid, MA eliminated
```
                ┌────────────────────────────────────────────────┐
                │  ESP32 Knob-Mat firmware                        │
                │                                                  │
                │  • volume / play / pause / next / prev / stop   │
                │      → direct Cast TLS:8009 (ArduCastControl)   │
                │                                                  │
                │  • source = direct stream URL                   │
                │      → direct: LAUNCH CC1AD845 + LOAD            │
                │                                                  │
                │  • source = YT Music preset                     │
                │      → HTTP POST to tiny helper                 │
                └────────────────────────────────────────────────┘
                                       ↓
                ┌────────────────────────────────────────────────┐
                │  Tiny helper on minipc (~80 LOC Python)         │
                │  FastAPI + pyytlounge                            │
                │  POST /play  {device, playlist_or_search}        │
                │   → pyytlounge.set_playlist(...)                 │
                └────────────────────────────────────────────────┘
                                       ↓
                ┌────────────────────────────────────────────────┐
                │  Nest Mini (pulls content itself after handoff)  │
                └────────────────────────────────────────────────┘
```

**Result:** Home Assistant gone. Music Assistant gone. One tiny Python service runs on the existing minipc, called only on source-change (latency-tolerant). All real-time control is ESP32 → Nest Mini direct.

### Trade-off: YT Music search vs preset
Current sources include free-form search strings like `"60s & 70s Summer Road Trip"`. The tiny helper has two reasonable shapes:
- **Preset list (simplest):** resolve search strings → playlist IDs once on a laptop, store playlist IDs in `media_sources.txt`. Helper just calls `setPlaylist(id)`. No YT auth needed for public playlists.
- **Live search (closer to current UX):** helper uses `ytmusicapi` or similar to convert search string → playlist on each call. Needs YT account auth (cookies + OAuth refresh). More moving parts.

### Sources
- [pychromecast youtube.py](https://github.com/home-assistant-libs/pychromecast/blob/master/pychromecast/controllers/youtube.py)
- [casttube YouTubeSession](https://github.com/ur1katz/casttube/blob/master/casttube/YouTubeSession.py)
- [pyytlounge](https://github.com/FabioGNR/pyytlounge)
- [Lounge API reverse-engineering writeup](https://bugs.xdavidhu.me/google/2021/04/05/i-built-a-tv-that-plays-all-of-your-private-youtube-videos/)
- [plaincast (Go) YouTube receiver impl](https://github.com/aykevl/plaincast/blob/master/apps/youtube/youtube.go)

---

## Phase 4 Research: Fully Server-Less (Direct URLs Only)

**Verdict: Achievable. ESP32 + Nest Mini + WiFi router is the entire stack.**

If YouTube/YT Music is dropped and only direct stream URLs (SomaFM, internet radio) are used, the tiny helper goes away too. Runtime requirements collapse to: **same WiFi, nothing else.**

### Runtime architecture
```
WiFi router
  ├── Nest Mini   (pulls stream from internet after Cast LOAD)
  └── ESP32       (stores source list in flash; Cast TLS:8009 → Nest Mini)
```

### What the ESP32 does standalone
1. **Discover the Nest Mini** — mDNS `_googlecast._tcp` via `ESPmDNS`, or hardcoded IP from router DHCP reservation (more reliable).
2. **Store source list** — options ranked by maintainability:
   - LittleFS file edited via a one-shot config page the ESP32 serves only while a setup button is held (clean UX, no persistent server)
   - GitHub raw URL fetched at boot (updates by editing a public repo; no local server, needs internet)
   - Hardcoded in firmware (simplest; reflash to change)
3. **Cast TLS:8009 with `setInsecure()`** for volume + transport
4. **`LAUNCH CC1AD845` + `LOAD <stream_url>`** on source change

### Caveats
- **Stream format support** — Default Media Receiver handles direct MP3/AAC HTTP(S) and most HLS fine. A few obscure formats may fail; verify each URL once before adding.
- **HTTPS vs HTTP** — Nest Mini accepts both; HTTPS preferred.
- **No NTP needed** when using `setInsecure()` (skipping cert validation skips time-sync requirement).
- **OTA updates** — ESP32 HTTPS-OTA can pull firmware from a public URL (e.g. GitHub releases); cloud-only, no local server.
- **First-time WiFi setup** — captive-portal AP mode is one-time (not a persistent server).
- **Router is technically "local"** but user-defined scope counts WiFi infra as fine.

### Trade-offs vs hybrid (with tiny helper)
- ✅ Zero local server, zero maintenance of a Python process
- ✅ Truly portable — works at a friend's house after WiFi re-setup
- ❌ No YT Music search strings (only fixed stream URLs)
- ❌ No dynamic playlists; source list is just a list of station URLs
- ❌ Updating the source list is less convenient than editing a text file on the minipc

---

## Decision Status

Resolved:
- ✅ Hardware: ESP32 swap (LOLIN D1 Mini ESP32 to preserve form factor)
- ✅ Volume + play/pause + FF/RW + stop: direct Cast from ESP32
- ✅ Direct-URL sources (SomaFM etc.): direct Cast from ESP32 (`LAUNCH CC1AD845` + `LOAD`)
- ✅ HA fully removed from runtime path

Two viable end-state architectures — user chooses based on whether YT Music sources are kept:

**Option A — Fully server-less (only direct stream URLs)**
- Runtime stack: WiFi + ESP32 + Nest Mini. Nothing else.
- Source list lives on ESP32 (LittleFS, GitHub-raw fetch, or firmware-hardcoded).
- Loses YT Music search strings.

**Option B — Hybrid (keep YT Music)**
- Runtime stack: above + tiny ~80-LOC Python helper on minipc (FastAPI + `pyytlounge`).
- ESP32 handles direct URLs itself; HTTP POST to helper only for YT presets.
- HA and MA both gone; minipc runs only this tiny service.

Resolved for Phase 5:
- ✅ **Option A** chosen — fully server-less
- ✅ **Source format:** URL-only plain text, one per line (`media_sources.txt`)
- ✅ **Source hosting:** GitHub raw URL fetched at boot, cached to LittleFS as fallback
- ✅ **Toolchain:** PlatformIO / Arduino C++
- ✅ **Cast library:** [ArduCastControl](https://github.com/andrasbiro/ArduCastControl) — handles persistent TLS session, PING/PONG, reconnect

---

## Phase 5: Implementation Plan

### Hardware bill of materials
| Item | Notes |
|---|---|
| LOLIN D1 Mini ESP32 | Same form factor as current ESP8266 D1 Mini — drop-in for the existing wiring |
| Existing rotary encoder (EC11) | Reused; both GND pins to ESP32 GND |
| Existing dedicated push button | Reused |
| Existing status LED | Reused |
| Router DHCP reservation | Pin the Nest Mini to a fixed IP (e.g. `192.168.50.40`) so the ESP32 doesn't need mDNS |

### Project structure
```
home-assistant-radio/        # existing GitHub repo, reused
├── media_sources.txt        # one stream URL per line — already in repo
├── firmware/                # NEW — PlatformIO project
│   ├── platformio.ini
│   ├── src/
│   │   ├── main.cpp         # boot, input loop, source cycling
│   │   ├── cast_client.cpp  # thin wrapper around ArduCastControl
│   │   ├── sources.cpp      # GitHub fetch + LittleFS cache
│   │   └── secrets.h        # gitignored — WiFi creds, Nest Mini IP
│   └── lib/
│       └── ArduCastControl/ # vendored or via lib_deps
└── knob.yaml                # existing ESPHome config — keep as historical reference; not used after migration
```

### Firmware flow

**Boot sequence:**
1. Mount LittleFS
2. WiFi connect (creds from `secrets.h`; ~3s)
3. Fetch `https://raw.githubusercontent.com/prentice78/home-assistant-radio/main/media_sources.txt`
   - On success: write to LittleFS as `sources.cache`, parse into `String[]`
   - On failure: load `sources.cache` from LittleFS, parse
4. Restore `currentSourceIdx` from NVS (Preferences API)
5. Connect to Nest Mini at hardcoded IP via ArduCastControl (`cast.begin(IP)`)
6. Enter input loop

**Input loop:**
| Trigger | Action |
|---|---|
| Encoder rotate CW | `cast.setVolume(currentVol + 0.05)` (clamp 0..1) |
| Encoder rotate CCW | `cast.setVolume(currentVol - 0.05)` |
| Encoder single click | `cast.next()` (QUEUE_UPDATE jump +1) |
| Encoder double click | `cast.prev()` (QUEUE_UPDATE jump -1) |
| Encoder long press | `cast.setVolume(0.4)` |
| Button single click | `cast.isPlaying() ? cast.pause() : cast.play()` |
| Button double click | `currentSourceIdx = (currentSourceIdx + 1) % sources.size(); cast.launchAndLoad("CC1AD845", sources[currentSourceIdx]); savePrefs();` |
| Button long press | `cast.stop()` |

**Idle / persistence:**
- ArduCastControl handles PING/PONG and reconnect inside its `loop()` method (call from Arduino `loop()`)
- LED reflects connection state (solid = connected & playing, blink = connecting, off = paused/stopped)

### Cast library wrapping notes
- `setInsecure()` on the underlying `WiFiClientSecure` — skip cert validation
- ArduCastControl's `setVolume(float)` is `RECEIVER`-namespace, no session needed
- `launchAndLoad` may need to be wrapped: if ArduCastControl doesn't expose a one-shot `LAUNCH CC1AD845 + LOAD <url>`, add a tiny helper that fires the two messages back-to-back with a 500ms gap
- If a session is already active (MA-driven or from a previous LOAD), the LAUNCH may be a no-op — verify on first hardware test

### What gets removed
- HA automation `automation_knob_mat.yaml` — delete
- `knob_mat_sensor.yaml` command_line sensor — delete from HA config
- `input_number.knob_mat_stream_index` helper — delete from HA helpers
- Music Assistant entry for this device — can stay, just not used by the radio
- Old `knob.yaml` ESPHome config — keep in repo as reference, but the ESP32 firmware replaces it

### Verification plan (incremental)

1. **WiFi + Cast connect.** Flash bare firmware that does WiFi + ArduCastControl `begin()` + `getStatus()`. Confirm via serial that the Nest Mini responds with current volume/playback state.
2. **Volume control.** Add encoder ISR; verify rapid spinning produces smooth volume changes (no audible stuttering or skipped ticks).
3. **Transport control.** Start music from the Google Home app on a phone, then test button single-click pause/resume and encoder click next/prev. Confirm transport commands hijack the active session.
4. **Source loading.** Add `launchAndLoad`; press button double-click to cycle to first SomaFM stream. Confirm playback starts within ~3s and continues without ESP32 needing to stay connected (test by power-cycling the ESP32 — Nest Mini should keep playing).
5. **Source list refresh.** Push a commit to `home-assistant-radio` adding a new stream URL; reboot ESP32; confirm new station appears in the cycle.
6. **Failure modes.** Block GitHub at the router; reboot; confirm fallback to LittleFS cache works. Power-cycle the Nest Mini mid-session; confirm ArduCastControl reconnects.
7. **Long-soak test.** Leave running for 24h with periodic volume/transport commands; confirm no memory leak, no reboot loops, persistent session stays healthy.

### Risks / open questions for implementation
- **ArduCastControl ESP32 port** — library is ESP8266-tested; needs `WiFiClientSecure` swap and `ESP8266mDNS` → `ESPmDNS` (we're skipping mDNS so that's moot, but the WiFi types matter). Budget 1 evening for porting fixes.
- **LAUNCH + LOAD wrapper** — ArduCastControl may or may not expose this cleanly; might need ~30 LOC of custom protobuf messages. Worst case, lift the message shape from pychromecast's `socket_client.py` and `media.py`.
- **Long-soak TLS heap** — if heap creeps over days, may need scheduled session refresh (close + reconnect every N hours).
- **CASTV2 wire format stability** — Google has not changed this in years, but keep OTA working (HTTPS-OTA from GitHub releases) as insurance.

---

## Sources

- [ArduCastControl repo](https://github.com/andrasbiro/ArduCastControl)
- [CastVolumeKnob Hackaday project](https://hackaday.io/project/163525-castvolumeknob)
- [ESPCaster repo](https://github.com/amitn/ESPCaster)
- [GHLocalApi docs](https://rithvikvibhu.github.io/GHLocalApi/)
- [pychromecast reference implementation](https://github.com/home-assistant-libs/pychromecast)
