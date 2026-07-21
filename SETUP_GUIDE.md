# Wazuh SIEM — Complete Setup Guide

> A step-by-step guide to deploying Wazuh Security Information and Event Management on your server using Dokploy. No prior experience required.

---

## Table of Contents

1. [What is Wazuh?](#what-is-wazuh)
2. [What You Need Before Starting](#what-you-need-before-starting)
3. [Step 1 — Prepare Your Server](#step-1--prepare-your-server)
4. [Step 2 — Clone the Repository](#step-2--clone-the-repository)
5. [Step 3 — Generate Security Certificates](#step-3--generate-security-certificates)
6. [Step 4 — Create Your Environment File](#step-4--create-your-environment-file)
7. [Step 5 — Push to GitHub](#step-5--push-to-github)
8. [Step 6 — Deploy on Dokploy](#step-6--deploy-on-dokploy)
9. [Step 7 — Access Your Dashboard](#step-7--access-your-dashboard)
10. [Step 8 — Install Agents on Your Devices](#step-8--install-agents-on-your-devices)
11. [Environment Variables Explained](#environment-variables-explained)
12. [What Each Container Does](#what-each-container-does)
13. [Port Reference](#port-reference)
14. [Troubleshooting](#troubleshooting)
15. [Security Checklist](#security-checklist)

---

## What is Wazuh?

Wazuh is a **free, open-source security platform** that protects your computers and servers. Think of it as a security guard that:

- **Watches** what happens on all your devices (servers, laptops, desktops)
- **Detects** suspicious activity, viruses, and security problems
- **Alerts** you when something needs attention
- **Shows everything** in a visual dashboard you can access from any web browser

You install a small program (called an **agent**) on each device you want to protect. All agents report back to your Wazuh server, which collects and analyzes the data.

---

## What You Need Before Starting

| Item | Details |
|------|---------|
| **A server** | A VPS or dedicated server running Linux (Ubuntu 22.04+ recommended) |
| **Minimum specs** | 8 GB RAM, 4 CPU cores, 50 GB disk space |
| **Dokploy installed** | Your server should have Dokploy set up and running |
| **A domain name** | Something like `wazuh.yourcompany.com` pointing to your server |
| **GitHub account** | To store the configuration files |
| **SSH access** | Terminal access to your server |

---

## Step 1 — Prepare Your Server

Your server needs one setting changed so Wazuh's internal database (OpenSearch) can work properly. This is a one-time change.

**Open a terminal and connect to your server:**

```bash
ssh root@your-server-ip
```

**Run these two commands:**

```bash
# Allow the system to use enough memory for Wazuh's database
sudo sysctl -w vm.max_map_count=262144

# Make this setting permanent (survives server restarts)
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
```

**What this does:** Wazuh stores security data in a search engine called OpenSearch. By default, Linux limits how much memory a program can map. This command increases that limit so OpenSearch doesn't crash.

---

## Step 2 — Clone the Repository

The repository contains all the configuration files Wazuh needs.

```bash
# Download the repository to your server
git clone https://github.com/somuq-web/wazuh.git

# Go into the folder
cd wazuh
```

---

## Step 3 — Generate Security Certificates

Wazuh encrypts all communication between its components. For this, it needs security certificates. You generate them once, and they last for years.

**Run this command:**

```bash
docker compose -f generate-indexer-certs.yml run --rm generator
```

**What this does:** This runs a temporary program that creates digital "ID cards" (certificates) for each Wazuh component. These certificates ensure that the manager, indexer, and dashboard can trust each other and communicate securely.

**After it finishes, you should see new files:**

```bash
ls config/wazuh_indexer_ssl_certs/
```

You should see files like `root-ca.pem`, `wazuh.manager.pem`, `admin.pem`, etc. These are your certificates. **Do not lose them.**

---

## Step 4 — Create Your Environment File

The environment file (`.env`) contains your passwords and settings. This is where you customize Wazuh for your setup.

**Copy the template:**

```bash
cp .env.example .env
```

**Open it in a text editor:**

```bash
nano .env
```

**You will see this:**

```env
WAZUH_VERSION=4.14.6
DASHBOARD_EXTERNAL_PORT=443
INDEXER_USERNAME=admin
INDEXER_PASSWORD=CHANGE_ME_STRONG_PASSWORD
API_USERNAME=wazuh-wui
API_PASSWORD=CHANGE_ME_STRONG_PASSWORD
DASHBOARD_USERNAME=kibanaserver
DASHBOARD_PASSWORD=CHANGE_ME_STRONG_PASSWORD
OPENSEARCH_JAVA_OPTS=-Xms1g -Xmx1g
```

**Change each `CHANGE_ME_STRONG_PASSWORD` to a real, strong password.** Use a password manager to generate and save these. All three passwords must be different.

**Save and exit:** Press `Ctrl+X`, then `Y`, then `Enter`.

---

## Step 5 — Push to GitHub

Now push the configuration (with your certificates) to GitHub so Dokploy can access it.

```bash
# Add all files (including the certificates you just generated)
git add .

# Save them with a message
git commit -m "Add certificates and environment config"

# Push to GitHub
git push
```

---

## Step 6 — Deploy on Dokploy

1. **Open your Dokploy dashboard** in a web browser

2. **Create a new Compose service:**
   - Click **"Create Service"** or **"New Application"**
   - Select **"Compose"** as the type

3. **Connect to GitHub:**
   - Under **Source**, choose **GitHub**
   - Select the repository: `somuq-web/wazuh`
   - Branch: `main`
   - Compose Path: `./compose.yml`

4. **Set your environment variables:**
   - Go to the **"Environment"** tab in your service
   - Copy each line from your `.env` file (the one you created in Step 4)
   - Paste them into the environment section

   **Example:**
   ```
   WAZUH_VERSION=4.14.6
   DASHBOARD_EXTERNAL_PORT=443
   INDEXER_USERNAME=admin
   INDEXER_PASSWORD=YourRealStrongPassword1
   API_USERNAME=wazuh-wui
   API_PASSWORD=YourRealStrongPassword2
   DASHBOARD_USERNAME=kibanaserver
   DASHBOARD_PASSWORD=YourRealStrongPassword3
   OPENSEARCH_JAVA_OPTS=-Xms1g -Xmx1g
   ```

5. **Set your domain:**
   - Go to the **"Domains"** tab
   - Add your domain: `wazuh.yourcompany.com`
   - Enable **HTTPS** (Let's Encrypt)

6. **Deploy:**
   - Click the **"Deploy"** button
   - Wait 3-5 minutes for all containers to start

---

## Step 7 — Access Your Dashboard

**Open your browser and go to:**

```
https://wazuh.yourcompany.com
```

**Log in with:**

| Field | Value |
|-------|-------|
| Username | `admin` |
| Password | The `INDEXER_PASSWORD` you set in Step 4 |

**First things to do after logging in:**

1. Change the admin password (click your profile icon → Security → internal users → admin)
2. Explore the dashboard — you'll see security alerts, system information, and compliance reports

---

## Step 8 — Install Agents on Your Devices

An **agent** is a small program you install on every device you want Wazuh to monitor. Once installed, it automatically reports security data back to your server.

### Linux Server (Ubuntu/Debian)

```bash
# Download and install the agent
curl -so wazuh-agent.deb https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.14.6-1_amd64.deb

# Install it
sudo dpkg -i ./wazuh-agent.deb

# Tell it where your Wazuh server is (replace with your domain or IP)
sudo sed -i "s/<address>.*<\/address>/<address>wazuh.yourcompany.com<\/address>/" /var/ossec/etc/ossec.conf

# Start the agent
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```

### Windows PC

1. Download: https://packages.wazuh.com/4.x/windows/wazuh-agent-4.14.6-1.msi
2. Run the installer
3. When asked for the server address, enter: `wazuh.yourcompany.com`
4. Complete the installation

### macOS

```bash
# Download the installer
curl -so wazuh-agent.pkg https://packages.wazuh.com/4.x/macos/wazuh-agent-4.14.6-1.pkg

# Install it
sudo installer -pkg wazuh-agent.pkg -target /

# Configure the server address
sudo sed -i '' 's/<address>.*<\/address>/<address>wazuh.yourcompany.com<\/address>/' /Library/Ossec/etc/ossec.conf

# Start the agent
sudo /Library/Ossec/bin/wazuh-control start
```

### After Installing an Agent

The agent will appear in your Wazuh dashboard within a few minutes. Go to **Agents** in the dashboard to see all connected devices.

---

## Environment Variables Explained

Here is what each setting in your `.env` file does:

| Variable | Default | What It Does |
|----------|---------|--------------|
| `WAZUH_VERSION` | `4.14.6` | The version of Wazuh to install. All three components (manager, indexer, dashboard) must use the same version. Change this only if you want a specific version. |
| `DASHBOARD_EXTERNAL_PORT` | `443` | The port your dashboard is accessible from outside. If your server already uses port 443 for something else, change this to another number like `8443`. This is the port Traefik/reverse proxy will forward traffic to. |
| `INDEXER_USERNAME` | `admin` | The admin username for Wazuh's search engine (OpenSearch). Used by all three containers to talk to each other. **Do not change unless you know what you're doing.** |
| `INDEXER_PASSWORD` | `CHANGE_ME_STRONG_PASSWORD` | **Must change.** The admin password for the search engine. Also used as your dashboard login password. Make it long and unique (16+ characters recommended). |
| `API_USERNAME` | `wazuh-wui` | The username the dashboard uses to talk to the manager's API. **Do not change.** |
| `API_PASSWORD` | `CHANGE_ME_STRONG_PASSWORD` | **Must change.** The password for the API user. The dashboard uses this to fetch data from the manager. Must match what you set in the manager's internal config. |
| `DASHBOARD_USERNAME` | `kibanaserver` | The internal username the dashboard uses to authenticate with the search engine. **Do not change.** |
| `DASHBOARD_PASSWORD` | `CHANGE_ME_STRONG_PASSWORD` | **Must change.** The password the dashboard uses to connect to the search engine. Must match what's in `internal_users.yml`. |
| `OPENSEARCH_JAVA_OPTS` | `-Xms1g -Xmx1g` | How much memory the search engine (indexer) can use. The format is `-Xms<minimum> -Xmx<maximum>`. Set this to half your server's RAM, but never more than 31 GB. Example: if you have 16 GB RAM, use `-Xms8g -Xmx8g`. |

### Password Summary

You need to set **three different passwords**. They should all be different from each other:

1. **INDEXER_PASSWORD** — Used to log into the dashboard AND by all containers to talk to the search engine
2. **API_PASSWORD** — Used by the dashboard to talk to the manager's API
3. **DASHBOARD_PASSWORD** — Used by the dashboard to authenticate with the search engine internally

---

## What Each Container Does

Wazuh runs as three separate programs (containers) that work together:

### Wazuh Manager — The Brain

- **What it does:** Receives data from all agents, analyzes it for threats, stores logs, and runs security rules
- **Default ports:** 1514 (agents talk to this), 1515 (new agents register here), 514 (syslog), 55000 (REST API)
- **Analogy:** Like a security operations center that receives reports from all guards and decides what to do

### Wazuh Indexer — The Filing Cabinet

- **What it does:** Stores all the security data in a searchable database (OpenSearch). When you search the dashboard, this is what gets queried.
- **Default port:** 9200
- **Analogy:** Like a massive, searchable filing cabinet where all security reports are stored

### Wazuh Dashboard — The Control Room

- **What it does:** Provides the web interface you see in your browser. Shows alerts, graphs, agent status, and compliance reports.
- **Default port:** 5601 (exposed as 443 or your configured port)
- **Analogy:** Like the screens in a security control room showing camera feeds and alerts

---

## Port Reference

### External Ports (accessible from the internet)

| Port | Protocol | Purpose | Needs to be open? |
|------|----------|---------|-------------------|
| 1514 | TCP | Agents send security data here | Yes, if agents are outside your network |
| 1515 | TCP | New agents register here | Yes, when enrolling new agents |
| 514 | UDP | Syslog messages from network devices | Only if you use syslog |
| 55000 | TCP | Wazuh REST API (for management) | Optional, for API access |
| 443 (or your `DASHBOARD_EXTERNAL_PORT`) | TCP | Dashboard web interface | Yes, to access the dashboard |

### Internal Ports (only used between containers)

| Port | Protocol | Purpose |
|------|----------|---------|
| 9200 | TCP | Indexer API — manager and dashboard query this |

**Note:** If your server has a firewall, you need to open ports 1514 and 1515 at minimum for agents to connect.

---

## Troubleshooting

### Dashboard won't load

1. Check if all containers are running: `docker compose ps`
2. Check dashboard logs: `docker compose logs wazuh.dashboard`
3. Make sure your domain's DNS points to your server
4. Wait 2-3 minutes after deploying — containers need time to initialize

### Agents can't connect

1. Make sure port 1514 is open on your server's firewall
2. Verify the agent is configured with the correct server address
3. Check manager logs: `docker compose logs wazuh.manager`
4. On the agent machine, check: `sudo systemctl status wazuh-agent`

### "Bad gateway" or connection errors

1. The indexer might still be starting — wait 2-3 minutes
2. Check indexer logs: `docker compose logs wazuh.indexer`
3. Verify passwords match between `.env` and what's in the containers

### Container keeps restarting

1. Check the logs for the failing container: `docker compose logs <container-name>`
2. Common cause: wrong passwords or missing certificates
3. Make sure you generated certificates in Step 3

### Dashboard shows "Plugin setting up..."

- This is normal on first startup. Wait 2-3 minutes and refresh the page.

---

## Security Checklist

Before going live, make sure you:

- [ ] Changed all three passwords from `CHANGE_ME_STRONG_PASSWORD`
- [ ] Used different passwords for each variable
- [ ] Generated TLS certificates (Step 3)
- [ ] Set up HTTPS on your domain in Dokploy
- [ ] Opened only necessary ports in your firewall (1514, 1515, 443)
- [ ] Changed the default `kibanaserver` password after first login
- [ ] Changed the default `admin` password after first login
- [ ] Your `.env` file is NOT committed to a public repository (it's in `.gitignore`)
- [ ] `vm.max_map_count=262144` is set on the server

---

## Quick Reference Card

| Item | Value |
|------|-------|
| **Dashboard URL** | `https://wazuh.yourcompany.com` |
| **Login username** | `admin` |
| **Login password** | Your `INDEXER_PASSWORD` |
| **Agent enrollment port** | 1515 |
| **Agent communication port** | 1514 |
| **REST API endpoint** | `https://wazuh.yourcompany.com:55000` |

---

*Built for Dokploy. Based on [Wazuh Docker](https://github.com/wazuh/wazuh-docker) v4.14.6.*
