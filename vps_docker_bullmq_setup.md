VPS Docker + BullMQ Email System Setup Guide

---

# 1. VPS Prerequisites

- Ubuntu 20/22 recommended
- Root access or sudo privileges
- Internet access

# 2. Update System

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install curl git -y
```

# 3. Install Docker

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo groupadd docker          # Only if 'docker' group doesn't exist
sudo usermod -aG docker $USER
newgrp docker                 # Apply new group
docker --version
```

# 4. Install Docker Compose

Option 1: Use standalone (older) version
```bash
sudo apt install docker-compose -y
docker-compose --version
```

Option 2: Use Docker Compose plugin (recommended)
```bash
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
docker compose version
```

# 5. Upload Project to VPS

- Clone via git or upload via SCP/FTP
```bash
cd /home/ubuntu
git clone <repo-url> fast-publisher
cd fast-publisher
```
- Ensure `.env` is present

# 6. Configure .env

```env
REDIS_HOST=redis
REDIS_PORT=6379
RESEND_API_KEY=<your_resend_key>
ADMIN_EMAIL=<your_email@domain.com>
```

# 7. Docker Compose File Example

```yaml
version: "3.9"

services:
  redis:
    image: redis:7
    container_name: redis
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data

  app:
    build: .
    container_name: email-app
    command: npx ts-node src/app.ts
    env_file:
      - .env
    ports:
      - "8080:8080"
    depends_on:
      - redis

  worker:
    build: .
    container_name: email-worker
    command: npx ts-node src/workers/email-worker.ts
    env_file:
      - .env
    depends_on:
      - redis

volumes:
  redis-data:
```

# 8. Build and Run Containers

```bash
# Using docker-compose (old)
docker-compose up --build -d

# Using docker compose plugin (new)
docker compose up --build -d
```

# 9. Verify Running Containers

```bash
docker ps
```
- Should see: `redis`, `email-app`, `email-worker`

# 10. View Logs

```bash
docker logs -f email-worker
docker logs -f email-app
```

# 11. Test API

```bash
curl http://<VPS-IP>:8080/send
```
- Queues emails in BullMQ, worker processes them

# 12. Restart Containers

```bash
# Restart single container
docker restart email-app
# Restart everything
docker-compose restart
```

# 13. Monitor Queue

- Use Bull Board UI or programmatically:
```ts
const waiting = await emailQueue.getWaitingCount();
const completed = await emailQueue.getCompletedCount();
```

# 14. Firewall (Optional)

```bash
sudo ufw allow 8080/tcp
sudo ufw enable
```

# Notes
- Run everything inside Docker for proper `REDIS_HOST=redis` resolution
- Respect Resend API rate limits (2 requests/sec for free tier)
- Use `.env` inside Docker containers
