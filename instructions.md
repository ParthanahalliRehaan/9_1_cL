# 📋 LUCIFER BUILD PLAYBOOK (v2 — updated from real build session)

> Changes from v1: CUDA install via winget pulled **13.3** (not 12.x — still fine, llama.cpp built clean against it), added the PATH-persistence steps that actually got CUDA working after multiple failed attempts, and added verification commands that catch silent failures (e.g. a clean build that's secretly CPU-only).

---

### **PHASE 0: SYSTEM SETUP (One Time)**

```powershell
# 1. Install Go 1.23+
winget install GoLang.Go
# Add Go to PATH if not automatic — verify with: go version

# 2. Install Node.js 20+
nvm install 20
# OR: winget install OpenJS.NodeJS.LTS

# 3. Install Git
winget install Git.Git

# 4. Install Build Tools
winget install Kitware.CMake
# Also install Visual Studio Build Tools (C++ compiler — cl.exe) if not already present

# 5. Install CUDA Toolkit via winget (pulled 13.3 in practice — that's fine)
winget install --id Nvidia.CUDA -e --source winget

# 6. IMPORTANT: winget install does NOT reliably put nvcc on PATH.
#    Find the actual installed version folder first:
Get-ChildItem "C:\Program Files\NVIDIA GPU Computing Toolkit\CUDA"
#    → note the folder name, e.g. v13.3

# 7. Set CUDA env vars PERMANENTLY at the SYSTEM (Machine) level.
#    Must be run in an ELEVATED PowerShell (Run as Administrator) or the
#    "Machine" scope write silently no-ops with no error.
[System.Environment]::SetEnvironmentVariable("CUDAToolkit_ROOT", "C:\Program Files\NVIDIA GPU Computing Toolkit\CUDA\v13.3", "Machine")
[System.Environment]::SetEnvironmentVariable("PATH", [System.Environment]::GetEnvironmentVariable("PATH", "Machine") + ";C:\Program Files\NVIDIA GPU Computing Toolkit\CUDA\v13.3\bin", "Machine")
# (Use GetEnvironmentVariable(...,"Machine") specifically, not $env:PATH —
#  $env:PATH is a merged User+Machine+Process view and appending that back
#  into "Machine" scope would duplicate all your user PATH entries.)

# 8. Close EVERY open terminal/IDE completely (not just a tab) — PATH is read
#    once at process start, so old windows (including editor-integrated
#    terminals like Cursor/VS Code) keep stale PATH until fully restarted.

# 9. Verify installations in a FRESH terminal
go version         # Should show go1.23+
node -v             # Should show v20+
cmake --version     # Should show 3.28+
nvidia-smi          # Should show your GPU
nvcc --version       # Should print CUDA compiler version — this is the one that
                     # silently fails if PATH wasn't actually refreshed
```

---

### **PHASE 1: BUILD INFERENCE ENGINES (C/C++) — ✅ llama.cpp done**

```powershell
mkdir C:\Projects\lucifer
cd C:\Projects\lucifer

# 1. CLONE LLAMA.CPP
git clone https://github.com/ggerganov/llama.cpp.git

# 2. BUILD LLAMA.CPP WITH CUDA
cd llama.cpp
cmake -B build -DGGML_CUDA=ON
cmake --build build --config Release -j
# NOTE: -DLLAMA_CUDA=ON is deprecated now, GGML_CUDA=ON alone is enough.
# If you hit "Could not find nvcc" / "CUDA Toolkit not found" here, it means
# Phase 0 step 7-9 wasn't actually complete in this terminal — go back and
# fix nvcc on PATH before touching cmake again, and wipe the stale cache:
#   Remove-Item -Recurse -Force build
# (cmake caches a failed CUDA detection — re-running in the same build/ dir
# after fixing PATH can silently reuse the bad cache.)

# 3. VERIFY — don't trust a clean build alone, confirm CUDA is actually linked in:
dir build\bin\Release\llama-server.exe
.\build\bin\Release\llama-server.exe --list-devices
# Should print: CUDA0: <your GPU name> (X MiB, Y MiB free)
# If this prints nothing, the binary compiled CPU-only despite no errors —
# `--version` alone does NOT confirm CUDA backend, --list-devices does.

# 4. CLONE WHISPER.CPP — ✅ done
cd ..
git clone https://github.com/ggerganov/whisper.cpp.git
cd whisper.cpp
cmake -B build -DGGML_CUDA=ON
cmake --build build --config Release -j
# whisper-cli.exe and whisper-server.exe built successfully.
# Full CUDA verification happens in Phase 7 when first transcribing a real file.

# 5. DOWNLOAD LLAMAFILE (for overnight 70B) — in progress
# Repo MOVED: now https://github.com/mozilla-ai/llamafile (was Mozilla-Ocho)
# Latest at time of writing: 0.10.5
# Use PowerShell's native cmdlet to list release assets (curl is aliased to
# Invoke-WebRequest in PowerShell and does NOT understand -s, breaks silently):
cd C:\Projects\lucifer
(Invoke-RestMethod https://api.github.com/repos/mozilla-ai/llamafile/releases/latest).assets.browser_download_url
# Grab the plain "llamafile-X.Y.Z" asset — NOT -thin, NOT .zip, NOT the
# diffusionfile/whisperfile/transcribefile variants (separate tools).
Invoke-WebRequest -Uri "https://github.com/mozilla-ai/llamafile/releases/download/0.10.5/llamafile-0.10.5" -OutFile "C:\Projects\lucifer\llamafile.exe"
.\llamafile.exe --version
# llamafile ships as a cross-platform APE binary — Defender/SmartScreen can
# block it on first run. If --version errors, don't just retry; check the
# actual error message first.
```

**Known constraint from this build:** your RTX 4050 Laptop GPU reports ~5080 MiB free VRAM. The planned 8B Q4_K_M model (~5.5GB) is already tight against that before any other process claims VRAM — if `-ngl 99` OOMs later, that's the expected cause, not a mystery bug. Be ready to drop a few offloaded layers.

---

### **PHASE 2: DOWNLOAD MODELS**

> TheBloke stopped quantizing models in late 2023 — none of the original URLs (Llama 3.1/3.2/3.3, Phi-4) exist under his repos and will 404. The community quantizer covering all of these now is **bartowski**. No Python/huggingface-cli needed — HuggingFace serves plain HTTPS files, so PowerShell's native downloader (or real `curl.exe`) works directly.

```powershell
# 0
mkdir C:\Projects\lucifer\models
cd C:\Projects\lucifer\models

# 1. URGENT MODE - 8B (~4.9GB) — already downloading via Invoke-WebRequest, let it finish, Done
curl.exe -L -C - -o "Meta-Llama-3.1-8B-Instruct-Q4_K_M.gguf" "https://huggingface.co/bartowski/Meta-Llama-3.1-8B-Instruct-GGUF/resolve/main/Meta-Llama-3.1-8B-Instruct-Q4_K_M.gguf"

# 2. DRAFT MODEL - 1B (~0.8GB), Done
curl.exe -L -C - -o "Llama-3.2-1B-Instruct-Q4_K_M.gguf" "https://huggingface.co/bartowski/Llama-3.2-1B-Instruct-GGUF/resolve/main/Llama-3.2-1B-Instruct-Q4_K_M.gguf"

# 3. Phi-4 (~9GB), Done
curl.exe -L -C - -o "phi-4-Q4_K_M.gguf" "https://huggingface.co/bartowski/phi-4-GGUF/resolve/main/phi-4-Q4_K_M.gguf"

# 4. Small model (~1.9GB) — replaces Phi-4-mini, Llama family = consistent tokenizer/chat template with your existing 1B/8B
curl.exe -f -L -o "Llama-3.2-3B-Instruct-Q4_K_M.gguf" "https://huggingface.co/bartowski/Llama-3.2-3B-Instruct-GGUF/resolve/main/Llama-3.2-3B-Instruct-Q4_K_M.gguf"

# 6. Whisper large-v3 (~1.1GB), Done
curl.exe -L -C - -o "ggml-large-v3-q5_0.bin" "https://huggingface.co/ggerganov/whisper.cpp/resolve/main/ggml-large-v3-q5_0.bin"

# 5. 70B — deferred, run later:
# curl.exe -L -C - -o "Llama-3.3-70B-Instruct-Q4_K_M.gguf" "https://huggingface.co/bartowski/Llama-3.3-70B-Instruct-GGUF/resolve/main/Llama-3.3-70B-Instruct-Q4_K_M.gguf"
```
---

### **PHASE 3: CREATE GO ORCHESTRATOR**

```powershell
cd C:\Projects\lucifer
go mod init lucifer

go get github.com/gin-gonic/gin
go get github.com/gorilla/websocket
go get github.com/jmoiron/sqlx
go get github.com/mattn/go-sqlite3
go get github.com/pkoukk/tiktoken-go
go get github.com/sashabaranov/go-openai
go get github.com/qmuntal/stateless
go get github.com/flosch/pongo2/v6
go get github.com/spf13/viper
go get github.com/google/uuid
```

**Directory structure:**
```powershell
mkdir cmd\lucifer
mkdir internal\agent
mkdir internal\harness
mkdir internal\loop
mkdir internal\prompt
mkdir internal\context
mkdir internal\tokenreducer
mkdir internal\models
mkdir internal\session
mkdir internal\voice
mkdir internal\mcp
mkdir ui\src\components
mkdir ui\src\hooks
mkdir scripts
mkdir tmp
```

| File | Path |
|------|------|
| `main.go` | `cmd/lucifer/main.go` |
| `pool.go` | `internal/models/pool.go` |
| `orchestrator.go` | `internal/agent/orchestrator.go` |
| `engine.go` (harness) | `internal/harness/engine.go` |
| `engine.go` (loop) | `internal/loop/engine.go` |
| `engine.go` (prompt) | `internal/prompt/engine.go` |
| `manager.go` | `internal/context/manager.go` |
| `engine.go` (tokenreducer) | `internal/tokenreducer/engine.go` |
| `db.go` | `internal/session/db.go` |
| `pipeline.go` | `internal/voice/pipeline.go` |

---

### **PHASE 4: CREATE REACT UI**

```powershell
cd C:\Projects\lucifer\ui
npx create-vite@latest . --template react-ts
npm install
npm install @radix-ui/react-dialog @radix-ui/react-tabs
npm install lucide-react zustand react-use-websocket
```

`ui/src/App.tsx`:
```tsx
import { useState } from 'react'
import './App.css'

function App() {
  const [mode, setMode] = useState<'urgent' | 'overnight'>('urgent')
  const [message, setMessage] = useState('')
  const [response, setResponse] = useState<any>(null)

  const sendMessage = async () => {
    const res = await fetch('http://localhost:8080/api/chat', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ message, mode })
    })
    const data = await res.json()
    setResponse(data)
  }

  const approve = async (stageId: string) => {
    await fetch(`http://localhost:8080/api/stage/${stageId}/approve`, {
      method: 'POST'
    })
  }

  const reject = async (stageId: string, feedback: string) => {
    await fetch(`http://localhost:8080/api/stage/${stageId}/reject`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ feedback })
    })
  }

  return (
    <div className="lucifer-ui">
      <h1>🔥 LUCIFER</h1>

      <div className="mode-toggle">
        <button
          className={mode === 'urgent' ? 'active' : ''}
          onClick={() => setMode('urgent')}
        >
          ⚡ URGENT (8B, 25 tok/s)
        </button>
        <button
          className={mode === 'overnight' ? 'active' : ''}
          onClick={() => setMode('overnight')}
        >
          🌙 OVERNIGHT (70B, 0.5 tok/s)
        </button>
      </div>

      <div className="chat-input">
        <textarea
          value={message}
          onChange={(e) => setMessage(e.target.value)}
          placeholder="Ask Lucifer anything..."
        />
        <button onClick={sendMessage}>Send</button>
      </div>

      {response && (
        <div className="stage-review">
          <h3>Stage: {response.stage?.name}</h3>
          <pre>{response.stage?.output}</pre>
          <p>{response.message}</p>

          {response.status === 'pending_approval' && (
            <div className="actions">
              <button onClick={() => approve(response.stage.id)}>
                ✅ Approve & Continue
              </button>
              <button onClick={() => {
                const fb = prompt('Feedback for revision:')
                if (fb) reject(response.stage.id, fb)
              }}>
                ❌ Reject & Revise
              </button>
            </div>
          )}
        </div>
      )}
    </div>
  )
}

