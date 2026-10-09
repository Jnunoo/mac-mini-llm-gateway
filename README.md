# Mac Mini Local LLM Gateway

Sets up a free, private, LAN-only inference backend on an Apple Silicon Mac Mini:
**Ollama** (serving Qwen 3.5 models, loopback only) fronted by **LiteLLM** (an
OpenAI-compatible proxy on port 4000), reachable from other machines on the LAN
at `http://<mini-ip>:4000/v1`.

Any client that speaks the OpenAI chat-completions API can use it. The gateway
exposes two model aliases:

| Alias | Backing model | Purpose |
|---|---|---|
| `local-balanced` | `qwen3.5:9b` | default / quality |
| `local-fast` | `qwen3.5:4b` | latency-sensitive calls |

## How to use this repo with a coding agent

Open a Claude Code (or similar) session **on the Mac Mini**, clone this repo,
and tell the agent:

> Follow SETUP.md top to bottom. Ask me for the gateway master key when you
> reach the LiteLLM step — it is not in this repo.

The agent can run every step itself: all installs are Homebrew/pipx, and the
services are user-level LaunchAgents (no sudo).

## Prerequisites (do these by hand first)

1. **Homebrew** installed (https://brew.sh).
2. **Xcode command-line tools**: run `xcode-select --install` — this pops a GUI
   dialog, so it cannot be fully automated.
3. A coding agent CLI (e.g. Claude Code: `npm install -g @anthropic-ai/claude-code`)
   if you want the agent to do the rest.
4. The **gateway master key** at hand. If you are rebuilding an existing
   gateway, reuse the key your client machines already have configured, so
   nothing on their side changes. Never commit it anywhere.

## Security model

- Ollama binds to `127.0.0.1` only — nothing on the LAN talks to it directly.
- LiteLLM is the only LAN-exposed service (port 4000) and requires the
  `Authorization: Bearer <master key>` header on every request.
- This is a LAN deployment. Do not port-forward 4000 to the internet.
