# 🦞 ReClaw — Revive Your Old Smartphone with AI

> **PRISM Hackathon 2026 | Theme 2: Daily Utility (Smartphones)**
> **Variant Used: PicoClaw (OpenClaw family)**
> **Team:** `MS Ramaiah Institute of Technology_OffGrid`

## 🎬 Demo Video

▶️ **[Watch the Demo Video](https://drive.google.com/drive/folders/1FOFfDPJjOPysMSvwJmd__ZJfvzm4hSYN)** — Full walkthrough showing Telegram master-slave control, profile switching, commute planning, notification digest, and on-device AI.

---

## 🎯 Theme & Problem Statement

**Theme Selected:** Theme 2 — Daily Utility (Smartphones)

**Problem Statements Addressed:**
1. **Intelligent Customizer** — Context-aware phone behavior rules based on user location, time, and daily routine. Creates personalized packs of wallpaper, notification settings, ringtones, and volume for each context.
2. **Daily Commute Planner** — Reads calendar for upcoming appointments, connects to real-time traffic & transit info, calculates optimal departure time, and pushes alerts via Telegram/WhatsApp 30 min before departure.

### The Problem

**Billions of old smartphones sit unused in drawers worldwide.** These devices still have functional processors, cameras, sensors, Wi-Fi, and displays — yet they become e-waste because their software is no longer supported. ReClaw gives them a new brain.

### Our Solution

**ReClaw** transforms any old Android smartphone into an intelligent, always-on personal assistant by combining:
- **A QAT-fine-tuned SLM (Qwen3-0.6B)** with **adaptive multi-runtime inference** — ExecuTorch, llama.cpp (GGUF), PicoLM (forked), LiteRT, or MNN — chosen per device capability
- **PicoClaw** — natively compiled via **Android NDK** as a lightweight agent runtime
- **Master-Slave architecture** — old phone runs as a **passive agent**, controlled from the user's primary phone
- **proot-based Linux distro** for advanced management without requiring root

---

## 🏗️ Detailed Architecture

### Master-Slave Model

```
┌─────────────────────┐          ┌──────────────────────────────────────┐
│  MASTER (Primary     │  Telegram │  SLAVE (Old Android Phone)           │
│  Phone / Desktop)    │◄────────►│                                      │
│                      │  / WS    │  ┌──────────────────────────────────┐│
│  • Sends commands    │          │  │  PicoClaw Agent (NDK binary)     ││
│  • Receives alerts   │          │  │  Compiled via Android NDK        ││
│  • Views dashboards  │          │  │  Native ARM · <15MB · <1s boot   ││
│  • Configures skills │          │  │                                  ││
│                      │          │  │  ┌────────────┐  ┌────────────┐ ││
│  No special app      │          │  │  │ QAT SLM    │  │ Skills     │ ││
│  needed — just       │          │  │  │ Qwen3-0.6B │  │ Routine Mgr│ ││
│  Telegram/WhatsApp   │          │  │  │            │  │ Commute    │ ││
│                      │          │  │  │ Runtime:   │  │ Customizer │ ││
└─────────────────────┘          │  │  │ auto-select│  │ Notif Hub  │ ││
                                 │  │  └──────┬─────┘  └────────────┘ ││
                                 │  │         ▼                        ││
                                 │  │  ┌──────────────────────────┐   ││
                                 │  │  │ Tool Router + Memory     │   ││
                                 │  │  │ Shell·Sensors·GPS·Camera │   ││
                                 │  │  └──────────────────────────┘   ││
                                 │  └──────────────────────────────────┘│
                                 │                                      │
                                 │  ┌──────────────────────────────────┐│
                                 │  │  proot Linux Distro (optional)   ││
                                 │  │  For advanced pkg mgmt & tools   ││
                                 │  └──────────────────────────────────┘│
                                 └──────────────────────────────────────┘
                                    │              │              │
                                    ▼              ▼              ▼
                              Google Maps    Google Calendar   Weather API
```

The **old phone operates as a passive agent (slave)** — it runs autonomously, sensing context and executing skills. The user's **primary phone (master)** interacts via Telegram/WhatsApp without needing any special app. The master can also push config updates and override profiles remotely.

### Core Components

| Component | Technology | Purpose |
|-----------|-----------|---------|
| **Agent Runtime** | PicoClaw (compiled via **Android NDK**) | Native ARM binary, no Go/Termux dependency |
| **Language Model** | Qwen3-0.6B (QAT fine-tuned) | On-device reasoning & NLU |
| **Inference Runtime** | **Adaptive** — see Runtime Selection below | Picked per device capability |
| **Fine-Tuning** | PyTorch + TorchAO QAT | Framework-agnostic quantization-aware training |
| **Messaging** | Telegram Bot API / WhatsApp | Master-slave communication & user alerts |
| **Device Control** | NDK APIs / proot + Termux:API | Volume, wallpaper, DND, alarms, brightness |
| **Memory** | PicoClaw Memory Extension | User profiles, routines, learned preferences |
| **Advanced Mgmt** | proot Linux distro | Full package management without root |

### Adaptive Runtime Selection

We do **NOT** lock into a single inference runtime. PicoClaw auto-selects the best backend based on device capabilities:

| Runtime | Format | Best For | Min RAM | Notes |
|---------|--------|----------|---------|-------|
| **ExecuTorch** | `.pte` | Newer ARM devices (Snapdragon 6xx+) | 2GB | Meta's production runtime, XNNPACK backend |
| **llama.cpp** | `.gguf` | Broad device compatibility | 1.5GB | Mature, wide model support, easy quantization |
| **PicoLM** (forked) | Custom | Ultra-constrained devices (<1GB RAM) | 512MB | Our customized fork for extreme edge |
| **LiteRT** | `.tflite` | Devices with GPU delegate support | 1.5GB | Google's TFLite successor, GPU acceleration |
| **MNN** | `.mnn` | Chinese SoC devices (MediaTek, Kirin) | 1GB | Alibaba's runtime, optimized for these chips |

```
Device Probe → Check: RAM, SoC, Android version, available acceleration
      │
      ├─ Snapdragon 6xx+ / 2GB+ RAM → ExecuTorch (.pte)
      ├─ General ARM64 / 1.5GB+ RAM → llama.cpp (.gguf)
      ├─ MediaTek / Kirin SoC        → MNN (.mnn)
      ├─ GPU delegate available       → LiteRT (.tflite)
      └─ Ultra-low (<1GB RAM)         → PicoLM (custom fork)
```

---

## 🧠 SLM Fine-Tuning Strategy

### Base Model
- **Qwen3-0.6B** — optimal balance of capability vs. on-device performance
- ~40 tok/s on Pixel 8 class hardware (ExecuTorch), varies by runtime

### QAT Training Pipeline (Framework-Agnostic)

No vendor lock-in. We use **PyTorch + TorchAO** directly for Quantization-Aware Training:

```
┌─────────────────────────────────────────────────────┐
│  Step 1: Data Curation (see section below)          │
│  Custom dataset in Alpaca/ShareGPT format           │
│  1,700+ curated examples across 6 categories        │
└──────────────────────┬──────────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────────┐
│  Step 2: QAT Fine-Tuning (PyTorch + TorchAO)       │
│                                                     │
│  from torchao.quantization import quantize_         │
│  quantize_(model, int8_dynamic_activation_int4w())  │
│                                                     │
│  • INT8 dynamic activation + INT4 weight quant      │
│  • Fake-quantization during training                │
│  • Recovers ~70% accuracy vs naive PTQ              │
│  • Full fine-tuning (not LoRA) for QAT compat       │
│  • Colab T4 GPU sufficient                          │
└──────────────────────┬──────────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────────┐
│  Step 3: Multi-Format Export                        │
│                                                     │
│  ┌─────────────┐ ┌──────────┐ ┌─────────────────┐  │
│  │ .pte        │ │ .gguf    │ │ .tflite / .mnn  │  │
│  │ ExecuTorch  │ │ llama.cpp│ │ LiteRT / MNN    │  │
│  │ XNNPACK     │ │ Q4_K_M   │ │ GPU delegate    │  │
│  │ ~472MB      │ │ ~400MB   │ │ ~450MB          │  │
│  └─────────────┘ └──────────┘ └─────────────────┘  │
│                                                     │
│  All formats from same QAT-trained checkpoint       │
└──────────────────────┬──────────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────────┐
│  Step 4: Deploy to Android                          │
│                                                     │
│  • PicoClaw NDK binary auto-detects device          │
│  • Loads best runtime + matching model format       │
│  • Runs under proot distro for advanced setups      │
│  • Fully offline, no cloud dependency               │
└─────────────────────────────────────────────────────┘
```

---

## 📊 Data Curation Plan

> **Priority task — to be completed Week 1**

### Approach

We curate a domain-specific instruction dataset for daily phone utility tasks. Data is created through a mix of **manual writing, template expansion, and LLM-assisted augmentation** (with human validation).

### Dataset Categories & Examples

#### Category 1: Profile Switching (~400 examples)

```jsonl
{"instruction": "Switch to office mode", "output": "[CALL device.volume.set(0)] [CALL device.dnd.toggle(true)] [CALL device.wallpaper.set('minimal_dark')] [CALL device.brightness.set(40)] Done. Office profile activated — silent mode, DND on."}
{"instruction": "I just got home", "output": "[CALL device.volume.set(70)] [CALL device.dnd.toggle(false)] [CALL device.wallpaper.set('family_photos')] [CALL device.brightness.set(60)] Welcome home! Switched to Home profile."}
{"instruction": "Set everything up for sleeping", "output": "[CALL device.volume.set(0)] [CALL device.dnd.toggle(true)] [CALL device.brightness.set(5)] [CALL device.alarm.set('07:00')] 🌙 Sleep mode activated. Alarm set for 7:00 AM. Goodnight!"}
{"instruction": "Main transit mode mein switch kar do", "output": "[CALL device.volume.set(40)] [CALL device.wallpaper.set('transit_map')] Transit profile on. Volume medium, transit display ready."}
```

#### Category 2: Commute Queries (~300 examples)

```jsonl
{"instruction": "When should I leave for my 3 PM meeting?", "output": "[CALL calendar.upcoming()] [CALL maps.directions(current, meeting_location, mode='transit')] Your 3 PM meeting at Connaught Place is 45 min away by metro. 🚇 Leave by 2:00 PM to be safe."}
{"instruction": "Aaj office kitna time lagega?", "output": "[CALL location.get()] [CALL maps.directions(current, office, mode='transit')] Current traffic is moderate. Metro + walk = ~75 min. 🚇 Suggested departure: 8:15 AM."}
{"instruction": "Is there traffic on my usual route?", "output": "[CALL maps.traffic(home, office)] Traffic is heavier than usual on NH-48. Expected delay: +15 min. Consider leaving 15 min earlier today."}
```

#### Category 3: Device Commands (~350 examples)

```jsonl
{"instruction": "Set volume to 50", "output": "[CALL device.volume.set(50)] Volume set to 50%."}
{"instruction": "Turn on Do Not Disturb", "output": "[CALL device.dnd.toggle(true)] DND enabled. Only starred contacts can reach you."}
{"instruction": "Brightness kam kar do", "output": "[CALL device.brightness.set(30)] Brightness reduced to 30%."}
{"instruction": "Change wallpaper to something minimal", "output": "[CALL device.wallpaper.set('minimal_dark')] Wallpaper changed to Minimal Dark."}
```

#### Category 4: Context Sensing & Auto-Triggers (~250 examples)

```jsonl
{"instruction": "[SYSTEM] Time: 09:00, Day: Monday, Location: Office GPS", "output": "[CALL device.volume.set(0)] [CALL device.dnd.toggle(true)] Auto-switched to Office profile. It's Monday 9 AM and you're at the office."}
{"instruction": "[SYSTEM] Time: 18:30, Day: Friday, Location: Home GPS", "output": "[CALL device.volume.set(70)] [CALL device.dnd.toggle(false)] Welcome home! TGIF 🎉 Switched to Home profile."}
{"instruction": "[SYSTEM] GPS: Moving, Speed: >20kmh, Not at home/office", "output": "[CALL device.volume.set(40)] Looks like you're commuting. Switched to Transit mode."}
```

#### Category 5: Notification Management (~200 examples)

```jsonl
{"instruction": "Summarize my messages", "output": "[CALL notifications.list()] You have 14 notifications: 3 urgent (Mom, Boss), 5 from Work Group about project deadline, 6 casual messages in Memes group. Want me to read the urgent ones?"}
{"instruction": "Kya miss kiya maine?", "output": "[CALL notifications.list()] While you were in DND: 8 messages — 2 urgent from Mom, 1 missed call from Boss, rest are group chats. Should I open the urgent ones?"}
```

#### Category 6: Master-Slave & Meta Commands (~200 examples)

```jsonl
{"instruction": "What's the status of my old phone?", "output": "[CALL slave.status()] Old phone status: Battery 73%, Profile: Office, DND: On, Last activity: 12 min ago. Model: Redmi Note 7."}
{"instruction": "Push the sleep profile to my old phone", "output": "[CALL slave.profile.set('sleep')] Sleep profile pushed to slave device. It will activate immediately."}
{"instruction": "Update the commute buffer to 45 minutes", "output": "[CALL memory.write('commute.buffer_minutes', 45)] Commute buffer updated to 45 min. Future alerts will factor this in."}
```

### Data Quality Pipeline

```
Manual Seed Examples (100 per category)
         │
         ▼
Template Expansion (regex + slot filling)
         │  e.g., "Set volume to {0-100}" → 100 variants
         ▼
LLM-Assisted Augmentation (GPT/Claude)
         │  Paraphrase, add Hinglish variants, edge cases
         ▼
Human Validation & Filtering
         │  Remove duplicates, fix tool-call formatting
         ▼
Final Dataset: ~1,700 examples
         │  Format: Alpaca JSONL
         │  Split: 85% train / 15% eval
         ▼
Versioned in model/dataset/
```

### Bilingual Support (English + Hinglish)

~30% of examples include Hinglish variants to support natural Indian user interaction:
- "Volume kam kar do" → decrease volume
- "Aaj office kitna time lagega?" → commute time query
- "Sab kuch night mode mein kar do" → activate sleep profile

---

## 📱 Key Features

### 1. Intelligent Customizer
- **Auto-profiles** based on time + GPS location:
  - 🏠 **Home** → Normal volume, personal wallpaper, all notifications
  - 🏢 **Office** → Silent mode, minimalist UI, work notifications only
  - 🚌 **Transit** → Medium volume, transit-focused display
  - 🌙 **Sleep** → Zero brightness, DND, blue-light filter
- **Personalized packs:** Wallpaper, ringtone, notification sounds, volume presets per profile
- **Adaptive learning:** Observes user patterns and suggests new profile rules

### 2. Daily Commute Planner
- Reads user's Google Calendar for upcoming appointments
- Calculates optimal departure time via Google Maps Directions API
- Factors in real-time traffic + preferred transit mode (metro/bus/drive)
- Pushes proactive alerts **30 min before departure** via Telegram/WhatsApp

### 3. Smart Notification Hub
- Aggregates and prioritizes notifications using on-device SLM
- Summarizes message threads on demand
- Filters noise based on learned user preferences

### 4. Master-Slave Remote Control
- **Old phone = Slave** — runs autonomously, sensing context, executing skills
- **Primary phone = Master** — sends commands, receives alerts, overrides profiles
- Master needs **zero special apps** — just Telegram/WhatsApp
- Remote status check: battery, active profile, last activity
- Push profile changes from master to slave in real-time

---

## 🛠️ Technical Implementation

### PicoClaw — Native NDK Build

PicoClaw is compiled directly via **Android NDK** as a native ARM binary. No Go runtime, no Termux dependency for the core agent.

```bash
# Cross-compile PicoClaw for Android ARM64
export ANDROID_NDK=/path/to/ndk
export CC=$ANDROID_NDK/toolchains/llvm/prebuilt/linux-x86_64/bin/aarch64-linux-android30-clang

# Build native binary
CGO_ENABLED=1 GOOS=android GOARCH=arm64 CC=$CC \
  go build -o picoclaw-android ./cmd/picoclaw

# Push to device
adb push picoclaw-android /data/local/tmp/
adb shell chmod +x /data/local/tmp/picoclaw-android
```

### proot Distro (Advanced Mode)

For advanced users or when more tooling is needed, PicoClaw can run inside a **proot-based Linux distro** on Android — no root required:

```bash
# Install proot-distro via Termux
pkg install proot-distro
proot-distro install alpine  # lightweight ~8MB base

# Enter proot environment
proot-distro login alpine

# Inside proot: full Linux package management
apk add python3 py3-pip git
pip install torch torchao  # for on-device model management

# Run PicoClaw inside proot
./picoclaw-android start --config ./picoclaw.json --skills ./skills/
```

Benefits of proot:
- Full Linux filesystem without root
- `apt`/`apk` package management
- Python, Node, and any Linux tool available
- Isolated environment — doesn't touch Android system

### Skill Files

**Routine Manager** (`skills/routine-manager/SKILL.md`):
```markdown
# Routine Manager
You manage the user's daily phone settings based on context.

## Tools: device.volume.set, device.dnd.toggle, device.wallpaper.set,
##        device.brightness.set, location.get, calendar.upcoming, cron.schedule

## Rules
1. At 9 AM weekdays → Office profile
2. At 6 PM weekdays → Home profile
3. GPS office geofence → enforce silent
4. "goodnight"/"sleep" → Sleep profile
5. Always confirm changes with summary
```

**Commute Planner** (`skills/commute-planner/SKILL.md`):
```markdown
# Commute Planner
Proactively plan commutes so the user never misses a departure.

## Tools: calendar.upcoming, location.get, maps.directions, telegram.send, cron.schedule

## Rules
1. 7 AM daily → check today's calendar
2. departure_time = event_time - commute_time - buffer
3. Push alert with route, mode, ETA
4. Emoji: 🚇 metro, 🚌 bus, 🚗 drive, 🚶 walk
5. Update if traffic changes significantly
```

### Memory Store

```json
{
  "profiles": {
    "office": { "volume": 0, "dnd": true, "brightness": 40, "wallpaper": "minimal_dark",
                "trigger_hours": ["09:00","18:00"], "trigger_location": [28.5355,77.3910] },
    "home":   { "volume": 70, "dnd": false, "brightness": 60, "wallpaper": "family_photos",
                "trigger_hours": ["18:30","08:30"], "trigger_location": [28.6139,77.2090] },
    "sleep":  { "volume": 0, "dnd": true, "brightness": 5, "trigger_keyword": "goodnight" }
  },
  "commute": { "home_coords": [28.6139,77.2090], "office_coords": [28.5355,77.3910],
               "preferred_mode": "transit", "buffer_minutes": 30 },
  "master_slave": { "master_chat_id": "<telegram_id>", "slave_device": "Redmi Note 7",
                    "sync_interval_s": 300, "allow_remote_override": true }
}
```

---

## 👥 User Stories & Journeys

### Story 1: Morning Commute Alert
> *As a professional, I want my old phone to tell me when to leave for work.*

1. 7 AM → ReClaw checks calendar → 10 AM meeting
2. Queries Maps: metro + walk = 75 min → alert at 8:15 AM
3. Telegram to master: *"🚇 Leave by 8:30. Blue Line from Rajiv Chowk, ~75 min."*
4. User leaves → GPS → Transit profile auto-activates on slave

### Story 2: Remote Profile Push (Master-Slave)
> *As a user, I want to text my old phone from my main phone and change its profile.*

1. User texts from primary phone: *"Push sleep mode to old phone"*
2. Master sends command → Slave receives → activates Sleep profile
3. Slave confirms: *"🌙 Sleep mode pushed. DND on, brightness 5%."*

### Story 3: Auto-Silent + Digest
> *As a student, I want auto-silent during class with a summary after.*

1. GPS detects campus → Office profile → silent + DND
2. After class → digest: *"📋 8 messages missed. 2 urgent from Mom."*

---

## 📅 Development Timeline

| Phase | Dates | Deliverables |
|-------|-------|-------------|
| **Phase 1: Registration** | Apr 24 (EOD) | ✅ Proposal, architecture, user stories |
| **Phase 2: Implementation** | Apr 25 – May 8 | Prototype, demo video, APK, skill files |
| **Phase 3: Demo** | May 19 | 10-min demo video, 3 scenarios, final report |

| Week | Tasks |
|------|-------|
| **Week 1** (Apr 25 – May 1) | **Data curation** (1700 examples), QAT pipeline on Colab, NDK-compile PicoClaw, proot setup on test device |
| **Week 2** (May 2 – 8) | SLM fine-tuning, multi-format export (.pte/.gguf/.tflite), skill files, Telegram integration, master-slave testing, demo video |
| **Week 3** (May 9 – 15) | Polish, benchmark per-runtime performance, documentation |
| **Week 4** (May 16 – 19) | Record 3 usage scenarios, implementation report, final demo |

---

## 🔧 Components & Dependencies

### Software Stack

| Component | Version | Purpose |
|-----------|---------|---------|
| **PyTorch** | 2.4+ | Model training framework |
| **TorchAO** | latest | Quantization-Aware Training (INT8-INT4) |
| **ExecuTorch** | latest | Runtime option 1 — .pte inference |
| **llama.cpp** | latest | Runtime option 2 — GGUF inference |
| **PicoLM** (fork) | custom | Runtime option 3 — ultra-constrained devices |
| **LiteRT** | latest | Runtime option 4 — GPU delegate |
| **MNN** | latest | Runtime option 5 — MediaTek/Kirin SoCs |
| **PicoClaw** | latest | Agent runtime (NDK-compiled) |
| **Android NDK** | r26+ | Native ARM compilation |
| **proot-distro** | latest | Rootless Linux on Android |

### API Dependencies

| API | Tier | Purpose |
|-----|------|---------|
| Telegram Bot API | Free | Master-slave comm & user alerts |
| Google Maps Directions | Free (2500/day) | Commute routes |
| Google Calendar API | Free | Appointment reading |
| OpenWeatherMap | Free (optional) | Weather-aware profiles |

### Hardware
- **Training:** Colab T4 (free) — sufficient for Qwen3-0.6B QAT
- **Inference:** Old Android 7+ phone, 1-2GB+ RAM, ARM64
- **Model Size:** ~400-472MB depending on runtime format

---

## 📊 Evaluation Criteria Alignment

| Criteria | Weight | Our Approach |
|----------|--------|-------------|
| **Working Prototype** | 35% | Functional on real old phone, multi-runtime, live demo |
| **Technical Depth** | 25% | QAT fine-tuning, NDK compilation, adaptive runtime, master-slave |
| **User Experience** | 15% | Natural language via Telegram, zero-config profiles, Hinglish support |
| **Theme & Biz Importance** | 15% | E-waste reduction, $0 cost, privacy-first |
| **Documentation** | 10% | This proposal + skills reference + architecture |

---

## 🤖 AI Disclosure

### AI Models in the Solution

| Component | Model / Tool | How It's Used |
|-----------|-------------|---------------|
| **On-device SLM** | Qwen3-0.6B | Core language model — interprets user commands, generates tool calls, provides natural language responses. Runs fully offline on the old phone. |
| **QAT Fine-Tuning** | PyTorch + TorchAO | Quantization-Aware Training with INT8 dynamic activation + INT4 weight quantization. Full fine-tuning (not LoRA) to maintain QAT compatibility. |
| **Inference Runtimes** | ExecuTorch / llama.cpp / PicoLM / LiteRT / MNN | Multiple runtime backends — PicoClaw auto-selects the best one after benchmarking the device hardware. |
| **Export Formats** | .pte, .gguf (Q4_K_M), .tflite, .mnn | All formats exported from the same QAT-trained checkpoint for runtime compatibility. |

### AI Used During Development

| Tool | Purpose | What It Did |
|------|---------|-------------|
| **Antigravity (Gemini)** | Documentation & code scaffolding | Generated README, proposal, skills reference, architecture docs, PPT generation script, and source code structure. All outputs were reviewed and edited by the team. |
| **Google Colab (T4 GPU)** | Model training | Used for running QAT fine-tuning on Qwen3-0.6B. Free tier T4 GPU was sufficient. |
| **LLM-assisted augmentation** | Dataset creation | Used GPT/Claude to paraphrase and expand seed training examples (manual → template expansion → LLM augmentation → human validation). ~30% of augmented examples are Hinglish variants. |

### What AI Did NOT Do

- **Architecture decisions** — Master-slave design, multi-runtime strategy, and PicoClaw agent design were made by the team
- **Skill rule design** — All SKILL.md behavior rules were manually authored based on real user scenarios
- **Hardware testing** — Device benchmarking, NDK compilation, and on-device testing done manually
- **Manual seed data** — Initial 100 examples per dataset category were hand-written

---

## 📋 Prerequisites

Before setting up ReClaw, ensure you have:

**Dev Machine:**
- Go 1.22+ ([install](https://go.dev/dl/))
- Python 3.10+ with pip
- Android NDK r26+ ([download](https://developer.android.com/ndk/downloads))
- ADB (Android Debug Bridge) from Android SDK Platform Tools
- Git

**Target Phone (Slave):**
- Android 7.0+ (API 24+) with ARM64 processor
- Minimum 1GB RAM (2GB+ recommended for ExecuTorch)
- USB debugging enabled
- ~500MB free storage for model + binary
- Termux app installed (for proot/advanced mode)
- Termux:API app installed (for device control)

**Optional:**
- Telegram account + bot token (via @BotFather) for master-slave communication
- Google Maps API key (for commute planning)
- Google Calendar API credentials (for calendar integration)

---

## 🔧 Setup & Installation

### Step 1: Clone the Repository

```bash
git clone https://github.com/<team>/reclaw.git && cd reclaw
```

### Step 2: Fine-Tune the Model (Colab)

```bash
# Upload model/ReClaw_QAT_Finetune.py to Google Colab
# Or run locally with a GPU:
pip install torch torchao transformers datasets accelerate
python model/ReClaw_QAT_Finetune.py \
    --model Qwen/Qwen3-0.6B \
    --dataset model/dataset/ \
    --output model/output/ \
    --epochs 3
```

### Step 3: Export Model to Target Format

```bash
# ExecuTorch (.pte) — for Snapdragon 6xx+ devices
python model/export_pte.py --model model/output/final --output model.pte

# GGUF (.gguf) — for broad ARM64 compatibility
python model/export_gguf.py --model model/output/final --output model.gguf

# TFLite (.tflite) — for GPU delegate devices
python model/export_tflite.py --model model/output/final --output model.tflite
```

### Step 4: Build PicoClaw for Android

```bash
# Set your NDK path
export ANDROID_NDK=/path/to/ndk

# Build using the NDK script
bash android/ndk-build.sh

# Or use the Makefile directly
cd picoclaw && make -f Makefile.ndk build
```

### Step 5: Deploy to Phone

```bash
# Option A: Direct NDK binary (minimal setup)
adb push picoclaw/build/picoclaw-android /data/local/tmp/
adb push model.gguf /data/local/tmp/llama/
adb push config/picoclaw.json /data/local/tmp/
adb push -r skills/ /data/local/tmp/skills/
adb shell chmod +x /data/local/tmp/picoclaw-android

# Option B: proot distro (advanced — full Linux environment)
# First install Termux on the phone, then:
adb push android/proot-setup.sh /data/local/tmp/
adb shell "cd /data/local/tmp && bash proot-setup.sh"
```

### Step 6: Configure

Edit `config/picoclaw.json` with your:
- Telegram bot token (from @BotFather)
- Master phone's Telegram chat ID
- Home/office GPS coordinates for geofencing

### Step 7: Start PicoClaw

```bash
# Direct mode
adb shell /data/local/tmp/picoclaw-android start \
    --config /data/local/tmp/picoclaw.json \
    --skills /data/local/tmp/skills/

# proot mode
adb shell "proot-distro login alpine -- /opt/picoclaw/start.sh"
```

---

## 📱 Usage

### Device Benchmarking

On first run, PicoClaw benchmarks the device and selects the optimal runtime:

```bash
adb shell /data/local/tmp/picoclaw-android benchmark

# Output:
# ═══════════════════════════════════════
#   PicoClaw Device Benchmark Report
# ═══════════════════════════════════════
#   Architecture:    arm64
#   CPU Cores:       8
#   Total RAM:       1836 MB
#   SoC Vendor:      qualcomm
#   GPU Delegate:    false
# ───────────────────────────────────────
#   Recommended:     llama.cpp (GGUF)
# ═══════════════════════════════════════
```

### Telegram Interaction (Master → Slave)

Once connected, chat with your old phone via Telegram:

```
You:    "Switch to office mode"
ReClaw: "✅ Office profile activated — silent mode, DND on, minimal wallpaper."

You:    "When should I leave for my 3 PM meeting?"
ReClaw: "🚇 Your 3 PM meeting at CP is 45 min away by metro. Leave by 2:00 PM."

You:    "Summarize my messages"
ReClaw: "📋 14 notifications: 3 urgent (Mom, Boss), 5 work, 6 casual."

You:    "Push sleep mode to old phone"
ReClaw: "🌙 Sleep profile pushed. DND on, brightness 5%."

You:    "Status?"
ReClaw: "📱 Redmi Note 7 | Battery: 73% | Profile: Office | Last: 12m ago"
```

### Automated Behaviors

These run automatically without user interaction:

| Time | What Happens |
|------|-------------|
| 7:00 AM daily | Checks calendar, plans commute, schedules departure alert |
| 9:00 AM weekdays | Auto-switches to Office profile (if at office GPS) |
| 6:00 PM weekdays | Auto-switches to Home profile |
| 8:00 PM daily | Sends evening notification digest |
| On GPS movement | Detects commute → Transit profile |
| On "goodnight" | Activates Sleep profile + sets alarm |

---

## 📁 Repository Structure

```
reclaw/
├── README.md                        # This file
├── ReClaw_Presentation.pptx         # Hackathon PPT (10 slides)
├── ReClaw_Architecture.png          # System architecture diagram
│
├── picoclaw/                        # Agent runtime (Go, NDK-compiled)
│   ├── go.mod                       # Go module definition
│   ├── Makefile.ndk                 # Android NDK cross-compilation
│   ├── cmd/picoclaw/main.go        # Entry point, config, CLI
│   └── runtime/                     # Multi-runtime inference
│       ├── router.go                # Runtime interface + auto-selection
│       ├── benchmark.go             # Device benchmarking (RAM, SoC, GPU)
│       ├── executorch.go            # ExecuTorch backend (.pte)
│       ├── llamacpp.go              # llama.cpp backend (.gguf)
│       ├── picolm.go                # PicoLM fork (ultra-constrained)
│       ├── litert.go                # LiteRT backend (.tflite)
│       └── mnn.go                   # MNN backend (.mnn)
│
├── skills/                          # Skill definitions (SKILL.md)
│   ├── routine-manager/SKILL.md     # Auto-profile switching
│   ├── commute-planner/SKILL.md     # Departure alerts
│   ├── smart-customizer/SKILL.md    # Personalized packs
│   └── notification-hub/SKILL.md    # Smart notification digest
│
├── model/                           # SLM training & export
│   ├── ReClaw_QAT_Finetune.py      # QAT training (PyTorch + TorchAO)
│   ├── export_pte.py                # ExecuTorch export
│   ├── export_gguf.py               # GGUF export (Q4_K_M)
│   ├── export_tflite.py             # LiteRT export
│   └── dataset/                     # Curated training data (80 seed examples)
│       ├── profile_switching.jsonl   # ~20 examples
│       ├── commute_queries.jsonl     # ~15 examples
│       ├── device_commands.jsonl     # ~15 examples
│       ├── context_sensing.jsonl     # ~10 examples
│       ├── notification_mgmt.jsonl   # ~10 examples
│       └── master_slave.jsonl        # ~10 examples
│
├── android/                         # Android deployment scripts
│   ├── ndk-build.sh                 # NDK cross-compilation
│   ├── proot-setup.sh               # proot-distro bootstrap
│   └── device-bridge.py             # Termux:API device control bridge
│
├── config/                          # Configuration files
│   ├── picoclaw.json                # Agent config (auto runtime, master-slave)
│   └── memory.json                  # Default user preferences & profiles
│
└── demo/                            # Demo assets
    ├── VIDEO_SCRIPT.md              # Video demo storyboard
    ├── videos/
    └── screenshots/
```

---

> **Team ReClaw** — PRISM Hackathon 2026
> *Giving old phones a new brain.* 🧠📱
>
> Built with: PyTorch · TorchAO · ExecuTorch · llama.cpp · PicoClaw · Qwen3