export default App
```

`ui/src/App.css`:
```css
.lucifer-ui {
  max-width: 800px;
  margin: 0 auto;
  padding: 20px;
  font-family: 'Segoe UI', system-ui, sans-serif;
  background: #0a0a0f;
  color: #e0e0e0;
  min-height: 100vh;
}

h1 {
  text-align: center;
  color: #ff4400;
  text-shadow: 0 0 20px #ff440044;
}

.mode-toggle {
  display: flex;
  gap: 10px;
  margin-bottom: 20px;
}

.mode-toggle button {
  flex: 1;
  padding: 15px;
  border: 2px solid #333;
  background: #1a1a2e;
  color: #888;
  cursor: pointer;
  border-radius: 8px;
  font-weight: bold;
  transition: all 0.3s;
}

.mode-toggle button.active {
  border-color: #ff4400;
  color: #ff4400;
  background: #ff440011;
}

.mode-toggle button:first-child.active {
  border-color: #00ff88;
  color: #00ff88;
  background: #00ff8811;
}

.chat-input {
  display: flex;
  flex-direction: column;
  gap: 10px;
  margin-bottom: 20px;
}

.chat-input textarea {
  min-height: 100px;
  padding: 15px;
  border-radius: 8px;
  border: 1px solid #333;
  background: #1a1a2e;
  color: #e0e0e0;
  font-size: 16px;
  resize: vertical;
}

