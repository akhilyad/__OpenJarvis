# OpenJarvis — Changes Notepad

A running log of every change made, grouped by feature. User will request `/summary` at the end.

---

## Feature 1: Voice Loop (`jarvis listen`)

**Goal**: Full wake-word → STT → Agent → TTS → Playback pipeline. Jarvis always listening.

### New Files

| File | Purpose |
|------|---------|
| `src/openjarvis/voice/__init__.py` | Voice package init, lazy-imports sub-modules |
| `src/openjarvis/voice/audio_io.py` | `MicrophoneStream` (sounddevice generator) + `AudioPlayer` (soundfile/sounddevice playback) |
| `src/openjarvis/voice/vad.py` | `VoiceActivityDetector` — energy-based by default, optional webrtcvad upgrade |
| `src/openjarvis/voice/wake_word.py` | `WakeWordDetector` — keyword-in-transcription mode + optional openwakeword mode |
| `src/openjarvis/voice/tts_discovery.py` | `get_tts_backend()` — mirrors STT `_discovery.py`, priority: kokoro → openai_tts → cartesia |
| `src/openjarvis/voice/loop.py` | `VoiceLoop` — orchestrates mic → VAD → STT → agent → TTS → playback |
| `src/openjarvis/cli/listen_cmd.py` | `jarvis listen` CLI command with `--wake-word`, `--no-wake-word`, `--tts`, `--agent`, `--once` |

### Modified Files

| File | Change |
|------|--------|
| `src/openjarvis/core/config.py` | Extended `SpeechConfig` with 11 new voice-loop fields (tts_backend, tts_voice_id, tts_speed, wake_word, wake_word_engine, vad_engine, vad_aggressiveness, silence_timeout_ms, min_speech_ms, input_device, output_device) |
| `src/openjarvis/cli/__init__.py` | Imported and registered `listen` command |
| `pyproject.toml` | Added `voice` optional extra (sounddevice, soundfile, numpy); added `voice-wakeword` bundle |

### Architecture

```
Microphone (sounddevice)
  ↓
VAD — energy RMS or webrtcvad (collect speech frames, stop on silence)
  ↓
PCM → WAV (stdlib wave module)
  ↓
STT — FasterWhisper / OpenAI Whisper / Deepgram (existing backends)
  ↓
Wake Word Check — "jarvis" in transcript (keyword mode) or openwakeword model
  ↓
Agent.run(command_text) — any registered agent (simple, orchestrator, react…)
  ↓
TTS — Kokoro (local) / Cartesia / OpenAI TTS (existing backends)
  ↓
Playback (soundfile decode → sounddevice play)
  ↓
Loop back ↑
```

---

## Feature 3: Personal Context Layer

**Goal**: Replace the flat USER.md bullet list with a structured, living personal profile that Jarvis always knows — identity, contacts, projects, preferences.

### New Files

| File | Purpose |
|------|---------|
| `src/openjarvis/profile/__init__.py` | Profile package init |
| `src/openjarvis/profile/store.py` | `ProfileStore` — section-aware read/write for USER.md |
| `src/openjarvis/cli/profile_cmd.py` | `jarvis profile` CLI command group |

### Modified Files

| File | Change |
|------|--------|
| `src/openjarvis/tools/user_profile_manage.py` | Enhanced with section-aware actions: `read_section`, `set_field`, `add` (to specific section), `remove` (pattern match). Legacy `read`/`update`/`remove` kept for backwards compatibility. |
| `src/openjarvis/cli/__init__.py` | Registered `profile` command |
| `src/openjarvis/cli/init_cmd.py` | `jarvis init` now calls `ProfileStore.ensure_template()` to write a structured USER.md instead of a blank file |

### Architecture

```
~/.openjarvis/USER.md  (structured Markdown)
       |
  ProfileStore
  ├── set_field(section, field, value)   -- "- Name: Akhil Yadav"
  ├── add_item(section, text)            -- append bullet
  ├── remove_item(section, pattern)      -- remove matching bullet
  ├── get_identity() -> dict             -- parse Identity key-values
  ├── render_summary(max_chars)          -- compact for voice/tokens
  └── render_full()                      -- complete markdown
       |
  Injected into system prompt via existing SystemPromptBuilder (unchanged)
       |
  Agents can read/write via user_profile_manage tool (section-aware)
```

### Profile Format (USER.md)

```markdown
# User Profile

## Identity
- Name: Akhil Yadav
- Timezone: Asia/Kolkata
- Preferred address: sir
- Role: Software Engineer

## Contacts
- Alice (boss): Weekly 1:1 Mondays. alice@corp.com
- Dev team (colleagues): Daily standup 9am IST

## Active Projects
- OpenJarvis [active]: Building open-source AI assistant
- Client Dashboard [review]: Pending sign-off by 2026-04-20

## Preferences
- Always schedule meetings after 10am
- Never send emails without explicit confirmation

## Notes
- Works from home on Mondays and Fridays
```

