# AWS EC2 Deployment Guide
### For Beginners — Docker + Nginx + Node.js API

> **Who is this for?**  
> You've never deployed to AWS before, or you keep forgetting the steps. This guide walks you through everything from connecting to your server to verifying your app is live.

---

## Prerequisites

Before you start, make sure you have:

- An AWS EC2 instance running **Ubuntu**
- Your `.pem` key file downloaded (e.g., `hostname.pem`)
- Your project code in a **GitHub repo**
- A `docker.env` (or `.env.example`) file ready to fill in

---

## Part 1 — Connect to Your Server

Open a terminal on your **local machine** and navigate to the folder where your `.pem` key lives:

```bash
cd /path/to/your/pem-file
```

Fix the key file permissions (required by SSH — do this once):

```bash
chmod 400 "hostname.pem"
```

Now connect to your EC2 instance:

```bash
ssh -i "hostname.pem" ubuntu@ec2-52-212-177-154.eu-west-1.compute.amazonaws.com
```

> 💡 Replace the hostname with **your own** EC2 Public DNS, found in the AWS Console under EC2 → Instances → your instance.

---

## Part 2 — Set Up the Server (First Time Only)

Once connected, run these steps **in order**.

### 2.1 Update the system

```bash
sudo apt update && sudo apt upgrade -y
```

### 2.2 Install Nginx (web server / reverse proxy)

```bash
sudo apt install nginx -y
sudo systemctl start nginx
sudo systemctl enable nginx
```

Verify it's running:

```bash
sudo systemctl status nginx
```

### 2.3 Install Docker

```bash
sudo apt install docker.io -y
sudo systemctl start docker
sudo systemctl enable docker
docker --version
```

### 2.4 Install Docker Compose

```bash
sudo apt install docker-compose -y
docker compose version
```

---

## Part 3 — Deploy Your Application

### 3.1 Clone your repo

```bash
cd /var/www
git clone <your-repo-url> AEORanker
cd AEORanker
```

### 3.2 Create your environment file

```bash
cp docker.env.example docker.env
nano docker.env
```

> ⚠️ Fill in all real values — database passwords, API keys, secrets, etc. Never commit this file to Git.

### 3.3 Start infrastructure (database, Redis, etc.)

```bash
sudo docker compose up -d
```

### 3.4 Build and start the app

```bash
sudo docker compose --profile production up -d --build
```

This command:
- Rebuilds the Docker image with your latest code
- Starts all containers in the background
- Keeps your database volumes safe (data is not deleted)

### 3.5 Seed the database (first time only)

```bash
sudo docker compose exec app pnpm seed
```

> Migrations run automatically on startup — no manual step needed.

---

## Part 4 — Configure Nginx

Nginx acts as the front door to your app. It receives all incoming HTTP requests and forwards API traffic to your Docker container.

Edit the default config:

```bash
sudo nano /etc/nginx/sites-available/default
```

Replace the contents with:

```nginx
server {
    listen 80 default_server;
    server_name _;

    location /api/ {
        proxy_pass http://localhost:8080;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

Test the config for errors, then reload:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

> If `nginx -t` shows errors, **do not reload** — fix the config first.

---

## Part 5 — Verify Everything Works

### Check your public IP

```bash
curl ifconfig.me
```

### Check running containers

```bash
sudo docker ps
```

### Check app logs

```bash
# Replace with your actual container name from `docker ps`
sudo docker logs aeoranker-app-1

# Live/streaming logs
sudo docker logs -f aeoranker-app-1
```

### Test your API endpoints

```bash
curl http://YOUR_IP/api/health
curl http://YOUR_IP/api/v1/auth/me
```

---

## Part 6 — Routine: How to Update After a Code Change

Every time you push new code, follow these steps:

```bash
# 1. Connect to EC2
ssh -i "hostname.pem" ubuntu@<your-ec2-dns>

# 2. Go to project folder
cd /var/www/AEORanker

# 3. Pull latest code
git pull origin main

# 4. Rebuild and restart app
sudo docker compose --profile production up -d --build

# 5. Verify containers are running
sudo docker ps

# 6. Watch logs for errors
sudo docker logs -f aeoranker-app-1
```

---

## Architecture Overview

```
Browser / Frontend (Vercel)
           │
           ▼
      /api/* requests
           │
           ▼
     Nginx on EC2 :80
           │
           ▼
  Docker — Express API :8080
           │
      ┌────┴────┐
      ▼         ▼
  Postgres    Redis
```

---

## Quick Reference — Common Commands

| Task | Command |
|------|---------|
| See running containers | `sudo docker ps` |
| View app logs | `sudo docker logs aeoranker-app-1` |
| Live logs | `sudo docker logs -f aeoranker-app-1` |
| Rebuild & restart app | `sudo docker compose --profile production up -d --build` |
| Edit Nginx config | `sudo nano /etc/nginx/sites-available/default` |
| Test Nginx config | `sudo nginx -t` |
| Reload Nginx | `sudo systemctl reload nginx` |
| Get server IP | `curl ifconfig.me` |
| Re-seed database | `sudo docker compose exec app pnpm seed` |

---

## Common Mistakes to Avoid

| ❌ Wrong | ✅ Correct |
|----------|-----------|
| `sudo docker logs app` | `sudo docker logs aeoranker-app-1` (use actual container name from `docker ps`) |
| `curl iconfig.me` | `curl ifconfig.me` (note the correct spelling) |
| Reloading Nginx before testing config | Always run `sudo nginx -t` first |
| Committing `docker.env` to Git | Add it to `.gitignore` — it holds secrets |

---

*Last updated: May 2026*