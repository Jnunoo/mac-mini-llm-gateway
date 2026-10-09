# Gateway Setup — agent runbook

Execute these steps in order on the Apple Silicon Mac Mini (16 GB). Each step
has a verification; do not move on until it passes. Replace `YOUR_USER` with
the actual macOS username (`whoami`) everywhere it appears.

## 1. Install software

```bash
brew install ollama pipx
pipx install 'litellm[proxy]'
pipx ensurepath
ollama --version
```

**Verify:** `ollama --version` reports **0.17.5 or newer**. Older versions emit
a model's tool calls as plain prose instead of structured `tool_calls`, which
silently breaks every tool-using client. If the version is older, `brew upgrade ollama`.

Then pull the models (several GB; do this before wiring services so failures
surface early):

```bash
ollama pull qwen3.5:9b
ollama pull qwen3.5:4b
```

## 2. Ollama service (loopback only)

Write `~/Library/LaunchAgents/com.agenticos.ollama.plist`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.agenticos.ollama</string>
    <key>ProgramArguments</key>
    <array>
        <string>/opt/homebrew/bin/ollama</string>
        <string>serve</string>
    </array>
    <key>EnvironmentVariables</key>
    <dict>
        <key>OLLAMA_HOST</key>
        <string>127.0.0.1:11434</string>
        <key>OLLAMA_KEEP_ALIVE</key>
        <string>-1</string>
        <key>OLLAMA_CONTEXT_LENGTH</key>
        <string>32768</string>
        <key>OLLAMA_MAX_LOADED_MODELS</key>
        <string>1</string>
    </dict>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
    <key>StandardOutPath</key>
    <string>/Users/YOUR_USER/Library/Logs/ollama.log</string>
    <key>StandardErrorPath</key>
    <string>/Users/YOUR_USER/Library/Logs/ollama.log</string>