.chat-input button {
  padding: 15px;
  background: #ff4400;
  color: white;
  border: none;
  border-radius: 8px;
  font-size: 16px;
  font-weight: bold;
  cursor: pointer;
}

.stage-review {
  background: #1a1a2e;
  padding: 20px;
  border-radius: 12px;
  border: 1px solid #333;
}

.stage-review pre {
  background: #0a0a0f;
  padding: 15px;
  border-radius: 8px;
  overflow-x: auto;
  white-space: pre-wrap;
  word-wrap: break-word;
}

.actions {
  display: flex;
  gap: 10px;
  margin-top: 15px;
}

.actions button {
  flex: 1;
  padding: 12px;
  border-radius: 8px;
  border: none;
  cursor: pointer;
  font-weight: bold;
}

.actions button:first-child {
  background: #00ff88;
  color: #0a0a0f;
}

.actions button:last-child {
  background: #ff4444;
  color: white;
}
```

---

### **PHASE 5: BUILD & RUN**

```powershell
# Terminal 1: Build and run Go server
cd C:\Projects\lucifer
go mod tidy
go build -o lucifer.exe ./cmd/lucifer
.\lucifer.exe

# Terminal 2: Run UI
cd C:\Projects\lucifer\ui
npm run dev

# Open browser: http://localhost:5173
```

---

### **PHASE 6: TESTING COMMANDS**

```powershell
# Test 1: Health check
curl http://localhost:8080/health

