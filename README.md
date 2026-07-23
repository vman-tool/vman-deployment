# VMAN3 Deployment Guide

This directory contains the orchestration and configuration files for deploying the VMAN3 application suite (Frontend, Backend, Database, and Background Workers).

## Prerequisites

- **Docker** & **Docker Compose**
- **.env** file (use `.env_sample` as a template)
- **config.json** file (use `config.json.sample` as a template)
- **.htpasswd** file (for Flower monitoring UI basic auth — see step 5)

---

## Deployment

### 1. Clone this repository

```bash
git clone https://github.com/vman-tool/vman-deployment
cd vman-deployment
```

### 2. Create environment file

```bash
cp .env_sample .env
```

Edit `.env` and set values for your environment. Key variables:

| Variable | Description |
| :--- | :--- |
| `DEFAULT_ACCOUNT_NAME` | Name of the default admin account created on first boot |
| `DEFAULT_ACCOUNT_EMAIL` | Email of the default admin account |
| `DEFAULT_ACCOUNT_PASSWORD` | Password of the default admin account |
| `ARANGO_ROOT_PASSWORD` | ArangoDB root password — must match what is stored in the database volume |
| `REDIS_PASSWORD` | Redis password — must match the value in `redis.command` in `docker-compose.yml` |
| `SECRET_KEY` | FastAPI secret key — generate a unique value per deployment |
| `JWT_SECRET` | JWT signing secret — generate a unique value per deployment |
| `CORS_ALLOWED_ORIGINS` | Comma-separated list of allowed frontend origins |

> **Do not change** `DB_URL`, `REDIS_URL`, or `REDIS_CELERY_URL` — these use Docker service names
> that are resolved internally and must match the service names in `docker-compose.yml`.

### 3. Create config.json

```bash
cp config.json.sample config.json
```

Update the API base URL inside `config.json` to point to your server's IP address or domain name.

### 4. Create .htpasswd for Flower auth

The Flower monitoring UI is protected by HTTP basic auth via nginx. The `.htpasswd` file is required — nginx will fail to start without it.

If `htpasswd` is installed locally:
```bash
htpasswd -c .htpasswd admin
```

If not, use Docker:
```bash
docker run --rm httpd:alpine htpasswd -nb admin yourpassword > .htpasswd
```

### 5. Start the application

```bash
docker compose up -d
```

Services start in dependency order. ArangoDB runs a health check before the backend starts, so the first boot may take 30–60 seconds before everything is ready.

### 6. Optional — Run documentation locally

```bash
git clone https://github.com/vman-tool/vman3-documentation
```

- In `nginx.conf`, uncomment the `docs_upstream` block and the `location /docs` block.
- In `docker-compose.yml`, uncomment the `docs` service.

Then run:
```bash
docker compose up -d
```

---

## Infrastructure Components

| Service | Image | Description |
| :--- | :--- | :--- |
| **nginx** | `nginx:alpine` | Reverse proxy routing traffic to all services |
| **frontend** | `ilyatuu/vman3_frontend` | Angular application |
| **backend** | `ilyatuu/vman3_backend:latest` | FastAPI REST API and WebSocket server |
| **celery-worker** | `ilyatuu/vman3_backend:latest` | Background task processor (CCVA, ODK sync) |
| **flower** | `mher/flower:master` | Celery task monitoring UI (at `/flower`, auth protected) |
| **arangodb** | `arangodb/arangodb:3.12.2` | Primary NoSQL database |
| **redis** | `redis:latest` | Message broker for Celery and WebSocket pub/sub |

### Startup order

```
arangodb (healthy) → backend → nginx
                   → celery-worker → flower → nginx
redis → celery-worker
```

---

## Ports and Reverse Proxy

### Docker nginx port

The nginx service is mapped to port **82** by default:

```yaml
ports:
  - "82:80"
```

This matches the production setup where an external reverse proxy (handling SSL/HTTPS) connects to port 82 on the host. The internal ports for backend (8080), ArangoDB (8529), and Redis (6379) are **not exposed** — they are only reachable within the Docker network.

For **local testing** without a reverse proxy, change the nginx port to `"80:80"` and optionally uncomment the backend/database ports for debugging.

### External reverse proxy (SSL + WebSocket)

Production runs an external nginx that terminates SSL and proxies to Docker nginx on port 82. WebSocket support (`wss://`) is required for the CCVA progress tracker — without it the progress bar stalls and the connection closes immediately.

