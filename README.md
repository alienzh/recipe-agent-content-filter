# Agora Conversational AI — Content Filter Recipe (Python)

[![MIT License](https://img.shields.io/badge/license-MIT-blue.svg)](./LICENSE)
[![python>=3.10](https://img.shields.io/badge/python-%3E%3D3.10-blue)](https://www.python.org/)
[![bun](https://img.shields.io/badge/bun-%E2%89%A51.0-black)](https://bun.sh/)

The **content-filter** recipe in the Agora Conversational AI recipes family. The
agent's LLM stage is pointed at a mock endpoint that echoes the user's words
and then runs a **sentence-level keyword filter** before the text is handed to TTS.
Flagged sentences are replaced with "Content filtered." — so you can hear the
redaction by voice just by saying a banned term. STT (Deepgram nova-3) and TTS
(MiniMax) stay Agora-managed.

**Zero-key:** no LLM API key, no external model account.

## Prerequisites

- [Python 3.10+](https://www.python.org/)
- [Bun](https://bun.sh/)
- [Agora CLI](https://github.com/AgoraIO/cli) (easiest way to get App ID + Certificate)
- [ngrok](https://ngrok.com/) (or any tunnel to expose localhost) — required so
  Agora cloud can reach the `/llm` endpoint on the backend

The same commands work on macOS, Linux, and Windows. On macOS/Linux, setup uses
`python3`; on Windows, it uses the Python launcher (`py`) or `python`. WSL and
virtualenv activation are not required.

## Run It

```bash
# 1. Install + create the server Python venv
bun run setup

# 2. Add Agora credentials (CLI), or edit server/.env.local by hand
agora login
agora project use <your-project>
agora project env write server/.env.local

# 3. Expose the backend publicly (Agora cloud calls /llm/chat/completions)
ngrok http 8000

# 4. Add the tunnel URL to server/.env.local
#    CUSTOM_LLM_URL=https://<your-tunnel>.ngrok-free.dev/llm/chat/completions

# 5. Run the backend and web
bun run dev
```

Open [http://localhost:3000](http://localhost:3000) → **Start Conversation** → speak.
Say "strawberries" (or any term in `FILTER_BANNED_TERMS`) to hear that sentence
redacted.

### Working from a clone

After `bun run setup` and credentials, `bun run dev` starts two services:

| Service | Port | Notes |
| --- | :---: | --- |
| Web (Next.js) | 3000 | Browser UI |
| Agent backend | 8000 | Token generation + agent lifecycle + mock `/llm` endpoint |
| API docs | 8000/docs | FastAPI auto-generated docs |

## Deploy

Deploy `web` (Next.js) and `server` (a single publicly reachable FastAPI
backend). The mock LLM endpoint is mounted at `/llm` in the same process, so
Agora cloud reaches it at `<public-url>/llm/chat/completions`. Set
`AGENT_BACKEND_URL` in the web deployment to point at your deployed backend.

A single-process Docker image is published to
`ghcr.io/AgoraIO-Conversational-AI/recipe-agent-content-filter` on `v*` tags.

> **Co-public caveat:** the server :8000 is now the public endpoint Agora calls
> (`/llm`), so the token endpoints are co-public; the App Certificate is only
> used in-memory to mint tokens (never on the wire); add auth/rate-limiting
> before a real deployment.

```bash
docker run -d -p 8000:8000 \
  -e AGORA_APP_ID=<your-app-id> \
  -e AGORA_APP_CERTIFICATE=<your-certificate> \
  -e CUSTOM_LLM_URL=https://<your-tunnel>.ngrok-free.dev/llm/chat/completions \
  -e CUSTOM_LLM_API_KEY=any-key-here \
  ghcr.io/AgoraIO-Conversational-AI/recipe-agent-content-filter:latest
```

## Environment variables

Backend env file: [`server/.env.example`](server/.env.example).

| Variable | Required | Default | Notes |
| --- | :---: | :---: | --- |
| `AGORA_APP_ID` | ✅ | — | Agora Console → Project → App ID |
| `AGORA_APP_CERTIFICATE` | ✅ | — | Agora Console → Project → App Certificate (server only) |
| `CUSTOM_LLM_URL` | ✅ | — | **Public** chat-completions URL of your `/llm` endpoint (`<tunnel>/llm/chat/completions`). Agora cloud calls it; cannot be `localhost`. |
| `CUSTOM_LLM_API_KEY` | ✅ | `any-key-here` | Forwarded by Agora cloud as `Authorization: Bearer`. Required by the `CustomLLM` vendor. |
| `CUSTOM_LLM_MODEL` |  | `filter-mock` | Model name passed to your endpoint |
| `AGENT_GREETING` |  | built-in | Optional opening line override |
| `PORT` |  | `8000` | Agent backend port |
| `FILTER_BANNED_TERMS` |  | `strawberries` | Comma-separated list of banned terms |
| `AGENT_BACKEND_URL` (web deploy) | ✅ | — | Required in a deployed `web` app when proxying to the backend |

## Commands

```bash
bun run setup            # install web deps + create server/ venv
bun run dev              # run backend (:8000, serves /llm) + web (:3000)

bun run doctor           # prerequisite check (no creds needed)
bun run doctor:local     # + .env.local + credentials + CUSTOM_LLM_URL checks

bun run verify           # web-only gate (no Agora creds needed)
bun run verify:local     # full local gate: backend compile + smoke tests + web build
bun run clean            # remove venvs and build artifacts
```

Tests run standalone (no Agora cloud needed): `pytest` in `server/`, plus
`bun run verify` in `web/`. CI runs them on Linux/macOS/Windows × Python 3.10 & 3.13.

## Architecture

```
Browser (localhost:3000)
  │  fetch /api/*
  ▼
Next.js  ──rewrite──▶  Agent backend  (server/, localhost:8000)
                          │  starts agent session (CustomLLM vendor)
                          ▼
                       Agora ConvoAI Cloud
                          │  POST <CUSTOM_LLM_URL>   (Authorization: Bearer)
                          ▼
                       Content-filter LLM endpoint  (mounted at /llm in server/, localhost:8000)
                          ▲  public via ngrok tunnel
```

The browser only ever calls Next `/api/*`, which rewrites to the agent backend.
The agent backend owns Agora tokens, agent lifecycle, and serves the
**content-filter LLM endpoint** mounted at `/llm`. Because Agora cloud — not the
browser — calls it, the backend must be publicly reachable (`ngrok http 8000`).
See [ARCHITECTURE.md](./ARCHITECTURE.md).

## Repo Map

```
recipe-agent-content-filter/
├── web/          # Next.js frontend (:3000)
├── server/       # Agent backend (:8000) — tokens + agent lifecycle + /llm mock endpoint
│   └── src/
│       ├── server.py          # FastAPI app + /llm mount
│       ├── agent.py           # Agora agent session management
│       └── llm.py             # OpenAI-compatible mock; filter seam; no Agora deps
├── ARCHITECTURE.md
└── AGENTS.md
```

## What You Get

- **Web client** — zero-config Next.js voice UI; token + session lifecycle handled
  behind Next rewrites.
- **Agent backend** (`server/`) — FastAPI service that generates Agora tokens and
  drives the agent session via the `CustomLLM` vendor.
- **API contract** — the `CustomLLM` vendor expects a public OpenAI-compatible
  `POST /chat/completions` endpoint; the mock fulfils that contract.
- **Output redaction** behind a pluggable `moderate()` seam inside `server/src/llm.py`
  with `FILTER_BANNED_TERMS` — swap the body for a real moderation model with no
  other changes required.
- **Zero-key mock** — no LLM API key or external model account needed to run the
  demo.

## How It Works

1. The browser opens the Agora RTC channel via the Next.js web client.
2. The agent backend (`:8000`) generates an Agora token and starts an agent session
   pointing the `CustomLLM` vendor at `CUSTOM_LLM_URL`.
3. Agora cloud transcribes speech (Deepgram STT) and sends the transcript to the
   content-filter LLM endpoint via `POST /llm/chat/completions`.
4. `echo_reply()` builds a reply that includes a sentence repeating what the user
   said (the sentence that may get flagged).
5. `filter_reply()` splits the reply at sentence boundaries (`.`, `!`, `?`) and
   passes each sentence to `moderate()`.
6. `moderate()` checks whether any term from `FILTER_BANNED_TERMS` (default:
   `strawberries`) appears in the sentence. Returning `False` replaces the sentence
   with `"Content filtered."`.
7. The filtered text streams back to Agora cloud via OpenAI SSE and is spoken by
   MiniMax TTS.

The content _generation_ is mocked; the filter is real code. Swap `moderate()`'s
body for a real moderator model (e.g. call an LLM) to get the "LLM-powered"
variant with no other changes.

### Replacing the mock

The filter + seam live in `run_agent_turn()` / `moderate()` / `filter_reply()` in
[`server/src/llm.py`](server/src/llm.py).

- Replace `moderate()`'s body with a real moderator model (e.g. an LLM
  classification call) to get the "LLM-powered" variant.
- Replace `echo_reply()` with a real LLM call to get a fully real response pipeline
  that still runs the filter.

The endpoint must keep speaking the OpenAI streaming `/chat/completions` contract.
A production endpoint should also validate the `Authorization: Bearer` header.

## Troubleshooting

| Problem | Fix |
| --- | --- |
| Agent starts but never speaks | `CUSTOM_LLM_URL` is not public or omits `/llm/chat/completions`. Use your ngrok URL. |
| `doctor:local` warns about localhost | Replace the local URL with your public tunnel URL. |
| Local calls fail / hang under a global proxy | Configure your proxy to route `127.0.0.1`, `localhost`, and RFC-1918 ranges DIRECT. |
| `Missing server/venv` during verify | Run `bun run setup` (creates the venv). |
| Windows asks to install a WSL distribution | Pull the latest changes and rerun `bun run setup`; the setup scripts do not require WSL. |
| Want different banned terms | Set `FILTER_BANNED_TERMS=term1,term2` in `server/.env.local`. |

## More Docs

- [ARCHITECTURE.md](./ARCHITECTURE.md)
- [AGENTS.md](./AGENTS.md)

## License

Released under the [MIT License](./LICENSE).
