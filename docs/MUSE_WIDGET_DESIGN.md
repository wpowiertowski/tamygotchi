# Muse Widget: Design

A Tamagotchi-style companion on the ESP32-S3-Touch-AMOLED-1.75C. A pixel-art Muse character lives on the round display. You hold a button to talk to it, its reply streams onto the screen as text, and it shows what the agent is doing in the background.

Status: design only. Nothing has been built. Status tags are the same as in `KNOWLEDGE_BASE.md`: [V] verified against a primary source, [S] secondary or third-party source, [A] assumption.

Researched: 2026-11-15.

---

## 1. Requirements

| # | Requirement |
|---|---|
| R1 | Push-to-talk: hold a hardware button, speak, release |
| R2 | Animated pixel-art Muse character whose mood follows agent state |
| R3 | Agent reply streamed and displayed as text (no speech output required) |
| R4 | Asynchronous activity the agent performs is pushed to the widget, including while no conversation is active |

---

## 2. What "Muse" offers today

Meta ships two different things under the Muse name, and they behave very differently for this project.

### 2.1 Muse, the consumer agent (Meta AI app)

- Runs tasks in the background in a cloud VM ("Muse Secure VM") and reports back when something changes or needs approval [S].
- Connects to outside services through connectors. Users can have Muse write a personal custom connector to any public API or hosted MCP server. Developers can also submit connectors for review on the Muse Connector Platform, opened 2026-09-18 [S].
- The VM runs in Meta's cloud, so a connector target must be reachable from the public internet. A server on your LAN or laptop will not work [S].
- **No documented API lets an outside device send messages to this agent or read its activity feed.** None of the sources found describe webhooks or event subscriptions out of Muse [S]. The only outbound path is Muse calling a connector you host.

### 2.2 Meta Model API (developer API)

| Item | Detail | Status |
|---|---|---|
| Base URL / auth | `https://api.meta.ai/v1`, bearer token | [V] |
| Protocols | Responses API (`POST /v1/responses`, recommended for agents; retrieve, cancel, delete), Chat Completions (OpenAI-compatible), Messages (Anthropic-compatible) | [V] |
| Agent models | `muse-spark-1.3` / `1.2` / `1.1`, `muse-wonder-1.0` (preview), `muse-ember-1.0` (low cost). 1M-token context | [V] |
| Agent features | Tool calling (parallel), built-in `web_search`, `previous_response_id` for server-side state, `reasoning_effort` | [V] |
| Text streaming | Docs mention streaming and streamed tool-call arguments. Event names not yet checked | [S] |
| Speech-to-text | `muse-voice-transcribe-1.0`. Realtime at `wss://api.meta.ai/v1/asr/realtime`, file upload at `POST /v1/asr/transcribe`. $0.18 per audio hour | [V] |
| ASR audio format | Mono PCM16. 24 kHz native, 16 kHz also accepted. Auth is sent in the first JSON frame, not the HTTP header. Sessions last up to 60 minutes | [S] (Pipecat and LiveKit plugin sources, a third-party blog) |
| Text-to-speech | Not offered | [V] |
| Background jobs / webhooks | Not documented | [V] (absent from docs) |
| Availability | Public preview, US developers | [V] |
| Pricing (Muse Spark) | From $1.25 input / $4.25 output per 1M tokens | [V] |

### 2.3 What this means for the requirements

| Req | Consumer Muse (2.1) | Model API (2.2) |
|---|---|---|
| R1 voice in | No inbound API | ✅ realtime ASR |
| R3 streamed reply | No inbound API | ✅ |
| R4 async activity | ⚠️ Only if Muse calls a connector you host, and only when told to. Best effort, not a full activity feed | ✅ your own bridge runs the agent and owns every event |

Conclusion: R1 and R3 need the Model API. R4 can come from either source: your own agent (complete and reliable) or the consumer Muse via a connector (partial, driven by prompts).

---

## 3. Architecture

The device never talks to Meta directly. It holds one WebSocket to a **bridge** service that you run.