The full external nginx configuration for `vman3.vatools.net`:

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name vman3.vatools.net www.vman3.vatools.net;
    client_max_body_size 100M;

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name vman3.vatools.net;
    client_max_body_size 100M;

    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;

    location / {
        proxy_pass http://localhost:82;
        proxy_redirect off;

        # Required for WebSocket (wss://) — CCVA progress tracker will not work without these
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_buffer_size 128k;
        proxy_buffers 8 128k;
        proxy_busy_buffers_size 256k;

        # CORS Headers
        add_header Access-Control-Allow-Origin *;
        add_header Access-Control-Allow-Methods 'GET, POST, PUT, PATCH, DELETE, OPTIONS';
        add_header Access-Control-Allow-Headers 'Origin, Content-Type, Accept, Authorization';
    }

    ssl_certificate /etc/letsencrypt/live/vman3.vatools.net/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/vman3.vatools.net/privkey.pem;
}
```

**The three WebSocket lines are mandatory:**

| Header | Purpose |
| :--- | :--- |
| `proxy_http_version 1.1` | WebSocket requires HTTP/1.1; the default is 1.0 which does not support upgrades |
| `proxy_set_header Upgrade $http_upgrade` | Forwards the `Upgrade: websocket` header from the browser to Docker nginx |
| `proxy_set_header Connection "upgrade"` | Signals the upgrade to the next hop; without it nginx replaces this with `Connection: close` |

After editing the external nginx config, reload without downtime:

```bash
sudo nginx -t && sudo nginx -s reload
```

---

## Data Directories

Uploads, CCVA output files, and the app settings file are stored as **bind mounts** — real folders that sit alongside the compose file on the host:

```
uploads/          ← user-uploaded files (images, documents)
ccva_files/       ← CCVA analysis output files
settings.json     ← app runtime settings
```

Docker creates these automatically on first boot. They are:
- **Visible and easy to back up** — just copy the folder
- **Persistent** across container restarts and image updates
- **Shared** between the backend and celery-worker (both read/write uploads)

Only the ArangoDB database is stored as a Docker named volume (`vman3db`) since it requires Docker to manage file permissions correctly.

> **Important:** If you see 404 errors for uploaded images (e.g. login screen background),
> the file is missing from the `./uploads/` folder. This happens when:
> - The file was uploaded on a different instance and never copied across
> - The uploads folder was cleared or the instance was rebuilt from scratch
>
> To fix: re-upload the image through the admin panel, or copy the file from the source instance's `./uploads/` folder.

---

## Running Multiple Instances

The compose file is designed to be portable. Each deployment folder is a fully isolated instance.

### Files to copy for a new instance

```
docker-compose.yml
.env
config.json
nginx.conf
.htpasswd
```

The `uploads/`, `ccva_files/`, and `settings.json` are created fresh on first boot. Copy them from an existing instance if you want to carry data across.

### Steps

```bash
cp -r /path/to/vman3_et /path/to/new_instance
cd /path/to/new_instance
```

Edit `.env` as needed (change `ARANGO_ROOT_PASSWORD`, `SECRET_KEY`, `JWT_SECRET`).

If the first instance is **shut down before starting the second**, no other changes are needed:

```bash
# Shut down current instance
docker compose down

# Start new instance
cd /path/to/new_instance
docker compose up -d
```

If both instances need to run **simultaneously**, change the nginx port in `docker-compose.yml` to avoid conflicts (the internal ports are not exposed, so only nginx needs changing):

```yaml
ports:
  - "83:80"   # or any other free port
```

### What keeps instances isolated

| Resource | How it's isolated |
| :--- | :--- |
| Container names | Auto-prefixed: `{foldername}-backend-1`, `{foldername}-redis-1`, etc. |
| Network | Auto-created: `{foldername}_default` — services in different instances cannot reach each other |
| Database volume | Auto-prefixed: `{foldername}_vman3db` — each instance has its own database |
| Uploads / files | Stored in `./uploads/`, `./ccva_files/` — isolated by folder location |

---

## Configuration Reference (.env)

```env
# Default admin account created on first boot
DEFAULT_ACCOUNT_NAME=admin
DEFAULT_ACCOUNT_EMAIL=admin@vman.net
DEFAULT_ACCOUNT_PASSWORD=Welcome2vman$

# Celery
USE_CELERY=True

# Redis — uses Docker service name, do not change
REDIS_URL=redis://redis:6379
REDIS_CELERY_URL=redis://redis:6379
REDIS_PASSWORD=vman@1029

# ArangoDB — uses Docker service name, do not change
DB_URL=http://arangodb:8529
DB_NAME=vman3
ARANGO_ROOT_USER=root
ARANGO_ROOT_PASSWORD=******   # must match the password stored in the database volume
DB_ROOT_USER=root

# Application
APP_NANE=VMan3
DEBUG=True
SECRET_KEY=<generate a unique secret>
CORS_ALLOWED_ORIGINS=http://localhost,http://localhost:4200,*

# JWT
JWT_SECRET=<generate a unique secret>
REFRESH_TOKEN_EXPIRE_MINUTES=15
ACCESS_TOKEN_EXPIRE_MINUTES=10

# SMTP
USE_CREDENTIALS=False
```

> **ArangoDB password note:** `ARANGO_ROOT_PASSWORD` is only used by ArangoDB to set the root
> password on **first initialization** (when the data volume is empty). On subsequent starts,
> the password is read from the volume — the env var is ignored. If you change the password
> after first boot, update it in the database directly:
> ```bash
> docker exec -it <arangodb-container> arangosh \
>   --server.endpoint tcp://127.0.0.1:8529 \
>   --server.username root \
>   --server.password 'current_password' \
>   --javascript.execute-string "require('@arangodb/users').update('root', 'new_password');"
> ```
> Then update `ARANGO_ROOT_PASSWORD` in `.env` to match.

---

## Useful Commands

### Status

```bash
docker compose ps
```

### Logs

```bash
docker compose logs -f backend        # Backend API logs
docker compose logs -f celery-worker  # Worker logs
docker compose logs -f arangodb       # Database logs
```

### Restart all services

```bash
docker compose up -d --force-recreate
```

### Stop and remove containers (data volumes are preserved)

```bash
docker compose down
```

### Stop and remove everything including data volumes

```bash
docker compose down -v
```

---

## Accessing Services

| Tool | URL |
| :--- | :--- |
| Application | `http://localhost` |
| Swagger UI | `http://localhost/vman/api/v1/docs` |
| ArangoDB UI | `http://localhost/_db` |
| Flower UI | `http://localhost/flower` (basic auth required) |
