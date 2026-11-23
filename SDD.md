# SDD — ACE-Step UI

## Context
AI music generation interface for ACE-Step 1.5 model. Runs on the Ubuntu 24.04 bare metal server with RTX 3070 GPU (since 22-11-2026; was Windows). Provides a web UI accessible at music.agenc.ia, integrated into Mission Monitor.

## Architecture
```
Browser → music.agenc.ia (Caddy)
    │
    ├── Frontend: React + Vite + TypeScript (port 3000)
    │
    └── Backend: Express + TypeScript (port 3001)
            └── @gradio/client → ACE-Step API (port 8001, external process)

Legacy Windows startup scripts (discontinued 22-11-2026; the Linux server runs systemd units `acestep-backend` + `acestep-frontend`, code in `/opt/audio/ace-step-ui`, ACE-Step in `/opt/audio/ACE-Step-1.5`; `start-all.sh`/`start.sh` exist for manual use):
    ├── start-all.bat  — launches all 3 services (ACE-Step API + backend + frontend)
    ├── start.bat      — frontend + backend only
    └── setup.bat      — install dependencies
```

## Stack
| Layer | Technology |
|-------|-----------|
| Frontend | React 19 + Vite 6 + TypeScript |
| API Server | Express 4 + TypeScript (tsx/ts-node) |
| AI Backend | ACE-Step 1.5 (Gradio, external process — not in this repo) |
| Auth | JWT (jsonwebtoken) |
| DB | SQLite (better-sqlite3) |
| GPU | RTX 3070 8GB VRAM |
| OS | Ubuntu 24.04 bare metal (was Windows Server until 22-11-2026) |
| Deploy | systemd (`acestep-backend.service`, `acestep-frontend.service`); legacy: Windows CMD scripts (start-all.bat) |
| Proxy | Caddy → music.agenc.ia |

## Key Files
- `App.tsx` — Root React component (frontend entrypoint at repo root)
- `components/` — UI components (Player, CreatePanel, LibraryView, TrainingPanel, etc.)
- `services/api.ts` — Frontend API client
- `services/geminiService.ts` — Gemini integration
- `context/` — AuthContext, I18nContext, ResponsiveContext
- `vite.config.ts` — Vite build config
- `server/src/index.ts` — Express server entrypoint
- `server/src/routes/` — API routes (generate, songs, playlists, lora, training, users, auth)
- `server/src/services/acestep.ts` — ACE-Step Gradio client wrapper
- `server/src/services/generationQueue.ts` — Generation queue management
- `server/src/db/` — SQLite pool and migrations
- `start-all.bat` — Starts ACE-Step API (external) + backend + frontend
- `start.bat` — Starts backend + frontend only
- `setup.bat` — Dependency installation

## API / Endpoints
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/generate` | Trigger music generation |
| GET | `/api/songs` | List generated songs |
| GET | `/api/playlists` | List playlists |
| POST | `/api/training` | Start LoRA training |
| GET | `/api/lora` | List LoRA models |
| POST | `/api/auth` | Authentication |
| GET | `/api/health` | Health check |
| POST | `/generation_wrapper` | Gradio direct (port 8001, internal — external process) |

### Key Gradio Parameters (proxied to ACE-Step)
- `prompt` — text description of music
- `duration` — output length (seconds)
- `steps` — inference steps
- `cfg_scale` — guidance scale

## Deploy
```
Linux (actual, desde 22-11-2026): systemctl status acestep-backend acestep-frontend
Windows (CMD, descontinuado):
  start-all.bat
    → ACE-Step API (external, port 8001) — requires ACESTEP_PATH set or ..\ACE-Step-1.5
    → npm run dev (server: tsx src/index.ts, port 3001)
    → npm run dev (frontend: vite, port 3000)

Ports (Linux: backend/frontend listen on 127.0.0.1, exposed only via Caddy):
  3000 (Frontend), 3001 (Backend API), 8001 internal only
  (Windows firewall rules TCP 3000/3001 applied only to the old Windows host)

Caddy:
  music.agenc.ia → localhost:3000
```

## Integration
- **Mission Monitor**: embedded as iframe in `#tools/music`
- **Caddy**: TLS + reverse proxy on the server (native `caddy.service`, /etc/caddy/Caddyfile)
- **ACE-Step**: external Gradio process (not in repo); path set via `ACESTEP_PATH` env var
- **GPU**: ~6GB VRAM per generation, ~2 min per 1 min of audio
- **Windows Firewall** (legacy, Windows host only): TCP 3000 + 3001 rules required (Python exe may trigger auto-block rules)

## Known Issues
- (Legacy, Windows host) auto-block firewall rules for Python executables — verify firewall before diagnosing port issues
- start-all.bat requires ACESTEP_PATH env var or ACE-Step-1.5 directory adjacent to repo
- RTX 3070 8GB limits concurrent generation; generationQueue.ts serializes requests
- Gradio API surface is brittle against ACE-Step version upgrades
- (Legacy, Windows host) SSH commands >10s to the Windows server had to use tmux; on Linux use `srv '<cmd>'`

## Verification
- `GET /api/health` → 200
- Submit test prompt → verify audio file returned within ~2 min
- Check GPU utilization via Task Manager during generation (~6GB VRAM)
- Confirm all services running: check open ports 3000, 3001, 8001