```
 ┌──────────── device (ESP32-S3) ────────────┐        ┌──────────────── bridge (always on) ─────────────────┐
 │ button ─► PTT ─► mic 16 kHz PCM16 ────────┼─ WSS ─►│ ASR relay ──► wss://api.meta.ai/v1/asr/realtime     │
 │                                           │        │                                                     │
 │ LVGL: sprite + mood + text ◄──────────────┼─ WSS ◄─│ agent loop ─► POST /v1/responses (stream, tools)    │
 │                                           │        │     │                                               │
 │ activity badge / feed ◄───────────────────┼─ WSS ◄─│ event log (seq) ◄── background jobs                 │
 └───────────────────────────────────────────┘        │                ◄── optional: MCP endpoint that the  │
                                                      │                    consumer Muse calls (connector)  │
                                                      └─────────────────────────────────────────────────────┘
```

Why a bridge instead of calling the API from the device:

- **Key safety.** The Meta API key stays off a device that anyone can pick up and dump the flash from.
- **Always on.** R4 needs something running while the device sleeps or is offline. The bridge logs events and replays them when the device reconnects.
- **One connection.** ASR (WebSocket) and agent output (HTTP streaming) are merged into one small protocol the firmware can handle easily.
- **Swappable backend.** The device protocol does not change whether events come from your own agent, the consumer Muse, or something else later.
- **Reachability.** If the consumer-Muse connector is used, its endpoint must be public. The bridge is the one public piece; the device stays on your LAN.

---

## 4. Device ↔ bridge protocol (v0 draft)

WebSocket over TLS. Control messages are JSON text frames. Microphone audio is sent as binary frames.

Device → bridge:

| Type | Fields | Notes |
|---|---|---|
| `hello` | `proto`, `device_id`, `fw`, `last_seq` | Sent on connect. The bridge replays activity after `last_seq` |
| `ptt_start` | `turn_id`, `rate: 16000`, `fmt: "pcm16"`, `ch: 1` | Button pressed |
| *(binary)* | raw PCM16 LE, 20–40 ms per frame | Only between `ptt_start` and `ptt_end` |
| `ptt_end` | `turn_id` | Button released |
| `cancel` | `turn_id` | Long-press or tap to abort a reply |
| `ack` | `seq` | Activity shown; the bridge can trim its replay window |

Bridge → device:

| Type | Fields | Notes |
|---|---|---|
| `state` | `mood`: `idle`, `listening`, `thinking`, `replying`, `working`, `alert`, `sleeping`, `error` | Drives the sprite |
| `transcript` | `turn_id`, `text`, `final` | Live caption while you speak |
| `reply_delta` | `turn_id`, `text` | Appended to the text view |
| `reply_done` | `turn_id` | |
| `activity` | `seq`, `id`, `phase`: `started`, `progress`, `done`, `failed`, `needs_input`; `title`, `detail`, `ts` | Monotonic `seq`; the same `id` is reused across phases |
| `error` | `code`, `message` | |

Keep messages short; the device only renders text. The bridge strips Markdown and citations before sending `reply_delta`.

---

## 5. Firmware

ESP-IDF v5.5.x project using the vendor BSP (see `KNOWLEDGE_BASE.md` section 1.1).

### 5.1 Tasks

| Task | Core | Owns |
|---|---|---|
| `ui` | 1 | All LVGL calls, under `bsp_display_lock()`. Consumes a UI event queue |
| `audio` | 1 | `esp_codec_dev_read()` from the ES7210 at 16 kHz mono → ring buffer |
| `net` | 0 | Wi-Fi, `esp_websocket_client`, reconnect with backoff, JSON encode and decode |
| `input` | any | PTT button debounce, long-press detection (`espressif/button` component) |

Wi-Fi runs on core 0 by default [A], so the UI and audio tasks go on core 1.

### 5.2 Push-to-talk

