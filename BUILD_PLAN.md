# Kai — Feature Build Plan

## Phases Overview

| # | Feature | Files Changed | Effort |
|---|---------|---------------|--------|
| 1 | **Memory Browser** | `web.py`, `index.html` | Small |
| 2 | **Session History** | `brain.py`, `web.py`, `kai/sessions.py` (new), `index.html` | Medium |
| 3 | **Proactive Monitoring** | `web.py`, `kai/monitor.py` (new), `config.py`, `index.html` | Medium |
| 4 | **Morning Briefing** | `web.py`, `index.html` | Small |
| 5 | **Voice I/O** | `web.py`, `kai/voice.py` (new), `index.html` | Large |
| 6 | **Screenshot Tool** | `kai/tools/screenshot.py` (new), `kai/tools/__init__.py`, `index.html` | Medium |

---

## Phase 1 — Memory Browser

**Goal:** Sidebar panel to inspect, edit, and delete what Kai knows about you.

### New API Endpoints (`web.py`)

```python
GET  /memory/facts            → list[{key, value, source, updated_at}]
PUT  /memory/facts/{key}      → body: {value: str} → updates fact
DELETE /memory/facts/{key}    → deletes fact
```

All backed by existing `semantic.list_facts()`, `semantic.set_fact()`, `semantic.delete_fact()`.

### UI Changes (`index.html`)

- Add "Memory" tab toggle in sidebar (alongside the existing stats)
- Table view: `key | value | source | [edit] [delete]`
- Inline edit: click value → becomes input → save on blur/Enter
- Delete: confirmation on click (highlight red → click again to confirm)
- Refresh button (re-fetches `/memory/facts`)
- Show count in tab label: "Memory (12)"

### No new Python files needed.

---

## Phase 2 — Session History

**Goal:** Persist conversations across server restarts. Browse and reopen past sessions.

### New file: `kai/sessions.py`

SQLite tables added to existing `kai.db`:

```sql
CREATE TABLE IF NOT EXISTS sessions (
    id         TEXT PRIMARY KEY,       -- UUID
    title      TEXT NOT NULL,          -- first user message (truncated to 60 chars)
    started_at TEXT NOT NULL,
    last_active TEXT NOT NULL,
    message_count INTEGER DEFAULT 0
);

CREATE TABLE IF NOT EXISTS session_messages (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    session_id  TEXT NOT NULL REFERENCES sessions(id),
    role        TEXT NOT NULL,         -- 'user' | 'assistant'
    content     TEXT NOT NULL,
    timestamp   TEXT NOT NULL,
    turn_order  INTEGER NOT NULL
);
```

Functions:
```python
def new_session(title: str) -> str                            # creates session, returns id
def append_message(session_id, role, content, turn_order)    # persist one message
def list_sessions(limit=50) -> list[dict]                    # [{id, title, started_at, message_count}]
def get_messages(session_id) -> list[dict]                   # [{role, content, timestamp}]
def update_last_active(session_id)                           # called after each turn
```

### `brain.py` changes

- Add `session_id: str | None = None` to `Brain.__init__`
- On first turn, call `sessions.new_session(title=user_input[:60])` → store as `self.session_id`
- After each `commit_turn`, call `sessions.append_message(...)` for both user + assistant
- `clear_history()` also resets `self.session_id = None` (next turn creates a fresh session)

### New API Endpoints (`web.py`)

```python
GET  /sessions                → list of {id, title, started_at, last_active, message_count}
GET  /sessions/{id}/messages  → list of {role, content, timestamp}
POST /sessions/{id}/load      → loads session messages into _brain._session_history, sets session_id
```

### UI Changes (`index.html`)

- Sidebar "History" section: scrollable list of past sessions
  - Each row: session title (first message), date, message count
  - Click → loads session → chat area fills with past messages (rendered as normal bubbles)
  - Active session highlighted
- "New chat" button (existing Clear button behavior) → archives current, starts fresh
- Sessions sorted by `last_active DESC`

---

## Phase 3 — Proactive Monitoring

**Goal:** Background thread watches CPU, GPU temp, disk. Pushes alert to UI when thresholds crossed.

