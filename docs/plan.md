# Gear S3 Frontier → Standalone Gemini Voice Assistant

Detailed plan from feasibility research (2026-10-09). The goal is to repurpose a Samsung Gear S3 Frontier (Tizen 4.0) as a push-to-talk voice assistant that calls Google Gemini directly from the watch. There is no relay server or companion app.

---

## 1. Goal and scope

### In scope (v1)
- Push-to-talk voice questions from the watch.
- The watch sends recorded audio directly to the **Gemini API** (REST, API key).
- The answer is shown as text on the round screen (scrollable with the bezel) and spoken with the **Tizen on-device TTS**.
- Short conversation memory, so follow-up questions work.
- The app is sideloaded (developer mode) and is for personal use only.
- **Quick launch** (see §3.4):
  - Double-pressing the Home key opens the app (a system setting, no code needed).
  - A **custom watch face** (digital time, date, battery) has a mic button that opens the app straight into listening.

### Out of scope (v1)
- The real Google Assistant (see §2.2).
- Personal Google data such as Calendar, Tasks, or Home (would need OAuth; possible v2).
- "Hey Google" hotword or always-listening mode (battery cost, and no API for it).
- Any relay server or phone companion app.
- Gemini Live streaming (WebSocket, real-time audio).

---

## 2. Research findings

### 2.1 Platform and distribution
| Topic | Finding | Impact |
|---|---|---|
| Watch OS | Gear S3 Frontier has received its last update to **Tizen 4.0 (wearable)**. Specs: dual-core ~1GHz, 768MB RAM, 4GB storage, mic, speaker, Wi-Fi b/g/n (2.4GHz), BT 4.2, LTE on SM-R765 variants. | The device is weak, but enough for a thin client. |
| App store | Samsung **no longer accepts new or updated Tizen watch apps** in Galaxy Store (Seller Portal notice). | We **sideload** with developer mode, `sdb`, and a Samsung certificate. |
| SDK | Tizen Studio is still downloadable from `https://download.tizen.org/sdk/Installer/`, latest **6.1** (5.6 and 6.0 are also listed). The old developer.tizen.org pages now redirect to samsungtizenos.com. | We can still build. Keep a local copy of the installer and packages. |
| Signing | Gear devices only accept apps signed with a **Samsung certificate** (author + distributor). The distributor certificate must include the watch **DUID**. It is created through the "Samsung Certificate Extension" in Tizen Studio Package Manager, which needs a Samsung account. | **Biggest risk.** If Samsung's certificate service has been retired, nothing installs. Verify this before any other work. |

### 2.2 Google Assistant vs Gemini
| Option | Finding | Verdict |
|---|---|---|
| Google Assistant SDK | Still documented and labeled **experimental, non-commercial**. It is gRPC (HTTP/2) only, needs OAuth 2.0 user credentials and device model registration, and has limited features (no hotword, timers, or alarms). | ❌ Impractical on Tizen 4. We would need gRPC/protobuf/nghttp2 cross-compiled for armv7 Tizen, plus an OAuth flow. Google is also migrating Assistant to Gemini. |
| **Gemini API (REST)** | `POST https://generativelanguage.googleapis.com/v1beta/models/{model}:generateContent` with header `x-goog-api-key`. Accepts **inline base64 audio** (WAV, MP3, AAC, OGG, FLAC) plus text and returns JSON text. A newer "Interactions" SDK surface exists, but plain REST `generateContent` remains the lowest common denominator. | ✅ Plain HTTPS + JSON is easy with libcurl + json-glib. |
| Gemini Live API | WebSocket, raw 16kHz PCM streaming, barge-in. | Possible v2. Too complex for v1. |
| Gemini TTS | A separate model returns PCM audio. | Not needed. Tizen TTS is free and offline. |

