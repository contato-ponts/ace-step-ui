# SDD — ACE-Step UI

## Context
AI music generation interface for ACE-Step 1.5 model. Runs on Windows server with RTX 3070 GPU. Provides a web UI accessible at music.agenc.ia, integrated into Mission Monitor.

## Architecture
```
Browser → music.agenc.ia (Caddy)
    │
    ├── Frontend: React (port 3000)
    │       └── Express static server (port 3001)
    │
    └── Gradio Backend: ACE-Step 1.5 (port 8001)
            └── /generation_wrapper (73 params)

Windows Scheduled Tasks:
    ├── ACEStepAPI     (ONLOGON)   — Gradio Python process
    ├── ACEStepBackend (ONSTART)   — Express server
    └── ACEStepFrontend (ONSTART)  — React dev/static server
```

## Stack
| Layer | Technology |
|-------|-----------|
| Frontend | React |
| API Server | Express.js (Node) |
| AI Backend | Gradio + ACE-Step 1.5 (Python) |
| GPU | RTX 3070 8GB VRAM |
| OS | Windows Server |
| Deploy | Windows Scheduled Tasks |
| Proxy | Caddy → music.agenc.ia |

## Key Files
- `frontend/src/App.jsx` — Main React UI, generation form
- `frontend/src/api.js` — Calls to Express API layer
- `server/index.js` — Express proxy to Gradio
- `start_api.bat` / `start_backend.bat` / `start_frontend.bat` — Launch scripts for Scheduled Tasks
- `gradio_app.py` — ACE-Step Gradio wrapper

## API / Endpoints
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/generate` | Trigger music generation (proxied to Gradio) |
| GET | `/api/status` | Job status |
| GET | `/api/health` | Health check |
| POST | `/generation_wrapper` | Gradio direct (73 params, internal) |

### Key Gradio Parameters
- `prompt` — text description of music
- `duration` — output length (seconds)
- `steps` — inference steps
- `cfg_scale` — guidance scale
- 69 additional generation parameters

## Deploy
```
Windows Scheduled Tasks:
  ACEStepAPI      → ONLOGON → python gradio_app.py
  ACEStepBackend  → ONSTART → node server/index.js
  ACEStepFrontend → ONSTART → node/serve frontend

Firewall rules:
  TCP 3000 inbound (Frontend)
  TCP 3001 inbound (Backend API)
  (Port 8001 internal only)

Caddy:
  music.agenc.ia → localhost:3000
```

## Integration
- **Mission Monitor**: embedded as iframe in `#tools/music`
- **Caddy**: TLS + reverse proxy on Windows server
- **GPU**: ~6GB VRAM per generation, ~2 min per 1 min of audio
- **Windows Firewall**: TCP 3000 + 3001 rules required (Python exe may trigger auto-block rules)

## Known Issues
- Windows auto-block firewall rules for Python executables — verify firewall before diagnosing port issues
- Scheduled Tasks with ONLOGON require active user session on Windows server
- RTX 3070 8GB limits concurrent generation; no queue implemented
- Gradio 73-param API surface is brittle against ACE-Step version upgrades

## Verification
- `GET /api/health` → 200
- Submit test prompt → verify audio file returned within ~2 min
- Check GPU utilization via Task Manager during generation (~6GB VRAM)
- Confirm Scheduled Tasks running: Task Scheduler → ACEStepAPI, ACEStepBackend, ACEStepFrontend
