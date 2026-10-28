# 🚀 DOCTRIOS AGENT PAIRING — Setup Prompt

**Paste this in the Telegram group. Both Hermes agents will read this and execute their roles.**

---

## Context

Two VMs (miles apart) run Hermes agents managing redundant Doctrios deployments behind Cloudflare Load Balancing.

- **VM1**: Primary — Hermes Agent PID 1464, serves `doctrios.xyz` via cloudflared tunnel `f680d174-5b8c-453d-8077-b58ab0ff505b`, origin `e8505dfe63f1212e8470f27f10f29c99`
- **VM2**: Secondary — Stand-in agent (this prompt) to be activated by user on second laptop
- **Shared DB**: Supabase/Cloud PostgreSQL at `postgresql://***@db.doctrios.xyz:5432/doctrios`
- **GitHub**: Token in `~/.bashrc` as `GITHUB_TOKEN`, repo `hodarihats/doctrios-agent-pairing` already created

## What Each Agent Does

### Both Agents (run on both VMs):

1. **Start Hermes backend server** (if not running):
   ```bash
   hermes serve --port 9119 --host 127.0.0.1
   ```

2. **Start cloudflared tunnel**:
   ```bash
   cloudflared tunnel --config ~/.cloudflared/config.yml run
   ```

3. **Start Doctrios service**:
   ```bash
   # Production (port 4100)
   systemctl is-active doctrios-app.service || systemctl start doctrios-app.service
   ```

4. **Coordinate via HTTP health API**:
   ```bash
   # Each agent exposes: http://localhost:9119/health
   # Check peer: curl -s http://<peer-ip>:9119/health
   curl -sf http://<peer-ip>:9119/health || echo "PEER DOWN"
   ```

### Agent Role Assignment:

- **Agent 1 (VM1 / Primary)**: Responds as "PRIMARY". Keeps cloudflared tunnel `f680d174` running. Routes `doctrios.xyz` to port 4100.
- **Agent 2 (VM2 / Secondary)**: Responds as "SECONDARY". Creates new cloudflared tunnel. Routes `vm2.doctrios.xyz` to port 4100.

## Health Check Script (run every minute via cron on both VMs):

```bash
#!/bin/bash
PEER_IP="${PEER_IP:-127.0.0.1}"  # Set via env or SSH tunnel
MY_ROLE="${ROLE:-primary}"         # primary or secondary

# Check local service
if curl -sf http://localhost:4100/health | grep -q "ok"; then
  LOCAL="healthy"
else
  LOCAL="down"
fi

# Check peer service
if curl -sf http://${PEER_IP}:9119/health | grep -q "ok"; then
  PEER="healthy"  
else
  PEER="down"
fi

# Report status to group (via Telegram bot)
curl -s "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage" \
  -d "chat_id=${GROUP_CHAT_ID}" \
  -d "text=[$(hostname)][$MY_ROLE] local=$LOCAL peer=$PEER @ $(date +%H:%M:%S)"

# If peer down and we are secondary → promote to primary mode
if [ "$PEER" = "down" ] && [ "$MY_ROLE" = "secondary" ]; then
  echo "$(date): Primary down, secondary taking over" >> /tmp/failover.log
  # Secondary's cloudflared tunnel inherits the load balancer origin
  # (Cloudflare LB health check will route traffic here)
fi

# If local down → alert
if [ "$LOCAL" = "down" ]; then
  echo "$(date): CRITICAL: local Doctrios service is down" >> /tmp/failover.log
fi
```

## Cloudflare Load Balancer Setup (user does this manually):

1. Go to Cloudflare Dashboard → Load Balancing → Create Load Balancer
2. **Hostname**: `doctrios.app` (CNAME at apex)
3. **Pool origins**:
   - Origin 1: VM1 tunnel ID `f680d174-5b8c-453d-8077-b58ab0ff505b` → `http://localhost:4100`
   - Origin 2: VM2 tunnel ID (TBD by Agent 2) → `http://localhost:4100`
4. **Health check**: HTTP GET `/`, expect 200, interval 30s
5. **Failover**: Both origins in pool, fallback to whichever is healthy

## What's Already Done:

- ✅ GitHub repo `hodarihats/doctrios-agent-pairing` created with this prompt
- ✅ VM1 cloudflared tunnel credentials at `~/.cloudflared/f680d174-...json`
- ✅ VM1 cloudflared config at `~/.cloudflared/config.yml` (routes to port 8080 — needs updating to 4100)
- ✅ GitHub token in `~/.bashrc` (line 123: `export GITHUB_TOKEN="..."`)
- ✅ VM1: Doctrios app on port 4100, QA on 4101
- ✅ Both agents share Telegram group for coordination

## Immediate Next Steps:

1. **Agent 1 (VM1)**: Update cloudflared config to route `doctrios.app` to port 4100, restart tunnel
2. **Agent 2 (VM2)**: Install cloudflared, authenticate, create tunnel, route `vm2.doctrios.app` to port 4100
3. **Both**: Start `hermes serve --port 9119`, set up cron health check, set PEER_IP
4. **User**: Create Cloudflare Load Balancer with both tunnels as origins
5. **Both**: Configure shared PostgreSQL database (update `DATABASE_URL` env var)

## Commands to Acknowledge:

- `AGENT 1 READY` — Agent 1 confirms setup complete
- `AGENT 2 READY` — Agent 2 confirms setup complete  
- `LB CONFIGURED` — User confirms Cloudflare load balancer is set up
- `FAILBACK TEST` — User wants to test failover (Agent 1 stops service, Agent 2 takes over)