- **Button:** the BSP does not wrap any buttons (`BSP_CAPS_BUTTONS 0`) [V]. Start with BOOT, assumed to be GPIO0 [A]. GPIO0 is a strapping pin, so it is only safe to read after boot. Check the schematic for whether PWR can be read as a GPIO or only through an AXP2101 PWRON interrupt. If it can, PWR is the nicer PTT key and BOOT stays free.
- **Audio:** `bsp_audio_init()` defaults to 22050 Hz [V]. Pass your own `i2s_std_config_t` for 16 kHz so the bridge can forward audio unchanged. I2S is shared duplex with the speaker, so both run at the same rate.
- **Bandwidth:** 16 kHz × 16 bit = 256 kbit/s, which is trivial on Wi-Fi. Stream it; do not buffer the whole utterance (30 s would be 960 KB).
- Cap a turn at 60 s on the device to match the ASR session limit [S].

### 5.3 Pixel-art Muse

- **Visible area:** a circle 466 px across. The largest inscribed square is about 329 × 329 px [A, geometry].
- **Sprite:** draw at 64 × 64 (or 48 × 48), scale ×4 with nearest-neighbour filtering (no antialiasing), which gives 256 × 256 (or 192 × 192). Put the sprite in the upper part of the circle and text in the lower arc.
- **Frames:** an indexed 16-colour format (`I4`) is about 2 KB per 64 × 64 frame plus palette, so dozens of frames fit in flash. Convert with LVGL's image converter. Each mood is 2–8 frames at 6–10 fps, which suits the pixel-art style and keeps QSPI traffic low.
- **AMOLED:** a black background turns pixels off, which saves power. Drift the sprite a few pixels every few minutes and dim it when idle to avoid burn-in.
- **Moods:**

| Mood | Trigger | Sprite idea |
|---|---|---|
| `idle` | nothing happening | breathing or blinking loop |
| `listening` | PTT held | ears up, pulsing ring around the edge of the display |
| `thinking` | `ptt_end` → first `reply_delta` | swirl, dots |
| `replying` | `reply_delta` stream | mouth or quill moving |
| `working` | an `activity` is open | carrying, building, small spinner |
| `alert` | `activity` `done`, `failed` or `needs_input` not yet acked | waving; badge count on the edge of the circle |
| `sleeping` | idle timeout | eyes closed, display dimmed |
| `error` | offline or bridge error | confused face, short reason |

### 5.4 Text view

- An LVGL label in the lower arc that scrolls as `reply_delta` text arrives. Wrap the text to fit the circle (narrower near the bottom).
- Swipe up opens the full reply; swipe sideways switches between the conversation view and the activity feed. Touch is CST9217 through `bsp_display_get_input_dev()` [V].
- Fonts: LVGL Montserrat, or a pixel font to match the sprite. Include only the glyphs you need to save flash.

### 5.5 Power and PMIC

- The BSP has no AXP2101 code [V]. Battery level and charging state need your own driver.
- Wi-Fi with modem sleep keeps the WebSocket alive for R4, but costs battery. Measure first (`KNOWLEDGE_BASE.md` section 8), then decide between always connected, waking periodically to fetch missed events, or always connected only while charging.

---

## 6. Bridge

Any always-on host with outbound internet access: a Raspberry Pi, a home server or a small VPS. It must be publicly reachable only if the consumer-Muse connector (6.2) is used.

Suggested language: Python (asyncio). Pipecat and LiveKit already ship Meta STT clients [S], and the OpenAI SDK works against `api.meta.ai/v1` [V].

### 6.1 Backend A: your own agent on the Model API (primary)

1. Relay PTT audio to the realtime ASR WebSocket. Send `transcript` updates to the device.
2. When the final transcript arrives, call `POST /v1/responses` with streaming on and `previous_response_id` for continuity. Forward text deltas as `reply_delta`.
3. Tools run on the bridge. Anything long-running becomes a background job that emits `activity` events. Status of each event comes from the bridge itself, so R4 is complete.
4. Persist the event log (SQLite) so replay survives a bridge restart.

