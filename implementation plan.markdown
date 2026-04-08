# Voice Based Virtual Assistant — Implementation Plan

> **Constraints**: Zero-cost APIs only · Cross-platform (Mac + Windows) · M1 8GB RAM · Siri-like UI · Online + Offline · Error handling

---

## Technology Stack (All Free)

| Layer | Technology | Cost | RAM Usage | Cross-Platform |
|---|---|---|---|---|
| Language | **Python 3.11+** | Free | — | ✅ Mac + Win |
| Wake Word | **OpenWakeWord** | Free, open-source | ~50 MB | ✅ Mac + Win |
| STT (Primary) | **Whisper.cpp** (small model) | Free, open-source | ~850 MB | ✅ Mac + Win |
| STT (Fallback) | **Vosk** (small model) | Free, open-source | ~50 MB | ✅ Mac + Win |
| TTS (Offline) | **Piper TTS** | Free, open-source | ~100 MB | ✅ Mac + Win |
| TTS (Fallback) | **gTTS** (Google Translate TTS) | Free, no API key | ~5 MB | ✅ Mac + Win |
| LLM (Primary) | **Gemini 2.5 Flash** (free tier) | Free — 10 RPM, 250 RPD | ~0 (cloud) | ✅ API-based |
| LLM (Fallback) | **Groq** (free tier — Llama 3.3 70B) | Free — 1,000 RPD | ~0 (cloud) | ✅ API-based |
| NLU / Intent | Hybrid (LLM + regex fallback) | Free | ~10 MB | ✅ Mac + Win |
| Memory | **ChromaDB** | Free, open-source | ~200 MB | ✅ Mac + Win |
| Image Gen | **Gemini image generation** (free tier) | Free | ~0 (cloud) | ✅ API-based |
| Web Search | **DuckDuckGo** (ddgs library) | Free, no API key | ~5 MB | ✅ Mac + Win |
| Weather | **Open-Meteo API** | Free, no API key | ~0 (cloud) | ✅ API-based |
| News | **NewsAPI.org** (free tier — 100 req/day) | Free | ~0 (cloud) | ✅ API-based |
| Translation | **Argos Translate** | Free, open-source | ~200 MB/lang | ✅ Mac + Win |
| GUI | **Electron + React** | Free | ~100 MB | ✅ Mac + Win |
| Task Queue | **asyncio** (Python built-in) | Free | ~0 | ✅ Mac + Win |
| Config | YAML + `.env` | Free | — | ✅ Mac + Win |
| Testing | pytest + unittest | Free | — | ✅ Mac + Win |

---

## Free API Keys Required

