# Ollama AI Deployment – Predator + WSL Setup

![Ollama Logo](https://ollama.com/assets/ollama-logo.png)

## Overview

This project demonstrates how to deploy **Ollama AI models** in a **distributed setup**:

- **WSL Node** → Model worker (Ollama runs here, handles inference)
- **Predator Server** → Dashboard & reverse proxy (Nginx forwards requests to WSL)
- **Browser / Clients** → Access models via Predator IP, get fast responses

> ⚡ Key Principle:  
> **Predator only proxies requests**, does not run Ollama itself.  
> **WSL node runs the models**, ensuring isolated, efficient inference.

---

## Table of Contents

- [Requirements](#requirements)  
- [Architecture](#architecture)  
- [Setup Guide](#setup-guide)  
  - [1. WSL Node – Ollama Installation](#1-wsl-node--ollama-installation)  
  - [2. Predator – Nginx Reverse Proxy](#2-predator--nginx-reverse-proxy)  
  - [3. Environment Variables](#3-environment-variables)  
  - [4. Running Ollama](#4-running-ollama)  
- [Performance Tips](#performance-tips)  
- [Dashboard Access](#dashboard-access)  
- [Permanent Setup](#permanent-setup)  
- [License](#license)

- ⚡ Important Notes on Ollama Services

Predator Node

Ollama should be OFF on Predator.

Disable service to prevent port conflicts with WSL:

sudo systemctl stop ollama
sudo systemctl disable ollama


Working Node (WSL)

Ollama should be ON on Working Node (WSL).

Enable permanent background service for production:

sudo systemctl enable ollama
sudo systemctl start ollama
sudo systemctl status ollama


Reason:

Predator only acts as a reverse proxy (nginx) to forward requests.

WSL is the actual node running the Ollama model.

Ensures single source of truth, avoids port conflicts, and keeps response consistent.

---

## Requirements

- Windows 11 with **WSL 2** (Ubuntu 24.04 recommended)  
- Predator Linux server (any Ubuntu/Debian flavor)  
- Nginx installed on Predator  
- Network connectivity: Predator ↔ WSL node  
- Ollama AI installed on WSL node

---
Windows machine (bridge setup)

👉 Yeh Windows Admin PowerShell me.

Port forward Windows → WSL
netsh interface portproxy add v4tov4 listenport=11434 listenaddress=0.0.0.0 connectport=11434 connectaddress=172.23.122.107

Allow firewall
netsh advfirewall firewall add rule name="Ollama11434" dir=in action=allow protocol=TCP localport=11434

Test Windows bridge

PowerShell:

curl http://localhost:11434/api/tags


👉 JSON aaye → bridge working.

🟢 STEP 3 — Predator machine (gateway setup)

👉 Predator Linux terminal me.

Edit nginx config:

sudo nano /etc/nginx/conf.d/ollama.conf


Put:

upstream ollama_cluster {
    server 10.0.0.18:11434;
}

server {
    listen 11434;

    location / {
        proxy_pass http://ollama_cluster;
    }
}


Restart nginx:

sudo systemctl restart nginx

🟢 STEP 4 — Final test

👉 Predator machine:

curl http://localhost:11434/api/tags


👉 Dashboard:

http://10.0.0.195:11434

🔥 Final flow (real)
Dashboard
   ↓
Predator nginx (10.0.0.195)
   ↓
Windows bridge (10.0.0.18)
   ↓
WSL Ollama

┌───────────────┐ ┌───────────────┐
│ Browser / │ HTTP │ Predator │
│ Client App ├────────►│ Nginx Proxy │
└───────────────┘ └─────┬─────────┘
│
│
▼
┌───────────────┐
│ WSL Node │
│ Ollama AI │
│ Models │
└───────────────┘


- Predator → receives requests, forwards via **reverse proxy**  
- WSL Node → serves actual model responses  
- Predator Ollama service → **OFF** to avoid conflicts  

---

## Setup Guide

### 1️⃣ WSL Node – Ollama Installation
```bash
# Install Ollama
curl -fsSL https://ollama.com/install.sh | bash

# Verify installation
ollama --version

WSL Permanent Backend Service (systemd)

Ye setup WSL par Ollama ko permanent background mai run karne ke liye hai, taake terminal open na rakho aur Predator nginx se directly connect ho sake.

Service file location:

/etc/systemd/system/ollama.service


Content example:

[Unit]
Description=Ollama AI Server
After=network.target

[Service]
Type=simple
User=jared
WorkingDirectory=/home/jared
ExecStart=/usr/local/bin/ollama serve
Environment="OLLAMA_HOST=0.0.0.0"
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target

🔹 Steps to Enable

Service file create karo:

sudo nano /etc/systemd/system/ollama.service


Above content paste karo → save (CTRL+X → Y → Enter)

systemd reload karo:

sudo systemctl daemon-reload


Service enable & start karo:

sudo systemctl enable ollama
sudo systemctl start ollama


Status check karo:

sudo systemctl status ollama


Port verify karo:

ss -tulnp | grep 11434


Expected:

*:11434   LISTEN

🔹 Notes

.bashrc se OLLAMA_HOST=0.0.0.0 ollama serve &` remove karo → port conflict avoid karne ke liye

Predator par Ollama OFF rakho → WSL hi main service hai

Auto start → WSL boot hone ke saath

2️⃣ Predator – Nginx Reverse Proxy
sudo apt update
sudo apt install nginx -y

# Create proxy config
sudo nano /etc/nginx/conf.d/ollama.conf
raja@ready-Predator-PO3-620:/etc/nginx$ sudo cat /etc/nginx/conf.d/ollama.conf
[sudo] password for raja:
upstream ollama_cluster {
    server  10.0.0.18:11434;
   # server WORKER2_IP:11434;
}

server {
    listen 11434;

    location / {
        proxy_pass http://ollama_cluster;
        proxy_set_header Host $host;
    }
}
raja@ready-Predator-PO3-620:/etc/nginx$
# Test and restart
sudo nginx -t
sudo systemctl restart nginx
3️⃣ Environment Variables (WSL Node)
echo 'export OLLAMA_HOST=0.0.0.0' >> ~/.bashrc
echo 'export OLLAMA_NUM_PARALLEL=2' >> ~/.bashrc
echo 'export OLLAMA_CONTEXT_LENGTH=1024' >> ~/.bashrc

source ~/.bashrc

4️⃣ Running Ollama
OLLAMA_HOST=0.0.0.0 ollama serve &


Check listening port:

ss -tulnp | grep 11434

Performance Tips

Use lightweight models (e.g., deepseek-r1:1.5b) for faster response

Adjust parallelism: export OLLAMA_NUM_PARALLEL=2

Context length balance: export OLLAMA_CONTEXT_LENGTH=1024

GPU acceleration: Enable Ollama GPU options if available
Dashboard Access

Browser / Clients:

http://<Predator_IP>:11434


Dashboard queries Ollama on WSL, Predator proxies responses.

Permanent Setup

WSL Node: Ollama runs automatically via .bashrc or systemd service

Predator: Nginx always ON, Ollama OFF

Browser / Apps: Always hit Predator IP → Nginx → WSL Ollama

##############################
WSL resources optimize karo

.wslconfig (Windows home folder me) banaye / edit kare:

[wsl2]
memory=16GB          # WSL ko 16GB RAM allocate
processors=8         # 8 CPU cores
localhostForwarding=true


WSL restart karo:

wsl --shutdown
wsl



How to third working machine
🌐 Scenario

Current setup:

Predator (Linux) → nginx reverse proxy

WSL (Windows) → Ollama running

Predator shows model via reverse proxy from WSL

Goal:

Add another Linux machine as a working node for Ollama.

Predator should reverse proxy requests to all working nodes.

Each working node has Ollama running as a systemd service.

1️⃣ On the New Linux Node (Working Node)

Install Ollama

Follow same installation as WSL node: download .deb or tar.gz and install.

Create permanent Ollama service

sudo nano /etc/systemd/system/ollama.service


Paste:

[Unit]
Description=Ollama AI Server
After=network.target

[Service]
Type=simple
User=jared           # replace with your user
WorkingDirectory=/home/jared
ExecStart=/usr/local/bin/ollama serve
Environment="OLLAMA_HOST=0.0.0.0"
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target


Enable and start service

sudo systemctl daemon-reload
sudo systemctl enable ollama
sudo systemctl start ollama
sudo systemctl status ollama   # check it’s running


Check if listening

ss -tulnp | grep 11434


Should show 0.0.0.0:11434

2️⃣ Update Predator nginx Reverse Proxy

Edit Ollama nginx upstream

sudo nano /etc/nginx/conf.d/ollama.conf


Example for two working nodes:

upstream ollama_cluster {
    server  172.23.112.1:11434;   # WSL node
    server  10.0.0.220:11434;     # New Linux node
}

server {
    listen 11434;

    location / {
        proxy_pass http://ollama_cluster;
        proxy_set_header Host $host;
    }
}


Test nginx config

sudo nginx -t


Reload nginx

sudo systemctl reload nginx

3️⃣ Verify Setup

Predator → All nodes

curl http://localhost:11434/api/tags


Response should show models from all nodes.

Load balancing:

nginx will round-robin by default across nodes.

Optional: configure least_conn or ip_hash if needed.

4️⃣ Notes / Best Practices

Predator Ollama OFF, only nginx runs → it proxies to nodes.

Each working node Ollama ON and enabled.

If you add more nodes, just add node IP:11434 to upstream and reload nginx.

Use same model names across nodes if you want uniform availability.

For Windows nodes, use portproxy if Predator can’t access WSL IP directly.

License

MIT License © 2026

Authors: Raja Ramees
Contact: rajaramees005@gmail.com

## Architecture

