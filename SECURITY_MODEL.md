# Threat model: CLIProxyAPI + Management Center

Repositories: `ckrkmz/CLIProxyAPI` (Go server, this repository), `ckrkmz/Cli-Proxy-API-Management-Center` (React UI). No separate infrastructure repository exists; the only deployment description is `CLIProxyAPI/Dockerfile` and `docker-compose.yml` / `docker-compose.cluster.yml`.

## System overview and trust boundaries

CLIProxyAPI is a self-hosted proxy that wraps upstream AI accounts (Claude Code, Codex/ChatGPT, Antigravity/Gemini, Grok, Muse Code, Devin, ...) authenticated through OAuth or provider API keys. It exposes them as OpenAI/Claude/Gemini/Codex-compatible APIs. The Management Center is a single-file React app (`management.html`) that the server hosts. It drives the server through the `/v8/management` API using the management key.

Trust boundaries:
1. **Internet → proxy API**: the proxy is reachable from the public internet (user-confirmed). Only holders of a client API key (`access.api-keys`) get past `AuthMiddleware` (`internal/api/server_middleware.go`, `sdk/access`).
2. **Network → management API**: `/v8/management/*` sits behind `Handler.Middleware` (`internal/api/handlers/management/handler.go`). That middleware requires the management key on every request and rejects non-localhost clients unless `remote-management.allow-remote` is set. Five failures from one IP trigger a 30-minute ban.
3. **Proxy → upstream providers**: requests leave the proxy carrying stored upstream credentials.
4. **Browser (Management Center) → management API**: the management key is held in browser storage, using only reversible obfuscation (`src/services/storage/secureStorage.ts`).
5. **Server → local disk / storage backends**: auth files are stored under `auths/` or `~/.cli-proxy-api`. Postgres, git, or object stores are optional (`internal/store`).

## Components

| Component | Code |
|---|---|
| **Proxy API** | `CLIProxyAPI/internal/api/server_routes.go`, `server_middleware.go`, `sdk/api`, `internal/runtime/executor`, `internal/translator`, `internal/thinking`, `internal/wsrelay`, `internal/wstunnel` |
| **Client Auth** | `CLIProxyAPI/internal/access`, `sdk/access`, `internal/safemode` |
| **Management API** | `CLIProxyAPI/internal/api/handlers/management`, `internal/api/server_management_v8.go`, `internal/managementasset` |
| **Credential Store & OAuth** | `CLIProxyAPI/internal/auth`, `sdk/auth`, `internal/store`, OAuth callback routes (`/anthropic/callback`, `/codex/callback`, `/antigravity/callback`, `/callback`, `/devin/callback`, `/v8/management/oauth/callback`) |
| **Server Core & Integrations** | `CLIProxyAPI/cmd`, `internal/config`, `internal/watcher`, `internal/home`, `internal/redisqueue`, `internal/registry`, `internal/usage`, `internal/logging`, other `internal/*` |
| **Management Center UI** | `Cli-Proxy-API-Management-Center/src/**` |
| **Out-of-scope components** | `/v0/management` (`internal/api/server_management.go`), plugins (`internal/plugin*`, `sdk/plugin*`, `src/features/plugins`), TUI and local CLI tooling (`internal/tui`, `cmd/fetch_*`, `cmd/validate_*`) |

## Entry points and who can reach them

- `/v1`, `/v1beta`, `/openai/v1`, `/backend-api/codex`, `/muse-code`, and the realtime/WebSocket routes: anyone on the internet can reach them. They need a client API key.
- `/healthz`, `/`, `/management.html`, and the OAuth callback routes: the internet can reach them without authentication.
- `/v8/management/*`: needs the management key. The user reaches the Management Center **through a reverse proxy or tunnel on the same host** (user-confirmed). Unless `trusted-proxies` is configured correctly, remote requests arrive from 127.0.0.1. They then count as local clients, which skips the `allow-remote` check and allows the local-password path. In practice, the management API and `/management.html` should be treated as **reachable from the internet, guarded by the management key alone**.
- OAuth callback listener ports 8085, 1455, 54545, 51121, and 11451 are published in `docker-compose.yml`. They are **publicly reachable** in the user's deployment (user-confirmed).

## Assets and sensitivity

1. **Upstream OAuth tokens, auth files, and provider API keys**: highest value. They grant use of the user's paid AI accounts.
2. **Management key** and **client API keys**.
3. **Request/response content and logs**: prompts and completions may contain sensitive data. Request logs and error logs are kept.
4. Config integrity: whoever controls the config controls routing, upstream URLs, and keys.
5. Availability and quota of the upstream accounts.

## Attacker profiles

- **Unauthenticated internet attacker** (primary): reaches the proxy port, the unauthenticated routes, the publicly published OAuth callback ports, and (through the user's same-host reverse proxy) the management API, which accepts the management key without a localhost restriction. Goals: bypass client auth, steal credentials, abuse the proxy (open relay, SSRF), read logs, or cause DoS.
- **Malicious website in the operator's browser**: CSRF, cross-origin requests to the management API (CORS is `*`), or XSS in the Management Center leading to theft of the management key.
- **Compromised or malicious upstream provider response**: data rendered in the UI, or parsed by translators.
- **Client API-key holders**: only the user holds these keys, and they are **fully trusted** (user-confirmed). Abuse that requires a valid client API key is not a threat in this deployment.

## Existing controls

- Client API-key auth on all proxy route groups. Safe mode blocks the proxy while example API keys are configured (`exampleAPIKeySafeModeMiddleware`).
- Management key on every management request (hashed when stored in config, constant-time compare), localhost-only by default, IP ban after 5 failures.
- Request body size limit (`maxRequestBodyBytesMiddleware`). The project convention forbids logging secrets.
- In the UI: `window.open(..., 'noopener,noreferrer')` for OAuth URLs, and all calls go through the central `apiClient`.

## Out of scope / accepted risk (user-confirmed)

- Deprecated `/v0/management` endpoints.
- Plugins: only the user installs them, so they are trusted code.
- TUI and local CLI tooling.
- Anything that requires a valid client API key, since key holders are trusted (the user only).

## Open questions

- Is `trusted-proxies` set for the reverse proxy, so the real client IP is used?
- Is TLS terminated in front of the proxy?
