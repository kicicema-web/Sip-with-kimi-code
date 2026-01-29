# Remote Kimi Code on VPS

Actually use Kimi (Moonshot AI) on a remote VPS with CLI tools. Code from your phone, tablet, or any device via SSH.

## Prerequisites

- VPS with 2GB+ RAM (4GB recommended)
- Ubuntu/Debian server
- [Moonshot API Key](https://platform.moonshot.cn) (free tier includes 15元 credits)

---

## Option 1: AIChat (Recommended)

Universal CLI tool that supports Kimi natively with file system access.

### Installation

\`\`\`bash
# SSH into your VPS
ssh root@your-vps-ip

# Download and install Aichat
curl -fsSL https://github.com/sigoden/aichat/releases/latest/download/aichat-x86_64-unknown-linux-musl.tar.gz | tar xz
sudo mv aichat /usr/local/bin/

# Create config directory
mkdir -p ~/.config/aichat
\`\`\`

### Configuration

\`\`\`bash
# Create config file
cat > ~/.config/aichat/config.yaml << 'EOF'
model: moonshot:kimi-k2-5-latest
clients:
  - type: openai-compatible
    name: moonshot
    api_base: https://api.moonshot.cn/v1
    api_key: $MOONSHOT_API_KEY
EOF

# Set your API key (add to ~/.bashrc to persist)
export MOONSHOT_API_KEY="sk-your-key-here"
\`\`\`

### Usage

\`\`\`bash
# Interactive mode
aichat

# Single question
aichat "Explain this Python code" < script.py

# With file context
aichat "@script.py refactor this to use async/await"

# Start persistent session
aichat --session coding
\`\`\`

---

## Option 2: Shell-GPT

Python-based CLI with optional integration (piping, git hooks).

### Installation

\`\`\`bash
pip install shell-gpt
\`\`\`

### Configuration

\`\`\`bash
mkdir -p ~/.config/shell_gpt
cat > ~/.config/shell_gpt/.sgptrc << 'EOF'
OPENAI_API_KEY=sk-your-moonshot-key
OPENAI_API_HOST=https://api.moonshot.cn/v1
DEFAULT_MODEL=kimi-k2-5-latest
EOF
\`\`\`

### Usage

\`\`\`bash
# Ask anything
sgpt "How do I fix 'permission denied' in Linux?"

# Code generation with file output
sgpt --code "python function to validate email" > validate.py

# Chat mode (context aware)
sgpt --chat project "explain the database schema"
sgpt --chat project "now write a query for user stats"
\`\`\`

---

## Option 3: Custom Minimal Client

Zero-dependency Python script.

### Setup

\`\`\`bash
# Create the CLI
cat > /usr/local/bin/kimi << 'EOF'
#!/usr/bin/env python3
import os, sys, json, urllib.request

API_KEY = os.getenv("MOONSHOT_API_KEY")
if not API_KEY:
    print("Error: Set MOONSHOT_API_KEY environment variable")
    sys.exit(1)

def ask(prompt):
    data = {
        "model": "kimi-k2-5-latest",
        "messages": [{"role": "user", "content": prompt}],
        "stream": False
    }
    req = urllib.request.Request(
        "https://api.moonshot.cn/v1/chat/completions",
        data=json.dumps(data).encode(),
        headers={"Authorization": f"Bearer {API_KEY}", "Content-Type": "application/json"},
        method="POST"
    )
    try:
        with urllib.request.urlopen(req) as resp:
            result = json.loads(resp.read())
            print(result["choices"][0]["message"]["content"])
    except Exception as e:
        print(f"Error: {e}")

if __name__ == "__main__":
    if not sys.stdin.isatty():
        ask(sys.stdin.read())
    elif len(sys.argv) > 1:
        ask(" ".join(sys.argv[1:]))
    else:
        print("Usage: kimi 'your question' or cat file | kimi")
EOF

chmod +x /usr/local/bin/kimi
export MOONSHOT_API_KEY="sk-your-key-here"
\`\`\`

### Usage

\`\`\`bash
# Direct question
kimi "optimize this nginx config for speed"

# Pipe content
cat error.log | kimi "analyze these errors"

# With code files
kimi "$(cat main.py)\n\nrefactor using classes"
\`\`\`

---

## Option 4: Full IDE (Code Server + Continue.dev)

GUI-like experience in your browser.

### Installation

\`\`\`bash
# Install code-server
curl -fsSL https://code-server.dev/install.sh | sh

# Install Continue extension
code-server --install-extension Continue.continue

# Set password
export PASSWORD="your-secure-password"
\`\`\`

### Launch (with tmux for persistence)

\`\`\`bash
tmux new -s code
code-server --bind-addr 0.0.0.0:8080 --auth password
\`\`\`

Then open `http://your-vps-ip:8080` in your browser.

### Configure Continue.dev

1. Click Continue icon in sidebar
2. Add model: Select "OpenAI-compatible API"
3. Set:
   - **Model**: kimi-k2-5-latest
   - **API Key**: your Moonshot key
   - **Base URL**: https://api.moonshot.cn/v1
4. Use Ctrl+L (chat) or Ctrl+I (inline edit)

---

## Mobile Setup with Terminus

### 1. Install tmux for persistence

\`\`\`bash
sudo apt install tmux -y
echo "set -g mouse on" >> ~/.tmux.conf
\`\`\`

### 2. Start your AI session

\`\`\`bash
tmux new -s kimi

# Inside tmux, run your chosen client:
aichat
# OR
sgpt
# OR
kimi
\`\`\`

### 3. Mobile Connection

**iOS/Android Options:**
- **Termius** (Free): Best UI, SFTP support
- **Blink Shell** (iOS/Paid): Mosh support for bad networks
- **JuiceSSH** (Android): Great key management

**Quick tmux commands:**

| Command | Action |
|---------|--------|
| `Ctrl+B D` | Detach (keep running) |
| `tmux attach -t kimi` | Reconnect |
| `tmux ls` | List sessions |

---

## Security Checklist

\`\`\`bash
# 1. Disable password auth (use SSH keys only)
sudo sed -i 's/^#*PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
sudo systemctl restart ssh

# 2. Install fail2ban
sudo apt install fail2ban -y

# 3. Firewall
sudo ufw allow 22/tcp
sudo ufw enable

# 4. For web IDE (Option 4), use SSH tunnel instead of opening port 8080
# From your local machine:
ssh -L 8080:localhost:8080 root@your-vps-ip
# Then access http://localhost:8080 locally
\`\`\`

---

## Environment Setup (Add to ~/.bashrc)

\`\`\`bash
# Kimi API
export MOONSHOT_API_KEY="sk-your-key-here"

# Aliases for quick access
alias kimi='aichat'  # If using Option 1
alias km='kimi'      # If using Option 3

# Auto-attach to tmux session
alias work='tmux attach -t kimi 2>/dev/null || tmux new -s kimi'
\`\`\`

---

## Troubleshooting

### "API key invalid" errors
- Verify key at https://platform.moonshot.cn
- Check key has `sk-` prefix
- Ensure billing is enabled (add 1元 if credits exhausted)

### Slow responses on VPS
- Kimi-K2.5 is powerful but slower than smaller models
- For faster responses, use `kimi-k2-5-32k` instead of `kimi-k2-5-128k`

### tmux scroll not working
\`\`\`bash
echo "set -g mouse on" >> ~/.tmux.conf
tmux source-file ~/.tmux.conf
\`\`\`

### High memory usage
Monitor with \`htop\`. If using Continue.dev in browser, expect 500MB-1GB RAM usage. CLI tools use <100MB.

---

## Recommended VPS Specs

| Provider | Plan | Price | Best For |
|----------|------|-------|----------|
| Hetzner CX21 | 2 vCPU/4GB | €5.35/mo | Europeans |
| DigitalOcean Basic | 1 vCPU/2GB | $6/mo | Beginners |
| Vultr Cloud Compute | 1 vCPU/2GB | $5/mo | Global network |
| AWS Lightsail | 2 vCPU/4GB | $10/mo | If you use AWS |

---

## Quick Start Commands (Copy-Paste Ready)

\`\`\`bash
# Complete setup for Option 1 (Aichat)
ssh root@your-vps-ip
curl -fsSL https://github.com/sigoden/aichat/releases/latest/download/aichat-x86_64-unknown-linux-musl.tar.gz | tar xz && sudo mv aichat /usr/local/bin/
mkdir -p ~/.config/aichat
cat > ~/.config/aichat/config.yaml << 'EOF'
model: moonshot:kimi-k2-5-latest
clients:
  - type: openai-compatible
    name: moonshot
    api_base: https://api.moonshot.cn/v1
    api_key: $MOONSHOT_API_KEY
EOF
echo 'export MOONSHOT_API_KEY="sk-your-key"' >> ~/.bashrc
source ~/.bashrc
sudo apt install tmux -y && tmux new -s kimi
aichat
\`\`\`

---

**License**: MIT  
**Note**: This guide uses real, working tools (Aichat, Shell-GPT) that integrate with Moonshot AI API. Kimi CLI does not exist as a standalone official product yet.