### CLI Commands

```bash
jarvis profile import                          # interactive first-run wizard
jarvis profile show                            # display current profile
jarvis profile edit                            # open USER.md in $EDITOR
jarvis profile set name "Akhil Yadav"          # set an Identity field
jarvis profile prefer "never send without confirmation"
jarvis profile note "working from home today"
jarvis profile contact add "Alice" --role boss --note "1:1 Mondays"
jarvis profile contact remove "Alice"
jarvis profile project add "OpenJarvis" --status active --desc "AI assistant"
jarvis profile project update "OpenJarvis" --status completed
```

### Key Design Decisions

1. **Markdown as source of truth** — injected by `SystemPromptBuilder` unchanged; no schema migration needed.
2. **Section-aware parser** — `ProfileStore._parse()` reads h2 headers as sections, preserves all other content exactly.
3. **Backwards compatible tool** — `user_profile_manage` tool's old `read`/`update`/`remove` actions still work unchanged.
4. **Interactive wizard** — `jarvis profile import` is the "Tony meets Jarvis for the first time" moment.
5. **Honorific propagation** — `import` wizard patches `SOUL.md` if it contains a `{honorific}` placeholder, so the Jarvis persona uses your preferred address immediately.

---

## Feature 2: Event-Driven Operators

**Goal**: Operators that react to real-world events (file changes, system alerts, URL changes, bus events) instead of only cron/interval schedules.

### New Files

| File | Purpose |
|------|---------|
| `src/openjarvis/operators/watchers.py` | 4 watcher classes: `FileWatcher`, `SystemMetricWatcher`, `HttpPollWatcher`, `BusEventWatcher` |
| `src/openjarvis/operators/event_engine.py` | `EventTriggerEngine` — manages watchers for one operator, fires tick callback on event |
| `configs/openjarvis/examples/event-driven-operator.toml` | Example manifest showing all 4 trigger types |

### Modified Files

| File | Change |
|------|--------|
| `src/openjarvis/operators/types.py` | Added `event_triggers: List[Dict[str, Any]]` field to `OperatorManifest` |
| `src/openjarvis/operators/loader.py` | Parse `[[operator.event_triggers]]` TOML array-of-tables into manifest |
| `src/openjarvis/operators/manager.py` | Added `_event_engines` dict; `activate()` starts engine if triggers present; `deactivate()` stops engine; `_start_event_engine()` / `_stop_event_engine()` helpers; tick callback calls `system.ask(prompt, agent="operative", operator_id=oid)` |
| `src/openjarvis/core/events.py` | Added `OPERATOR_EVENT_FIRED = "operator_event_fired"` to `EventType` enum |
| `pyproject.toml` | Added `operators-events` optional extra (psutil, watchdog) |

### Architecture

```
OperatorManifest.event_triggers (list of dicts)
  -> OperatorManager.activate() -> EventTriggerEngine.start()
       -> FileWatcher (watchdog or polling fallback)
       -> SystemMetricWatcher (psutil)
       -> HttpPollWatcher (httpx, hash-based change detection)
       -> BusEventWatcher (EventBus subscription)

When event fires:
  -> EventTriggerEngine._on_event_fired()
     -> EventBus.publish(OPERATOR_EVENT_FIRED)
     -> tick_callback(operator_id, enriched_prompt)
        -> system.ask(prompt, agent="operative", operator_id=oid)
```

### TOML Manifest Syntax

```toml
[[operator.event_triggers]]
type             = "file"
path             = "~/Documents/inbox"
pattern          = "*.pdf"
events           = ["created"]
check_interval_s = 5
cooldown_s       = 60

[[operator.event_triggers]]
type      = "system_metric"
metric    = "cpu_percent"      # cpu_percent | memory_percent | disk_percent
threshold = 85.0
operator  = ">"                # > | < | >= | <=
check_interval_s = 30

[[operator.event_triggers]]
type           = "http_poll"
url            = "https://example.com/feed"
fire_on_change = true
check_interval_s = 300

[[operator.event_triggers]]
type         = "bus_event"
event_type   = "channel_message_received"
filter_key   = "channel"       # optional field filter
filter_value = "telegram"
```

### Key Design Decisions

1. **Cooldown per trigger** — prevents rapid-fire repeated activations (configurable `cooldown_s`).
2. **watchdog optional** — FileWatcher falls back to `os.scandir` polling; watchdog is in new `operators-events` extra.
3. **psutil optional** — SystemMetricWatcher silently skips if psutil not installed.
4. **Backwards compatible** — existing operators with no `event_triggers` are completely unaffected; all 52 operator tests pass.
5. **Event context injected into prompt** — when an event fires, the tick prompt includes the event description and data so the agent knows what triggered it.

