# Kai Remote Access & PC Control — Project Plan

> Source of truth for multi-phase work to make Kai phone-accessible,
> remote-accessible, and capable of controlling external machines.
> Generated 2026-05-18. Update this file as decisions are made.

---

## Hardware Inventory

| Component       | Spec                                                        | Notes                                            |
|-----------------|-------------------------------------------------------------|--------------------------------------------------|
| CPU             | Intel i7-8700 (6C/12T, Coffee Lake, 2017)                  | No AVX-512. Adequate for orchestration + SSE.    |
| RAM             | 64 GB DDR4                                                  | Plenty for Ollama + Kai + container overhead.    |
| GPU             | NVIDIA RTX 5060 Ti 16 GB (Blackwell, GDDR7, 448 GB/s)      | ~15.5 GB usable VRAM after driver/ECC reserve.   |
| PCIe            | Host: PCIe 3.0 · Card: PCIe 5.0 x8 (electrical)            | 3.0 x16 ≈ 5.0 x8 bandwidth. No bottleneck.      |
| Platform caveat | Z370/B360-era motherboard                                   | Resizable BAR likely unavailable. See risk table. |
| Hypervisor      | Proxmox VE (planned)                                        | Status: not yet installed. See open questions.    |
| Network         | Home LAN, topology TBD                                      | See open questions.                               |

---

## Architecture Overview

```
┌──────────────────────────────────────────────────────┐
│  Proxmox VE Host  (i7-8700 / 64 GB / RTX 5060 Ti)   │
│                                                       │
│  ┌─────────────────────────────────────────────────┐  │
│  │  LXC Container  (Debian/Ubuntu, GPU passthrough) │  │
│  │                                                   │  │
│  │  ┌──────────┐    HTTP     ┌──────────────────┐   │  │
│  │  │  Ollama   │◄──────────►│  Kai (Python)    │   │  │
│  │  │  (GPU)    │ /api/chat  │  brain.py        │   │  │
│  │  └──────────┘  (native)   │  web.py (FastAPI)│   │  │
│  │                           │  events.py       │   │  │
│  │                           │  memory/ (SQLite) │   │  │
│  │                           └────────┬─────────┘   │  │
│  │                                    │ :7860       │  │
│  └────────────────────────────────────┼─────────────┘  │
│                                       │                 │
│  ┌────────────────────────────────────┼─────────────┐  │
│  │  Reverse proxy (Caddy/nginx)       │             │  │
│  │  TLS termination · :443 → :7860    │             │  │
│  └────────────────────────────────────┼─────────────┘  │
└───────────────────────────────────────┼─────────────────┘
                                        │
             LAN / Tailscale ───────────┘
                    │
            ┌───────┴────────┐
            │  Phone browser │
            │  (mobile-first │
            │   chat UI)     │
            └────────────────┘


Phase 3 addition:

┌──────────────────────────────┐
│  Gaming PC (separate machine)│
│  ┌────────────────────────┐  │
│  │  kai-pc-daemon (Python) │  │
│  │  Listens on Tailscale   │  │
│  │  only. mTLS or HMAC.    │  │
│  └────────────────────────┘  │
└──────────────────────────────┘
        ▲
        │  Tailscale-only
        │  commands from Kai
        ▼
   Kai LXC container
```

---

## Locked Decisions

**Decision:** LXC container, not VM.
**Why:** GPU passthrough with kernel sharing. <1% perf overhead vs ~5-10% for full VM. Simpler driver story for single-GPU homelab.
**Risk if wrong:** If NVIDIA driver requires full VM isolation (unlikely with 5060 Ti + Proxmox 8.x), migration to VM adds ~4h work.

**Decision:** Auth = API key for REST + short-lived signed HTTP-only cookie for SSE streams.
**Why:** Browser `EventSource` API cannot send custom headers. Cookie-exchange endpoint trades API key for stream cookie. Single-user system — no Auth0, no user management.
**Risk if wrong:** None meaningful. Pattern is standard and already partially implemented in `web.py` (session cookie auth exists today).

**Decision:** SSE for streaming, not WebSockets.
**Why:** Unidirectional server→client fits `run_stream()` generator shape. Simpler than WS for chat tokens. Automatic reconnection via `EventSource`. The existing `/chat` endpoint already returns `text/event-stream`.
**Risk if wrong:** If bidirectional real-time features are needed later (e.g., interrupt mid-generation), adding a WS endpoint alongside SSE is additive, not breaking.

