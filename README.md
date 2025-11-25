# Deploy Guide – ModSecurity-protected YAS stack

This document summarizes the end-to-end deployment used in our project (Tân's section). It covers bringing up YAS services, hardening them with ModSecurity on the bastion, exposing the storefront via SSH tunnel, and wiring the attack-simulator stack.

---

## System Architecture & Topology

### Overview
The deployment consists of 4 main components:
1. **Application Host** (`192.168.100.10`): Docker Compose stack running YAS microservices
2. **Bastion/WAF Server** (`100.101.119.23`): Nginx reverse proxy with ModSecurity WAF
3. **Control Panel**: Web-based ModSecurity management (backend on bastion, frontend accessible locally)
4. **Attack Simulator**: C2 server + bot agents for security testing

### Network Topology

```
┌─────────────────────────────────────────────────────────────────┐
│                    Local Workstation (taotentanyb)              │
│  ┌─────────────────┐  ┌──────────────────┐  ┌──────────────┐  │
│  │  Web Browser    │  │  C2 Server       │  │  Local Bot   │  │
│  │  (storefront)   │  │  :5050           │  │  :6060       │  │
│  └────────┬────────┘  └────────┬─────────┘  └──────┬───────┘  │
│           │                     │                    │          │
│           │ SSH Tunnel :8080    │ HTTP API           │          │
└───────────┼─────────────────────┼────────────────────┼──────────┘
            │                     │                    │
            │                     │                    │
┌───────────▼─────────────────────▼────────────────────▼──────────┐
│              Bastion/WAF Server (100.101.119.23)                │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │  Nginx Reverse Proxy (:8080)                             │ │
│  │  ┌────────────────────────────────────────────────────┐  │ │
│  │  │  ModSecurity WAF (OWASP CRS)                       │  │ │
│  │  │  - SecRuleEngine: On/Off                            │  │ │
│  │  │  - Audit Log: /var/log/modsec_audit.log            │  │ │
│  │  └────────────────────────────────────────────────────┘  │ │
│  └──────────────────────────────────────────────────────────┘ │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │  ModSecurity Control Panel                                │ │
│  │  - Backend API (:4000)                                    │ │
│  │  - Frontend (dev mode :5173, or via tunnel)              │ │
│  └──────────────────────────────────────────────────────────┘ │
└───────────────────────────┬────────────────────────────────────┘
                            │
                            │ HTTP Proxy (port 80)
                            │
┌───────────────────────────▼────────────────────────────────────┐
│         Application Host (192.168.100.10)                     │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │  Docker Compose Stack (yas-main)                         │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │ │
│  │  │  Storefront  │  │  Backoffice  │  │  API Gateway │  │ │
│  │  │  (Next.js)   │  │              │  │  (api.yas.*) │  │ │
│  │  └──────────────┘  └──────────────┘  └──────────────┘  │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │ │
│  │  │  PostgreSQL  │  │  Kafka       │  │  Monitoring  │  │ │
│  │  └──────────────┘  └──────────────┘  └──────────────┘  │ │
│  └──────────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                    Kali Bot (100.122.20.18)                     │
│  ┌──────────────────┐                                           │
│  │  Bot Agent :6060 │                                           │
│  │  - XSS attacks   │                                           │
│  │  - SQLi (sqlmap) │                                           │
│  │  - L4 DDoS       │                                           │
│  └────────┬─────────┘                                           │
│           │                                                      │
│           │ SSH Tunnel :8080                                     │
└───────────┼──────────────────────────────────────────────────────┘
            │
            │
    ┌───────▼──────────────────────────────────────────────────────┐
    │  Same Bastion (100.101.119.23) → Application (192.168.100.10)│
    └──────────────────────────────────────────────────────────────┘
```

### Component Details

#### 1. Application Host (`192.168.100.10`)
- **Role**: Runs the YAS microservices application stack
- **Services**: 
  - Storefront (Next.js frontend)
  - Backoffice (admin interface)
  - API Gateway (`api.yas.local`)
  - PostgreSQL database
  - Kafka message broker
  - Monitoring stack (Grafana, Loki, Tempo, Elasticsearch)
- **Access**: Only accessible from bastion server via private network

#### 2. Bastion/WAF Server (`100.101.119.23`)
- **Role**: Public-facing reverse proxy with Web Application Firewall
- **Components**:
  - **Nginx** (port 8080): Reverse proxy to application host
  - **ModSecurity**: WAF engine with OWASP Core Rule Set (CRS)
  - **Control Panel Backend** (port 4000): REST API for WAF management
  - **Control Panel Frontend** (port 5173): React UI (dev mode)
- **Security**: All incoming traffic is inspected by ModSecurity before forwarding

#### 3. Control Panel
- **Backend** (`~/modsec-panel/server`):
  - Express.js API server
  - Manages ModSecurity configuration (`/etc/nginx/modsec/main.conf`)
  - Controls Nginx reload/status
  - Streams audit logs
- **Frontend** (`~/modsec-panel/client`):
  - React + Vite application
  - Real-time WAF status monitoring
  - Rule engine toggle (On/Off)
  - Attack log viewer

#### 4. Attack Simulator Stack
- **C2 Server** (runs on local workstation, port 5050):
  - Orchestrates attack missions
  - Distributes commands to registered bots
  - Collects and aggregates results
  - Manages mission lifecycle
- **Bot Agents** (port 6060):
  - **Local Bot** (`taotentanyb`): Runs on local workstation
  - **Kali Bot** (`100.122.20.18`): Runs on Kali VM
  - Execute attack payloads:
    - **Layer 7**: XSS (Cross-Site Scripting), SQLi (SQL Injection)
    - **Layer 4**: DDoS (SYN flood via `hping3`)

### Data Flow

#### Normal User Access
```
Browser → SSH Tunnel (localhost:8080) → Bastion Nginx (8080) 
  → ModSecurity Inspection → Application Host (80) → Docker Services
```

#### Attack Simulation Flow
```
C2 Server → HTTP POST /missions → Bot Agents (HTTP API)
  → Bot executes attack → SSH Tunnel → Bastion Nginx 
  → ModSecurity (blocks/detects) → Application Host
  → Bot collects response → Returns to C2 → Results aggregated
```

#### Control Panel Access
```
Browser → SSH Tunnel (localhost:4000) → Control Panel Backend (4000)
  → Reads/Writes ModSecurity config → Reloads Nginx
  → Streams audit logs → Frontend displays status
```

### Port Mapping

| Service | Host | Port | Protocol | Access |
|---------|------|------|----------|--------|
| Storefront (via WAF) | Bastion | 8080 | HTTP | SSH tunnel → `localhost:8080` |
| Control Panel API | Bastion | 4000 | HTTP | SSH tunnel → `localhost:4000` |
| Control Panel UI | Bastion | 5173 | HTTP | Direct or SSH tunnel |
| C2 Server | Local | 5050 | HTTP | `localhost:5050` |
| Local Bot | Local | 6060 | HTTP | `localhost:6060` |
| Kali Bot | Kali VM | 6060 | HTTP | `http://100.122.20.18:6060` |
| Application Stack | App Host | 80 | HTTP | Internal (via bastion) |

### Security Boundaries

1. **Public Internet** → **Bastion Server**: Only SSH (22) and optionally HTTP (8080, 4000, 5173) if exposed
2. **Bastion** → **Application Host**: Private network (192.168.100.0/24), HTTP proxy only
3. **Local/Kali** → **Bastion**: SSH tunnels for secure access
4. **ModSecurity**: Inspects all HTTP traffic at Layer 7 before reaching application

### Prerequisites

- **Network Access**:
  - SSH access to `thethinh@100.101.119.23` (bastion)
  - SSH access to `thethinh@192.168.100.10` (application host)
  - Network connectivity to `kali@100.122.20.18` (Kali bot)
- **Software**:
  - Docker & Docker Compose on application host
  - Node.js 20+ on bastion and local machines
  - `sshpass` for non-interactive SSH
  - `sqlmap` and `hping3` on bot machines
- **Configuration**:
  - `/etc/hosts` entries on local workstation and bot machines
  - SSH tunnels established and kept open during testing

---

## 1. Bring up YAS (application host `192.168.100.10`)
1. SSH:
   ```bash
   sshpass -p 'password' ssh thethinh@192.168.100.10
   ```
2. Start the docker compose stack (needs sudo):
   ```bash
   cd ~/yas-main
   echo password | sudo -S docker compose -f docker-compose.yml up -d
   ```
3. Verify:
   ```bash
   sudo docker ps | grep yas-main
   ```

## 2. Configure nginx + ModSecurity (bastion `100.101.119.23`)
1. Install packages (already done once):
   ```bash
   echo password | sudo -S apt install -y \
     nginx libnginx-mod-http-modsecurity libnginx-mod-http-ndk modsecurity-crs
   ```
2. Prepare ModSecurity files:
   ```bash
   echo password | sudo -S mkdir -p /etc/nginx/modsec
   echo password | sudo -S cp /opt/ModSecurity/modsecurity.conf-recommended /etc/nginx/modsec/modsecurity.conf
   echo password | sudo -S sed -i 's/SecRuleEngine DetectionOnly/SecRuleEngine On/' /etc/nginx/modsec/modsecurity.conf
   echo password | sudo -S cp /etc/nginx/unicode.mapping /etc/nginx/modsec/unicode.mapping
   ```
3. Include CRS:
   ```bash
   echo password | sudo -S bash -c 'cat <<EOF > /etc/nginx/modsec/main.conf
   Include /etc/nginx/modsec/modsecurity.conf
   Include /etc/nginx/modsec/coreruleset/crs-setup.conf
   Include /etc/nginx/modsec/coreruleset/rules/*.conf
   EOF'
   ```
4. Nginx adjustments:
   - In `/etc/nginx/nginx.conf` inside `http {}` add:
     ```
        modsecurity on;
        modsecurity_rules_file /etc/nginx/modsec/main.conf;
     ```
   - Site `storefront-waf` (`/etc/nginx/sites-available/storefront-waf`):
     ```nginx
     server {
         listen 8080;
         server_name _;
         location / {
             proxy_pass http://192.168.100.10;
             proxy_http_version 1.1;
             proxy_set_header Host storefront;
             proxy_set_header X-Real-IP $remote_addr;
             proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
             proxy_set_header X-Forwarded-Proto $scheme;
             proxy_set_header Connection "";
         }
     }
     ```
   - Enable sites:
     ```bash
     echo password | sudo -S ln -sf /etc/nginx/sites-available/storefront-waf /etc/nginx/sites-enabled/storefront-waf
     echo password | sudo -S ln -sf /etc/nginx/sites-available/yas-project /etc/nginx/sites-enabled/yas-project
     echo password | sudo -S rm -f /etc/nginx/sites-enabled/default
     ```
5. Validate & restart:
   ```bash
   echo password | sudo -S nginx -t
   echo password | sudo -S systemctl restart nginx
   ```

## 3. ModSecurity Control Panel (backend + UI)
Located in `~/modsec-panel`.

1. Backend API (`server/`):
   ```bash
   cd ~/modsec-panel/server
   npm install
   echo "PORT=4000" > .env   # plus MODSEC paths as in README
   # create systemd service (modsec-panel.service) as described in README_modsecurity.md
   echo password | sudo -S systemctl enable --now modsec-panel.service
   ```
   This exposes `http://100.101.119.23:4000/api/...` for status/config/logs.

2. Frontend (`client/`):
   ```bash
   cd ~/modsec-panel/client
   npm install
   npm run dev -- --host 0.0.0.0 --port 5173   # dev mode
   ```
   Script `npm run dev` already sets `VITE_API_BASE=http://100.101.119.23:4000`.

## 4. Attack Simulator Stack
### C2 (runs on taotentanyb)
```bash
cd ~/Downloads/modsec-panel/attack-c2
npm install
cp bots.sample.json bots.json   # edit baseUrl for each bot
npm run dev   # exposes http://localhost:5050
```

### Bot agents
Run on **two machines** (local + Kali):
```bash
cd ~/Downloads/modsec-panel/attack-bot   # or copied folder
npm install
BOT_HOSTNAME=<bot-name> BOT_PORT=6060 npm start
```
Each bot must have:
- `/etc/hosts` entries mapping `storefront`, `api.yas.local` to `127.0.0.1`.
- An SSH tunnel to the bastion:
  ```bash
  sshpass -p 'password' ssh -L 8080:192.168.100.10:80 thethinh@100.101.119.23
  ```
Keep tunnel terminals open while missions run.

## 5. Expose storefront via SSH tunnel (workstation)
1. Add to `/etc/hosts` (local machine):
   ```
   127.0.0.1 storefront
   127.0.0.1 api.yas.local
   127.0.0.1 backoffice
   127.0.0.1 pgadmin.yas.local
   ```
2. Open tunnel:
   ```bash
   sshpass -p 'password' ssh -L 8080:192.168.100.10:80 thethinh@100.101.119.23
   ```
3. Browse `http://storefront:8080/` to reach the WAF-protected site.

## 6. Example test flow
1. Verify WAF blocking:
   ```bash
   curl -I -H 'Host: storefront' http://127.0.0.1:8080/
   curl -I -H 'Host: storefront' "http://127.0.0.1:8080/?q=<script>alert(1)</script>"
   tail -n 40 /var/log/modsec_audit.log
   ```
2. Launch attack mission:
   ```bash
   curl -X POST http://localhost:5050/missions \
     -H 'Content-Type: application/json' \
     -d '{
           "type":"xss",
           "targetUrl":"http://api.yas.local/storefront/catalog-search?keyword=",
           "payload":{"param":"keyword","value":"<script>alert(1)</script>","requestsPerBot":20,"timeoutMs":5000}
         }'
   ```
   Inspect bot logs (`[BOT bot-...] POST /execute 200`) and C2 mission output to compare SecRuleEngine On/Off.

---

With these steps the deployment is reproducible: docker services on 192.168.100.10, WAF/nginx/control-panel on 100.101.119.23, SSH tunnels for UI access, and attack simulator (C2 + bots) for Layer 7/Layer 4 testing.


