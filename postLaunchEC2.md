# OpenClaw EC2 — Post Launch Setup Checklist

After launching EC2 with the user data script, follow these steps in order.

---

## Step 1 — SSH Into Server

```bash
ssh -i /path/to/your-key.pem ubuntu@YOUR_EC2_IP
```

---

## Step 2 — Source PATH

User data script installs everything but PATH isn't active yet in your session:

```bash
source ~/.bashrc
```

---

## Step 3 — Verify User Data Script Worked

```bash
node -v && claude --version && openclaw --version && ls ~/.npm-global/bin/ | grep claude
```

**Expected output:**
```
v22.22.1
2.1.x (Claude Code)
🦞 OpenClaw 2026.x.x ...
claude-max-api
```

**If openclaw not found:**
```bash
export PATH=$HOME/.npm-global/bin:$PATH
npm install -g openclaw@latest
npm install -g claude-max-api-proxy
```

**If claude not found:**
```bash
export PATH=$HOME/.local/bin:$PATH
```

---

## Step 4 — Login to Claude Code

```bash
claude
```

- Opens browser OAuth flow
- Login with Claude Max account
- Once logged in exit with `/exit` or `Ctrl+C`

---

## Step 5 — Start the Proxy

```bash
nohup ~/.npm-global/bin/claude-max-api > /dev/null 2>&1 &
```

**Verify proxy is working:**
```bash
curl http://localhost:3456/v1/models
```

**Expected output:**
```json
{"object":"list","data":[{"id":"claude-opus-4",...},{"id":"claude-sonnet-4",...},{"id":"claude-haiku-4",...}]}
```

**If connection refused:**
```bash
# Proxy not running — start it
nohup ~/.npm-global/bin/claude-max-api > /dev/null 2>&1 &
# Wait 3 seconds then retry
curl http://localhost:3456/v1/models
```

**If still failing:**
```bash
# Claude Code session may have expired — re-login
claude
# Then restart proxy
pkill -f claude-max-api
nohup ~/.npm-global/bin/claude-max-api > /dev/null 2>&1 &
```

---

## Step 6 — Run OpenClaw Onboard

```bash
openclaw onboard
```

Follow the wizard exactly as below:

| Prompt | Answer |
|---|---|
| Security warning — Continue? | **Yes** |
| Onboarding mode | **QuickStart** |
| Model/auth provider | **Custom Provider** |
| API Base URL | `http://localhost:3456/v1` |
| How to provide API key | **Paste API key now** |
| API Key | `not-needed` |
| Endpoint compatibility | **OpenAI-compatible** |
| Model ID | `claude-opus-4` |
| Endpoint ID | Press Enter (accept `custom-localhost-3456`) |
| Model alias | Press Enter (skip) |
| Select channel | **Skip for now** |
| Search provider | **Skip for now** |
| Configure skills now? | **Yes** |
| Install missing skill dependencies | **Skip for now** |
| Set GOOGLE_PLACES_API_KEY | **No** |
| Set GEMINI_API_KEY | **No** |
| Set NOTION_API_KEY | **No** |
| Set OPENAI_API_KEY (image-gen) | **No** |
| Set OPENAI_API_KEY (whisper) | **No** |
| Set ELEVENLABS_API_KEY | **No** |
| Enable hooks? | **Select all 4:** boot-md, bootstrap-extra-files, command-logger, session-memory |
| Gateway service runtime | **Node (recommended)** |
| How do you want to hatch your bot? | **Do this later** |

> **Note:** Wizard will auto-install the gateway systemd service and enable systemd lingering so the gateway survives logout.

> **If you pick wrong option:** Press `Ctrl+C` to exit and rerun `openclaw onboard` from scratch.

---

## Step 7 — Fix Context Window

Default values are too low — always fix after onboard:

```bash
sed -i 's/"contextWindow": 16000/"contextWindow": 200000/' ~/.openclaw/openclaw.json
sed -i 's/"maxTokens": 4096/"maxTokens": 200000/' ~/.openclaw/openclaw.json
```

---

## Step 8 — Run Doctor

```bash
openclaw doctor
```

**When asked:**
- Archive orphan transcripts → **No**
- Enable bash completion → **Yes**
- Install gateway service → **Yes**
- Gateway runtime → **Node (recommended)**

---

## Step 9 — Start Gateway

```bash
openclaw gateway restart
openclaw status
```

**Expected status output:**
- Gateway service: running
- Channels: OK (WhatsApp/Slack linked)
- Model: claude-sonnet-4-5 or claude-opus-4

---

## Step 10 — Test Agent

```bash
openclaw tui
```

Type a message — if agent responds, everything is working.

---

## Full Verification Commands

```bash
node -v                                   # Node installed
claude --version                          # Claude Code installed + logged in
openclaw --version                        # OpenClaw installed
curl -s http://localhost:3456/v1/models   # Proxy working
openclaw status                           # Gateway running, channels connected
```

---

## Common Issues & Fixes

| Problem | Fix |
|---|---|
| `openclaw: command not found` | `source ~/.bashrc` or `export PATH=$HOME/.npm-global/bin:$PATH` |
| `claude: command not found` | `source ~/.bashrc` or `export PATH=$HOME/.local/bin:$PATH` |
| Proxy `Connection refused` | Start with `nohup ~/.npm-global/bin/claude-max-api > /dev/null 2>&1 &` |
| Proxy running but no models returned | Re-login to Claude Code with `claude`, then restart proxy |
| `openclaw status` shows gateway unreachable | Run `openclaw doctor` then `openclaw gateway restart` |
| Context window too small | Run Step 7 fix commands then `openclaw gateway restart` |
| WhatsApp not connecting | Run `openclaw channels login whatsapp` and scan QR code |

---

## Proxy Auto-Start Verification

Confirm proxy will auto-start on every login:

```bash
grep 'claude-max-api' ~/.bashrc
```

Expected:
```
nohup ~/.npm-global/bin/claude-max-api > /dev/null 2>&1 &
```

If missing, add it:
```bash
echo 'nohup ~/.npm-global/bin/claude-max-api > /dev/null 2>&1 &' >> ~/.bashrc
```