**Decision:** Ollama native `/api/chat`, not OpenAI-compatible endpoint.
**Why:** Kai's Brain already uses native Ollama HTTP. Native endpoint exposes thinking tokens, tool calls, and model-specific features that OpenAI-compat drops.
**Risk if wrong:** Off-the-shelf chat UIs (Open WebUI, etc.) cannot connect directly. Acceptable — Kai has its own UI.

**Decision:** YuE music gen and Kai are sequential, not co-resident in VRAM.
**Why:** Both exceed 8 GB individually. 16 GB cannot hold both. Kai unloads → YuE runs → Kai reloads (~10-15s on GDDR7).
**Risk if wrong:** Reload time increases if model is larger than expected. Mitigated by `OLLAMA_KEEP_ALIVE=0` on YuE side.

**Decision:** Default model target: Qwen3 14B at Q4_K_M (~10 GB working set with context).
**Why:** Best quality-per-VRAM at 16 GB. Leaves ~5 GB for KV cache + overhead.
**Risk if wrong:** If 14B Q4 + long context exceeds 15.5 GB usable, fall back to Q3 or smaller context window. `[CONFIRM: needs current web search — verify Qwen3 14B Q4_K_M actual VRAM usage with 8K context on Ollama]`

---

## Phase 1: Phone-Accessible Chat UI

### 1.1 Existing `web.py` Inspection

Current state (inspected 2026-05-18):
- **Entry point:** `web.py` — FastAPI app on `:7860`
- **Auth:** Session-cookie based (`kai_session` HTTP-only cookie), DB-backed tokens (`session_tokens` table), 7-day TTL, login rate-limiting (5 attempts / 15 min per IP)
- **Chat endpoint:** `POST /chat` → `StreamingResponse` with `text/event-stream`. Runs `brain.run_stream()` in a background thread, pushes events via `asyncio.Queue`.
- **Event types in SSE:** `token`, `done`, `status`, `think_step`, `think`, `error`
- **Event bus:** `kai/events.py` — separate system for Kai's Computer (WebSocket-based, `ws/activity/{session_id}`). Persists to SQLite. Events: `tool.start`, `tool.end`, `think`, `stream.token`, `stream.end`, `memory.read`, `memory.write`, `model.switch`, `face`, `error`.
- **CORS:** Locked to `localhost:{port}` and `127.0.0.1:{port}` only.
- **Security headers:** CSP, X-Frame-Options DENY, nosniff, strict referrer, permissions-policy.
- **TLS:** Self-signed cert generation built in (`--tls` flag). Enforces TLS when binding non-localhost.
- **Static files:** Mounted at `/static/`.
- **Auth guard:** Raw ASGI middleware. Public routes: `/login`, `/users/*`, `/api/show-window`, `/static/*`, `/ws/*`, `/api/events/*`, `/computer`. Optional auth: `/`, `/users/logout`.

**Key finding:** Most of the Phase 1 backend already exists. The chat SSE endpoint, auth middleware, rate limiting, and TLS support are implemented. Changes needed are narrower than expected.

### 1.2 Backend Additions / Changes

| Change | Description | Effort |
|--------|-------------|--------|
| **CORS for LAN** | `setup_app()` currently allows only `localhost` + `127.0.0.1`. Add the LXC container's LAN IP and optionally `*.local` hostname. Best approach: accept a `--cors-origins` CLI flag or read from env var `KAI_CORS_ORIGINS`. | Small |
| **Bind to 0.0.0.0** | Currently defaults to `127.0.0.1`. Inside LXC, must bind `0.0.0.0` to be reachable. The TLS enforcement already gates this (refuses non-localhost without TLS). | Trivial |
| **API-key auth endpoint** | Add `POST /api/auth/token-exchange` — accepts `Authorization: Bearer <api-key>` header, returns `kai_session` cookie. This is the "cookie exchange" that lets the phone's `EventSource` authenticate via cookie after initial API-key login. The existing `/users/login` (name + PIN) can remain as a secondary path. | Small |
| **API key storage** | Store a hashed API key in the DB (or config file). Single key, generated on first run, displayed once. `secrets.token_urlsafe(32)`. Hash with `hashlib.sha256`. | Small |
| **SSE `Last-Event-Id` support** | Add `id:` field to SSE events in `/chat` stream. On reconnect, `EventSource` sends `Last-Event-Id` header — resume from that point in the queue. Prevents lost tokens on flaky mobile connections. | Medium |
| **Health endpoint** | `GET /api/health` — returns `{"status": "ok", "model": "...", "ollama": true/false}`. Public (no auth). Used by mobile UI for connection status indicator. | Trivial |
| **CORS preflight** | Verify `OPTIONS` requests work correctly with the auth middleware. Current `_AuthGuard` may block preflight requests that lack cookies. Add `OPTIONS` to `_PUBLIC` methods or let CORSMiddleware handle it (it runs after AuthGuard in the middleware stack — need to verify ordering). | Small |

