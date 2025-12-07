# F1 Store – Dockerized DevOps Demo

This repository is a Docker-based implementation of the **F1 Store** project, originally created for DevOps training and cloud deployment practice.  
It focuses on running the full stack locally using Docker and Docker Compose: two Django backends, PostgreSQL, Redis, and an Nginx-based static frontend.

> Original project: https://github.com/THE-GAME-DEVOPS/f1-store

---

## 1. Architecture Overview

The system consists of five main services:

- **PostgreSQL** – relational database for the RDS backend.
- **Redis** – in-memory data store for caching and fast access.
- **Backend RDS** – Django application connected to PostgreSQL.
- **Backend Redis** – Django application connected to Redis.
- **Frontend** – static HTML application served via Nginx.

High-level data flow:

```text
Browser
   │
   ▼
Nginx (Frontend, :8080)
   │
   ├──> Backend RDS (Django, :8000) ───> PostgreSQL (:5432)
   │
   └──> Backend Redis (Django, :8001) ─> Redis (:6379)
```

You can also see an architectural diagram in `diagram.png`.

---

## 2. Tech Stack

**Languages & Frameworks**

- Python 3.10
- Django 3.2

**Datastores**

- PostgreSQL 16
- Redis 7 (Alpine image)

**Infrastructure & Tooling**

- Docker & Docker Compose v2
- Nginx (Alpine) as static web server for the frontend
- `.env` / `.env.example` configuration pattern

---

## 3. Repository Structure

```text
f1-store-devops/
├── backend_rds/
│   ├── backend_rds/          # Django project (RDS-focused backend)
│   ├── manage.py
│   ├── requirements.txt
│   ├── Dockerfile
│   ├── .env.example          # Template for local environment variables
│   └── ...                   # Django apps, settings, urls, etc.
│
├── backend_redis/
│   ├── backend_redis/        # Django project (Redis-focused backend)
│   ├── manage.py
│   ├── requirements.txt
│   ├── Dockerfile
│   ├── .env.example          # Template for local environment variables
│   └── ...                   # Django apps, settings, urls, etc.
│
├── frontend/
│   ├── index.html            # Static frontend
│   └── Dockerfile            # Nginx-based static site
│
├── docker-compose.yml        # Multi-service definition for local stack
├── diagram.png               # Architecture diagram (from the original project)
└── README.md                 # This file
```

---

## 4. Getting Started (Local Environment)

### 4.1. Prerequisites

Make sure you have the following installed:

- [Git](https://git-scm.com/)
- [Docker](https://docs.docker.com/get-docker/)
- Docker Compose v2 (usually available as `docker compose` command)

You can verify your setup:

```bash
docker --version
docker compose version
```

### 4.2. Clone the repository

Clone your fork (example using HTTPS):

```bash
git clone https://github.com/AndriiDorohov/f1-store-devops-andrii.git
cd f1-store-devops-andrii
```

You can verify the structure with:

```bash
ls
# backend_rds/ backend_redis/ frontend/ docker-compose.yml diagram.png README.md
```

### 4.3. Configure environment variables

This project uses the pattern:

- `*.env.example` – template committed to Git
- `*.env` – real local configuration (not tracked by Git)

Create real `.env` files from templates:

```bash
cp backend_rds/.env.example backend_rds/.env
cp backend_redis/.env.example backend_redis/.env
```

You can then customize them as needed.

**Example: `backend_rds/.env.example`**

```env
DB_NAME=f1store
DB_USER=f1user
DB_PASSWORD=change_me
DB_HOST=postgres
DB_PORT=5432

CORS_ALLOWED_ORIGINS=http://localhost:8080
```

**Example: `backend_redis/.env.example`**

```env
SECRET_KEY=change_me
DEBUG=True
CORS_ALLOWED_ORIGINS=http://localhost:8080

REDIS_HOST=redis
REDIS_PORT=6379
REDIS_DB=0
REDIS_PASSWORD=
```

> Do not commit `.env` files. Git is configured via `.gitignore` to ignore them.

---

## 5. Running the Full Stack with Docker Compose

### 5.1. Build and start all services

From the repository root:

```bash
docker compose up -d --build
```

This will:

- build Docker images for the three application services:
  - `backend_rds`
  - `backend_redis`
  - `frontend`
- start the infrastructure containers:
  - `postgres`
  - `redis`

You can see running containers with:

```bash
docker ps
```

### 5.2. Default ports

After `docker compose up`, services are available on:

- **Frontend (Nginx)**:  
  `http://localhost:8080/`
- **Backend RDS (Django)**:  
  `http://localhost:8000/`
- **Backend Redis (Django)**:  
  `http://localhost:8001/`
- **PostgreSQL**:  
  `localhost:5432`
- **Redis**:  
  `localhost:6379`

---

## 6. Verifying the Setup

### 6.1. Frontend “API Connection Tester”

Open the frontend in your browser:

```text
http://localhost:8080/
```

You should see a simple “API Connection Tester” UI.  
It allows you to test:

- connectivity to `backend_rds` (RDS backend),
- connectivity to `backend_redis` (Redis backend).

When both are working, you will see `200 OK` responses with messages such as:

- `Connection to Backend RDS is successful!`

### 6.2. Backend health endpoints

You can also test the health checks from the command line:

```bash
# Backend RDS health endpoint
curl http://localhost:8000/test_connection/

# Backend Redis health endpoint
curl http://localhost:8001/test_connection/
```

Expected output:

```json
{"message": "Connection to Backend RDS is successful!"}
```

(The same message is reused in both services for simplicity.)

---

## 7. Database Migrations (Backend RDS)

If you see Django warnings about unapplied migrations for `admin`, `auth`, `contenttypes`, or `sessions`, you can run migrations inside the Docker network.

From the project root:

```bash
# Ensure Postgres is running
docker compose up -d postgres

# Apply migrations for backend_rds
docker compose run --rm backend_rds python manage.py migrate
```

After that, you can restart the backend:

```bash
docker compose up -d backend_rds
```

> The same pattern can be used for future Django apps that require migrations.

---

## 8. Docker Services Details

### 8.1. PostgreSQL

Defined in `docker-compose.yml` as:

- **Image**: `postgres:16`
- **Environment**:
  - `POSTGRES_DB=f1store`
  - `POSTGRES_USER=f1user`
  - `POSTGRES_PASSWORD=f1password` (for local development)
- **Volume**:
  - `postgres_data:/var/lib/postgresql/data`

Data is persisted in the `postgres_data` Docker volume.

### 8.2. Redis

Defined as:

- **Image**: `redis:7-alpine`
- **Command**:
  - `redis-server --appendonly yes` (AOF enabled)
- **Volume**:
  - `redis_data:/data`

This gives a simple but robust Redis instance with append-only persistence.

### 8.3. Backend RDS (Django + Postgres)

- Built from `backend_rds/Dockerfile`.
- Uses `backend_rds/.env` for database and CORS configuration.
- Runs development server with:

  ```bash
  python manage.py runserver 0.0.0.0:8000
  ```

- Accessible on `http://localhost:8000` from the host.

### 8.4. Backend Redis (Django + Redis)

- Built from `backend_redis/Dockerfile`.
- Uses `backend_redis/.env` for Redis and CORS configuration.
- Runs development server with:

  ```bash
  python manage.py runserver 0.0.0.0:8000
  ```

- Exposed as `http://localhost:8001` on the host.

### 8.5. Frontend (Nginx)

- Built from `frontend/Dockerfile`.
- Serves `frontend/index.html` using `nginx:alpine`.
- Exposed as `http://localhost:8080`.

---

## 9. Useful Docker Compose Commands

From the repository root:

```bash
# Stop all containers
docker compose down

# Stop and remove containers + volumes (including DB data)
docker compose down -v

# View logs for all services
docker compose logs -f

# View logs for a specific service
docker compose logs -f backend_rds
docker compose logs -f backend_redis
docker compose logs -f frontend
docker compose logs -f postgres
docker compose logs -f redis
```

---

## 10. Future Improvements (DevOps Roadmap)

This repository is primarily focused on **local containerized setup**.  
Some natural next steps for a real-world DevOps portfolio could include:

- **Cloud deployment**:
  - Build and push Docker images to AWS ECR.
  - Run backends on ECS, EC2, or Kubernetes.
  - Serve the frontend via S3 + CloudFront (as in the original assignment).
- **CI/CD pipelines**:
  - GitHub Actions workflows for:
    - building images,
    - running tests and linters,
    - deploying to staging/production environments.
- **Observability**:
  - Structured logging, log aggregation (e.g. CloudWatch, ELK, Loki).
  - Metrics and monitoring (Prometheus, Grafana).
  - Health checks, readiness/liveness probes.
- **Security & configuration**:
  - Use SSM Parameter Store or AWS Secrets Manager instead of plain `.env` in production.
  - Harden Docker images (non-root user, minimal base images).
  - Add HTTPS and secure headers on the frontend.

These enhancements are not implemented yet in this fork, but they are a natural evolution path if you want to present this project as a full DevOps case study.

---

## 11. Credits

- **Original concept and code**: [THE-GAME-DEVOPS / f1-store](https://github.com/THE-GAME-DEVOPS/f1-store)
- **This fork / implementation**: focuses on a clean Docker-based local setup and DevOps-friendly structure.

---

## 12. License & Usage

The original repository does not explicitly declare a license at the time of writing.  
Treat this project as **educational material**, and refer to the upstream repository or contact the original authors if you plan to use the code in a commercial or production context.

