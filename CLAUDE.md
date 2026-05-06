# CC Gateway — Project Instructions

CC Gateway is a reverse proxy that sits between Claude Code and `api.anthropic.com`, normalizing device identity, environment fingerprints, and process telemetry so a single Anthropic account can be used from multiple machines without triggering account bans. It centralizes OAuth token refresh and strips the `x-anthropic-billing-header` (matching `CLAUDE_CODE_ATTRIBUTION_HEADER=false`) for cross-session prompt-cache sharing.

## Origin & Ownership

This is an **adopted third-party tool** (upstream owner: `motiful`), MIT-licensed. It is integrated into the AIBC platform at the deployment layer only — it is **not** a first-party AIBC component, does not carry AIBC branding, and does not participate in platform-level disciplines (papa-git trailers, FYI.md journals, CPM channels). Treat it like any other vendor tool: keep changes focused on configuration and local fixes; prefer upstream PRs for feature work.

## Tech Stack

- TypeScript (Node.js 22+, ESNext modules, strict mode)
- Runtime deps: `https-proxy-agent` (Clash/V2Ray support), `yaml`
- Dev: `tsx` (watch mode), `typescript` 5.7+
- Deploy: Docker + Docker Compose (multi-stage build, PM2 in production)
- Default port: `8443`
- Version: 0.2.0 (alpha)

## Build / Run / Test

```bash
npm install                          # install deps
npm run dev                          # tsx watch (auto-reload)
npm run build                        # compile to dist/
npm test                             # rewriter test suite (16 cases)
npm start                            # run compiled dist/index.js
npm run generate-identity            # mint a 64-char canonical device_id
npm run generate-token               # mint a client auth token

bash scripts/quick-setup.sh          # one-command local dev setup
bash scripts/admin-setup.sh          # interactive Docker production deploy
bash scripts/add-client.sh <name>    # generate a new client launcher
bash scripts/extract-token.sh        # extract OAuth from Keychain or ~/.claude/.credentials.json

docker compose up -d --build         # production deploy
docker compose logs -f               # tail logs
docker compose restart               # apply config.yaml changes
```

## Architecture

| Module | Purpose |
|---|---|
| `src/index.ts` | Entry: load config, init OAuth, start proxy |
| `src/proxy.ts` | HTTP/HTTPS server; auth → rewrite → upstream stream (handles SSE) |
| `src/rewriter.ts` | Body / header / event-log rewrites: identity, env, process metrics, billing-header strip, CCH (billing hash) |
| `src/oauth.ts` | OAuth lifecycle: reuse Keychain token at startup, auto-refresh 5 min before expiry, 30 s retry on failure |
| `src/auth.ts` | Client auth via `x-api-key` header (matches CC's native header) |
| `src/config.ts` | Strict YAML config validation |
| `src/proxy-agent.ts` | Outbound HTTPS proxy support (Clash, V2Ray) via `HTTPS_PROXY` / `HTTP_PROXY` / `ALL_PROXY` |
| `src/logger.ts` | Structured logging + audit trails |

**Configuration**: `config.yaml` (template: `config.example.yaml`). Sections: `server`, `upstream`, `oauth`, `auth.tokens`, `identity`, `env`, `prompt_env`, `process`, `logging`. No identity values are hardcoded — everything is declarative.

**Endpoints**:
- `/_health` — no auth, JSON status (used by Docker healthcheck)
- `/_verify` — auth required, returns inspected rewrites without forwarding (debug aid)
- All other paths → authenticated → rewriter → `api.anthropic.com`

## Conventions

- Strict TypeScript (`strict: true`, `declaration: true`)
- ESNext modules (`type: "module"`)
- Logging via `logger.ts` (debug/info/warn/error, ISO timestamps)
- Config-as-code: identity, env, process metrics all in `config.yaml`
- Streams responses (SSE) directly — preserve upstream behavior
- Tests cover identity rewriting, env normalization, event-logging payload rewrites, billing-header handling

## Sibling Repos in the AIBC Platform

| Repo | How it relates |
|---|---|
| `claude_code_router` | Alternative path: routes CC to non-Anthropic providers. Choose **one** per CC client — gateway = Anthropic with privacy; router = multi-provider. They are **not** chained in production. |
| `claude-code-config` | The CC client whose traffic this gateway handles. Gateway is configured at the launcher level (per-client tokens), not in `~/.claude/`. |
| `SCRIM`, `opencode_CUSTOM`, `firecracker_MicroVMs` | Orthogonal — different layers of the stack (secrets, alt agent CLI, sandbox VM). |

## Caveats

- `mcp-proxy.anthropic.com` is hardcoded in CC and bypasses this gateway — documented behavior, not a bug
- New CC telemetry fields may not be covered by the rewriter; in production, monitor Clash REJECT logs
- Refresh-token expiry (rare, months) requires re-running `extract-token.sh` on the admin machine
- Alpha-status — test against a non-primary account before pointing a primary account through it

## Security Stance

- No fake locations, no IP rotation, no rate-limit bypass — this is a privacy/identity-normalization tool, not an evasion tool
- Single canonical IP/device per organization (Anthropic sees one device)
- Billing-header strip matches the official `CLAUDE_CODE_ATTRIBUTION_HEADER=false` knob — not a bypass
- Never commit `config.yaml`, `clients/`, `certs/`, `.env`, or extracted OAuth tokens (`.gitignore` enforces this)