### 1.3 Frontend Approach

**Decision:** Single HTML file + vanilla JS (no framework). Tailwind CSS via CDN for styling.
**Why:** The existing `web.py` already serves HTML from `/static/`. The current app uses inline JS + Tailwind CDN (see CSP headers allowing `cdn.tailwindcss.com`). Adding React/Vue/Svelte introduces a build step, node_modules, and complexity for a single-page chat UI that needs: a message list, an input box, a status bar, and a settings panel. Vanilla JS with `EventSource` is ~300 lines for full streaming chat. Mobile-first layout is a CSS concern, not a framework concern.
**Risk if wrong:** If the UI grows significantly (file browser, settings panels, multi-session), migrating to a framework later is straightforward — the API contract doesn't change.

**Frontend requirements:**
- Mobile-first responsive layout (min-width: 320px, breakpoint at 768px for tablet/desktop)
- `EventSource` connection to `POST /chat` (note: standard `EventSource` is GET-only; use `fetch()` with `ReadableStream` for POST + SSE, or use a library like `eventsource-polyfill` that supports POST)
- Visible thinking blocks (collapsible `<details>` element, shows `think_step` events)
- Tool status indicators (icon + label for `tool.start` / `tool.end` via event bus)
- Memory state panel (slide-out drawer showing last memory reads/writes from event bus)
- Auto-scroll with "scroll to bottom" button when user scrolls up
- Dark mode default (OLED-friendly for phone use)
- Markdown rendering for responses (use `marked.js` or similar lightweight lib)
- Input: auto-growing `<textarea>`, send on Enter (Shift+Enter for newline), send button for touch

**Correction on EventSource:** The existing `/chat` endpoint is `POST`. Browser `EventSource` only supports `GET`. Two options:
1. **Option A:** Use `fetch()` + `response.body.getReader()` to consume the SSE stream from a POST. This is what the existing desktop UI likely does. Works on all modern mobile browsers.
2. **Option B:** Add a `GET /chat/stream?msg=...` endpoint (URL-encoded message). Simpler `EventSource` usage but message length limited by URL length (~2048 chars) and message visible in logs.

**Recommendation:** Option A (`fetch` + `ReadableStream`). No URL length limit, POST body is not logged, existing endpoint works unchanged. ~20 lines of JS to parse SSE from a ReadableStream.

### 1.4 Auth Implementation Steps

1. **Generate API key on first run.** On startup, if no key exists in DB, generate `secrets.token_urlsafe(32)`, hash it with SHA-256, store the hash. Print the raw key to the console once: `[!] Your API key: kai_xxxxxxxxxxxx — save this, it won't be shown again.`
2. **Add `POST /api/auth/token-exchange` endpoint.**
   - Accepts `Authorization: Bearer <api-key>`.
   - Hashes the provided key, compares against stored hash.
   - On match: calls existing `_issue_token()` to create a session, sets `kai_session` cookie (HTTP-only, SameSite=Strict, Secure if TLS).
   - On failure: 401 with rate limiting (reuse existing `_check_login_rate()`).
3. **Phone login flow:**
   - User opens `https://<kai-ip>/` on phone.
   - Login page shows a single "API Key" field + "Connect" button.
   - JS sends `POST /api/auth/token-exchange` with the key.
   - On success, cookie is set, page redirects to chat UI.
   - Subsequent requests (including `fetch` to `/chat`) include the cookie automatically.
4. **Existing name+PIN login remains** as an alternative path (useful for desktop app where the user may not want to type a 43-char key).
5. **Logout:** Existing `/users/logout` endpoint revokes the token. Add a logout button to mobile UI.

### 1.5 Deployment Inside LXC

**Assumption:** Proxmox is a fresh install (confirmed as open question — these steps assume Debian-based LXC).

**LXC setup steps (Phase 1 scope):**