### 2.3 Watch capabilities (Tizen 4.0 native APIs)
| Need | Tizen API | Notes |
|---|---|---|
| Record mic | `recorder` (audio-only), codec AAC, container MP4/M4A, 16kHz mono | Privilege `http://tizen.org/privilege/recorder`. An alternative is `audio_in` (raw PCM) + a WAV header, which is larger to upload. |
| HTTPS | `libcurl` (bundled in the native SDK) | TLS 1.2 is supported. The old CA bundle is a risk → ship our own `cacert.pem` and set `CURLOPT_CAINFO`. Google's GTS Root R1 is cross-signed by GlobalSign Root CA R1, which is valid until 2028. |
| JSON | `json-glib` (bundled) | Build the request and parse the response. |
| Base64 | `g_base64_encode` (GLib) | |
| Speech out | `tts` API (Samsung on-device engine) | Check the available voices and languages on the device. |
| UI | EFL / Elementary (`elm_*`, `eext_*` circle UI + rotary events) | `eext_circle_surface`, `eext_rotary_object_event_callback_add`. |
| Network | Wi-Fi, LTE, or phone BT proxy via Galaxy Wearable | Privileges `internet`, `network.get`. |
| Keep CPU awake | `device_power_request_lock(POWER_LOCK_CPU, …)` | Hold the lock while a request is in flight. |
| Secret storage | `key-manager` (`ckmc_*`) or app data dir | Store the API key. |
| Async work | `ecore_thread_run` / `ecore_main_loop_thread_safe_call_async` | Keep curl off the UI thread. |

---

## 3. Architecture

```
┌──────────────── Gear S3 (native C app "gemwatch") ────────────────┐
│ UI (EFL circle)                                                   │
│  idle ──tap/bezel──▶ listening ──tap/timeout 30s──▶ thinking      │
│                                                    │              │
│ recorder → data/q.m4a (AAC 16kHz mono)             │              │
│ base64 encode                                      ▼              │
│ request builder (system prompt + history + audio)                 │
│ libcurl HTTPS POST (worker thread, CA bundle, 30s timeout)        │
│ json-glib parse → {heard, answer}                                 │
│  ▼                                                                │
│ answering: tts speak + scrollable label (bezel)                   │
│ history ring buffer (last N text turns)                           │
└───────────────────────────────────────────────────────────────────┘
              │ HTTPS (TLS 1.2)
              ▼
   generativelanguage.googleapis.com  (Gemini Flash model)
```

### 3.1 Request shape
```json
POST /v1beta/models/{MODEL}:generateContent
x-goog-api-key: <KEY>
{
  "systemInstruction": {"parts":[{"text":
    "You are a voice assistant on a smartwatch. Answer the user's spoken question. Reply in the user's language, plain text, max 2 short sentences, no markdown, no emojis."}]},
  "contents": [
    {"role":"user","parts":[{"text":"<previous question transcript>"}]},
    {"role":"model","parts":[{"text":"<previous answer>"}]},
    {"role":"user","parts":[{"inlineData":{"mimeType":"audio/aac","data":"<base64>"}}]}
  ],
  "generationConfig": {
    "maxOutputTokens": 200, "temperature": 0.6,
    "responseMimeType": "application/json",
    "responseSchema": {"type":"OBJECT","properties":{
       "heard":{"type":"STRING"}, "answer":{"type":"STRING"}}}}
}
```
- Structured output returns `heard` (a transcript of the question) and `answer`. `heard` goes into the history as text, so audio is never re-sent. It is also shown on screen for confirmation.
- Verify the accepted MIME type for M4A/AAC (`audio/aac` vs `audio/mp4`) in the audio spike.
- Keep the model name configurable, because Gemini model IDs change often.

### 3.2 App states and errors
| State | UI | Exit |
|---|---|---|
| idle | Big mic button, last answer dimmed | tap / bezel click → listening |
| listening | Pulsing ring, elapsed seconds | tap → stop; 30s auto-stop; back → cancel |
| thinking | Spinner | response / error / 30s timeout |
| answering | `heard` small on top, `answer` scrollable; TTS speaking | tap → stop TTS; mic → new question |
| error | Short message + spoken ("No internet", "Quota exceeded", "Key invalid", "Timeout") | tap → idle |

HTTP mapping: 400 → bad audio or request, 401/403 → bad key, 429 → quota, 5xx → retry once.

### 3.3 Config
- `config.json` in app data: `{ "api_key": "...", "model": "...", "api_version": "v1beta", "lang": "en_US", "history_turns": 4 }`.
- Provisioning in v1: `sdb push config.json /opt/usr/home/owner/apps_rw/<pkg>/data/`. A v2 option is an on-watch settings screen.
- Restrict the API key to the Generative Language API. Keep `config.json` gitignored.

