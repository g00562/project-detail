# Voice Based Virtual Assistant

A modular, AI-powered voice assistant for **macOS and Windows** featuring a Siri-like transparent overlay GUI, conversational AI, and 9+ built-in skills. Designed specifically to run on minimal hardware (e.g., M1 Mac 8GB RAM) and gracefully degrade into **100% free, offline operation**.

## Features

- 🎤 **Wake Word Activation** — Always-listening ("Hey Jarvis" / "Hey Assistant")
- 🧠 **Conversational AI** — Gemini/Groq free tier with sliding multi-turn dialog memory
- 🔇 **Offline Mode Built-In** — Works without internet (STT, TTS, system control, timers, calendar, translation)
- 🔮 **Siri-like GUI** — Electron-based transparent overlay with a glowing orb, animations, and frosted glass components
- 🛡️ **Error Resilient & Safe** — Built-in health monitors prevent RAM spikes, graceful fallbacks at every layer
- 💻 **Cross-Platform** — macOS + Windows support

## Tech Stack Overview

| Component | Technology | Offline? |
|---|---|---|
| **STT** | Whisper.cpp (primary) → Vosk (fallback) | ✅ Yes |
| **TTS** | Piper TTS (primary) → gTTS (fallback) | ✅ Yes (Piper) |
| **LLM** | Gemini 2.5 Flash → Groq Llama 3.3 → Regex | ❌ LLMs rule / ✅ Regex |
| **GUI** | Electron (transparent WebSocket overlay) | ✅ Yes |
| **Memory**| ChromaDB (vector store) | ✅ Yes |
| **Search**| DuckDuckGo (free, no API keys) | ❌ No |

## Quick Setup & Usage

### 1. Prerequisites
- **Python 3.11+**
- **Node.js 18+** (for Electron GUI)
- Free API keys (Optional but recommended): [Gemini](https://aistudio.google.com), [Groq](https://console.groq.com)

### 2. Initialization

**macOS:**
```bash
# 1. Setup virtual environment and install Python dependencies
chmod +x scripts/setup.sh && ./scripts/setup.sh

# 2. Add your free API keys to .env
cp config/.env.example config/.env
# Edit config/.env

# 3. Setup and start
python main.py
```

**Windows:**
```cmd
# 1. Setup virtual environment and install Python dependencies
scripts\setup.bat

# 2. Add your free API keys to .env
copy config\.env.example config\.env
# Edit config\.env

# 3. Setup and start
python main.py
```

### 3. Usage & Offline Mode Guide

When running `main.py`, the backend starts alongside the transparent Electron GUI. 

- **Wake the Assistant:** Say *"Hey Jarvis"*, or press `Cmd+Space` (Mac) / `Ctrl+Space` (Windows), or click the orb!
- **Offline Operations:** Unplug your network to test offline mode. The orb will tint **amber**. The system automatically cascades down to Regex-driven intents, Whisper.cpp (STT), Piper (TTS), and offline skills (Calendar, Media, System Control, Translation).
- **Settings:** Right-click the system tray (or use the GUI panel) to change TTS speed, force offline-mode, or adjust wake word sensitivity.
- **Graceful Shutdown:** The system captures `Ctrl+C` inputs, flushing logs, executing garbage collection, stopping the GUI cleanly, and saving any memory to ChromaDB.

## Packaging for Distribution

You can build a standalone backend binary and a GUI executable:
```bash
# macOS
./scripts/build.sh

# Windows
# Similar build scripts are available. It runs PyInstaller and Electron Packager.
```
*Executables will be generated in `dist/`.*

## Available Skills

1. **System Control**: Open/close apps, volume adjust, brightness, screenshot, battery level
2. **Calendar**: SQLite-based offline timers, alarms, event tracking
3. **Media Player**: Play, pause, skip commands directly via OS keys
4. **Translation**: Argos offline translation (16+ languages) with LLM fallbacks
5. **Image Generation**: Gemini Image APIs
6. **Code Assistant**: Specialized LLM conversational agents
7. **News & Weather**: Open-Meteo & DuckDuckGo (Zero Keys!)
8. **Web Search**: DuckDuckGo text/snippet integrations

## Architecture Mapping

```
VoiceAssistant/
├── config/          # .yaml configurations, .env secrets
├── core/            # Audio pipeline, STT, TTS, Intent recognition
├── skills/          # Custom skill implementations (drop-in plugins)
├── gui/             # Electron transparent overlay
├── utils/           # LRU Caches, Memory limits, Network state
├── tests/           # Unit tests asserting offline + online capabilities
├── scripts/         # Bootstrapping
└── main.py          # Unified Thread Bootstrapper
```

## Documentation
- [Architecture Details](docs/ARCHITECTURE.md)
- [Implementation Iteration Plan](docs/IMPLEMENTATION_PLAN.md)
- [Task List Tracking](docs/TASKS.md)

---
MIT License