| Service | How to Get | Daily Limit |
|---|---|---|
| **Gemini** (Google AI Studio) | [aistudio.google.com](https://aistudio.google.com) → Create API Key | 250 RPD (Flash) |
| **Groq** (fallback LLM) | [console.groq.com](https://console.groq.com) → Create API Key | 1,000 RPD |
| **NewsAPI** | [newsapi.org](https://newsapi.org) → Register | 100 RPD |

> All other services (Whisper, Piper, Vosk, DuckDuckGo, Open-Meteo, Argos, ChromaDB) require **no API keys**.

---

## Project Directory Structure

```
VoiceAssistant/
├── config/
│   ├── settings.yaml           # Master config (wake word, model sizes, timeouts)
│   ├── prompts.yaml            # System prompts for LLM persona
│   └── .env.example            # API key template (Gemini, Groq — all free)
├── core/
│   ├── __init__.py
│   ├── audio/
│   │   ├── microphone.py       # Mic capture & VAD (sounddevice + Silero)
│   │   ├── wake_word.py        # OpenWakeWord always-listening engine
│   │   ├── barge_in.py         # Interrupt-while-speaking detection
│   │   └── audio_utils.py      # Format conversion, resampling
│   ├── stt/
│   │   ├── engine.py           # Abstract STT interface
│   │   ├── whisper_stt.py      # Whisper.cpp offline primary
│   │   └── vosk_stt.py         # Vosk offline fallback
│   ├── tts/
│   │   ├── engine.py           # Abstract TTS interface
│   │   ├── piper_tts.py        # Piper offline primary
│   │   └── gtts_tts.py         # gTTS online fallback
│   ├── brain/
│   │   ├── intent_classifier.py # Hybrid LLM + regex intent routing
│   │   ├── dialog_manager.py   # Conversation state machine
│   │   ├── llm_client.py       # Gemini/Groq free-tier with auto-fallback
│   │   ├── context.py          # Short-term sliding context window
│   │   └── memory.py           # ChromaDB long-term vector memory
│   ├── platform/
│   │   ├── __init__.py         # OS detection & platform loader
│   │   ├── mac.py              # macOS: AppleScript, CoreAudio, osascript
│   │   └── windows.py          # Windows: PowerShell, WASAPI, pycaw
│   └── pipeline.py             # End-to-end audio pipeline orchestrator
├── skills/
│   ├── __init__.py
│   ├── base_skill.py           # Abstract skill interface + is_offline_capable
│   ├── system_control.py       # Volume, brightness, apps (cross-platform)
│   ├── web_search.py           # DuckDuckGo (free, no API key)
│   ├── calendar.py             # Calendar & reminders (local SQLite)
│   ├── media_player.py         # Cross-platform media control
│   ├── image_gen.py            # Gemini free image generation
│   ├── code_assistant.py       # Code gen via Gemini/Groq free tier
│   ├── news_weather.py         # NewsAPI (free) + Open-Meteo (free)
│   ├── translation.py          # Argos Translate (free, offline)
│   └── custom_skill_loader.py  # Dynamic plugin loader with error isolation
├── gui/
│   ├── frontend/               # Electron transparent overlay
│   │   ├── src/
│   │   │   ├── main.js         # Electron main process (transparent window)
│   │   │   ├── renderer.js     # Orb animations, waveform, state management
│   │   │   ├── orb.js          # Glowing orb component with 7 states
│   │   │   ├── waveform.js     # Canvas-based audio visualizer
│   │   │   ├── settings.js     # Settings panel (frosted glass popup)
│   │   │   └── styles.css      # Transparent bg, orb gradients, animations
│   │   ├── public/
│   │   │   ├── index.html      # Root HTML (transparent body)
│   │   │   └── sounds/         # Chimes, error sounds, processing sounds
│   │   └── package.json
│   └── bridge.py               # Python ↔ Electron WebSocket IPC bridge
├── api/
│   ├── __init__.py
│   ├── server.py               # FastAPI REST server
│   └── websocket.py            # Real-time WS endpoint for GUI bridge
├── utils/
│   ├── logger.py               # Structured logging with rotation
│   ├── secrets.py              # API key manager (keyring cross-platform)
│   ├── cache.py                # LRU response caching
│   ├── network.py              # Online/offline detection + auto-switch
│   └── helpers.py              # Shared utilities
├── tests/
│   ├── unit/                   # Unit tests per module
│   ├── integration/            # End-to-end pipeline tests
│   └── conftest.py             # Pytest fixtures
├── scripts/
│   ├── setup.sh                # macOS one-command setup
│   ├── setup.bat               # Windows one-command setup
│   └── run.py                  # Cross-platform launcher
├── requirements.txt
├── pyproject.toml
├── .gitignore
└── README.md
```

---

## Feature Implementation Details

### 1. Core Audio Pipeline
- **Microphone capture** via `sounddevice` (cross-platform CoreAudio/WASAPI)
- **Voice Activity Detection (VAD)** via Silero VAD (free, ~2 MB)
- **Wake word detection** via OpenWakeWord — always-listening, Siri-like activation
- **Immediate chime response** on wake word (<100ms) before STT starts
- **Barge-in** — interrupt the assistant while it is speaking
- **Multi-turn listening** — stays active for follow-up questions after responding
- **"Cancel"** — stops the currently running command/action
- **"Bye"** — ends conversation and returns to sleep
- **Audio feedback** — chime on wake, processing indicator, error sounds

### 2. Speech-to-Text (STT)
- **Primary**: Whisper.cpp `small` model (offline, free, ~850 MB, quantized for M1)
- **Fallback**: Vosk `small` model (offline, free, ~40 MB)
- **Error chain**: Whisper fail → retry → Vosk → "I didn't catch that"
- **Streaming recognition**: real-time partial results for low latency
- **Multi-language support**: configurable language codes

### 3. Text-to-Speech (TTS)
- **Primary**: Piper TTS (offline, free, natural voices, ~100 MB)
- **Fallback**: gTTS via Google Translate (free, online, no API key)
- **Error chain**: Piper fail → gTTS → system `say` command → GUI text only
- **Audio caching**: cache repeated TTS outputs to disk
- **Voice selection**: multiple Piper voices, adjustable speed

### 4. Conversational AI (Brain)
- **LLM**: Gemini 2.5 Flash free tier (10 RPM, 250 RPD) → Groq free tier → regex
- **Intent classification**: hybrid LLM + regex pattern matching for fast routing
- **Offline engine**: regex patterns + pre-built response templates
- **System prompt management**: persona, tone, safety guardrails via `prompts.yaml`
- **Conversation memory**: short-term sliding window + long-term ChromaDB vectors
- **Proactive suggestions**: context-aware notifications
- **Emotional tone detection**: adjust response style based on user mood

### 5. Skills & Plugins (10+ Skills)

| Skill | Technology (Free) | Capabilities | Offline? |
|---|---|---|---|
| **System Control** | `subprocess` + platform module | Open/close apps, volume, brightness, screenshot | ✅ |
| **Web Search** | DuckDuckGo (`ddgs` library) | Search + LLM-summarized answers | ❌ |
| **Calendar & Reminders** | SQLite (local) | Create/query events, timers, alarms | ✅ |
| **Media Player** | `subprocess` + OS APIs | Play/pause/skip, queue management | ✅ |
| **Image Generation** | Gemini free tier | Generate images from voice prompts | ❌ |
| **Code Assistant** | Gemini / Groq free tier | Explain code, generate snippets, debug | ❌ |
| **News & Weather** | NewsAPI + Open-Meteo | Headlines, forecasts, personalized news | ❌ |
| **Translation** | Argos Translate (offline) | Voice translation, 50+ languages | ✅ |
| **Custom Plugins** | Python SDK | Users write and load their own skills | ✅ |

### 6. Desktop GUI — Siri-Like Transparent Overlay
- **Transparent frameless window** — Electron with `frame: false`, `transparent: true`
- **Glowing orb** with 7-state animations (Sleeping, Wake, Listening, Processing, Speaking, Error, Offline)
- **Frosted glass text pill** — `backdrop-filter: blur(20px)`, light theme, `rgba(255,255,255,0.85)`
- **Real-time waveform visualizer** — Canvas + Web Audio API
- **Professional light color palette** — soft purple, sky blue, pearl white, lavender accents
- **Dark text on light frosted glass** — `#2D2D3F` typography, Inter/SF Pro
- **Soft shadows** — `rgba(150, 130, 200, 0.3)` glow effects
- **Click-through** — transparent areas pass clicks to desktop
- **Draggable orb** — reposition anywhere on screen
- **System tray** — right-click for settings, quit, toggle
- **Keyboard shortcut** — `Cmd+Space` (Mac) / `Ctrl+Space` (Win)
- **Settings panel** — separate frosted-glass popup window
- **Chat history** — expands upward from text pill, auto-fades

### 7. Security & Privacy
- **API keys** via `keyring` (macOS Keychain / Windows Credential Locker)
- **On-device processing** by default (Whisper, Piper, Vosk — all local)
- **Conversation data** stored locally in SQLite + ChromaDB
- **No paid services** — zero billing risk

### 8. Infrastructure
- **Structured logging** with log levels and file rotation
- **Response caching** (LRU) — saves API quota
- **Health monitoring** — RAM tracking, enforce <4 GB ceiling
- **Rate limit awareness** — track Gemini/Groq daily quotas
- **Network monitor** — auto-detect online/offline, switch modes
- **Global error handler** — never crash, never go silent

---

## Recommended Model Sizes for M1 8GB

| Model | Recommended Size | RAM | Notes |
|---|---|---|---|
| Whisper.cpp | `small` (quantized q5) | ~500 MB | Best accuracy/size tradeoff |
| Piper TTS | `medium` quality voices | ~20 MB/voice | Avoid `high` quality |
| Vosk | `vosk-model-small-en-us-0.15` | ~40 MB | Lightweight fallback |
| OpenWakeWord | Default model | ~50 MB | Single model loaded |
| Argos Translate | Per language pair | ~200 MB/pair | Download on demand |