### 3.4 Quick launch: shortcut and watch face
Both entry points launch the main UI app with `app_control`. They pass the extra data `"action"="listen"`, so the app skips idle and starts recording immediately.

| Entry point | Tizen mechanism | Notes |
|---|---|---|
| Double-press Home | Watch Settings → Advanced → *Double press Home key* → choose "Gemwatch" | No code. Document it in the README. |
| Watch face | Native **watch application** (`watch_app_main`, `watch_time_get_*`, ambient mode callbacks). Digital time, date, battery, and a mic button at the bottom. | Tap the mic → launch with `action=listen`. Ambient mode draws a minimal time only (low-bit color, no mic). Privilege: `http://tizen.org/privilege/alarm.get` for time ticks where needed. |

- Packaging: one **multi-app TPK**. The UI app is the main project, and the watch face is a referenced project. One install covers both.
- Launch privilege: `http://tizen.org/privilege/appmanager.launch`.
- The UI app handles `app_control` in `app_control_cb`. If `action=listen` arrives while the app is already running, it restarts recording.

---

## 4. Proposed repo layout
```
gear-s3-gemini/
├── README.md                  # setup, signing, deploy, config
├── docs/
│   ├── plan.md                # this file
│   └── signing.md             # Samsung cert step-by-step + troubleshooting
├── watch/                     # Tizen native project
│   ├── tizen-manifest.xml     # pkg id, privileges, profile wearable-4.0
│   ├── project_def.prop
│   ├── inc/  app.h ui.h recorder_ctl.h gemini.h tts_ctl.h config.h history.h
│   ├── src/
│   │   ├── main.c             # app lifecycle (ui_app_main)
│   │   ├── ui.c               # EFL circle UI, states, rotary
│   │   ├── recorder_ctl.c     # recorder setup/start/stop
│   │   ├── gemini.c           # request build, curl, parse, errors
│   │   ├── tts_ctl.c          # TTS init/speak/stop
│   │   ├── config.c           # load config.json
│   │   └── history.c          # ring buffer of turns
│   ├── res/cacert.pem         # CA bundle (curl.se), refresh periodically
│   └── shared/res/icon.png
├── watchface/                 # Tizen native watch app (time/date/battery + mic)
│   ├── src/watchface.c
│   └── tizen-manifest.xml
├── tools/
│   ├── gemini_smoke.sh        # curl test from laptop with sample audio
│   └── deploy.sh              # build-native + package + install
├── config.example.json
└── samples/question.m4a
```

---

## 5. Implementation steps (todos)

| # | ID | Task | Done when |
|---|---|---|---|
| 1 | env-setup | Install Tizen Studio 6.1 + Wearable 4.0 Native + Samsung Wearable Extension + Samsung Certificate Extension. CLI `tizen`, `sdb`. | `tizen version`, `sdb version` work |
| 2 | device-connect | On the watch: Settings → About → Software → tap version 5x → Debugging ON; Wi-Fi on the same LAN. `sdb connect <ip>:26101`, accept on the watch. Read the DUID. Create the Samsung author + distributor certs. Deploy the "Basic UI" template. | Hello-world runs on the watch |
| 3 | https-spike | Laptop `gemini_smoke.sh` confirms the request format and MIME type. Watch: curl POST (text) with bundled CA, log via `dlog`. | 200 OK from the watch |
| 4 | audio-spike | Record 5–10s AAC on the watch, base64, send it to Gemini, log `heard`/`answer`. Measure size and latency over Wi-Fi and the BT proxy. Fallback: `audio_in` PCM → WAV. | Correct answer to a spoken question |
| 5 | tts-spike | `tts_create`, list supported voices, speak a sample. | Watch speaks text |
| 6 | app-core | `gemini.c` (builder, curl worker thread, parser, error mapping, retry), `history.c`, `config.c`, power lock. | Works with canned audio |
| 7 | app-ui | Circle UI states (§3.2), rotary scroll, back key, haptic feedback, runtime privacy permission (`ppm_request_permission`). | Full flow works without a laptop |
| 8 | launch-intent | UI app handles `app_control` `action=listen` (cold start + already running). | `app_launcher` test with extra data starts listening |
| 9 | watchface | Watch face: digital time/date/battery, mic button → launch `action=listen`, ambient mode. Added to the package. | Selectable face; tapping the mic starts listening |
| 10 | e2e-test | Wi-Fi / BT proxy / LTE × short/long questions × errors (airplane mode, bad key, silence) × all launch paths. Battery per 10 queries, plus watch-face battery impact over a day. | Checklist passes |
| 11 | docs | README (including the double-press Home setup) + signing.md. | Reproducible by someone else |

