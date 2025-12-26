
# AWS Backend Deployment Guide

**EC2 + Nginx + systemd + Docker (Reusable SOP)**

---

## 0. Assumptions (Applies to All Projects)

| Item            | Standard                           |
| --------------- | ---------------------------------- |
| OS              | Ubuntu 22.04 LTS                   |
| Cloud           | AWS EC2                            |
| Reverse Proxy   | Nginx                              |
| Process Manager | systemd                            |
| App Runtime     | Python (FastAPI / Uvicorn)         |
| Containers      | Docker (host-level)                |
| DB              | Dockerized (Postgres / MySQL etc.) |
| App Binding     | `127.0.0.1:<PORT>`                 |

---

## 1. EC2 Provisioning (One-Time Per Server)

### 1.1 Launch Instance

* AMI: Ubuntu 22.04
* Open inbound ports:

  * 22 (SSH)
  * 80 (HTTP)
  * 443 (HTTPS)

### 1.2 SSH Access

```bash
ssh -i <key>.pem ubuntu@<PUBLIC_IP>
sudo su -
```

---

## 2. Base System Preparation (Always Required)

```bash
apt update
apt install -y \
  nginx \
  curl \
  ca-certificates \
  net-tools \
  certbot \
  python3-venv
```

Verify:

```bash
nginx -v
python3 --version
```

---

## 3. Docker Installation (Standard Runtime Layer)

```bash
apt update
apt install -y docker.io
systemctl enable --now docker
```

Validate:

```bash
docker ps
```

---

## 4. Standard Project Directory Layout (MANDATORY)

**All applications must live under:**

```bash
/var/www/vhosts/<PROJECT_NAME>
```

Setup:

```bash
mkdir -p /var/www/vhosts
cd /var/www/vhosts
git clone <REPO_URL> <PROJECT_NAME>
cd <PROJECT_NAME>
```

---

## 5. Application Environment Setup (Python / FastAPI)

### 5.1 Virtual Environment

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

### 5.2 Environment File

Create `.env` (optional but recommended):

```env
APP_ENV=production
APP_HOST=127.0.0.1
APP_PORT=8000
DATABASE_URL=postgres://user:pass@127.0.0.1:5432/db
```

---

## 6. Database via Docker (Reusable Pattern)

```bash
docker run -d \
  --name <PROJECT>-db \
  -e POSTGRES_USER=<user> \
  -e POSTGRES_PASSWORD=<pass> \
  -e POSTGRES_DB=<db> \
  -p 127.0.0.1:5432:5432 \
  postgres:latest
```

Rules:

* Database is **never publicly exposed**
* Always bind to `127.0.0.1`

---

## 7. systemd Service (PRODUCTION-READY – FINAL)

### 7.1 Service File

Path:

```bash
/etc/systemd/system/<PROJECT>.service
```

### **Authoritative Template (Use This Exactly)**

```ini
[Unit]
Description=<PROJECT> Backend FastAPI Service
After=network.target docker.service

[Service]
User=www-data
Group=www-data
WorkingDirectory=/var/www/vhosts/<PROJECT>

# ---- IMPORTANT ----
# systemd does NOT load shell profiles.
# Use a SINGLE PATH with venv first and system paths included.
Environment=PATH=/var/www/vhosts/<PROJECT>/.venv/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

ExecStart=/var/www/vhosts/<PROJECT>/.venv/bin/uvicorn \
  agent_v1.api.main:app \
  --host 127.0.0.1 \
  --port 8000 \
  --log-level info

# ---- Logging ----
StandardOutput=append:/var/log/<PROJECT>.out.log
StandardError=append:/var/log/<PROJECT>.err.log

Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

### 7.2 Enable Service

```bash
systemctl daemon-reexec
systemctl daemon-reload
systemctl enable <PROJECT>
systemctl start <PROJECT>
systemctl status <PROJECT>
```

---

## 8. Nginx Reverse Proxy (HTTPS + WebSockets)

### 8.1 Config File

```bash
/etc/nginx/conf.d/<PROJECT>.conf
```

### **Standard HTTPS Configuration (Recommended)**

```nginx
# ----------------------------------------------------
# HTTP → HTTPS redirect
# ----------------------------------------------------
server {
    listen 80;
    server_name <DOMAIN>;

    access_log /var/log/nginx/<PROJECT>.access.log;
    error_log  /var/log/nginx/<PROJECT>.error.log;

    location / {
        return 301 https://$host$request_uri;
    }
}

# ----------------------------------------------------
# HTTPS reverse proxy to FastAPI (Uvicorn)
# ----------------------------------------------------
server {
    listen 443 ssl http2;
    server_name <DOMAIN>;

    access_log /var/log/nginx/<PROJECT>.access.log;
    error_log  /var/log/nginx/<PROJECT>.error.log;

    ssl_certificate     /etc/letsencrypt/live/<DOMAIN>/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/<DOMAIN>/privkey.pem;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers off;

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_http_version 1.1;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;

        # WebSocket support (FastAPI / ASGI)
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        proxy_redirect off;
    }
}
```

Reload Nginx:

```bash
nginx -t
systemctl restart nginx
```

---

## 9. SSL (After DNS Is Ready)

```bash
certbot certonly -d <DOMAIN>
```

Certificates are stored under:

```bash
/etc/letsencrypt/live/<DOMAIN>/
```

---

## 10. Permissions & Runtime Access (MANDATORY)

### 10.1 File Ownership

```bash
chown -R www-data:www-data /var/www/vhosts/<PROJECT>
chmod -R 755 /var/www/vhosts/<PROJECT>
```

### 10.2 Docker Access for Backend User

```bash
usermod -aG docker www-data
systemctl restart <PROJECT>
```

Validate:

```bash
sudo -u www-data docker ps
```

---

## 11. Logs & Debugging (Daily Operations)

### Application Logs

```bash
tail -f /var/log/<PROJECT>.out.log
tail -f /var/log/<PROJECT>.err.log
```

### systemd Logs

```bash
journalctl -u <PROJECT> -f
```

### Nginx Logs

```bash
tail -f /var/log/nginx/<PROJECT>.*.log
```

---

## 12. Restart & Recovery SOP

| Scenario    | Command                       |
| ----------- | ----------------------------- |
| App crash   | `systemctl restart <PROJECT>` |
| DB crash    | `docker start <PROJECT>-db`   |
| Nginx issue | `systemctl restart nginx`     |
| Full reset  | `reboot`                      |

---

## 13. Mandatory Pre-Production Checklist

| Check                               | Status |
| ----------------------------------- | ------ |
| App runs on `127.0.0.1`             | ☐      |
| systemd service stable              | ☐      |
| Docker accessible from backend user | ☐      |
| HTTPS enabled                       | ☐      |
| WebSockets working                  | ☐      |
| DB not publicly exposed             | ☐      |

---

## 14. Reuse Rule (Very Important)

For every **new backend**, only update:

* `<PROJECT>`
* `<REPO_URL>`
* `<DOMAIN>`
* App import path in `ExecStart`
* Database credentials

Everything else **remains identical**.

---


