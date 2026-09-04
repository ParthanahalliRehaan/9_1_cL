# 🔥 Lucifer v2.0 — No Python, All Speed, 70B on 4GB

## ⚡ The Core Insight: Why Python Sucks for This

| Python Problem | Real Impact on Lucifer |
|----------------|------------------------|
| **GIL** | Can't do true async model loading/unloading |
| **Memory bloat** | PyTorch overhead eats 1-2GB before the model even loads |
| **Slow startup** | 5-10s to import transformers, torch, etc. |
| **Packaging hell** | `pip install` breaks every 3 months |

**The fix:** Go for orchestration, C++ for inference, TypeScript for UI. Zero Python in production.

---

## 🏗️ LUCIFER ARCHITECTURE (Zero Python)

```
┌─────────────────────────────────────────────────────────────┐
│  REACT/TS UI (Lucifer Face)                                 │
│  ├── Overnight Mode Toggle                                  │
│  ├── Urgent Mode Toggle                                     │
│  ├── MCP Tool Panel                                         │
│  └── Voice Input (Whisper.cpp)                              │
└──────────────────────┬──────────────────────────────────────┘
                       │ WebSocket / HTTP
┌──────────────────────▼──────────────────────────────────────┐
│  GO ORCHESTRATOR (Lucifer Brain)                            │
│  ├── State Machine (human-gated stages)                     │
│  ├── Model Pool Manager (load/unload)                       │
│  ├── Prompt Engine (Jinja2-style templates in Go)           │
│  ├── Context Window Manager (token counting, summarization) │
│  ├── Token Reducer (LLMLingua port or custom)               │
│  ├── Loop Engine (iterative refinement)                     │
│  ├── Harness Engine (agent scaffolding)                     │
│  └── Session Logger (SQLite, session-keyed)                 │
└──────────────────────┬──────────────────────────────────────┘
                       │ HTTP / gRPC
┌──────────────────────▼──────────────────────────────────────┐
│  INFERENCE BACKENDS (C/C++ binaries, called as subprocess)  │
│  ├── llama.cpp server (chat, planning, prompt engineering)  │
│  ├── llama.cpp server #2 (grammar correction, token reduce) │
│  ├── ComfyUI (image generation, external process)           │
│  └── whisper.cpp (STT, compiled binary)                     │
└─────────────────────────────────────────────────────────────┘
```

---

## 🚀 TIER 1: Core Infrastructure (Install First)

| # | Tool | Language | Why | Install |
|---|------|----------|-----|---------|
| 1 | **Go 1.23+** | Go | Orchestrator, state machine, API server | `winget install GoLang.Go` |
| 2 | **Node.js 20+** | JS/TS | Lucifer UI | `nvm install 20` |
| 3 | **Git** | — | Version control | `winget install Git.Git` |
| 4 | **NVIDIA Driver + CUDA 12.x** | C++ | GPU acceleration | `nvidia.com/drivers` |
| 5 | **CMake + Visual Studio Build Tools** | — | Compile C++ inference engines | `winget install Kitware.CMake` |

---

## 🤖 TIER 2: Inference Engines (C/C++ Only)

| # | Engine | Language | Role | Why This One |
|---|--------|----------|------|-------------|
| 6 | **llama.cpp** (build from source) | **C/C++** | LLM inference server | The fastest, most mature, zero Python. OpenAI-compatible API via `llama-server`.  |
| 7 | **llamafile** | **C/C++** (Cosmopolitan) | Portable 70B runner | Single binary, no install, runs 70B on CPU. Air-gapped, zero deps.  |
| 8 | **whisper.cpp** | **C/C++** | Speech-to-text | Local, accurate, compiled binary. No Python. |
| 9 | **ComfyUI** | **Python (unavoidable)** | Image generation | Only Python piece. Runs as external process, orchestrator talks via HTTP. |

> **Note:** ComfyUI is the only Python in the stack. It's isolated — Lucifer's Go orchestrator spawns it as a subprocess and talks via its HTTP API. You never touch Python.

---

## 🧠 TIER 3: The 70B-on-4GB Secret Weapons

### 🔴 OVERNIGHT MODE: AirLLM-Style Layer Offloading

**How it works:** Load only 1-2 layers of a 70B model into VRAM at a time. Compute layer → offload to CPU RAM → load next layer. **~0.5 tok/s** but it **runs**.

| Technique | Implementation | Speed | Quality |
|-----------|---------------|-------|---------|
| **llama.cpp `-ngl 2`** | Only 2 layers on GPU, rest on CPU | ~1-2 tok/s | 100% |
| **llama.cpp with mmap** | Memory-mapped weights, OS handles paging | ~0.5-1 tok/s | 100% |
| **llamafile (CPU-only)** | Entire model on CPU, AVX-512 optimized | ~0.3-0.8 tok/s | 100% |

