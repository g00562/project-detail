# HIK — Project Plan & Architecture

---

## 1. Project Overview

**Project Title:** HIK — Voice-Based Virtual Assistant with Hybrid Online/Offline Architecture

**Objective:** Build a cross-platform (macOS + Windows), privacy-first voice assistant that works **with or without internet** — matching the capabilities of commercial assistants like Siri, while requiring **zero API keys** and **zero cost**.

**Key Principles:**
- 🔒 **Privacy First** — All processing happens locally on the user's machine
- 🔌 **Zero API Keys** — No accounts, no subscriptions, no cloud dependency
- 🌐 **Hybrid Mode** — Auto-switches between online and offline seamlessly
- 💻 **Cross-Platform** — Identical experience on macOS and Windows
- 🧩 **Modular & Extensible** — Plugin-based skill system for easy expansion
- 🛡️ **Error Resilient** — Multi-level fallback chains at every layer

---

## 2. System Architecture

### 2.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         USER LAYER                              │
│         Voice Input │ Keyboard Shortcut │ Mouse Click            │
├─────────────────────────────────────────────────────────────────┤
│                      PRESENTATION LAYER                         │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │              Electron GUI (Transparent Overlay)          │   │
│   │   Animated Orb │ Waveform │ Chat │ Settings │ Tray      │   │
│   └──────────────────────┬──────────────────────────────────┘   │
│                          │ WebSocket (ws://127.0.0.1:8765)      │
│   ┌──────────────────────┴──────────────────────────────────┐   │
│   │              GUI Bridge (Python WebSocket Server)        │   │
│   └──────────────────────┬──────────────────────────────────┘   │
├──────────────────────────┼──────────────────────────────────────┤
│                      APPLICATION LAYER                          │
│   ┌──────────────────────┴──────────────────────────────────┐   │
│   │              main.py — Master Orchestrator               │   │
│   │     Thread Bootstrapper │ Signal Handler │ Lifecycle     │   │
│   └──┬────────┬────────────┬────────────┬───────────────────┘   │
│      │        │            │            │                       │
│   ┌──┴──┐  ┌──┴──┐    ┌───┴───┐   ┌────┴────┐                 │
│   │AUDIO│  │BRAIN│    │SKILLS │   │UTILITIES│                  │
│   │     │  │     │    │       │   │         │                  │
│   │ Mic │  │Regex│    │6 Built│   │ Logger  │                  │
│   │ VAD │  │Intnt│    │  -in  │   │ Cache   │                  │
│   │ Wake│  │Contx│    │Plugins│   │ Network │                  │
│   │Barge│  │Memry│    │       │   │ Health  │                  │
│   │Chime│  │Dialg│    │       │   │ Error   │                  │
│   └──┬──┘  └──┬──┘    └───┬───┘   └────┬────┘                 │
├──────┼────────┼────────────┼────────────┼──────────────────────┤
│      │    PROCESSING LAYER │            │                      │
│   ┌──┴──────────┐  ┌──────┴──────┐  ┌──┴───────────┐          │
│   │  STT Manager │  │ TTS Manager │  │Platform Layer│          │
│   │  Whisper│Vosk│  │Piper│gTTS  │  │ macOS│Windows│          │
│   └─────────────┘  └────────────┘  └──────────────┘           │
├─────────────────────────────────────────────────────────────────┤
│                        DATA LAYER                               │
│   SQLite (Calendar) │ ChromaDB (Memory) │ WAV Cache (TTS)      │
│   YAML (Config)     │ .env (Secrets)     │ Logs (Rotating)     │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 Data Flow Pipeline

```
┌────────┐    ┌──────────┐    ┌─────────┐    ┌──────────┐    ┌─────────┐
│  User  │───▶│   Mic    │───▶│  Wake   │───▶│   VAD    │───▶│   STT   │
│ Speaks │    │ Capture  │    │  Word   │    │ Silence  │    │Whisper/ │
│        │    │ 16kHz    │    │Detection│    │Detection │    │  Vosk   │
└────────┘    └──────────┘    └─────────┘    └──────────┘    └────┬────┘
                                                                  │
                                                            Text  │
                                                                  ▼
┌────────┐    ┌──────────┐    ┌─────────┐    ┌──────────┐    ┌─────────┐
│Speaker │◀───│   TTS    │◀───│Response │◀───│  Skill   │◀───│ Intent  │
│Playback│    │Piper/gTTS│    │  Text   │    │ Handler  │    │Classify │
│        │    │/System   │    │         │    │          │    │ (Regex) │
└────────┘    └──────────┘    └─────────┘    └──────────┘    └─────────┘
     │
     │ Audio playing
     ▼
┌──────────┐
│ Barge-In │───▶ If user speaks → Interrupt TTS → Re-enter Listening
│ Detector │
└──────────┘
```

### 2.3 State Machine

```
                    ┌─────────────────────────────┐
                    │                             │
                    ▼                             │
            ┌──────────────┐                      │
     ┌─────▶│   SLEEPING   │◀──── "bye" ─────────┤
     │      │  (Idle/Pulse) │                      │
     │      └──────┬───────┘                      │
     │             │                              │
     │     Wake Word / Click / Shortcut           │
     │             │                              │
     │      ┌──────▼───────┐                      │
     │      │WAKE_DETECTED │                      │
     │      │  (Chime ♪)   │                      │
     │      └──────┬───────┘                      │
     │             │                              │
     │      ┌──────▼───────┐                      │
     │      │  LISTENING   │◀─── Barge-In ──┐     │
     │      │ (Waveform)   │                │     │
     │      └──────┬───────┘                │     │
     │             │                        │     │
     │      5s silence / speech end         │     │
     │             │                        │     │
     │      ┌──────▼───────┐                │     │
     │      │ PROCESSING   │                │     │
     │      │ (Spinning)   │                │     │
     │      │ STT→Intent→  │                │     │
     │      │ Skill→Response│               │     │
     │      └──────┬───────┘                │     │
     │             │                        │     │
     │      ┌──────▼───────┐                │     │
     │      │  SPEAKING    │────────────────┘     │
     │      │ (Breathing)  │                      │
     │      │   TTS plays  │──── follow-up ──▶ LISTENING
     │      └──────┬───────┘                      │
     │             │                              │
     │       No follow-up                         │
     │             │                              │
     │      ┌──────▼───────┐                      │
     │      │   SLEEPING   │──────────────────────┘
     │      └──────────────┘
     │
     │      ┌──────────────┐
     └──────│    ERROR     │
            │ (Red Flash)  │
            └──────────────┘
```

**7 GUI States:** Sleeping → Wake → Listening → Processing → Speaking → Error → Offline

---

## 3. Module Breakdown

### 3.1 Audio Pipeline (`core/audio/` + `core/pipeline.py`)

| File | Lines | Purpose |
|------|-------|---------|
| `microphone.py` | ~270 | Thread-safe mic capture via `sounddevice`, 16kHz mono int16 |
| `wake_word.py` | ~166 | OpenWakeWord "Hey Jarvis" detection with sensitivity tuning |
| `barge_in.py` | ~85 | Energy-based interruption detector during TTS playback |
| `audio_utils.py` | ~200 | Chime generation, RMS, WAV I/O, resampling, format conversion |
| `pipeline.py` | ~446 | 6-state FSM orchestrator, VAD, state transitions |

**Design Decisions:**
- Queue-based audio buffering (producer-consumer pattern)
- VAD: Silero neural VAD (primary) → energy-based (fallback if no `torch`)
- Chimes synthesized via NumPy sine waves — no external audio files needed

---

### 3.2 Speech-to-Text (`core/stt/`)

| File | Lines | Purpose |
|------|-------|---------|
| `engine.py` | ~30 | Abstract base class: `load_model()`, `transcribe()`, `unload_model()` |
| `whisper_stt.py` | ~100 | OpenAI Whisper (Small, ~500MB) — high accuracy offline |
| `vosk_stt.py` | ~120 | Vosk (~40MB) — lightweight offline fallback |
| `__init__.py` | ~125 | STTManager: Whisper → Vosk auto-failover chain |

**Fallback Chain:**
```
Whisper (95% accuracy, 500MB) → Vosk (85% accuracy, 40MB) → STTError
```

---

### 3.3 Text-to-Speech (`core/tts/`)

| File | Lines | Purpose |
|------|-------|---------|
| `engine.py` | ~67 | Abstract base class: `synthesize()`, `get_sample_rate()` |
| `piper_tts.py` | ~136 | ONNX-based offline TTS, natural English voice |
| `gtts_tts.py` | ~137 | Google Translate TTS (free, no API key, needs internet) |
| `__init__.py` | ~240 | TTSManager: 4-level fallback + disk cache |

**Fallback Chain:**
```
Piper (offline, natural) → gTTS (online, free) → System say/SAPI → GUI text-only
```

**TTS Cache:** SHA256-keyed WAV files on disk, LRU eviction at 500MB.

---

### 3.4 Brain — NLU Engine (`core/brain/`)

| File | Lines | Purpose |
|------|-------|---------|
| `llm_client.py` | ~105 | Regex-based response engine (time, date, greetings, jokes, help) |
| `intent_classifier.py` | ~175 | Pre-compiled regex patterns → 9 intents with entity extraction |
| `context.py` | ~96 | Sliding-window context manager (10 messages) |
| `memory.py` | ~156 | ChromaDB vector store for long-term conversation memory |
| `dialog_manager.py` | ~253 | Central orchestrator: intent → skill → response |

**Intent Classification (Regex, 9 intents):**

| Intent | Example Triggers |
|--------|-----------------|
| `SYSTEM_CONTROL` | "open Chrome", "set volume to 50", "take screenshot" |
| `WEB_SEARCH` | "search for Python", "what is AI", "who is Elon Musk" |
| `CALENDAR` | "set timer", "remind me", "what's my schedule" |
| `MEDIA_PLAYER` | "play music", "next song", "pause" |
| `NEWS_WEATHER` | "weather", "latest news", "temperature" |
| `TRANSLATION` | "translate hello to Spanish" |
| `CANCEL` | "cancel", "stop" |
| `BYE` | "bye", "goodbye" |
| `GENERAL_CHAT` | Everything else → regex response engine |

---

### 3.5 Skills (`skills/`)

| File | Skill | Offline | Key Technology |
|------|-------|---------|----------------|
| `system_control.py` | System Control | ✅ | AppleScript (Mac) / PowerShell (Win) |
| `web_search.py` | Web Search | ❌ | DuckDuckGo (free, no key) |
| `calendar_skill.py` | Calendar & Timers | ✅ | SQLite + `threading.Timer` |
| `media_player.py` | Media Player | ✅ | OS media keys / Music.app |
| `news_weather.py` | News & Weather | ❌ | Open-Meteo + DuckDuckGo (free) |
| `translation.py` | Translation | ✅ | Argos Translate (16 languages) |
| `base_skill.py` | Base Class | — | `safe_execute()` error isolation |
| `skill_loader.py` | Plugin Loader | — | `importlib` dynamic loading |

**Plugin System:** Drop a `.py` file in `skills/custom/` → auto-discovered at startup.

---

### 3.6 GUI (`gui/`)

| File | Lines | Purpose |
|------|-------|---------|
| `bridge.py` | ~197 | Python WebSocket server (asyncio, websockets v16) |
| `main.js` | ~235 | Electron main process — window, tray, shortcuts, WS client |
| `preload.js` | ~28 | Context bridge — safe IPC between main and renderer |
| `renderer.js` | ~376 | UI state machine, orb animations, canvas waveform, chat |
| `index.html` | ~100 | Layout: orb container, text pill, chat history, settings |
| `styles.css` | ~500 | Glassmorphism, keyframe animations, dark theme |

**Communication:**
```
Python Backend ←──WebSocket JSON──→ Electron Frontend
     (port 8765, 127.0.0.1)
```

**Message Types:**
| Direction | Type | Data |
|-----------|------|------|
| Backend → Frontend | `state_change` | `{state: "listening"}` |
| Backend → Frontend | `transcription` | `{text: "hello"}` |
| Backend → Frontend | `response` | `{text: "Hi!", user_text: "hello"}` |
| Backend → Frontend | `error` | `{message: "STT failed"}` |
| Frontend → Backend | `trigger_wake` | `{}` |
| Frontend → Backend | `setting` | `{key: "tts_speed", value: 1.2}` |

---

### 3.7 Utilities (`utils/`)

| File | Lines | Purpose |
|------|-------|---------|
| `logger.py` | ~110 | Rotating file + console logger, colored output |
| `helpers.py` | ~90 | YAML config loader, `get_config()` dot-path accessor |
| `secrets.py` | ~140 | OS keychain (Keyring) + `.env` fallback |
| `cache.py` | ~120 | LRU response cache, SHA256 keys, persistent JSON |
| `network.py` | ~187 | Background connectivity checker, force modes |
| `health.py` | ~123 | RAM monitor, auto model unloading, GC trigger |
| `error_handler.py` | ~137 | Custom exceptions, safe error handling |

---

### 3.8 Platform Abstraction (`core/platform/`)

| File | macOS | Windows |
|------|-------|---------|
| `mac.py` | `open -a`, AppleScript, `pmset`, `screencapture`, `say` | — |
| `windows.py` | — | `Start-Process`, PowerShell, pycaw, WMI, SAPI |
| `__init__.py` | Auto-detects OS → exports correct module | Same |

---

## 4. Technology Stack

| Layer | Technology | Purpose | Cost |
|-------|-----------|---------|------|
| **Language** | Python 3.11 | Backend logic | Free |
| **GUI Framework** | Electron + Node.js | Transparent overlay window | Free |
| **Mic Capture** | sounddevice | Cross-platform audio input | Free |
| **Wake Word** | OpenWakeWord | "Hey Jarvis" detection | Free |
| **STT Primary** | Whisper (openai-whisper) | High-accuracy transcription | Free |
| **STT Fallback** | Vosk | Lightweight transcription | Free |
| **TTS Primary** | Piper TTS | Natural offline speech | Free |
| **TTS Fallback** | gTTS | Google Translate TTS | Free |
| **Response Engine** | Regex Patterns | Local pattern matching | Free |
| **Memory** | ChromaDB | Vector similarity search | Free |
| **Database** | SQLite | Calendar, timers, reminders | Free |
| **Web Search** | DuckDuckGo | No-key search API | Free |
| **Weather** | Open-Meteo | No-key weather API | Free |
| **Translation** | Argos Translate | Offline, 16 languages | Free |
| **IPC** | WebSocket (ws) | Python ↔ Electron comm | Free |
| **API Server** | FastAPI + Uvicorn | REST API (optional) | Free |
| **Health** | psutil | RAM/CPU monitoring | Free |
| **Testing** | pytest + pytest-asyncio | Unit tests | Free |
| **Build** | PyInstaller | Standalone binary | Free |

**Total Cost: $0.00 — Everything is free and open source.**

---

## 5. Online vs Offline Modes

```
┌──────────────────────────────────────────────────────────────┐
│                    ONLINE MODE                                │
│                                                               │
│  STT:     Whisper → Vosk              (both offline anyway)  │
│  TTS:     Piper → gTTS → System      (gTTS needs internet)  │
│  Brain:   Regex patterns              (always offline)        │
│  Skills:  ALL 6 active                                       │
│  Search:  DuckDuckGo ✅                                       │
│  Weather: Open-Meteo ✅                                       │
│  News:    DuckDuckGo ✅                                       │
│                                                               │
│  GUI Orb: Purple glow                                        │
├──────────────────────────────────────────────────────────────┤
│                    OFFLINE MODE                               │
│                                                               │
│  STT:     Whisper → Vosk              (fully offline)        │
│  TTS:     Piper → System say/SAPI     (fully offline)        │
│  Brain:   Regex patterns              (fully offline)        │
│  Skills:  4 of 6 active                                      │
│           ✅ System Control                                   │
│           ✅ Calendar / Timers                                │
│           ✅ Media Player                                     │
│           ✅ Translation (Argos)                              │
│           ❌ Web Search (needs internet)                      │
│           ❌ News/Weather (needs internet)                    │
│                                                               │
│  GUI Orb: Amber glow                                         │
└──────────────────────────────────────────────────────────────┘

Auto-switch: NetworkMonitor checks connectivity every 30 seconds.
```

---

## 6. Project File Structure

```
HIK/
│
├── main.py                          # Entry point — thread bootstrapper
│
├── config/
│   ├── settings.yaml                # All configuration (audio, STT, TTS, GUI, skills)
│   ├── prompts.yaml                 # System prompts, error messages, offline responses
│   └── .env.example                 # Template (no API keys needed)
│
├── core/
│   ├── __init__.py
│   ├── pipeline.py                  # 6-state audio FSM
│   │
│   ├── audio/
│   │   ├── __init__.py
│   │   ├── microphone.py            # Mic capture (sounddevice)
│   │   ├── wake_word.py             # "Hey HIK" (OpenWakeWord)
│   │   ├── barge_in.py              # Voice interruption detector
│   │   └── audio_utils.py           # Chimes, WAV I/O, RMS
│   │
│   ├── stt/
│   │   ├── __init__.py              # STT Manager (failover chain)
│   │   ├── engine.py                # Abstract STT interface
│   │   ├── whisper_stt.py           # Whisper offline STT
│   │   └── vosk_stt.py              # Vosk lightweight STT
│   │
│   ├── tts/
│   │   ├── __init__.py              # TTS Manager (4-level fallback + cache)
│   │   ├── engine.py                # Abstract TTS interface
│   │   ├── piper_tts.py             # Piper offline TTS
│   │   └── gtts_tts.py              # Google Translate TTS (free)
│   │
│   ├── brain/
│   │   ├── __init__.py
│   │   ├── llm_client.py            # Regex response engine
│   │   ├── intent_classifier.py     # Regex intent detection (9 intents)
│   │   ├── context.py               # Sliding window (10 messages)
│   │   ├── memory.py                # ChromaDB vector memory
│   │   └── dialog_manager.py        # Central conversation orchestrator
│   │
│   └── platform/
│       ├── __init__.py              # OS auto-detection
│       ├── mac.py                   # macOS APIs (AppleScript)
│       └── windows.py               # Windows APIs (PowerShell)
│
├── skills/
│   ├── __init__.py                  # Skill wiring function
│   ├── base_skill.py                # Abstract skill with safe_execute
│   ├── skill_loader.py              # Dynamic plugin loader
│   ├── system_control.py            # Apps, volume, brightness, screenshot
│   ├── calendar_skill.py            # Timers, reminders, events (SQLite)
│   ├── media_player.py              # Play, pause, skip (OS media keys)
│   ├── web_search.py                # DuckDuckGo (free, no key)
│   ├── news_weather.py              # Open-Meteo + DDG News (free)
│   ├── translation.py               # Argos Translate (offline)
│   └── custom/                      # User plugin directory
│
├── gui/
│   ├── bridge.py                    # Python WebSocket server
│   └── frontend/
│       ├── main.js                  # Electron main process
│       ├── preload.js               # IPC security bridge
│       ├── renderer.js              # UI state machine + animations
│       ├── index.html               # Layout
│       ├── styles.css               # Glassmorphism theme
│       └── package.json             # Node.js dependencies
│
├── utils/
│   ├── __init__.py
│   ├── logger.py                    # Rotating file + console logging
│   ├── helpers.py                   # Config loader, path utilities
│   ├── secrets.py                   # Keychain + .env secret manager
│   ├── cache.py                     # LRU response cache
│   ├── network.py                   # Online/offline monitor
│   ├── health.py                    # RAM monitor, auto model unloading
│   └── error_handler.py             # Custom exceptions
│
├── tests/
│   ├── conftest.py
│   └── unit/                        # 7 test files
│
├── scripts/
│   ├── setup.sh / setup.bat         # Environment bootstrap
│   ├── build.sh                     # Distribution build
│   ├── run.py                       # Dev launcher
│   └── echo_test.py                 # Audio diagnostic
│
├── data/                            # SQLite databases (calendar.db)
├── cache/                           # TTS audio cache
├── chroma_data/                     # ChromaDB vector store
├── logs/                            # Rotating log files
├── models/                          # Downloaded AI models
├── requirements.txt                 # Python dependencies (46 packages)
├── pyproject.toml                   # Project metadata
└── README.md                        # Documentation
```

---

## 7. Development Phases

| Phase | Duration | Deliverables |
|-------|----------|-------------|
| **Phase 1** — Foundation | Week 1-2 | Project setup, config system, logger, secrets, health monitor |
| **Phase 2** — Audio Pipeline | Week 3 | Mic capture, wake word, VAD, chimes, echo test |
| **Phase 3** — STT | Week 4 | Whisper + Vosk with auto-failover |
| **Phase 4** — TTS | Week 5 | Piper + gTTS + system fallback + disk cache |
| **Phase 5** — Brain | Week 6 | Regex engine, intent classifier, context, ChromaDB memory |
| **Phase 6** — Skills | Week 7-8 | 6 skills + plugin framework + platform abstraction |
| **Phase 7** — GUI | Week 9 | Electron overlay, WebSocket bridge, animations |
| **Phase 8** — Integration | Week 10 | State machine, barge-in, multi-turn, main orchestrator |
| **Phase 9** — Testing | Week 11 | 7 unit test files, build scripts, deployment |
| **Phase 10** — QA | Week 12 | Bug audit, fixes, documentation, final polish |

---

## 8. Error Handling Strategy

Every critical layer has a multi-level fallback chain:

```
STT:     Whisper ──▶ Vosk ──▶ STTError (user message)
TTS:     Piper ──▶ gTTS ──▶ System say ──▶ GUI text-only
Brain:   Regex match ──▶ Generic "I can help with..." message
Skills:  safe_execute() ──▶ User-friendly error message
Network: Online ──▶ Offline (auto-switch, seamless)
Memory:  RAM > 3GB ──▶ Auto-unload models ──▶ GC ──▶ Self-terminate
```

**The assistant never crashes silently.** Every error produces a spoken or displayed message.

---

## 9. Key Metrics

| Metric | Value |
|--------|-------|
| Total Source Files | 35+ |
| Total Lines of Code | ~6,000+ |
| Built-in Skills | 6 |
| Unit Test Files | 7 |
| STT Engines | 2 |
| TTS Engines | 4 (with fallbacks) |
| Intent Types | 9 |
| GUI States | 7 |
| API Keys Required | **0** |
| Total Cost | **$0.00** |
| Platforms | macOS + Windows |
| Features | **80** |