### New file: `kai/monitor.py`

```python
import threading, time, psutil

class SystemMonitor:
    def __init__(self, alert_fn):
        self.alert_fn = alert_fn  # callback: alert_fn(title, body, level)
        self._thread = None
        self._stop = threading.Event()

    def start(self): ...
    def stop(self): ...
    def _loop(self): ...  # checks every MONITOR_INTERVAL_SECONDS
```

Checks (using existing psutil + temps tool logic, called directly — not via LLM):
- CPU % > `ALERT_CPU_PCT` (default: 90) for 3 consecutive checks
- GPU temp > `ALERT_GPU_TEMP_C` (default: 85°C)
- Disk free < `ALERT_DISK_FREE_GB` (default: 10 GB)
- RAM % > `ALERT_RAM_PCT` (default: 92)

Cooldown: same alert type won't re-fire within `ALERT_COOLDOWN_SECONDS` (default: 300).

### `config.py` additions

```python
# ── Monitoring ─────────────────────────────────────────────────────────────────
MONITOR_INTERVAL_SECONDS = 60     # how often the background thread checks
ALERT_CPU_PCT            = 90     # CPU usage alert threshold
ALERT_GPU_TEMP_C         = 85     # GPU temp alert threshold (°C)
ALERT_DISK_FREE_GB       = 10     # free disk alert threshold (GB)
ALERT_RAM_PCT            = 92     # RAM usage alert threshold
ALERT_COOLDOWN_SECONDS   = 300    # min seconds between repeat alerts
```

### `web.py` changes

- Import and start `SystemMonitor` in `_init()`
- Add a global `asyncio.Queue` for alerts: `_alert_queue`
- `alert_fn` callback puts events onto the queue (thread-safe, via `run_coroutine_threadsafe`)
- New endpoint:

```python
GET /alerts   →  SSE stream, yields {"type":"alert","level":"warn","title":"...","body":"..."}
```

Browser keeps a persistent SSE connection to `/alerts` and shows toast notifications when events arrive.

### UI Changes (`index.html`)

- On page load, open `EventSource('/alerts')` and keep it alive
- Toast notification component: slides in from top-right, auto-dismisses after 8s
- Levels: `warn` (yellow), `critical` (red)
- Clicking the toast opens a chat message asking Kai to explain/fix it
- Small indicator dot in header when monitoring is active

---

## Phase 4 — Morning Briefing

**Goal:** First browser open of a new server session → Kai greets with a live status snapshot.

### `web.py` changes

- Add `_briefing_done: bool = False` global flag (resets on server restart)
- New endpoint:

```python
GET /briefing   →  SSE stream (same format as /chat)
```

On first call:
1. Set `_briefing_done = True`
2. Run `system.info` and `temps` tools directly (not via LLM — just call the Python functions)
3. Call `_brain.run_stream(briefing_prompt, on_status=...)` with a preset prompt:
   ```
   "Give me a quick morning briefing. Use the tool results already provided.
    Lead with anything notable, otherwise just say things look good."
   ```
   Inject tool results as pre-resolved tool messages in the conversation before calling.
4. Stream response as SSE tokens

On subsequent calls: return `{"type":"skip"}` immediately (client shows nothing).

### UI Changes (`index.html`)

- On page load, call `/briefing` — if not `skip`, render the streamed response as the first Kai bubble (with waking face + animation)
- Briefing bubble has subtle "[morning briefing]" label in the corner

---

## Phase 5 — Voice I/O

**Goal:** Speak to Kai, she speaks back. Zero external API dependency.

### Dependencies

```
pip install faster-whisper kokoro-onnx sounddevice soundfile
```

- **STT**: `faster-whisper` (CPU-capable, ~300MB for `base` model, ~1s for short clips)
- **TTS**: `kokoro-onnx` (~80MB, high quality, fast on CPU)
- Model download on first use (automatic).

### New file: `kai/voice.py`