Dependencies: 1→2→3→4→6→7→8→9→10→11, and 2→5→7.

---

## 6. Reference commands
```bash
sdb connect 192.168.1.50:26101
sdb devices
sdb capability                      # platform version
sdb dlog -v time GEMWATCH:D *:S

tizen build-native -a arm -c gcc -C Debug -- watch
tizen package -t tpk -s <cert-profile> -- watch/Debug
tizen install -n <pkg>-1.0.0-arm.tpk -- watch/Debug

sdb push config.json /opt/usr/home/owner/apps_rw/<pkg>/data/config.json

# Laptop smoke test
B64=$(base64 -i samples/question.m4a | tr -d '\n')
curl -s "https://generativelanguage.googleapis.com/v1beta/models/$MODEL:generateContent" \
  -H "x-goog-api-key: $GEMINI_API_KEY" -H 'Content-Type: application/json' \
  -d "{\"contents\":[{\"parts\":[{\"text\":\"Answer briefly\"},{\"inlineData\":{\"mimeType\":\"audio/aac\",\"data\":\"$B64\"}}]}]}"
```

### Manifest privileges (draft)
```xml
<privileges>
  <privilege>http://tizen.org/privilege/internet</privilege>
  <privilege>http://tizen.org/privilege/network.get</privilege>
  <privilege>http://tizen.org/privilege/recorder</privilege>
  <privilege>http://tizen.org/privilege/mediastorage</privilege>
  <privilege>http://tizen.org/privilege/display</privilege>
  <privilege>http://tizen.org/privilege/haptic</privilege>
  <privilege>http://tizen.org/privilege/appmanager.launch</privilege> <!-- watchface -->
  <privilege>http://tizen.org/privilege/alarm.get</privilege>         <!-- watchface -->
</privileges>
```
`recorder` is a privacy privilege on Tizen 4. The app must request it at runtime via `privacy_privilege_manager`.

---

## 7. Risks and mitigations
| Risk | Likelihood | Mitigation |
|---|---|---|
| Samsung cert service retired / login fails | Medium | Test first (step 2). Look for community workarounds and older Tizen Studio versions. Without signing, the project stops. |
| TLS handshake fails (old curl/OpenSSL/CA) | Low–Med | Bundle the CA. If the cipher is unsupported, statically link a newer curl + mbedTLS. Last resort: a small relay. |
| AAC MIME type rejected | Low | Use WAV PCM 16kHz (10s ≈ 320KB, ~430KB base64). |
| Slow over the BT proxy | Medium | Flash model, short answers, short recordings, show `heard` early. |
| API key on the device | Low (personal) | Restrict the key; config is gitignored. |
| Model names / endpoints change | High over time | Configurable model and API version. |
| TTS lacks your language | Low | Fallback: Gemini TTS → PCM → `audio_out`. |
| Battery | Medium | No background listening. Release the power lock promptly. |
| Watch face drains the battery | Medium | Redraw once per minute, minimal ambient mode, no network on the face. |
| Watch face can't launch an app from a tap | Low | Fallback: double-press Home (widget in v2). |

---

## 8. v2 ideas
- Widget-board widget (mic button + last answer).
- Personal Google data via the OAuth device flow (RFC 8628) + Gemini function calling (Calendar, Tasks).
- Google Search grounding for up-to-date answers.
- Gemini Live (WebSocket) streaming conversation.
- On-watch settings screen.

---

## 9. Sources
- Samsung Galaxy Watch (Tizen) store notice: https://developer.samsung.com/galaxy-watch-tizen/notice.html
- Tizen SDK installer mirror: https://download.tizen.org/sdk/Installer/
- Google Assistant SDK overview: https://developers.google.com/assistant/sdk/overview
- Gemini API audio understanding: https://ai.google.dev/gemini-api/docs/audio
- Gemini Live API: https://ai.google.dev/gemini-api/docs/live-api

