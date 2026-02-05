# Multi-Agent Dashboard 🐦‍🔥🌿

Dashboard unificado para monitorear múltiples agentes OpenClaw en tiempo real.

## Features

- 🎯 **Fleet Status**: Estado consolidado de todos los agentes
- 📊 **Agent Cards**: Métricas individuales por agente (modelo, CPU, RAM, temperatura)
- 🔄 **Auto-refresh**: Actualización automática cada 30 segundos
- 📱 **PWA Ready**: Instalable en dispositivos móviles con icono personalizado
- 🌐 **Tailscale**: Accesible via red privada

## Architecture

### Data Flow

```
Agent (Raspberry Pi) → generates metrics JSON → shared folder
                                                      ↓
Dashboard Server (Mac) ← pulls via SSH ← cron (30s)
                                ↓
                          merges to master JSON
                                ↓
                          Frontend renders
```

### Components

**Backend:**
- `server.py` - Lightweight HTTP server (Python)
- `update.sh` - Metrics collector (bash)
- SSH key-based auth between agents

**Frontend:**
- `index.html` - Single-page dashboard with live updates
- `manifest.json` - PWA configuration
- `phoenix-icon.svg` - App icon

## Setup

### 1. Agent Side (Generate Metrics)

Create a script on each agent to generate `status.json`:

```bash
#!/bin/bash
# generate-metrics.sh

# System metrics
CPU=$(top -bn1 | grep "Cpu(s)" | awk '{print $2}' | cut -d'%' -f1)
RAM=$(free | grep Mem | awk '{print int($3/$2 * 100)}')
TEMP=$(cat /sys/class/thermal/thermal_zone0/temp 2>/dev/null | awk '{print int($1/1000)}' || echo 0)

# OpenClaw context (from session)
CONTEXT=$(cat ~/.openclaw/workspace/.agent-context 2>/dev/null || echo "0")

# Generate JSON
cat > ~/.openclaw/workspace/shared/agent-status.json << EOF
{
  "agent": "AgentName",
  "status": "online",
  "timestamp": "$(date -Iseconds)",
  "system": {
    "cpu": $CPU,
    "ram": $RAM,
    "temp": $TEMP
  },
  "openclaw": {
    "version": "$(openclaw --version 2>/dev/null | cut -d' ' -f2)",
    "model": "anthropic/claude-opus-4-5",
    "contextPercent": $CONTEXT
  }
}
EOF
```

Run via cron every minute:
```bash
* * * * * /path/to/generate-metrics.sh
```

### 2. Dashboard Side (Collect & Display)

**Install dependencies:**
```bash
# Python 3.8+
python3 --version
```

**Configure SSH access to agents:**
```bash
# Generate key if needed
ssh-keygen -t ed25519 -f ~/.ssh/id_dashboard

# Copy to agents
ssh-copy-id -i ~/.ssh/id_dashboard.pub agent@agent-host
```

**Create update script (`update.sh`):**
```bash
#!/bin/bash
DATA_FILE="./data/dashboard.json"

# Pull metrics from agents via SSH
AGENT1=$(ssh agent1@host1 "cat ~/.openclaw/workspace/shared/agent-status.json" 2>/dev/null || echo '{"status":"offline"}')
AGENT2=$(ssh agent2@host2 "cat ~/.openclaw/workspace/shared/agent-status.json" 2>/dev/null || echo '{"status":"offline"}')

# Extract values
STATUS1=$(echo "$AGENT1" | jq -r '.status')
# ... (extract other fields)

# Build consolidated JSON
cat > "$DATA_FILE" << EOF
{
  "fleet": {
    "status": "🟢 All systems operational"
  },
  "agent1": $(echo "$AGENT1"),
  "agent2": $(echo "$AGENT2"),
  "timestamp": "$(date -Iseconds)"
}
EOF
```

**Run updater:**
```bash
# One-time
./update.sh

# Continuous (recommended)
while true; do ./update.sh; sleep 30; done

# Or via LaunchAgent (macOS)
```

**Start server:**
```bash
python3 server.py 3334
# Dashboard at http://localhost:3334
```

### 3. Frontend Integration

Update `index.html` to read your agent names and metrics:

```javascript
// Iris
if (data.iris) {
    document.getElementById('irisStatus').textContent = data.iris.status === 'online' ? 'Online 🌿' : 'Offline';
    document.getElementById('irisCpu').textContent = data.iris.system.cpu + '%';
    // ...
}
```

## Access

- **Local**: http://localhost:3334
- **Tailscale**: http://<tailscale-ip>:3334
- **Mobile**: Install as PWA (Add to Home Screen)

## Customization

### Add New Agent

1. **Metrics generator** on agent machine
2. **SSH pull** in `update.sh`
3. **Frontend card** in `index.html`:
```html
<div class="agent-section">
  <div class="agent-name">New Agent 🤖</div>
  <div class="agent-status" id="newagentStatus">Loading...</div>
  <!-- metrics -->
</div>
```
4. **JavaScript updater**:
```javascript
if (data.newagent) {
    document.getElementById('newagentStatus').textContent = data.newagent.status;
}
```

### Styling

Edit CSS in `<style>` block of `index.html`. Key variables:

```css
/* Agent colors */
.phoenix-section { background: rgba(59, 130, 246, 0.08); }
.iris-section { background: rgba(52, 211, 153, 0.08); }

/* Theme */
body { background: linear-gradient(135deg, #1a1a2e 0%, #16213e 100%); }
```

## Production Tips

1. **Use LaunchAgent/systemd** for auto-start
2. **Monitor SSH timeouts** (set ConnectTimeout=2)
3. **Log errors** for debugging
4. **Backup dashboard.json** periodically
5. **Set up Tailscale ACLs** for secure remote access

## Tech Stack

- **Backend**: Python 3 (stdlib only), Bash
- **Frontend**: Vanilla JS, CSS Grid/Flexbox
- **Data**: JSON over SSH
- **Deployment**: Standalone (no external dependencies)

## Security

- ✅ SSH key-based auth (no passwords)
- ✅ Tailscale for remote access (encrypted WireGuard)
- ✅ No exposed ports (Tailscale or localhost only)
- ✅ Read-only SSH access to metrics files
- ⚠️ **Note**: This assumes trusted agents. For untrusted sources, add input validation.

## License

MIT

## Credits

Built by Phoenix 🐦‍🔥 & Iris 🌿 for multi-agent OpenClaw deployments.

---

**Use Case**: Monitor a fleet of AI agents across multiple machines (laptops, servers, Raspberry Pis) from a single unified dashboard.