1. **Create LXC container:**
   ```bash
   pct create 100 local:vztmpl/debian-12-standard_12.x_amd64.tar.zst \
     --hostname kai --memory 8192 --cores 4 --rootfs local-lvm:32 \
     --net0 name=eth0,bridge=vmbr0,ip=dhcp --unprivileged 0
   ```
   Note: `--unprivileged 0` (privileged container) is required for GPU passthrough. `[CONFIRM: needs current web search — verify RTX 5060 Ti GPU passthrough in Proxmox LXC with privileged container, driver version requirements]`

2. **GPU passthrough (privileged LXC):**
   - Install NVIDIA driver on Proxmox host.
   - Add to LXC config (`/etc/pve/lxc/100.conf`):
     ```
     lxc.cgroup2.devices.allow: c 195:* rwm
     lxc.cgroup2.devices.allow: c 509:* rwm
     lxc.mount.entry: /dev/nvidia0 dev/nvidia0 none bind,optional,create=file
     lxc.mount.entry: /dev/nvidiactl dev/nvidiactl none bind,optional,create=file
     lxc.mount.entry: /dev/nvidia-uvm dev/nvidia-uvm none bind,optional,create=file
     lxc.mount.entry: /dev/nvidia-uvm-tools dev/nvidia-uvm-tools none bind,optional,create=file
     ```
   - Inside container: install matching NVIDIA driver userspace libs (same version as host kernel driver). `[CONFIRM: needs current web search — exact steps for RTX 5060 Ti / Blackwell driver in Proxmox 8.x LXC]`

3. **Install Ollama inside LXC:**
   ```bash
   curl -fsSL https://ollama.com/install.sh | sh
   ```
   Verify GPU detection: `ollama run qwen3:14b "hello"` — should show GPU layers in logs.

4. **Deploy Kai:**
   ```bash
   git clone https://github.com/supasoulja/newb_agent.git /opt/kai
   cd /opt/kai
   python3 -m venv .venv
   source .venv/bin/activate
   pip install -r requirements.txt  # or equivalent
   ```

5. **Systemd unit for Kai:**
   ```ini
   # /etc/systemd/system/kai.service
   [Unit]
   Description=Kai AI Assistant
   After=network.target ollama.service
   Wants=ollama.service

   [Service]
   Type=simple
   User=kai
   WorkingDirectory=/opt/kai
   Environment=OLLAMA_KV_CACHE_TYPE=q8_0
   ExecStart=/opt/kai/.venv/bin/python web.py --host 0.0.0.0 --port 7860 --tls
   Restart=on-failure
   RestartSec=5

   [Install]
   WantedBy=multi-user.target
   ```

6. **Reverse proxy decision:**

   **Decision:** Caddy as reverse proxy.
   **Why:** Automatic TLS cert management (even self-signed for LAN), HTTP/2, zero-config for simple proxying. Lighter than nginx for this use case. Single binary, no dependencies.
   **Risk if wrong:** If advanced proxy features are needed later, nginx is a drop-in replacement. The Caddyfile is ~5 lines.

   ```
   # /etc/caddy/Caddyfile
   https://kai.local {
     tls internal
     reverse_proxy localhost:7860
   }
   ```

   Alternative: skip the reverse proxy entirely in Phase 1. Kai's built-in `--tls` already generates a self-signed cert. Add Caddy in Phase 2 when Tailscale is introduced. **Recommended: skip Caddy in Phase 1, use Kai's built-in TLS directly.** One fewer moving part.

   **Decision:** Skip reverse proxy in Phase 1. Use Kai's built-in TLS (`--tls` flag).
   **Why:** Fewer components to debug. Built-in cert generation already works. Add Caddy in Phase 2 if needed.
   **Risk if wrong:** Self-signed cert means phone will show a browser warning on first connect. Acceptable for LAN use — tap "Advanced → Proceed" once.

7. **Port:** Kai listens on `:7860`. Firewall: allow `:7860` from LAN only.

### 1.6 Acceptance Criteria

Phase 1 is complete when ALL of the following are true:

- [ ] Kai runs inside a Proxmox LXC container with GPU passthrough confirmed (`nvidia-smi` shows RTX 5060 Ti inside container)
- [ ] Ollama inside the container loads Qwen3 14B Q4 and responds to a test prompt
- [ ] Phone on home WiFi can open `https://<kai-lan-ip>:7860` in mobile browser
- [ ] Phone sees a login screen, enters the API key, receives a session cookie
- [ ] Phone sends a message and receives streamed tokens in real time (no full-response delay)
- [ ] Thinking blocks are visible and collapsible in the phone UI
- [ ] Tool calls show a status indicator (tool name + spinner → result summary)
- [ ] Memory state is inspectable via a slide-out panel (last N memory reads/writes)
- [ ] Session persists across page reloads (cookie-based, no re-login required for 7 days)
- [ ] TLS is active (no plaintext HTTP on the network)
- [ ] Kai auto-starts on LXC boot via systemd

---

## Phase 2: Tailscale Remote Access

### 2.1 What to Install

- **Tailscale on the Proxmox host** (or inside the LXC container — inside LXC is simpler, avoids host-level changes):
  ```bash
  # Inside the LXC container:
  curl -fsSL https://tailscale.com/install.sh | sh
  tailscale up --hostname=kai
  ```
- **Tailscale on the phone:** Install Tailscale app (iOS/Android), join the same tailnet.

### 2.2 How Auth Carries Over

No changes needed. The existing API-key → cookie flow works identically over Tailscale because:
- Tailscale provides a WireGuard tunnel — traffic is encrypted end-to-end.
- The phone accesses Kai via its Tailscale IP (e.g., `https://100.x.y.z:7860` or `https://kai.tail-xxxxx.ts.net:7860`).
- Cookies are sent to the same origin. SameSite=Strict is fine because the user navigates directly.
- TLS (self-signed or Tailscale HTTPS cert) remains active.

**Optional improvement:** Use Tailscale's built-in HTTPS certs (`tailscale cert kai.tail-xxxxx.ts.net`) to eliminate the self-signed cert browser warning. This gives a real Let's Encrypt cert valid for the Tailscale hostname. `[CONFIRM: needs current web search — verify Tailscale HTTPS cert provisioning current steps and limitations]`

### 2.3 What NOT to Expose

- **Do NOT** open any port on the home router. No port forwarding. No UPnP. No DynDNS.
- **Do NOT** expose Ollama's port (`:11434`) to Tailscale. Only Kai's port (`:7860`) should be reachable.
- **Do NOT** enable Tailscale's subnet routing unless explicitly needed for other devices.
- **Do NOT** share the Tailscale node with other users (keep the tailnet single-user or use ACLs to restrict access).

### 2.4 Tailscale ACL Configuration

```json
{
  "acls": [
    {
      "action": "accept",
      "src": ["tag:phone"],
      "dst": ["tag:kai:7860"]
    }
  ],
  "tagOwners": {
    "tag:phone": ["autogroup:admin"],
    "tag:kai":   ["autogroup:admin"]
  }
}
```

This restricts traffic so only the phone can reach Kai, and only on port 7860.

---

## Phase 3: PC Control Daemon

### 3.1 Protocol Between Kai and PC Daemon

```
Kai (LXC)  ──────────────►  kai-pc-daemon (Gaming PC)
            Tailscale-only
            HTTPS + HMAC
            JSON-RPC 2.0
```

- **Transport:** HTTPS over Tailscale. The daemon binds to the Tailscale interface only (`100.x.y.z`), not `0.0.0.0`.
- **Auth:** HMAC-SHA256 signed requests. Kai and the daemon share a pre-shared key (generated once, stored in both configs). Every request includes a timestamp + HMAC signature. Daemon rejects requests where `|now - timestamp| > 30s` (replay window).
- **Protocol:** JSON-RPC 2.0 over HTTPS. Simple, stateless, easy to debug with `curl`.
- **Direction:** Kai calls the daemon. The daemon never calls Kai. The daemon is a pure command executor.

### 3.2 Wake-on-LAN Flow

```
Phone → Kai: "wake up my PC"
Kai tool: pc.wake()
  └── Kai sends WoL magic packet to Gaming PC's MAC address
      (WoL is a broadcast on the LAN — Kai's LXC must be on the
       same L2 network as the Gaming PC, or a directed broadcast
       must be configured on the router)
Kai polls: pc.status() → daemon not responding
Kai waits 30s, retries
Kai polls: pc.status() → daemon responds "online"
Kai → Phone: "Your PC is awake."
```

**Requirement:** Gaming PC's BIOS must have WoL enabled. The NIC must support WoL in standby. Verify in BIOS settings.

### 3.3 Security Model