**Command for Overnight 70B:**
```bash
# llama.cpp server with only 2 GPU layers, rest CPU
./llama-server \
  -m models/llama-3.3-70b-Q4_K_M.gguf \
  -ngl 2 \
  --host 0.0.0.0 \
  --port 8081 \
  -c 4096
```

> **VRAM math:** 70B Q4_K_M ≈ 42GB file. With `-ngl 2`, only ~1.2GB in VRAM. Rest paged via mmap to system RAM (you need 48GB+ RAM).

---

### 🟢 URGENT MODE: Fast Inference on 6GB VRAM

| Technique | Implementation | Speed | Model Size |
|-----------|---------------|-------|------------|
| **Q4_K_M quantization** | llama.cpp default | ~20-30 tok/s | 8B fits in 5.5GB |
| **Speculative decoding** | Draft model (1B) + target (8B) | ~40-60 tok/s | Draft runs parallel |
| **KV-cache quantization** | `--cache-type-k q4_0` | +15% speed | Same model |
| **FlashAttention** | Built into llama.cpp | +20% speed | Same model |

**Urgent Mode Command:**
```bash
# 8B model, max GPU layers, speculative decoding
./llama-server \
  -m models/llama-3.1-8b-Q4_K_M.gguf \
  -ngl 99 \
  --host 0.0.0.0 \
  --port 8080 \
  -c 8192 \
  --draft models/llama-3.2-1b-Q4_K_M.gguf \
  --draft-ngram 1
```

---

## 🔧 TIER 4: Go Libraries for Lucifer's Brain

```bash
# Core
go get github.com/gin-gonic/gin           # HTTP API server
go get github.com/gorilla/websocket       # Real-time UI comms
go get github.com/jmoiron/sqlx            # SQLite session logs
go get github.com/pkoukk/tiktoken-go      # Token counting (OpenAI-compatible)
go get github.com/sashabaranov/go-openai  # OpenAI client (talks to llama-server)

# State machine
go get github.com/qmuntal/stateless       # Finite state machine

# Prompt templating
go get github.com/flosch/pongo2           # Jinja2 for Go

# Config
go get github.com/spf13/viper             # Config management
```

---

## 🎨 TIER 5: UI Stack (TypeScript/React)

```bash
# Lucifer UI
npx create-vite@latest lucifer-ui --template react-ts
cd lucifer-ui
npm install @radix-ui/react-dialog @radix-ui/react-tabs  # UI components
npm install lucide-react                                  # Icons
npm install zustand                                       # State management
npm install react-use-websocket                           # WebSocket hook
```

---

## 🎙️ TIER 6: Voice Pipeline (C++ → Go)

| Component | Tool | Language | Role |
|-----------|------|----------|------|
| STT | **whisper.cpp** | C++ | Audio → text |
| Grammar Fix | **llama.cpp (Phi-4-mini)** | C++ | Broken English → correct English |
| TTS | **Piper** (compiled) | C++ | Text → speech |

**Pipeline:**
```
[Microphone] → whisper.cpp → [Raw text]
                              ↓
                    llama.cpp (grammar model)
                    System: "Fix grammar only. Preserve meaning."
                              ↓
                    [Corrected text] → Main Agent
                              ↓
                    [Response] → Piper TTS → [Speaker]
```

---

## 📋 COMPLETE INSTALL CHECKLIST (No Python Touching)

```bash
# === 1. SYSTEM ===
winget install GoLang.Go
winget install OpenJS.NodeJS.LTS
winget install Git.Git
winget install Kitware.CMake
# Install NVIDIA drivers + CUDA 12.x from nvidia.com

# === 2. BUILD LLAMA.CPP ===
git clone https://github.com/ggerganov/llama.cpp.git
cd llama.cpp
cmake -B build -DGGML_CUDA=ON
cmake --build build --config Release -j

# === 3. BUILD WHISPER.CPP ===
git clone https://github.com/ggerganov/whisper.cpp.git
cd whisper.cpp
cmake -B build -DGGML_CUDA=ON
cmake --build build --config Release -j

# === 4. DOWNLOAD LLAMAFILE (for portable 70B) ===
# From: https://github.com/Mozilla-Ocho/llamafile/releases
# Download .exe, rename to .llamafile, run

# === 5. PULL MODELS (via Ollama or manual download) ===
# Download GGUFs from HuggingFace:
# - llama-3.1-8b-Q4_K_M.gguf (urgent mode)
# - llama-3.3-70b-Q4_K_M.gguf (overnight mode)
# - phi-4-mini-Q4_K_M.gguf (grammar correction)
# - whisper-large-v3-q5_0.bin (STT)

# === 6. GO ORCHESTRATOR ===
mkdir lucifer && cd lucifer
go mod init lucifer
# go get all packages listed above

# === 7. UI ===
npx create-vite@latest ui --template react-ts
cd ui && npm install
```