# Test 2: Urgent mode chat
curl -X POST http://localhost:8080/api/chat `
  -H "Content-Type: application/json" `
  -d '{\"message\":\"Create a learning plan for Go programming\",\"mode\":\"urgent\"}'

# Test 3: Approve stage (use ID from response)
curl -X POST http://localhost:8080/api/stage/YOUR_STAGE_ID/approve

# Test 4: Reject with feedback
curl -X POST http://localhost:8080/api/stage/YOUR_STAGE_ID/reject `
  -H "Content-Type: application/json" `
  -d '{\"feedback\":\"Add more projects, less theory\"}'

# Test 5: Overnight mode (70B - SLOW)
curl -X POST http://localhost:8080/api/chat `
  -H "Content-Type: application/json" `
  -d '{\"message\":\"Explain quantum computing deeply\",\"mode\":\"overnight\"}'
```

---

### **PHASE 7: VOICE SETUP (Later)**

```powershell
# whisper.cpp built in Phase 1

.\whisper.cpp\build\bin\Release\whisper-cli.exe `
  -m .\models\ggml-large-v3-q5_0.bin `
  -f your_audio.wav `
  --no-timestamps

# Voice endpoint accepts multipart/form-data
curl -X POST http://localhost:8080/api/voice `
  -F "audio=@your_audio.wav"
```

---

### **PHASE 8: COMFYUI FOR IMAGES (External Process)**

```powershell
git clone https://github.com/comfyanonymous/ComfyUI.git
cd ComfyUI
pip install -r requirements.txt

# Download SD 1.5 checkpoint (6GB VRAM compatible)
# Place in: ComfyUI\models\checkpoints\
# https://civitai.com/models/4384?modelVersionId=128641 (DreamShaper)