1. **Daemon binds Tailscale IP only.** Not reachable from the LAN or internet.
2. **HMAC-signed requests.** No unsigned command is ever executed.
3. **Command allowlist.** The daemon has a hardcoded list of allowed commands. Anything not on the list is rejected with a 403. No shell exec. No arbitrary command passthrough.
4. **No inbound connections from the internet.** Tailscale only.
5. **Daemon runs as a limited user** (not root/admin). Commands that need elevation (e.g., shutdown) use pre-configured `sudoers` entries or Windows scheduled tasks.

### 3.4 Command Surface (v1)

| Command          | JSON-RPC Method   | Description                                      | Params                    |
|------------------|-------------------|--------------------------------------------------|---------------------------|
| Wake PC          | `pc.wake`         | WoL magic packet (sent from Kai, not daemon)     | None (MAC in Kai config)  |
| Status           | `pc.status`       | CPU/GPU/RAM usage, uptime, logged-in user        | None                      |
| Launch Steam     | `pc.launch_steam` | Start Steam client (or bring to foreground)      | None                      |
| Launch Discord   | `pc.launch_discord` | Start Discord (or bring to foreground)         | None                      |
| Launch game      | `pc.launch_game`  | Launch a game by Steam App ID                    | `{"app_id": "730"}`       |
| List games       | `pc.list_games`   | Installed Steam games (reads libraryfolders.vdf)  | None                      |
| Shutdown         | `pc.shutdown`     | Graceful OS shutdown (60s warning)               | `{"delay_seconds": 60}`   |
| Sleep            | `pc.sleep`        | Put PC to sleep                                  | None                      |

### 3.5 Daemon Implementation Notes

