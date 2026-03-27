# VMAN3 Deployment Guide

This directory contains the orchestration and configuration files for deploying the VMAN3 application suite (Frontend, Backend, Database, and Background Workers).

## 📋 Prerequisites
- **Docker** & **Docker Compose**
- **Git** (for code updates)
- **.env** file (use `.env_sample` as a template)
- **config.json** file (use `config.json.sample` as a template)

---

## Deployment

### 1. Clone this repository

```bash
git clone https://github.com/vman-tool/vman-deployment
```

### 2. Get inside vman-deployment directory
```bash
cd vman-deployment
```

### 3. Create environment file
```bash
cp .env_sample .env
```

Update values accordingly:

```

# DEFAULT ACCOUNT INFORMATION
DEFAULT_ACCOUNT_NAME=admin
DEFAULT_ACCOUNT_EMAIL=admin@vman.net
DEFAULT_ACCOUNT_PASSWORD=Welcome2vman$

USE_CELERY=True
# Redis - use container_name as hostname inside Docker network (Do not change these unless you know what you're doing)
REDIS_URL=redis://redis-vman3:6379
REDIS_CELERY_URL=redis://redis-vman3:6379
REDIS_PASSWORD=vman@1029
DB_URL=http://arango-db:8529

# DBS
DB_NAME=vman3
DB_PASSWORD=password
ARANGO_ROOT_USER=root
ARANGO_ROOT_PASSWORD=******
ARANGODB_URL=http://localhost:8529/
DB_ROOT_USER=root



CORS_ALLOWED_ORIGINS=http://localhost,http://localhost,http://localhost:4200,http://127.0.0.1:4200,*

APP_NANE=DescriblyFastAPIApp
DEBUG=True
SECRET_KEY=8deadce9449770680910741063cd0a3fe0acb62a8978661f421bbcbb66dc41f1




# SMTP Local Config
USE_CREDENTIALS=False

# JWT Secret
JWT_SECRET=649fb93ef34e4fdf4187709c84d643dd61ce730d91856418fdcf563f895ea40f
REFRESH_TOKEN_EXPIRE_MINUTES=15
ACCESS_TOKEN_EXPIRE_MINUTES=10
```

### 4. Create config.json file
```bash
cp config.json.sample config.json
```

> NB: Not recommended to change values here unless you know what you're doing.

### 5. Run the application
> Uses pre-built images.

```bash
docker compose up -d
```
---

## 🛠 Infrastructure Components

| Service | Image | Description |
| :--- | :--- | :--- |
| **nginx** | `nginx:alpine` | Reverse proxy handling traffic for all services. |
| **backend** | `backend` | FastAPI application serving the REST API. |
| **celery-worker** | `celery-worker` | Background task processor (CCVA, ODK Fetch). |
| **flower** | `mher/flower:master` | Monitoring UI for Celery tasks (at port `:5555`). |
| **arango-db** | `arangodb:3.11` | Primary NoSQL database. |
| **redis-vman3** | `redis:alpine` | Message broker for Celery and WebSocket pub/sub. |
| **frontend** | `frontend` | Angular application (accessible via Nginx). |

---

## ⚙️ Configuration (.env)

Ensure these variables are set correctly for your environment:

- `USE_CELERY=True`: Enables background processing for CCVA.
- `REDIS_URL`: `redis://redis-vman3:6379` (Internal Docker network).
- `ARANGODB_URL`: `http://arango-db:8529`.

---

## 🔦 Useful Commands

### Check Status
```bash
docker compose ps
```

### View Logs (Real-time)
```bash
docker compose logs -f backend         # Backend logs
docker compose logs -f celery-worker  # Worker logs 
```

### Restart Services (Force Recreate)
```bash
docker compose up -d --force-recreate
```

### Accessing Tools
- **Swagger UI**: `http://localhost:82/vman/api/v1/docs`
- **ArangoDB UI**: `http://localhost:82`
- **Flower UI**: `http://localhost:82`

---
**Note**: For CCVA analysis, memory optimizations ("Fetch-in-Worker") are enabled by default through the `.env` and Celery worker settings.
