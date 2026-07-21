# Wazuh SIEM — Dokploy Deployment

Single-node Wazuh SIEM (v4.14.6) for Dokploy. Three containers: manager, indexer, dashboard.

## Requirements

- **RAM:** 8 GB minimum (16 GB recommended)
- **CPU:** 4 cores minimum
- **Disk:** 50 GB minimum
- **Docker host kernel setting:**

```bash
sudo sysctl -w vm.max_map_count=262144
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
```

## Deploy on Dokploy

### 1. Generate TLS certificates (one-time, on the server)

```bash
git clone https://github.com/somuq-web/Wazuh.git
cd Wazuh
docker compose -f generate-indexer-certs.yml run --rm generator
```

This creates self-signed certs in `config/wazuh_indexer_ssl_certs/`.

### 2. Create a `.env` file with your secrets

```bash
cp .env.example .env
# Edit .env with strong passwords
```

### 3. Push to GitHub

```bash
git add .
git commit -m "Add generated certificates"
git push
```

### 4. Deploy on Dokploy

- Create a new Compose service
- Provider: GitHub → `somuq-web/Wazuh` → branch `main`
- Compose Path: `./compose.yml`
- Paste environment variables from `.env` into the Environment tab
- Deploy

### 5. Access the dashboard

Open `https://wazuh.yourdomain.com` (or the Dokploy-assigned domain).

- **Username:** `admin`
- **Password:** The `INDEXER_PASSWORD` you set

Change the password after first login.

## Ports

| Port | Protocol | Purpose |
|---|---|---|
| 1514 | TCP | Agent communication |
| 1515 | TCP | Agent enrollment |
| 514 | UDP | Syslog |
| 55000 | TCP | Wazuh REST API |
| 9200 | TCP | Indexer API |
| 443 | TCP | Dashboard web UI |

## Install Agents

### Linux

```bash
WAZUH_MANAGER='your-server-ip' WAZUH_AGENT_NAME='my-server' \
  wget https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.14.6-1_amd64.deb && \
  sudo dpkg -i ./wazuh-agent_4.14.6-1_amd64.deb
sudo systemctl enable wazuh-agent && sudo systemctl start wazuh-agent
```

### Windows

Download from: https://packages.wazuh.com/4.x/windows/wazuh-agent-4.14.6-1.msi

```cmd
wazuh-agent-4.14.6-1.msi /q WAZUH_MANAGER="your-server-ip" WAZUH_AGENT_NAME="my-windows"
```

## Included Capabilities

- File Integrity Monitoring (FIM)
- Vulnerability Detection
- Security Configuration Assessment (SCA)
- System Inventory (Syscollector)
- Rootkit Detection
- Active Response
- Log Data Analysis

## Useful Commands

```bash
# Check container health
docker compose ps

# View logs
docker compose logs -f wazuh.manager
docker compose logs -f wazuh.indexer
docker compose logs -f wazuh.dashboard

# List enrolled agents
curl -k -u wazuh-wui:YOUR_API_PASSWORD https://localhost:55000/agents

# Restart stack
docker compose restart
```

## License

GPLv2 — Same as Wazuh.