python main.py --listen 0.0.0.0 --port 8188
# Lucifer's orchestrator calls the ComfyUI API at localhost:8188
```

---

## 🎯 DAILY WORKFLOW

| Task | Command |
|------|---------|
| Start Lucifer | `.\lucifer.exe` |
| Start UI | `cd ui && npm run dev` |
| Start ComfyUI (for images) | `cd ComfyUI && python main.py` |
| Check logs | `sqlite3 lucifer.db "SELECT * FROM stages ORDER BY timestamp DESC LIMIT 10;"` |

---

## 📊 FILE STRUCTURE (Final)

```
lucifer/
├── cmd/lucifer/main.go          # Entry point
├── internal/
│   ├── agent/orchestrator.go    # Human-gated state machine
│   ├── harness/engine.go        # Agent scaffolding
│   ├── loop/engine.go           # Iterative refinement
│   ├── prompt/engine.go         # Template system
│   ├── context/manager.go       # Window management
│   ├── tokenreducer/engine.go   # Token compression
│   ├── models/pool.go           # Model load/unload
│   ├── session/db.go            # SQLite logging
│   └── voice/pipeline.go        # STT → Grammar → Text
├── ui/                          # React frontend
│   ├── src/App.tsx
│   ├── src/App.css
│   └── ...
├── llama.cpp/                   # C++ inference engine ✅ built + CUDA-verified
├── whisper.cpp/                 # C++ speech recognition ✅ built
├── llamafile.exe                # Portable 70B runner
├── models/                      # GGUF model files
│   ├── llama-3.1-8b-Q4_K_M.gguf
│   ├── llama-3.3-70b-Q4_K_M.gguf
│   ├── phi-4-Q4_K_M.gguf
│   ├── phi-4-mini-Q4_K_M.gguf
│   └── ggml-large-v3-q5_0.bin
├── ComfyUI/                     # Image generation (external)
├── lucifer.exe                  # Built binary
├── lucifer.db                   # Session database
└── go.mod, go.sum               # Go dependencies
```

---

## ⚡ KEY FEATURES

| Feature | File | How It Works |
|---------|------|--------------|
| **Overnight vs Urgent** | `models/pool.go` | `-ngl 2` (2 GPU layers) for 70B vs `-ngl 99` (all layers) for 8B |
| **Human-gated stages** | `agent/orchestrator.go` | Each stage surfaces output → user approves/rejects → next stage |
| **Harness Engine** | `harness/engine.go` | Pre-configured agents with system prompts per task |
| **Loop Engine** | `loop/engine.go` | Retry with feedback, self-reflection, chain-of-thought |
| **Prompt Engine** | `prompt/engine.go` | Jinja2-style templates with JSON schema validation |
| **Context Manager** | `context/manager.go` | Token counting, summarization at 80% threshold |
| **Token Reducer** | `tokenreducer/engine.go` | 4 strategies: whitespace, key sentences, keywords, LLM compression |
| **Session logging** | `session/db.go` | SQLite, session-keyed, not append-only diary |

---

## 🔧 ENVIRONMENT GOTCHAS LEARNED THIS SESSION

- `winget install --id Nvidia.CUDA` installs the **latest available version** (13.3 at time of writing), not 12.x — this is fine, llama.cpp compiles clean against it.
- Setting `$env:CUDAToolkit_ROOT` / `$env:PATH` only persists for the **current terminal process**. It's gone the moment you close that window.
- For a permanent, all-terminals fix, write to the **Machine**-scope environment variable using `[System.Environment]::SetEnvironmentVariable(..., "Machine")` in an **elevated (Admin)** PowerShell — non-elevated writes to "Machine" scope fail silently.
- Any terminal open *before* the env var change — including IDE-integrated terminals (Cursor, VS Code) — keeps stale PATH. Fully quit and reopen the app, not just the terminal tab.
- `cmake` caches a failed CUDA detection in the `build/` directory. If you fix PATH after a failed run, delete `build/` (`Remove-Item -Recurse -Force build`) before re-running `cmake -B build`, or it may silently reuse the bad cache.
- `llama-cli.exe --version` does **not** confirm CUDA is compiled in — recent llama.cpp versions no longer print backend info there. Use `llama-server.exe --list-devices` instead; it should print `CUDA0: <GPU name>` if the backend is actually linked.
- RTX 4050 Laptop GPU (this machine) reports ~5080 MiB free VRAM — tight against the planned 8B Q4_K_M model's ~5.5GB footprint. Expect to tune `-ngl` down from 99 if you OOM.
- `curl` in PowerShell is an alias for `Invoke-WebRequest`, which does not understand `-s` or other real-curl flags — use `curl.exe` explicitly for real curl, or better, use native `Invoke-RestMethod` / `Invoke-WebRequest` syntax.
- The llamafile repo moved from `Mozilla-Ocho/llamafile` to `mozilla-ai/llamafile`. Old URLs 404.
- `TheBloke` (original Phase 2 model source) stopped quantizing in late 2023 — no repos for Llama 3.1/3.2/3.3 or Phi-4. Use `bartowski` instead, who covers all current model families.
- Model downloads don't need Python/`huggingface-cli` — HuggingFace serves plain HTTPS, so `Invoke-WebRequest` or real `curl.exe` pulls `.gguf`/`.bin` files directly. Keeps the whole stack to JS/TS + Go + C++ as intended (only Python left is ComfyUI in Phase 8, which has no non-Python alternative for that exact tool).

---

**Status:** Phase 0 ✅ · Phase 1 ✅ complete (llama.cpp, whisper.cpp, llamafile v0.10.5 all built/verified) · Phase 2 in progress (8B downloading; 70B deferred until off the clock) · Phases 3-8 not started.