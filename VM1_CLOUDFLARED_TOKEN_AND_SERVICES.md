# VM1 Cloudflared Token & Service Inventory
# Henrios LLC — Doctrios Dual-VM Deployment

## ⚠️ DO NOT MODIFY ROUTING for `doctrios.xyz` or `*.doctrios.xyz`
These are live production routes serving real traffic. The entries below
are read-only documentation. VM2 adds its own routes; it does not touch these.

---

## 1. Cloudflare Credentials (for VM2 to bootstrap its own tunnel)

### Cloudflare API Token
```
[REDACTED — see local file at /home/hype/.hermes/cache/scratch/vm1_cloudflared_token_and_services.md]
```
> **Note:** GitGuardian detected this token in the GitHub repo. The token has been redacted from the public repo and should be rotated.

- **Account ID**: `e8505dfe63f1212e8470f27f10f29c99`
- **Account Name**: Shanonblessed47@gmail.com's Account
- **Zone**: `doctrios.xyz` (Zone ID: `7fec0f5b4f3fbba18a417567d69131b4`)
- **Permissions needed**: Zone:DNS:Edit, Zone:Load Balancer:Edit, Account:Tunnel:Edit

### VM1 Tunnel Credentials (VM1 only — do not share externally)
```json
{
  "AccountTag": "e8505dfe63f1212e8470f27f10f29c99",
  "TunnelID": "f680d174-5b8c-453d-8077-b58ab0ff505b",
  "TunnelSecret": "[REDACTED — see local file]"
}
```
- **Tunnel CNAME**: `f680d174-5b8c-453d-8077-b58ab0ff505b.cfargotunnel.com`
- **Credentials file**: `~/.cloudflared/f680d174-5b8c-453d-8077-b58ab0ff505b.json`

---

## 2. VM1 — Currently Running Services

| Port | Service | Process | Purpose |
|------|---------|---------|---------|
| 80 | nginx | root | Reverse proxy — terminates Cloudflare Tunnel TLS |
| 22 | sshd | root | SSH access (key-based) |
| 4100 | node | pid 1291 | **Doctrios app** (production) |
| 8080 | python3 | pid 1119 | Hermes gateway / dashboard backend |
| 8081 | python3 | pid 1118 | Hermes API |
| 8120 | node | pid 1121 | Fashion / LIFTED FROM POVERTY service |
| 8130 | node | pid 1127 | PEGIONS — WhatsApp campaign SaaS |
| 9119 | hermes | pid 226649 | Hermes backend server (agent-to-agent API) |
| 20241 | cloudflared | pid 1329 | Cloudflare Tunnel metrics endpoint |

**DNS records for doctrios.xyz (do NOT modify):**
```
CNAME    doctrios.xyz       -> f680d174-5b8c-453d-8077-b58ab0ff505b.cfargotunnel.com   (proxied)
CNAME    *.doctrios.xyz     -> f680d174-5b8c-453d-8077-b58ab0ff505b.cfargotunnel.com   (proxied)
TXT      doctrios.xyz       -> "google-site-verification=..."                            (not proxied)
```

**nginx vhost routing (via port 80 — managed by cloudflared):**
```
doctrios.xyz            -> Doctrios app (:4100)
www.doctrios.xyz        -> Marketing landing (:80 static files)
henrios.doctrios.xyz    -> Henrios studio landing (:80 static files)
fashion.doctrios.xyz    -> Fashion service (:8120)
pegions.doctrios.xyz    -> PEGIONS SaaS (:8130)
safari.doctrios.xyz     -> SafariAfricanGiant (:5174)
tradios.henrios.xyz     -> Tradios (:3000)
gen-z-research.doctrios.xyz -> Hermes dashboard (:8080, :8081)
```

**Additional cloudflared ingress routes (direct to service ports):**
```
doctrios.app            -> http://localhost:4100  (Doctrios app, bypass nginx)
vm1.doctrios.app        -> http://localhost:4100  (VM1-specific Doctrios)
vm1-api.doctrios.app    -> http://localhost:9119  (VM1 Hermes backend API)
```

---

## 3. How VM2 Exposes Services (Without Interfering)

VM2 runs its OWN cloudflared tunnel and adds NEW subdomain CNAME records
under `doctrios.xyz`. Specific CNAMEs override the `*.doctrios.xyz` wildcard,
so existing traffic to `doctrios.xyz` and `*.doctrios.xyz` is unaffected.

### Steps for VM2:

```bash
# 1. Install cloudflared
curl -L https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb -o /tmp/cloudflared.deb
sudo dpkg -i /tmp/cloudflared.deb

# 2. Authenticate (VM2 uses the SAME Cloudflare account)
# [REDACTED Token — obtain from VM1 local file]
cloudflared tunnel login   # This creates cert.pem

# 3. Create a NEW tunnel on VM2
cloudflared tunnel create
# Output: New tunnel ID: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
# Save this as VM2_TUNNEL_ID

# 4. Configure VM2's tunnel
cat > ~/.cloudflared/config.yml << 'EOF'
tunnel: <VM2_TUNNEL_ID>
credentials-file: ~/.cloudflared/<VM2_TUNNEL_ID>.json
ingress:
  - hostname: vm2.doctrios.xyz
    service: http://localhost:4100
  - hostname: vm2-api.doctrios.app
    service: http://localhost:9119
  - service: http_status:404
EOF

# 5. Add DNS records
cloudflared tunnel route dns <VM2_TUNNEL_ID> vm2.doctrios.xyz
cloudflared tunnel route dns <VM2_TUNNEL_ID> vm2-api.doctrios.app

# 6. Start tunnel
cloudflared tunnel run <VM2_TUNNEL_ID>
```

### VM2 service mapping:
```
vm2.doctrios.xyz       -> VM2's Doctrios app (port 4100 on VM2)
vm2-api.doctrios.app   -> VM2's Hermes backend API (port 9119 on VM2)
```

---

## 4. Cloudflare Load Balancer (Optional)

### Pool Origin 1 (VM1):
- Tunnel ID: f680d174-5b8c-453d-8077-b58ab0ff505b
- DNS: vm1.doctrios.app -> f680d174-5b8c-453d-8077-b58ab0ff505b.cfargotunnel.com

### Pool Origin 2 (VM2):
- Tunnel ID: <VM2_TUNNEL_ID>
- DNS: vm2.doctrios.app -> <VM2_TUNNEL>.cfargotunnel.com

---

## 5. Agent-to-Agent Communication

- VM1 API: http://localhost:9119/health
- VM2 API: http://localhost:9119/health
- Cross-VM health check: curl -sf http://<PEER_IP>:9119/health

---

## 6. What NOT to Touch on VM1

| Config | Path | Status |
|--------|------|--------|
| System cloudflared config | /etc/cloudflared/config.yml | LIVE — do not modify |
| Systemd cloudflared service | systemctl status cloudflared | Running |
| Cloudflared watchdog | /usr/local/bin/cloudflared-watchdog.sh | Running |
| nginx config | /etc/nginx/sites-enabled/henrios-services.conf | LIVE — do not modify |
| Doctrios service | systemctl status doctrios-app | Running on :4100 |
| PEGIONS service | port 8130 | Running |
| Fashion service | port 8120 | Running |
| Hermes gateway | port 8080 | Running |
| Hermes API | port 8081 | Running |

---
*Generated: 2026-10-28 — Redacted for GitHub safety*