---

## 🧠 THE ENGINES YOU ASKED FOR (Implementation in Go)

### 1. **Harness Engine** — Agent Scaffolding
```go
// Go struct for agent harness
type Harness struct {
    ID          string
    SystemPrompt string
    ModelEndpoint string  // "http://localhost:8080/v1/chat/completions"
    Tools       []MCPTool
    MaxTokens   int
    Temperature float32
}
```

### 2. **Loop Engine** — Iterative Refinement
```go
type LoopConfig struct {
    MaxIterations int
    ExitCondition string  // "plan_validated" | "image_approved" | "user_stop"
    RetryStrategy string  // "same_input" | "with_feedback" | "manual_edit"
}
```

### 3. **Prompt Engine** — Structured Output
```go
type PromptTemplate struct {
    Name     string
    Template string  // Jinja2-style: "Generate image prompt for {{style}}: {{topic}}"
    Schema   json.RawMessage  // JSON schema for structured output
    Validate func(string) error
}
```

### 4. **Context Window Manager**
```go
type ContextWindow struct {
    ModelContextLimit int     // 8192 for 8B
    CurrentTokens     int
    SummarizeThreshold float64 // 0.8 = summarize at 80% full
    History           []Message
}
```

### 5. **Token Reducer** — LLMLingua-style Compression
```go
type TokenReducer struct {
    Method string  // "drop" | "summarize" | "keyword_extract"
    Ratio  float64 // Target compression ratio (e.g., 0.5 = 50% tokens)
}
// Calls a small model (Phi-4-mini) to compress prompts before sending to big model
```

---

## 🌙 OVERNIGHT vs ⚡ URGENT — UI Toggle Logic

```go
// In Go orchestrator
func (l *Lucifer) SelectMode(req Request) Mode {
    if req.Mode == "overnight" {
        return Mode{
            InferenceEngine: "llamafile",      // or llama.cpp with -ngl 2
            Model: "llama-3.3-70b-Q4_K_M.gguf",
            MaxTokens: 4096,
            Speed: "0.5 tok/s (leave it running)",
            VRAM: "1.2GB GPU + 45GB RAM",
        }
    }
    // Urgent mode
    return Mode{
        InferenceEngine: "llama.cpp",
        Model: "llama-3.1-8b-Q4_K_M.gguf",
        MaxTokens: 2048,
        Speed: "25 tok/s (instant)",
        VRAM: "5.5GB GPU",
    }
}
```

---

## 🗺️ IMPLEMENTATION PHASES (Ship v1 First)

```
Phase 1: Core API + Model Switching
  └─ Go server → llama-server processes (spawn/kill per request)
  └─ SQLite session logs
  └─ REST API for chat

Phase 2: Human-Gated State Machine
  └─ Go stateless state machine
  └─ WebSocket to React UI
  └─ Approve/Reject buttons per stage

Phase 3: Prompt + Loop + Harness Engines
  └─ Jinja2 templates in Go
  └─ Iterative refinement loop
  └─ Agent harness with MCP tool registration

Phase 4: Context + Token Reducer
  └─ tiktoken-go for counting
  └─ Summarization fallback
  └─ Prompt compression via small model

Phase 5: Overnight Mode
  └─ llamafile integration
  └─ 70B model download + Q4_K_M quant
  └─ Background job queue

Phase 6: Voice Pipeline
  └─ whisper.cpp binary integration
  └─ Grammar correction model
  └─ Piper TTS

Phase 7: Lucifer UI + MCP
  └─ React UI with mode toggle
  └─ MCP server integration
  └─ Tool marketplace
```

---

## 🎯 WHY THIS STACK WINS

| Decision | Why |
|----------|-----|
| **Go over Python** | True async, single binary deployment, 10x faster startup, no GIL |
| **llama.cpp over Ollama** | Direct control over `-ngl`, speculative decoding, KV cache quant. Ollama hides these knobs.  |
| **llamafile for overnight** | Single binary, no daemon, perfect for "start and forget" jobs.  |
| **whisper.cpp over Python whisper** | Compiled binary, no Python env, faster startup |
| **React over vanilla JS** | Component ecosystem, MCP UI panels, state management |

