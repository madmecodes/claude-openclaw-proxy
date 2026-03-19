# OpenClaw Server Setup — GCP to AWS EC2 Migration Guide

Quick reference for setting up OpenClaw on a fresh Ubuntu EC2 and migrating from an existing server.

---

## New EC2 Setup (AWS Console)

- **AMI:** Ubuntu 24.04 LTS
- **Instance type:** t3.medium (minimum) or m7i-flex.large
- **Region:** Match source server region (e.g. ap-southeast-1 for Singapore)
- **Storage:** 30 GiB minimum
- **Security group inbound ports:** 22, 18789, 18792, 20201, 20202
- **Key pair:** Create new `.pem`, download and store in `aws_keys/`

---

## 1. SSH Into New EC2

```bash
chmod 400 aws_keys/your-key.pem
ssh -i aws_keys/your-key.pem ubuntu@YOUR_EC2_IP
```

---

## 2. Install Base Stack

```bash
# System packages
sudo apt update && sudo apt install -y git curl unzip jq

# Node.js 22
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo bash -
sudo apt install -y nodejs

# Fix npm global prefix (avoid sudo issues)
mkdir -p ~/.npm-global
npm config set prefix '~/.npm-global'
echo 'export PATH="$HOME/.npm-global/bin:$PATH"' >> ~/.bashrc && source ~/.bashrc

# Claude Code
curl -fsSL https://claude.ai/install.sh | bash
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc && source ~/.bashrc

# OpenClaw
npm install -g openclaw@latest

# Verify
node -v && claude --version && openclaw --version
```

---

## 3. Migrate OpenClaw State from Source Server

### On source server — create tarball
```bash
openclaw gateway stop
cd ~ && tar -czf openclaw-state.tgz .openclaw
```

### On local machine — transfer via local
```bash
# Download from source
scp ubuntu@SOURCE_IP:~/openclaw-state.tgz /tmp/

# Upload to EC2
scp -i aws_keys/your-key.pem /tmp/openclaw-state.tgz ubuntu@EC2_IP:~/
```

### On EC2 — extract
```bash
cd ~ && tar -xzf openclaw-state.tgz && ls ~/.openclaw/
```

---

## 4. Copy Claude Code Auth

```bash
# On local machine
scp ubuntu@SOURCE_IP:~/.claude/.credentials.json /tmp/
scp ubuntu@SOURCE_IP:~/.claude.json /tmp/claude-config.json

scp -i aws_keys/your-key.pem /tmp/.credentials.json ubuntu@EC2_IP:~/.claude/
scp -i aws_keys/your-key.pem /tmp/claude-config.json ubuntu@EC2_IP:~/.claude.json
```

---

## 5. Copy Google Credentials (if applicable)

```bash
scp ubuntu@SOURCE_IP:~/google_credentials.json /tmp/
scp -i aws_keys/your-key.pem /tmp/google_credentials.json ubuntu@EC2_IP:~/
```

---

## 6. Set Up claude-max-api Proxy

Allows OpenClaw to use Claude Max subscription instead of paid API key.

```bash
# Install proxy
npm install -g claude-max-api-proxy

# Start silently (nohup required — avoids breaking SCP)
nohup ~/.npm-global/bin/claude-max-api > /dev/null 2>&1 &

# Auto-start on login
echo 'nohup ~/.npm-global/bin/claude-max-api > /dev/null 2>&1 &' >> ~/.bashrc

# Verify
curl http://localhost:3456/v1/models
```

> **Note:** Binary is named `claude-max-api`, not `claude-max-api-proxy`.  
> **Never** add it to `.bashrc` without `nohup` + output redirect — it will break SCP with `Received message too long` error.

---

## 7. Run OpenClaw Doctor

```bash
openclaw doctor
# When asked:
# - Archive orphan transcripts → No
# - Enable bash completion → Yes
# - Install gateway service → Yes
# - Gateway runtime → Node (recommended)
```

---

## 8. Start Gateway

```bash
openclaw gateway restart
openclaw status
```

Expected output:
- Gateway: running
- WhatsApp: linked
- Slack: connected (if configured)

---

## 9. Enable Password SSH Login (for client access)

```bash
# Set password
sudo passwd ubuntu

# Enable password auth (Ubuntu 24.04 uses override file)
sudo sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/' /etc/ssh/sshd_config.d/60-cloudimg-settings.conf
sudo systemctl restart ssh
```

Test from local:
```bash
ssh ubuntu@EC2_IP
# Should prompt for password
```

---

## 10. Fix OpenClaw Context Window (if fresh onboard)

If you ran `openclaw onboard` instead of migrating, the default context window is too low:

```bash
sed -i 's/"contextWindow": 16000/"contextWindow": 200000/' ~/.openclaw/openclaw.json
sed -i 's/"maxTokens": 4096/"maxTokens": 200000/' ~/.openclaw/openclaw.json
openclaw gateway restart
```

---

## Verification Checklist

```bash
claude --version                          # Claude Code installed
openclaw status                           # Gateway running, channels connected
curl -s http://localhost:3456/v1/models   # Proxy serving models
```

| Check | Command |
|---|---|
| Claude Code auth | `claude --version` → shows account name |
| Gateway running | `openclaw status` → Gateway: running |
| WhatsApp linked | `openclaw status` → WhatsApp: OK |
| Proxy working | `curl localhost:3456/v1/models` → returns model list |
| Test agent | `openclaw tui` → send a message, agent responds |

---

## Go-Live

1. Verify EC2 is fully working via `openclaw tui`
2. Confirm with manager before stopping source server
3. Stop source server gateway: `openclaw gateway stop`
4. Have client test by sending a WhatsApp message
5. Once confirmed — shut down or snapshot source server

---

## Key Files Reference

| File | Purpose |
|---|---|
| `~/.openclaw/openclaw.json` | Main OpenClaw config |
| `~/.openclaw/credentials/` | Channel auth (WhatsApp keys etc.) |
| `~/.openclaw/workspace/` | Agent identity, memory, tools |
| `~/.claude/.credentials.json` | Claude Code OAuth token |
| `~/.claude.json` | Claude Code config + account info |
| `~/google_credentials.json` | Google OAuth for integrations |

---

## Common Issues

| Problem | Fix |
|---|---|
| `npm install -g` permission denied | Set up `~/.npm-global` prefix first |
| SCP `Received message too long` | Proxy printing to stdout — use `nohup ... > /dev/null 2>&1 &` |
| `openclaw: command not found` | `source ~/.bashrc` or check `~/.npm-global/bin` in PATH |
| WhatsApp disconnected after migration | Credentials copied correctly — restart gateway, WhatsApp should reconnect automatically |
| Context window too small | Set `contextWindow` and `maxTokens` to `200000` in `openclaw.json` |