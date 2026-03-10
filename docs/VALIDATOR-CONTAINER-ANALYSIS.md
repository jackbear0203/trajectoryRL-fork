# Validator Container Analysis

Analysis of `Dockerfile.validator` and `entrypoint-validator.sh`.

## Processes Overview

The container runs three processes managed by a single entrypoint script:

| # | Process | Command | Port | Mode |
|---|---------|---------|------|------|
| 1 | mock-tools (ClawBench) | `python -m mock_tools.server` | 3001 | Background |
| 2 | OpenClaw Gateway | `node dist/index.js gateway --allow-unconfigured --bind loopback` | 18789 | Background |
| 3 | Validator | `python -u neurons/validator.py` | — | Foreground (PID 1 via `exec`) |

## User Roles

**All three processes run as `root`.**

- The Dockerfile (Stage 2) uses `python:3.10-slim` as base image, default user is `root`.
- No `USER` directive in the Dockerfile.
- No `adduser` / `useradd` to create other users.
- No privilege-dropping in entrypoint (`su`, `gosu`, `runuser`, etc.).
- `chmod -R 777 "$WORKSPACE_DIR"` confirms root execution context.

## Installation Directories (under `/app`)

| Component | Path | Notes |
|-----------|------|-------|
| OpenClaw | `/app/openclaw` | Copied from Stage 1 builder (`/openclaw`) via `COPY --from=openclaw-builder` |
| OpenClaw entry | `/app/openclaw/dist/index.js` | Compiled JS output |
| ClawBench | `/app/clawbench` | Python package, installed via `pip install -e .` |
| mock-tools | `/app/clawbench` | Part of ClawBench, run as `python -m mock_tools.server` |
| Validator | `/app/neurons/validator.py` | Python script |
| Fixtures | `/app/clawbench/fixtures` | Scenario fixtures |
| Scenarios | `/app/clawbench/scenarios` | Scenario definitions |

## Runtime Directories (outside `/app`)

| Directory | Created By | Purpose |
|-----------|-----------|---------|
| `/workspace` | Dockerfile + entrypoint | Runtime workspace, `chmod 777` |
| `/openclaw-home` | Entrypoint (`$OPENCLAW_HOME`) | OpenClaw home dir |
| `/openclaw-home/.openclaw` | `init_workspace.py` | OpenClaw state/config dir |

## OpenClaw Config Resolution

Both sides now use `OPENCLAW_HOME` to resolve the config path consistently.

### Write side: `init_workspace.py`

```python
OPENCLAW_HOME = Path(os.environ.get("OPENCLAW_HOME", str(Path.home())))
OPENCLAW_CONFIG_DIR = OPENCLAW_HOME / ".openclaw"
```

### Read side: OpenClaw gateway (Node.js)

Resolution chain (`src/infra/home-dir.ts` → `src/config/paths.ts`):

1. `OPENCLAW_HOME=/openclaw-home` (set in entrypoint)
2. `resolveRequiredHomeDir()` → `/openclaw-home`
3. `resolveStateDir()` → `/openclaw-home/.openclaw`

### Resolved Path

| Scenario | `OPENCLAW_HOME` | Config Path |
|----------|----------------|-------------|
| Docker container | `/openclaw-home` (env var) | `/openclaw-home/.openclaw/openclaw.json` |
| No env var set | `Path.home()` / `os.homedir()` | `~/.openclaw/openclaw.json` |

## Build Stages

### Stage 1: `openclaw-builder` (node:22-bookworm)

- Installs Bun + pnpm
- Builds OpenClaw in `/openclaw`
- Strips source files, keeps `dist/` + `node_modules/`

### Stage 2: Runtime (python:3.10-slim)

- Installs Node.js 22, git, curl
- Installs Python dependencies (`pyproject.toml`, `requirements.txt`, ClawBench deps)
- Copies pre-built OpenClaw from Stage 1
- Sets environment defaults and entrypoint

## Startup Sequence

1. **Environment setup** — export paths and API keys
2. **Init workspace** — `python /app/clawbench/scripts/init_workspace.py`
3. **Start mock-tools** — background, health-check loop (30s timeout)
4. **Start OpenClaw** — background, health-check loop (60s timeout)
5. **Register signal handler** — `trap cleanup EXIT INT TERM`
6. **Start validator** — foreground via `exec` (replaces shell as PID 1)
