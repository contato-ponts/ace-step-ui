# Windows Setup — ACE-Step UI (Ponts deployment)

Fork de `ace-step/ace-step-ui` integrado ao stack Ponts. Música via ACE-Step 1.5, exposto em `music.agenc.ia` via Cloudflare tunnel.

> Para docs do projeto upstream, ver `README.md`. Este arquivo cobre apenas a deployment Ponts no Windows server.

## Hardware

- Windows 10 Pro 19045+
- Ryzen 3 3200G+ / RAM 16GB / RTX 3070 8GB VRAM
- Tailscale instalado + reachable em `100.x.x.x`

## Portas

- `8001` — Python backend (ACE-Step inference)
- `3000` — Frontend Next.js
- `3001` — Node backend (auth/JWT)

Caddy + Cloudflare tunnel mapeia `music.agenc.ia` → localhost:3000 (frontend) com proxy interno para :3001 (API) e :8001 (inference).

## Instalação

```cmd
git clone https://github.com/contato-ponts/ace-step-ui.git C:\Users\sshuser\ace-step-ui
cd C:\Users\sshuser\ace-step-ui

REM Backend Node (3001)
cd server
npm install
copy .env.example .env
REM Preencher .env (do vault):
REM   NODE_ENV=production
REM   JWT_SECRET=<64-char rotated 2026-06-03>
REM   DATABASE_URL=sqlite:./acestep.db

REM Frontend (3000)
cd ..
npm install
npm run build

REM ACE-Step Python (8001) — submódulo upstream
REM Ver upstream README para venv + model weights download.
```

## Boot via schtasks

Três schtasks separadas (controle independente de restart):

```cmd
schtasks /create /tn "ACEStepBackend" /tr "C:\Users\sshuser\start_acestep_backend.bat" /sc ONSTART /ru SYSTEM
schtasks /create /tn "ACEStepFrontend" /tr "C:\Users\sshuser\ace-step-ui\start_frontend.bat" /sc ONSTART /ru SYSTEM
schtasks /create /tn "ACEStepInference" /tr "C:\Users\sshuser\ACE-Step-1.5\start_inference.bat" /sc ONSTART /ru SYSTEM
```

`start_acestep_backend.bat` usa loop com backoff exponencial (5/10/20/40/60s, cap 60s, reset após 10 falhas) — restart resiliente.

## Health checks

```bash
curl -m 5 http://localhost:3001/health
curl -m 5 http://localhost:8001/health
curl -m 5 https://music.agenc.ia/health
```

## Models

ACE-Step 1.5 weights download via upstream procedure. Cache em `C:\Users\sshuser\ACE-Step-1.5\models\`.

Ver `rpi-server/infra/MODELS.md` (a criar — Tier 3) para SHA256 + versões pinned.

## Disaster recovery

- Stack via `rpi-server/infra/runbook.md` Phase 3.
- Secrets via `rpi-server/infra/VAULT.md` (entry: "ACE-Step JWT_SECRET").
- WoL MAC desktop em `HARDWARE.md`.

## Tools integration

Exposto em Mission Monitor (`tools.agenc.ia`) na seção `#tools/music`. Ver `project_acestep.md` em memory para wiring details.