---

---

## Feature 4: Screen Awareness

**Goal**: Let Jarvis "see" the user's screen — capture a screenshot (full screen or region), optionally OCR text from it, and inject the visual context into any query. Real Jarvis sees the displays in the lab.

### New Files

| File | Purpose |
|------|---------|
| `src/openjarvis/tools/screen_capture.py` | `ScreenCaptureTool` registered as `"screen_capture"` — capture screenshot, optional OCR, return base64 PNG |

### Modified Files

| File | Change |
|------|--------|
| `src/openjarvis/cli/ask.py` | Added `--screenshot`, `--screenshot-ocr`, `--screenshot-region` flags; `_prepend_screen_context()` helper |
| `src/openjarvis/cli/listen_cmd.py` | Added `--screenshot`, `--screenshot-ocr` flags; passes to `VoiceLoop` |
| `src/openjarvis/voice/loop.py` | Added `screenshot_context` / `screenshot_ocr` params; prepends screen context before agent call |
| `pyproject.toml` | Added `screen` extra (mss + Pillow) and `screen-ocr` extra (+ pytesseract) |

### Architecture

```
jarvis ask "what is on my screen?" --screenshot --screenshot-ocr
  |
  +--> ScreenCaptureTool.execute(ocr=True)
         |
         +--> mss.mss().grab()  (primary, fast, multi-monitor)
         |    OR PIL.ImageGrab.grab()  (fallback, Windows/macOS)
         |
         +--> _maybe_resize(max_width=1280)  -- keeps token cost reasonable
         |
         +--> pytesseract / easyocr  (optional OCR)
         |
         +--> base64-encoded PNG  +  OCR text
  |
  +--> _prepend_screen_context()
         |
         +--> [SCREEN CONTEXT]
              Screenshot: WxH px
              Visible text: <ocr_text>
              Image: data:image/png;base64,...
              [END SCREEN CONTEXT]
              <original query>
  |
  +--> Engine / Agent  (multimodal models consume the image directly)
```

### Key Design Decisions

1. **mss as primary backend** — cross-platform, fast, no GUI dependency, multi-monitor support. Pillow ImageGrab as fallback (Windows/macOS only, slower).
2. **Auto-resize to 1280px wide** — reduces base64 token cost by ~4x for typical 2560px displays. Configurable via `max_width` parameter.
3. **OCR is optional** — enables text extraction without multimodal LLM. Supports pytesseract (fast, requires Tesseract binary) and easyocr (pure-Python, slower).
4. **Structured `[SCREEN CONTEXT]` block** — clearly delimited so models know what is visual context vs. the actual query. Works with all models, not just multimodal.
5. **Region capture support** — `--screenshot-region left,top,width,height` or via `region` parameter, useful for capturing just one window or monitor area.
6. **Voice loop integration** — `jarvis listen --screenshot` captures the screen at each voice command cycle, giving Jarvis persistent visual awareness during a session.
7. **Tool registry** — `screen_capture` is a proper `ToolRegistry` tool, so agents can call it autonomously: `"Use screen_capture to see what the user is working on."`.

### CLI Examples

```bash
# Ask about what's on screen
jarvis ask "what application is open?" --screenshot

# Extract and describe text on screen
jarvis ask "summarize what's on my screen" --screenshot --screenshot-ocr

# Capture only a region (left monitor)
jarvis ask "what does this error say?" --screenshot --screenshot-region 0,0,1920,1080

# Voice loop with persistent screen awareness
jarvis listen --screenshot --screenshot-ocr

# Save screenshot to file
# (via the tool directly in an agent session)
# tool: screen_capture, params: {output_path: "~/Desktop/snap.png", ocr: true}
```

---

### Key Design Decisions (Voice Loop)

1. **Energy-based VAD by default** — no C extension required, works cross-platform (Windows 11 compatible). Optional `webrtcvad` upgrade via `voice-vad` extra.
2. **Keyword-in-transcription wake word** — transcribe every utterance, check if "jarvis" appears in it, extract command after the keyword. No extra model download needed.
3. **WAV format requested from TTS** — avoids mp3 decoding. All backends (Kokoro, Cartesia, OpenAI) can produce WAV or raw PCM.
4. **Graceful degradation** — missing STT/TTS produces a clear install hint; Ctrl-C exits cleanly; broken utterances are silently skipped.
5. **`--no-wake-word` flag** — every detected speech goes straight to agent (good for demos / private use).
6. **`--once` flag** — listen for one utterance, respond, exit (useful for shell scripting / cron).
7. **`--speak` / `--no-speak`** — optionally disable TTS output (just print response to terminal).