- **Language:** Python (reuse Kai's ecosystem). Single-file `kai_pc_daemon.py`, ~200 lines.
- **Framework:** `http.server` or FastAPI (if already available). Lean toward `http.server` for zero dependencies on the gaming PC.
- **Install:** Copy script + config file. No pip install needed if using stdlib only.
- **Windows service:** Run via Task Scheduler (start on login, restart on failure). Or NSSM for a proper Windows service.
- **Config file:** `kai_daemon.json` — contains HMAC key, Tailscale bind IP, allowed commands.

---

## Phase 4: Voice

### 4.1 Model Loading Strategy in 16 GB VRAM

| Component    | Model                | VRAM Estimate | Notes                                  |
|-------------|----------------------|---------------|----------------------------------------|
| Kai LLM     | Qwen3 14B Q4_K_M     | ~10 GB        | With 8K context KV cache               |
| Whisper STT | whisper-large-v3      | ~3 GB         | `[CONFIRM: needs current web search]`  |
| Sesame TTS  | CSM-1B                | ~2 GB         | `[CONFIRM: needs current web search]`  |
| **Total**   |                      | **~15 GB**    | Tight but within 15.5 GB usable        |

**Strategy: sequential loading, not co-resident.**

Despite the table suggesting it might fit, running all three simultaneously is risky due to:
- KV cache growth beyond 8K tokens
- CUDA memory fragmentation
- Driver overhead not accounted for

**Pipeline:** Load/unload models per phase of the interaction:

```
1. User speaks → mic captures audio
2. Load Whisper (if not loaded) → transcribe → unload Whisper
3. Kai LLM generates response text (already loaded)
4. Load CSM-1B (if not loaded) → synthesize speech → unload CSM-1B
5. Play audio to speaker/phone
```

Ollama's `OLLAMA_KEEP_ALIVE` controls how long models stay in VRAM. Set to `0` for Whisper and CSM-1B (immediate unload after use). Set to `5m` for Kai LLM (stays loaded between turns).

**Latency estimate:**
- Whisper load: ~2-3s, transcribe: ~1-2s for a 10s utterance
- Kai LLM response: already loaded, ~2-5s for a typical response
- CSM-1B load: ~1-2s, synthesize: ~1-3s for a sentence
- **Total round-trip: ~7-13s.** Acceptable for voice interaction. `[CONFIRM: needs current web search — actual Whisper/CSM-1B load times on RTX 5060 Ti]`

**Alternative (try co-resident first):**
Before committing to sequential loading, test co-resident loading on actual hardware:
```bash
# Load all three, check actual VRAM
ollama run qwen3:14b "test" &
# Load Whisper and CSM-1B via their respective servers
nvidia-smi  # check total usage
```
If all three fit with acceptable KV cache, skip the load/unload orchestration.

### 4.2 Pipeline Shape

```
                    Phone / Local Mic
                          │
                          ▼
                   ┌──────────────┐
                   │  Audio input  │  WebSocket binary frames (PCM/opus)
                   │  (browser    │  or local ALSA/PulseAudio capture
                   │   MediaStream│
                   │   API)       │
                   └──────┬───────┘
                          │
                          ▼
                   ┌──────────────┐
                   │  Whisper STT │  whisper-large-v3 (via faster-whisper
                   │  (GPU)       │  or Ollama if supported)
                   └──────┬───────┘
                          │  text transcript
                          ▼
                   ┌──────────────┐
                   │  Kai Brain   │  brain.run_stream()
                   │  (GPU)       │
                   └──────┬───────┘
                          │  response text
                          ▼
                   ┌──────────────┐
                   │  Sesame      │  CSM-1B TTS
                   │  CSM-1B      │  `[CONFIRM: needs current web search
                   │  (GPU)       │   — Sesame CSM-1B inference server
                   └──────┬───────┘    setup, API, Ollama support status]`
                          │  audio (PCM/opus)
                          ▼
                   ┌──────────────┐
                   │  Audio output │  WebSocket binary frames → phone speaker
                   │  (browser    │  or local ALSA/PulseAudio playback
                   │   AudioContext│
                   │   API)       │
                   └──────────────┘
```

**Phone voice flow:** Requires a WebSocket endpoint for bidirectional audio streaming. This is the one case where WS is justified over SSE (binary audio frames, bidirectional). Add `ws/voice/{session_id}` endpoint in Phase 4.

---

## Risk Register

| # | Risk | Phase | Likelihood | Impact | Mitigation |
|---|------|-------|------------|--------|------------|
| 1 | **GPU driver/kernel version drift in LXC.** Host kernel driver updates but container userspace libs don't match → Ollama/CUDA fails. | 1 | M | H | Pin NVIDIA driver version on host. After host driver update, immediately update container userspace libs. Add a startup check script: `nvidia-smi` must succeed or Kai refuses to start with a clear error. |
| 2 | **Resizable BAR unavailable.** Z370/B360 boards may not support ReBAR. RTX 5060 Ti performance may be reduced without it. | 1 | H | L | ReBAR primarily affects gaming; impact on LLM inference is minimal (inference is compute-bound, not PCIe-bandwidth-bound). Verify with benchmarks after setup. No mitigation needed unless benchmarks show >5% regression. `[CONFIRM: needs current web search — ReBAR impact on LLM inference specifically]` |
| 3 | **Tailscale ACL misconfiguration.** Wrong ACL exposes Kai to unintended Tailscale users or exposes Ollama port. | 2 | L | H | Use the ACL template in Phase 2 section. Test by attempting to connect from a non-tagged device — must be rejected. Audit ACLs quarterly. |
| 4 | **PC daemon command injection.** Malformed JSON-RPC payload escapes the allowlist and executes arbitrary commands. | 3 | L | H | Hardcoded command allowlist (not configurable at runtime). No `subprocess.Popen(shell=True)`. All commands are predefined functions, not string-interpolated shell commands. Params are validated against schemas before use. |
| 5 | **Model OOM with co-resident voice models.** Kai LLM + Whisper + CSM-1B exceed 15.5 GB usable VRAM. | 4 | M | M | Default to sequential loading (load/unload per phase). Test co-resident only after measuring actual VRAM usage on hardware. Ollama `OLLAMA_KEEP_ALIVE=0` for voice models. |
| 6 | **Self-signed TLS cert friction on mobile.** Phone browsers show scary warnings, may block `EventSource`/`fetch` to self-signed origins. | 1 | M | M | On first connect: user taps through the warning once. On iOS Safari, must manually trust the cert in Settings. Document the one-time setup steps. In Phase 2, Tailscale HTTPS certs eliminate this entirely. |
| 7 | **LXC privileged container security.** Privileged LXC for GPU passthrough means container root = host root. Container escape = host compromise. | 1 | L | H | Run Kai as a non-root user inside the container. Minimize installed packages. No SSH into the container from the internet (Tailscale only). Long-term: investigate unprivileged LXC with GPU passthrough (may become possible with future NVIDIA driver updates). |
| 8 | **Ollama native API breaking changes.** Ollama updates `/api/chat` response format, breaking Brain's parser. | 1-4 | L | M | Pin Ollama version. Test Ollama updates in a staging container before applying to production. Brain's parser is isolated in `brain.py` — single file to update if API changes. |
| 9 | **GDDR7 / Blackwell driver maturity.** RTX 5060 Ti is new (2025). Linux driver support may have bugs or missing features. | 1 | M | M | Use the latest NVIDIA driver branch (560+). Check NVIDIA forums and Proxmox forums for known issues before installing. `[CONFIRM: needs current web search — RTX 5060 Ti Linux driver status, Proxmox compatibility reports]` |
| 10 | **WoL fails across VLANs/subnets.** If Gaming PC and Kai LXC are on different subnets, broadcast WoL packets won't reach the PC. | 3 | M | L | Ensure both are on the same L2 network segment. If VLANs are in use, configure a directed broadcast on the router, or use a WoL relay. Alternatively, send WoL from a device on the same subnet as the gaming PC. |

---

## Open Questions

These must be answered before Phase 1 implementation can begin:

- [ ] **Contents of existing frontend HTML files.** What's in `kai/static/`? The plan assumes a new mobile-first page will be created, but the existing desktop UI structure may influence decisions (shared components, shared CSS, etc.).
- [ ] **Brain class exact signatures and event subscription mechanism.** Specifically: how does `brain.run_stream()` handle `session_id` assignment? Is it set on the Brain instance or passed per-call? The Phase 1 SSE endpoint needs to wire events correctly.
- [ ] **Is the Proxmox host a fresh install or existing?** Affects: disk partitioning, existing VMs/containers to preserve, network bridge configuration.
- [ ] **Network topology / subnet / VLAN situation.** Affects: LXC network config, CORS origins, WoL routing (Phase 3), Tailscale placement (Phase 2).
- [ ] **RTX 5060 Ti — do you already have the card, or is it on order?** Affects timeline. If not in hand, Phase 1 can begin with the existing GPU (memory says RTX with 8 GB VRAM — models will need to stay at current sizes until the 5060 Ti arrives).
- [ ] **Existing `users` table schema.** The auth system uses `session_tokens` and `users` tables. Need to understand the current user model to decide where to store the API key hash.
- [ ] **`requirements.txt` or equivalent.** Need the dependency list to replicate the environment inside LXC.

---

## Glossary

| Term | Definition |
|------|------------|
| **LXC** | Linux Containers. OS-level virtualization that shares the host kernel. Lower overhead than a full VM. Used here to isolate Kai while allowing direct GPU access. |
| **IOMMU** | Input/Output Memory Management Unit. Hardware feature that allows safe device passthrough to VMs/containers by isolating device DMA. Required for GPU passthrough. Intel's implementation is called VT-d. |
| **ReBAR** | Resizable BAR (Base Address Register). PCIe feature that allows the CPU to access the full GPU VRAM in one mapping instead of 256 MB windows. Requires motherboard firmware + GPU driver support. Primarily benefits gaming; minimal impact on LLM inference. |
| **SSE** | Server-Sent Events. HTTP-based protocol for unidirectional server→client streaming. The server sends `data:` lines over a long-lived HTTP response. Browser API: `EventSource`. Simpler than WebSockets for one-way data. |
| **JWT** | JSON Web Token. A signed token encoding claims (user ID, expiration). Not used in this project — listed for contrast. Kai uses opaque session tokens stored in the DB instead, which are simpler and revocable. |
| **WoL** | Wake-on-LAN. Network standard that allows a machine to be powered on remotely by sending a "magic packet" (6x `0xFF` + 16x target MAC address) to the broadcast address. Requires NIC and BIOS support. |
| **Tailscale** | Mesh VPN built on WireGuard. Creates a private network (tailnet) between your devices without port forwarding or dynamic DNS. Traffic is encrypted end-to-end. Used here instead of exposing Kai to the public internet. |
| **ACL** | Access Control List. In Tailscale, a JSON policy defining which devices can talk to which other devices on which ports. Used to restrict access so only the phone can reach Kai. |
| **HMAC** | Hash-based Message Authentication Code. A cryptographic signature using a shared secret key. Used by the PC daemon to verify that commands came from Kai and were not tampered with. |
| **mTLS** | Mutual TLS. Both client and server present certificates to authenticate each other. An alternative to HMAC for the PC daemon auth — more complex to set up but eliminates the shared-key management problem. Listed as an alternative in Phase 3. |
