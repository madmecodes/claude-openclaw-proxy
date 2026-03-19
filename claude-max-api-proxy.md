---
name: claude-max-api-proxy
description: Set up the claude-max-api proxy on a new Linux server so that OpenClaw (or any OpenAI-compatible tool) can use a Claude Max subscription instead of a paid API key. Use this skill whenever setting up a new EC2/VPS/GCP instance that needs OpenClaw + Claude integration without per-token billing.
---

This skill sets up the `claude-max-api` proxy — a local Node.js service that wraps the Claude Code CLI and exposes an OpenAI-compatible API on `localhost:3456`. This allows OpenClaw to use a Claude Max subscription (flat-rate) instead of paying per API call.

## How It Works

```
OpenClaw → localhost:3456 (claude-max-api) → Claude Code CLI → Anthropic (Max subscription)
```

- OpenClaw thinks it's talking to an OpenAI-compatible endpoint
- The proxy wraps Claude Code CLI as a subprocess
- Claude Code uses the OAuth session from the logged-in Max account
- No API key needed — auth is handled by the Claude Code session

## Prerequisites

- Ubuntu/Debian or Amazon Linux server
- Node.js 22+ installed
- Claude Code installed and logged in with a Claude Max account

## Step 1 — Install Claude Code

```bash
# Install Claude Code (native installer)
curl -fsSL https://claude.ai/install.sh | bash

# Add to PATH
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc && source ~/.bashrc

# Verify
claude --version
```

Then login with the Claude Max account:
```bash
claude
# Opens browser OAuth flow — login with Max account
# Once logged in, exit with /exit or Ctrl+C
```

## Step 2 — Set Up npm Global Prefix (avoid sudo issues)

```bash
mkdir -p ~/.npm-global
npm config set prefix '~/.npm-global'
echo 'export PATH="$HOME/.npm-global/bin:$PATH"' >> ~/.bashrc && source ~/.bashrc
```

## Step 3 — Install the Proxy

```bash
npm install -g claude-max-api-proxy
```

The binary installs as `claude-max-api` (not `claude-max-api-proxy`). Verify:
```bash
ls ~/.npm-global/bin/ | grep claude
# Should show: claude-max-api
```

## Step 4 — Start the Proxy (Silent Mode)

Always start with nohup and output redirected — otherwise it prints to stdout and breaks SCP/non-interactive SSH sessions:

```bash
nohup ~/.npm-global/bin/claude-max-api > /dev/null 2>&1 &
```

Verify it's running:
```bash
curl http://localhost:3456/v1/models
# Should return: {"object":"list","data":[{"id":"claude-opus-4",...},...]}
```

## Step 5 — Auto-Start on Login

Add to `.bashrc` so it starts automatically on every login/reboot:

```bash
echo 'nohup ~/.npm-global/bin/claude-max-api > /dev/null 2>&1 &' >> ~/.bashrc
```

**Important:** Never add it as `~/.npm-global/bin/claude-max-api &` (without nohup/redirect) — this causes the `Received message too long` SCP error because the process prints output during non-interactive SSH sessions.

## Step 6 — Configure OpenClaw to Use the Proxy

When running `openclaw onboard` or `openclaw configure`, set:

- **Provider type:** Custom Provider (OpenAI-compatible)
- **API Base URL:** `http://localhost:3456/v1`
- **API Key:** `not-needed`
- **Endpoint compatibility:** OpenAI-compatible
- **Model ID:** `claude-opus-4` (or `claude-sonnet-4`, `claude-haiku-4`)

Or manually in `~/.openclaw/openclaw.json`:

```json
"models": {
  "providers": {
    "custom-localhost-3456": {
      "baseUrl": "http://localhost:3456/v1",
      "apiKey": "not-needed",
      "api": "openai-completions",
      "models": [
        {
          "id": "claude-opus-4",
          "contextWindow": 200000,
          "maxTokens": 200000
        }
      ]
    }
  }
}
```

Always set `contextWindow` and `maxTokens` to `200000` — the default of `4096/16000` is too low for real usage.

## Verification Checklist

```bash
# Proxy running
curl -s http://localhost:3456/v1/models | head -1

# Claude Code authenticated
claude --version

# OpenClaw using proxy
openclaw status
# Should show gateway running and model as claude-sonnet-4-5 or claude-opus-4
```

## Common Issues

**`claude-max-api-proxy: command not found`**
Binary is named `claude-max-api`, not `claude-max-api-proxy`. Run:
```bash
~/.npm-global/bin/claude-max-api &
```

**`Received message too long` during SCP**
The proxy is printing startup output to stdout. Fix:
```bash
pkill -f claude-max-api
# Update .bashrc to use nohup with output redirect
sed -i 's|~/.npm-global/bin/claude-max-api &|nohup ~/.npm-global/bin/claude-max-api > /dev/null 2>\&1 \&|' ~/.bashrc
nohup ~/.npm-global/bin/claude-max-api > /dev/null 2>&1 &
```

**Proxy running but OpenClaw still fails**
Check context window — default is too low:
```bash
sed -i 's/"contextWindow": 16000/"contextWindow": 200000/' ~/.openclaw/openclaw.json
sed -i 's/"maxTokens": 4096/"maxTokens": 200000/' ~/.openclaw/openclaw.json
openclaw gateway restart
```

**Claude Code session expired**
Re-login:
```bash
claude
# Login again via browser
pkill -f claude-max-api
nohup ~/.npm-global/bin/claude-max-api > /dev/null 2>&1 &
```