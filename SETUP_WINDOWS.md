# Kai — Windows Setup Guide

A step-by-step guide for getting Kai running on Windows 10 / 11.

If you just want the fast path, jump to the [TL;DR](#tldr) at the bottom. The
easiest route of all is the [one-click `start.bat`](#easiest-one-click-startbat).

---

## Requirements

- **Python 3.12+**
- **Ollama** (Kai auto-starts it if not running)
- **A GPU is recommended** (NVIDIA or AMD, 8 GB VRAM). CPU-only works, just slower.
- ~15 GB free disk for the models.

---

## 1. Install Python

1. Download Python 3.12+ from <https://www.python.org/downloads/>.
2. Run the installer.
3. **IMPORTANT:** on the first screen, tick **"Add python.exe to PATH"** before
   clicking *Install Now*. If you skip this, the commands below won't be found.

Verify in a new PowerShell or Command Prompt window:

```powershell
python --version
```

> If `python` opens the Microsoft Store instead of printing a version, disable the
> Store alias: **Settings → Apps → Advanced app settings → App execution aliases**,
> and turn off `python.exe` / `python3.exe`. Then reinstall with "Add to PATH" ticked.

---

## 2. Install Ollama

1. Download the installer from <https://ollama.com/download>.
2. Run it (installs to your user profile, no admin needed).
3. Ollama starts automatically and lives in the system tray.

Verify in a new terminal:

```powershell
ollama --version
```

### GPU acceleration

- **NVIDIA:** install the latest GeForce/Studio driver. Ollama detects CUDA automatically.
- **AMD:** install the latest Adrenalin driver. Ollama detects supported cards automatically.
- **No GPU:** everything still works on CPU, just slower.

---

## 3. Get the code

Using Git (recommended):

```powershell
git clone https://github.com/supasoulja/newb_agent
cd newb_agent
```

No Git? Download the repo ZIP from GitHub, extract it, and `cd` into the folder.

---

## Easiest: one-click `start.bat`

The repo ships a launcher that does everything below for you — checks Python,
installs dependencies, starts Ollama, pulls the models, and launches Kai:

```
Double-click start.bat
```

(or run `.\start.bat` from a terminal). It auto-detects a `.venv` if one exists.
The first run takes a while because it downloads several GB of models.

If you'd rather do it by hand, continue with the manual steps below.

---

## 4. Create and activate a virtual environment

A venv keeps Kai's packages isolated from your system Python.

```powershell
python -m venv .venv
.venv\Scripts\activate
```

After activating, your prompt shows `(.venv)`.

> **PowerShell "running scripts is disabled" error?** Allow local scripts for
> your user once:
> ```powershell
> Set-ExecutionPolicy -Scope CurrentUser RemoteSigned
> ```
> Then run `.venv\Scripts\activate` again. (In Command Prompt / `cmd.exe` use
> `.venv\Scripts\activate.bat` instead — no policy change needed.)

---

## 5. Install Python dependencies

With the venv **active** (you see `(.venv)`):

```powershell
pip install -r requirements.txt
```

This includes `pywebview` (the native window) and `keyboard` (the global
hotkey), both of which work out of the box on Windows.

---

## 6. Pull the AI models

```powershell
ollama pull qwen3.5:9b            # primary model (~6.3 GB) — required

# Optional extras:
ollama pull qwen3:8b              # reasoning model (~6.0 GB)
ollama pull qwen3-embedding:4b    # HQ embedding for shutdown re-embed (~2.5 GB)
```

Only the first model is needed to start.

---

## 7. Run Kai

With the venv active:

```powershell
python app.py     # native desktop app (recommended) — window, tray, hotkey
# or
python web.py     # browser UI at http://localhost:7860
# or
python cli.py     # terminal REPL
```

On first run, Kai downloads a small (~25 MB) ONNX embedding model and prompts
you to register an account (name + PIN).

### 8 GB VRAM tip

Halve the KV-cache memory usage by setting this before launching:

```powershell
$env:OLLAMA_KV_CACHE_TYPE = "q8_0"
python app.py
```

---

## Coming back later

Every new terminal session, re-activate the venv first:

```powershell
cd newb_agent
.venv\Scripts\activate
python app.py
```

Or just double-click `start.bat`, which handles it.

---

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| `'python' is not recognized` | Python isn't on PATH. Reinstall and tick "Add python.exe to PATH", then open a **new** terminal. |
| `python` opens Microsoft Store | Disable the `python.exe` app execution alias (see step 1). |
| `activate ... cannot be loaded because running scripts is disabled` | `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned`, or use `cmd.exe`. |
| `ollama` not recognized | Reopen the terminal after installing Ollama so PATH refreshes. |
| Desktop app won't launch / silent exit | `pythonw.exe` hides errors. Run `python app.py` in a terminal to see them. Kill stale Python on port 7860 first: `taskkill /F /IM python.exe`. |
| Port 7860 already in use | `netstat -ano | findstr :7860`, then `taskkill /F /PID <pid>`. |
| Pull is very slow | Models are several GB — give it time, and keep a stable connection. Re-run `ollama pull qwen3.5:9b` to resume. |

---

## TL;DR

```powershell
# 1. Install Python 3.12+ (tick "Add to PATH") and Ollama (ollama.com/download)

# 2. Code
git clone https://github.com/supasoulja/newb_agent
cd newb_agent

# 3. Venv
python -m venv .venv
.venv\Scripts\activate

# 4. Deps + model
pip install -r requirements.txt
ollama pull qwen3.5:9b

# 5. Run
python app.py        # or: python web.py
```

…or just double-click **`start.bat`** and let it do all of the above.
