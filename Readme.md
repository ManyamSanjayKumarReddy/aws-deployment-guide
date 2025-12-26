Understood.
Below is a **reusable, project-agnostic AWS deployment guide** that you can **apply unchanged to any future backend project** (FastAPI / Django / Node.js can fit with small swaps).

This is **not explanatory training material**.
This is a **standard operating guide (SOP / Runbook)** you can follow every time you deploy a new backend to AWS.

You can copy this as **`AWS_NGINX_BACKEND_DEPLOYMENT_GUIDE.md`** and reuse it for every project.

---

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
| App Runtime     | Python / Node / Any                |
| DB              | Dockerized (Postgres / MySQL etc.) |
| App Binding     | `127.0.0.1:<PORT>`                 |

---

## 1. EC2 Provisioning (One-Time Per Server)

### 1.1 Launch Instance

* AMI: Ubuntu 22.04
* Open inbound:

  * 22 (SSH)
  * 80 (HTTP)
  * 443 (HTTPS)

### 1.2 SSH Access

```bash
ssh -i <key>.pem ubuntu@<PUBLIC_IP>
sudo su -
```

---

## 2. Base System Preparation (Always Do This)

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
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc
```

```bash
tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME}")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF
```

```bash
apt update
apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
systemctl enable --now docker
```

Validate:

```bash
docker ps
```

---

## 4. Standard Project Directory Layout (Mandatory)

**Always deploy apps here:**

```bash
/var/www/vhosts/<PROJECT_NAME>
```

Create:

```bash
mkdir -p /var/www/vhosts
cd /var/www/vhosts
```

Clone project:

```bash
git clone <REPO_URL> <PROJECT_NAME>
cd <PROJECT_NAME>
```

---

## 5. Application Environment Setup (Template)

### 5.1 Python Example

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

### 5.2 Environment File

Create `.env`:

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

* DB **never exposed publicly**
* Bind only to `127.0.0.1`

---

## 7. systemd Service (Mandatory for All Projects)

### 7.1 Service File

Path:

```bash
/etc/systemd/system/<PROJECT>.service
```

Template:

```ini
[Unit]
Description=<PROJECT> Backend Service
After=network.target docker.service

[Service]
User=www-data
Group=www-data
WorkingDirectory=/var/www/vhosts/<PROJECT>
EnvironmentFile=/var/www/vhosts/<PROJECT>/.env
ExecStart=/var/www/vhosts/<PROJECT>/.venv/bin/python \
  -m uvicorn main:app \
  --host 127.0.0.1 \
  --port 8000
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

### 7.2 Enable Service

```bash
systemctl daemon-reload
systemctl enable <PROJECT>
systemctl start <PROJECT>
systemctl status <PROJECT>
```

---

## 8. Nginx Reverse Proxy (Reusable Config)

### 8.1 Config File

```bash
/etc/nginx/conf.d/<PROJECT>.conf
```

```nginx
server {
    listen 80;
    server_name <DOMAIN>;

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Reload:

```bash
systemctl restart nginx
```

---

## 9. SSL (Always Do After DNS)

```bash
certbot certonly -d <DOMAIN>
```

(Optional: convert to full HTTPS redirect later.)

---

## 10. Permissions & Runtime Access

### 10.1 File Ownership

```bash
chown -R www-data:www-data /var/www/vhosts/<PROJECT>
chmod -R 755 /var/www/vhosts/<PROJECT>
```

### 10.2 Docker Access

```bash
usermod -aG docker www-data
systemctl restart <PROJECT>
```

Validate:

```bash
sudo -u www-data docker ps
```

---

## 11. Logs & Monitoring (Standard Commands)

### App Logs

```bash
journalctl -u <PROJECT> -f
```

### Nginx Logs

```bash
tail -f /var/log/nginx/*.log
```

### Port Validation

```bash
netstat -tulnp
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

## 13. Mandatory Checklist (Before Go-Live)

| Check                     | Status |
| ------------------------- | ------ |
| App runs on localhost     | ☐      |
| Nginx proxies correctly   | ☐      |
| systemd enabled           | ☐      |
| Docker auto-start enabled | ☐      |
| SSL issued                | ☐      |
| DB not public             | ☐      |

---

## 14. How You Reuse This

For **every new project**, you only change:

* `<PROJECT_NAME>`
* `<REPO_URL>`
* `<DOMAIN>`
* App start command
* Database credentials

Everything else stays **identical**.

---

## 15. This Document Is Your Rulebook

* No manual `uvicorn` in production
* No apps binding to `0.0.0.0`
* No DB ports open publicly
* No background shells
* Everything controlled by `systemd`

---

If you want next, I can give you:

* **One-page printable checklist**
* **Production Nginx SSL config**
* **Docker + DB backup strategy**
* **Multi-project server layout**
* **Zero-downtime deployment pattern**

Tell me which one.