### 6.2 Backend B: relay for the consumer Muse (optional add-on)

1. The bridge exposes a small remote MCP server (streamable HTTP, bearer-token auth) on a public HTTPS URL.
2. Add it to Muse as a personal custom connector, entering the token through Muse's credential prompt, never in chat [S].
3. Tools: `widget_post_activity(title, phase, detail)` and `widget_say(text)`. Each becomes an `activity` event for the device.
4. Instruct Muse, for example as a standing preference: "post progress on long tasks to the widget".
5. Optional reverse path: a `widget_inbox()` tool that returns voice notes recorded on the device, which Muse checks on a schedule. This adds a lot of latency and is not a conversation.

Limits: this only reports what Muse decides to report. It is not a full feed of Muse's activity. Muse can change behaviour without notice.

---

## 7. Repository layout

The repo will hold firmware, bridge and assets, so the ESP-IDF project moves from the repo root (as in `KNOWLEDGE_BASE.md` section 5) to `firmware/`:

```
.
├── firmware/            # ESP-IDF project (CMakeLists.txt, main/, components/, sdkconfig.defaults, partitions.csv)
├── bridge/              # Python service
├── assets/sprites/      # source pixel art (.aseprite / .png) + conversion script
├── proto/               # protocol spec + JSON schemas shared by both sides
└── docs/
```

CI: `espressif/esp-idf-ci-action@v1` with `path: firmware`.

---

## 8. Milestones

| # | Goal | Done when |
|---|---|---|
| M0 | Hardware bring-up | `KNOWLEDGE_BASE.md` section 8 checklist through the colour test and mic loopback |
| M1 | Character on screen | Sprite animates; moods switch from a test menu; no network |
| M2 | Bridge link | Device connects to a stub bridge over WSS; scripted `state`, `reply_delta` and `activity` events render; `last_seq` replay works |
| M3 | Voice in | Hold PTT → live `transcript` on screen from Meta realtime ASR |
| M4 | Agent replies | Final transcript → Responses API → streamed text on device |
| M5 | Async activity | Background job on the bridge → `working` / `alert` moods, feed view, ack |
| M6 | Consumer Muse relay | Optional backend B connector posts events to the device |
| M7 | Power | Sleep policy, battery readout through AXP2101, burn-in mitigation, measured runtime |

---

## 9. Open questions and risks

| Item | Notes |
|---|---|
| Model API region | Public preview for US developers [V]. Confirm your account has access |
| Responses streaming format | Confirm event names and whether background mode exists before writing the bridge |
| ASR protocol details | Audio format, handshake and session limits come from third-party sources [S]. Check against Meta's docs |
| PWR button wiring | GPIO or AXP2101 PWRON? Decides which button is PTT |
| Mic gain and noise | ES7210 dual mic; tune gain. Echo cancellation is not needed while there is no speech output |
| Speech output later | No Meta TTS [V]. Would need another provider or on-device chimes only |
| Battery life with Wi-Fi always on | Unknown until measured (M7) |
| Consumer Muse behaviour | Connector behaviour depends on prompts and can change without notice |

---

## 10. References

- Meta Model API overview: https://dev.meta.ai/docs/overview
- Muse Spark on Model API (blog, 2026-07-08): https://dev.meta.ai/resources/blog/build-with-muse-spark/
- Responses API reference: https://dev.meta.ai/docs/api-reference/responses
- Pipecat Meta STT (audio format): https://docs.pipecat.ai/api-reference/server/services/stt/meta
- LiveKit Meta STT plugin: https://docs.livekit.io/agents/models/stt/plugins/meta/
- Muse custom integrations (third party): https://parallel.ai/articles/meta-muse-custom-integrations
- Muse connectors opened (2026-09-18): https://runtimewire.com/article/meta-opens-muse-connectors-developers
- Vendor BSP v3.0.0: https://components.espressif.com/components/waveshare/esp32_s3_touch_amoled_1_75c
