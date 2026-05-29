# Kai — Linux Setup Guide

A step-by-step guide for getting Kai running on Linux (Ubuntu, Mint, Debian,
Fedora, Arch). Tested on Ubuntu 24.04 / Mint 22.

If you just want the fast path, jump to the [TL;DR](#tldr) at the bottom.

> **Why a separate guide?** On modern Debian/Ubuntu/Mint, `pip install` into the
> system Python is blocked (PEP 668 "externally-managed-environment"), and
> `python3 -m venv` fails until you install the `python3-venv` package. This
> guide walks you around both so you never see those errors.

---

## Requirements

- **Python 3.12+**
- **Ollama** (Kai auto-starts it if not running)
- **A GPU is recommended** (AMD or NVIDIA, 8 GB VRAM). CPU-only works, just slower.
- ~15 GB free disk for the models.

---

## 1. Install system packages

You need Python, the `venv` module, `pip`, and `git`. On Debian/Ubuntu/Mint the
`venv` module ships **separately** from Python — this is the step most people miss.

**Ubuntu / Mint / Debian:**

```bash
sudo apt update
sudo apt install -y python3 python3-venv python3-pip git curl
```

**Fedora:**

```bash
sudo dnf install -y python3 python3-pip git curl
```

**Arch:**

```bash
sudo pacman -S --needed python python-pip git curl
```

> ### A note on `python` vs `python3`
> On most Linux distros the command is **`python3`**, not `python`. If you type
> `python` and get *"Command 'python' not found"*, that's expected — just use
> `python3`. (Optional: `sudo apt install python-is-python3` makes `python`
> an alias.)

---

## 2. Install Ollama

Ollama runs the local models. One command:

```bash
curl -fsSL https://ollama.com/install.sh | sh
```

Verify it installed:

```bash
ollama --version
```

The installer also sets up a background service. If `ollama` commands say the
server isn't running, start it in another terminal with `ollama serve` (Kai will
also try to start it automatically).

### GPU acceleration (optional but recommended)

- **NVIDIA:** install the proprietary driver + CUDA. Ollama detects it automatically.
- **AMD:** install ROCm. Ollama detects supported cards automatically.
- **No GPU:** everything still works on CPU, just slower.

---

## 3. Get the code

```bash
git clone https://github.com/supasoulja/newb_agent
cd newb_agent
```

---

## 4. Create and activate a virtual environment

This is the key to avoiding the `externally-managed-environment` error. **Never
`pip install` into your system Python** — always work inside a venv.

```bash
python3 -m venv .venv
source .venv/bin/activate
```

After activating, your prompt shows `(.venv)`. Inside the venv, `python` and
`pip` both work and point at the right place — no `--break-system-packages`, no
`sudo`.

> **If `python3 -m venv .venv` fails** with *"ensurepip is not available"* or
> *"you need to install the python3-venv package"*, you skipped step 1. Run:
> ```bash
> sudo apt install python3.12-venv   # match your Python version
> ```
> then delete the half-made venv and retry: `rm -rf .venv && python3 -m venv .venv`

---

## 5. Install Python dependencies

With the venv **active** (you see `(.venv)`):

```bash
pip install -r requirements.txt
```

> The `keyboard` package (used for the global hotkey) needs root to grab key
> events on Linux, and `pywebview` needs a system WebView. If you only plan to
> use the **web UI** (recommended on Linux), you can ignore hotkey warnings —
> they don't affect `web.py`.

### pywebview on Linux (only needed for the native desktop app)

The native desktop app (`app.py`) needs GTK + WebKit bindings, which come from
the system, not pip:

```bash
# Ubuntu / Mint / Debian
sudo apt install -y python3-gi gir1.2-webkit2-4.1 libgirepository1.0-dev
```

If you don't install these, just use the **web UI** instead — it's the same
interface in your browser and is the smoothest option on Linux.

---

## 6. Pull the AI models

```bash
ollama pull qwen3.5:9b            # primary model (~6.3 GB) — required

# Optional extras:
ollama pull qwen3:8b              # reasoning model (~6.0 GB)
ollama pull qwen3-embedding:4b    # HQ embedding for shutdown re-embed (~2.5 GB)
```

The first model is the only one you need to start. The others enable the
reasoning mode and higher-quality memory embeddings.

---

## 7. Run Kai

With the venv active:

```bash
python web.py     # browser UI at http://localhost:7860  ← recommended on Linux
# or
python cli.py     # terminal REPL
# or
python app.py     # native desktop window (needs the GTK/WebKit packages above)
```

On first run, Kai downloads a small (~25 MB) ONNX embedding model and prompts
you to register an account (name + PIN).

### 8 GB VRAM tip

Halve the KV-cache memory usage by exporting this before launching:

```bash
export OLLAMA_KV_CACHE_TYPE=q8_0
python web.py
```

---

## Coming back later

Every new terminal session, re-activate the venv before running Kai:

```bash
cd newb_agent
source .venv/bin/activate
python web.py
```

---

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| `error: externally-managed-environment` | You ran `pip` outside a venv. Run `source .venv/bin/activate` first (step 4). Don't use `--break-system-packages`. |
| `ensurepip is not available` / `install the python3-venv package` | `sudo apt install python3-venv` (or `python3.12-venv`), then recreate the venv. |
| `Command 'python' not found` | Use `python3`, or `sudo apt install python-is-python3`. |
| `Command 'pip' not found` | Activate the venv (step 4); inside it `pip` exists. System-wide: `sudo apt install python3-pip`. |
| Ollama "connection refused" | Run `ollama serve` in another terminal, or `sudo systemctl start ollama`. |
| Port 7860 already in use | Kill the stale process: `kill $(lsof -t -i:7860)` |
| `app.py` fails / blank window | Install the GTK/WebKit packages in step 5, or just use `python web.py`. |
| Global hotkey (Ctrl+Shift+K) doesn't work | The `keyboard` lib needs root on Linux. Run with `sudo`, or ignore it and use the web UI. |

---

## TL;DR

```bash
# 1. System packages (Ubuntu/Mint/Debian)
sudo apt update && sudo apt install -y python3 python3-venv python3-pip git curl

# 2. Ollama
curl -fsSL https://ollama.com/install.sh | sh

# 3. Code
git clone https://github.com/supasoulja/newb_agent && cd newb_agent

# 4. Venv (avoids the PEP 668 / externally-managed error)
python3 -m venv .venv && source .venv/bin/activate

# 5. Deps + model
pip install -r requirements.txt
ollama pull qwen3.5:9b

# 6. Run
python web.py        # http://localhost:7860
```