```python
class STT:
    def __init__(self, model_size="base"):   # base, small, medium
        ...
    def transcribe(self, audio_bytes: bytes, mime: str) -> str:
        # Save to temp .wav, run faster-whisper, return text
        ...

class TTS:
    def __init__(self, voice="af_heart"):    # kokoro voice ID
        ...
    def synthesize(self, text: str) -> bytes:
        # Returns WAV bytes
        ...
```

Both are initialized once at startup (singleton-style) to avoid model reload overhead.

### `web.py` new endpoints

```python
POST /voice/transcribe
    # Body: audio blob (webm or wav, from MediaRecorder)
    # Returns: {"text": "what the user said"}

POST /voice/speak
    # Body: {"text": "..."}
    # Returns: audio/wav stream
```

### UI Changes (`index.html`)

**Input area changes:**
- Mic button (🎤) next to send button
- Click → starts recording (MediaRecorder, 16kHz mono)
- Button pulses red while recording
- Click again → stops, POSTs blob to `/voice/transcribe`
- Transcribed text fills the textarea, user can edit before sending
- Auto-send option (toggle in settings): send immediately after transcription

**Response playback:**
- Toggle in sidebar: "Speak responses" (off by default)
- When on: after Kai's full response arrives, POST to `/voice/speak`, play returned audio
- Playback indicator in the Kai bubble (small speaker icon)
- Can be interrupted by clicking the bubble or pressing Escape

**Settings panel (new small gear icon in sidebar):**
- Whisper model size: base / small / medium (tradeoff: speed vs accuracy)
- TTS voice selector (kokoro voices)
- Auto-send after transcription: on/off
- Speak responses: on/off

---

## Phase 6 — Screenshot Tool

**Goal:** Kai can capture the screen and analyze it, or the user can paste/drag an image.

### Approach

The active model (gpt-oss:20b) is text-only. Vision analysis uses a dedicated lightweight model:
- `moondream2` (~1.8GB) — runs fast, designed for scene description
- Or `llava:7b` (~4GB) — more general

Swap approach: capture → run vision model → get description → pass description to main conversation as tool result.

### Dependencies

```
pip install mss pillow
ollama pull moondream2   # or llava:7b
```

### New file: `kai/tools/screenshot.py`

```python
@registry.tool(
    name="screenshot.capture",
    description="Take a screenshot of the screen and describe what's visible. "
                "Use when the user says 'look at this', 'what's wrong here', or pastes an image.",
)
def capture_and_describe(monitor: int = 1) -> str:
    # 1. Capture with mss → PIL Image
    # 2. Save to temp file
    # 3. Call Ollama vision model with the image
    # 4. Return the description as text
    ...
```

### `config.py` addition

```python
VISION_MODEL = "moondream2"   # dedicated vision model for screenshot analysis
```

### `kai/tools/__init__.py`

Add import for `screenshot`.

### `web.py` new endpoint

```python
POST /upload/image
    # Body: multipart file upload
    # Saves to temp file, calls vision model, returns {"description": "..."}
    # Client can then inject "Analyze this: {description}" into chat
```

### UI Changes (`index.html`)

- Paste support: if user Ctrl+V with an image on clipboard → intercept, POST to `/upload/image`, inject description into next message
- Screenshot button (📷) in input area → calls `/screenshot` endpoint directly → same flow
- Image preview thumbnail shown above the input before sending
- Drag-and-drop onto chat area: same as paste

---

## Build Order

Work phases in this sequence — each is functional and testable before starting the next:

```
Phase 1: Memory Browser          ← quick win, no new files
Phase 2: Session History         ← most important, sets persistence foundation
Phase 3: Proactive Monitoring    ← background thread, SSE alerts
Phase 4: Morning Briefing        ← small, builds on Phase 3's direct tool calls
Phase 5: Voice I/O               ← largest, independent, can be skipped and returned to
Phase 6: Screenshot Tool         ← requires vision model to be installed first
```

## Dependency Summary

```
# Already installed
fastapi uvicorn psutil sqlite-vec pydantic

# Phase 5 (Voice)
faster-whisper kokoro-onnx sounddevice soundfile

# Phase 6 (Screenshot)
mss pillow
ollama pull moondream2
```