</dict>
</plist>
```

Rules that are load-bearing, not style:

- The env var is **`OLLAMA_CONTEXT_LENGTH`** — not `OLLAMA_NUM_CTX`. With the
  wrong name Ollama silently serves a 2,048-token window and truncates every
  long prompt. The tell is `usage.prompt_tokens` pinned at ~2050 regardless of
  prompt size.
- `OLLAMA_MAX_LOADED_MODELS=1` stays at 1 on a 16 GB machine — two resident
  9B-class models do not fit alongside the OS and LiteLLM.
- If the Ollama **menu-bar app** is installed, quit it and remove it from
  Login Items first — its server fights this one for port 11434.

Load it:

```bash
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.agenticos.ollama.plist
```

**Verify:**

```bash
curl -s http://127.0.0.1:11434/api/tags | head -c 400
```

returns JSON listing both pulled models.

## 3. LiteLLM gateway

Ask the operator for the **master key** now. If this is a rebuild, it is the
key the client machines already use. For a fresh deployment generate one with
`openssl rand -hex 24` and hand it to the operator. Never write it into this
repo or any log output.

Write `~/litellm/config.yaml`:

```yaml
model_list:
  - model_name: local-balanced
    litellm_params: { model: ollama_chat/qwen3.5:9b, api_base: http://127.0.0.1:11434, reasoning_effort: none }
  - model_name: local-fast
    litellm_params: { model: ollama_chat/qwen3.5:4b, api_base: http://127.0.0.1:11434, reasoning_effort: none }
litellm_settings:
  drop_params: true
  request_timeout: 180
general_settings:
  master_key: os.environ/LITELLM_MASTER_KEY
```

Two settings here are required for correct behavior:

- `drop_params: true` — OpenAI SDK clients send request fields Ollama's
  OpenAI-compat surface rejects; without this, requests 400.
- `reasoning_effort: none` — turns Qwen 3.5's thinking mode off at the gateway
  (maps to Ollama `think: false`). Thinking scales with prompt size: measured
  49 s vs 8 s on an ~11k-token prompt. Leave it off for pipeline use.

Write `~/Library/LaunchAgents/com.agenticos.litellm.plist`, pasting the master
key into `LITELLM_MASTER_KEY` (find the litellm binary path with
`which litellm`; pipx installs to `~/.local/bin/litellm`):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.agenticos.litellm</string>
    <key>ProgramArguments</key>
    <array>
        <string>/Users/YOUR_USER/.local/bin/litellm</string>
        <string>--config</string>
        <string>/Users/YOUR_USER/litellm/config.yaml</string>
        <string>--host</string>
        <string>0.0.0.0</string>
        <string>--port</string>
        <string>4000</string>
    </array>
    <key>EnvironmentVariables</key>
    <dict>
        <key>LITELLM_MASTER_KEY</key>
        <string>PASTE_THE_KEY_HERE</string>
    </dict>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
    <key>StandardOutPath</key>
    <string>/Users/YOUR_USER/Library/Logs/litellm.log</string>
    <key>StandardErrorPath</key>
    <string>/Users/YOUR_USER/Library/Logs/litellm.log</string>
</dict>
</plist>
```

```bash
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.agenticos.litellm.plist
```

**Verify locally** (use the real key; LiteLLM can take ~20 s to come up):

```bash
curl -s -H "Authorization: Bearer $KEY" http://127.0.0.1:4000/v1/models
```

returns a model list containing `local-balanced` and `local-fast`.

## 4. Network

1. **Firewall**: System Settings → Network → Firewall — if enabled, allow
   incoming connections for `litellm` (port 4000). Ollama needs no rule; it is
   loopback-only.
2. **Stable address**: set a DHCP reservation for this Mac on the router so
   its LAN IP never changes under the gateway URL. Optionally also give it a
   stable mDNS hostname: `sudo scutil --set HostName <name>` (clients can then
   use `http://<name>.local:4000/v1` where mDNS resolution works).
3. Report the Mac's current LAN IP to the operator: `ipconfig getifaddr en0`
   (try `en1` if empty). If this is a rebuild, the client machines expect the
   OLD IP — the operator must either make the reservation match it or update
   the clients' configured base URL.

## 5. Acceptance checks

Run from the Mac, then have the operator repeat the two curls from a client
machine against `http://<mini-ip>:4000/v1`.

1. Chat round-trip:

```bash
curl -s http://127.0.0.1:4000/v1/chat/completions \
  -H "Authorization: Bearer $KEY" -H "Content-Type: application/json" \
  -d '{"model":"local-balanced","messages":[{"role":"user","content":"Reply with exactly: OK"}]}'
```

Expect a completion whose content is `OK`.

2. **Context window is really 32k**: send one prompt of ~10,000+ tokens (e.g.
   a long document pasted into `content`, asking for a one-line summary) and
   read `usage.prompt_tokens` in the response. It must reflect the real prompt
   size. If it is pinned at ~2050, step 2's context variable is wrong — fix
   the plist, `launchctl kickstart -k gui/$(id -u)/com.agenticos.ollama`, retest.

3. **Tool calls are structured**: send a chat request with a `tools` array
   defining one function and a prompt that forces a call. The response must
   carry a structured `tool_calls` entry, not prose that looks like a function
   call. If prose: Ollama is older than 0.17.5 — upgrade.

4. **Model is on GPU**: after a request, `ollama ps` shows the model at
   `100% GPU`. If partially on CPU, memory is under pressure: check what else
   is running, or add `OLLAMA_FLASH_ATTENTION=1` and `OLLAMA_KV_CACHE_TYPE=q8_0`
   to the Ollama plist's env block and kickstart it.

## 6. Troubleshooting

- **Service not running**: `launchctl list | grep agenticos` should show both
  jobs with a blank/`0` status. Logs: `~/Library/Logs/ollama.log`,
  `~/Library/Logs/litellm.log`.
- **401 from clients**: the key the client sends must equal
  `LITELLM_MASTER_KEY` in the LiteLLM plist, byte for byte.
- **Client can't reach the gateway but local curl works**: macOS firewall rule
  for litellm, wrong LAN IP, or the router changed the Mac's address (no
  reservation).
- **Client can't reach the gateway, with heavy packet loss in BOTH directions
  and neither machine able to learn the other's MAC (`arp -a` shows no entry),
  yet both reach the router and other devices fine**: a Wi-Fi **range extender**
  (classic repeater, e.g. EX7000-class) on the same SSID can poison ARP between
  wireless clients via its MAC translation — even when neither machine is
  associated to it. Internet and router access keep working, so nothing looks
  misconfigured. Diagnose: ping each machine from the other, check `arp -a` for
  the peer, and confirm both BSSIDs are on the main router (Option-click the
  Wi-Fi icon on the Mac). Fix: unplug the extender and retest (the change is
  immediate), then either retire it or give its extended network its own SSID
  so clients never silently roam onto it.
- **Gateway goes "offline" whenever it sits idle**: a fresh macOS install
  sleeps after a few idle minutes. A headless gateway must not sleep:
  `sudo pmset -a sleep 0 disksleep 0`, then verify with `pmset -g | grep sleep`.
- **Slow first response after idle or after switching aliases**: expected —
  `OLLAMA_MAX_LOADED_MODELS=1` means an alias switch unloads/reloads (~5–10 s).
- **Everything is slow**: check acceptance checks 2 and 4 — a truncated
  context window and thinking-mode-on are the two silent misconfigurations
  that both present as "the local model is slow".

## Performance expectations (measured on this class of hardware)

| | `local-balanced` (9B) | `local-fast` (4B) |
|---|---|---|
| 500-word draft | ~12.5 tok/s | ~21 tok/s |
| 12.7k-token prompt ingest | ~110 s | ~67 s |
| JSON extraction | ~4 s | ~2.5 s |
| Single tool call | ~4.4 s | ~2.4 s |

Prompt ingest dominates agent workloads: a 10k+ token system prompt means
**1–2 minutes before the first output token**. That is normal, not a fault.
