# F1 Store DevOps – Multi-Container AWS Deployment

A production-style DevOps project that demonstrates how to take a small multi-service application from local Docker Compose setup to a full AWS deployment with CI/CD.

The application simulates a simple “F1 Store” platform with:
- Two Django-based backends:
  - `backend_rds` – uses PostgreSQL (RDS-style) for persistence.
  - `backend_redis` – uses Redis for fast in-memory access.
- A static frontend served via Nginx that provides an **API Connection Tester** UI to verify connectivity to both backends.

This repository focuses on **infrastructure, automation, and deployment patterns**, not on complex business logic.

---

## Architecture Overview

The project follows a classic multi-container architecture:

- **Frontend (Nginx)**
  - Serves a static HTML/CSS/JS application.
  - Exposes the API Connection Tester UI on port `8080` (local) or `:8080` on EC2.
- **Backend RDS (Django)**
  - Connects to PostgreSQL.
  - Exposes a `/test_connection/` endpoint to validate DB connectivity.
- **Backend Redis (Django)**
  - Connects to Redis.
  - Exposes a `/test_connection/` endpoint to validate Redis connectivity.
- **PostgreSQL**
  - Stores relational data for the RDS backend.
- **Redis**
  - Used as an in-memory data store/cache for the Redis backend.

### AWS Architecture

On AWS, the stack looks like this:

- **ECR (Elastic Container Registry)**
  - Stores container images:
    - `f1-backend-rds`
    - `f1-backend-redis`
    - `f1-frontend`
- **EC2**
  - Hosts the runtime Docker Compose stack:
    - pulls images from ECR;
    - runs all services (`postgres`, `redis`, `backend_rds`, `backend_redis`, `frontend`).
- **S3 + CloudFront**
  - S3 bucket (static website hosting) used for storing a built frontend variant.
  - CloudFront distribution in front of the S3 bucket for global caching and delivery.

The diagram below is captured in the repository as `diagram.png`:

- It shows local Docker Compose topology.
- It shows AWS components: ECR, EC2, S3, CloudFront, GitHub Actions.

> See `diagram.png` in the root of the repository for a visual architecture overview.

---

## Tech Stack

**Languages & Frameworks**

- Python 3
- Django 3.2
- JavaScript / HTML / CSS (static frontend)

**Databases & Caches**

- PostgreSQL 16
- Redis 7 (Alpine-based)

**Containers & Orchestration**

- Docker
- Docker Compose

**Web & Networking**

- Nginx (for static frontend)
- HTTP-based health and connection test endpoints

**Cloud & DevOps**

- AWS EC2 (compute)
- AWS ECR (container registry)
- AWS S3 (static website hosting)
- AWS CloudFront (CDN)
- AWS IAM (access control)
- GitHub Actions (CI/CD)
- Git + GitHub (version control)

---

## Local Development

### Prerequisites

- Docker
- Docker Compose plugin (`docker compose` command)
- Git

### 1. Clone the repository

```bash
git clone https://github.com/AndriiDorohov/f1-store-devops-andrii.git
cd f1-store-devops-andrii
```

### 2. Environment configuration

Copy example environment files and adjust values if needed:

```bash
cp backend_rds/.env.example backend_rds/.env
cp backend_redis/.env.example backend_redis/.env
```

Default `.env.example` values are configured for local Docker Compose usage:

- `backend_rds/.env.example`:
  - PostgreSQL connection (`DB_NAME`, `DB_USER`, `DB_PASSWORD`, `DB_HOST`, `DB_PORT`)
  - `CORS_ALLOWED_ORIGINS=http://localhost:8080`
- `backend_redis/.env.example`:
  - Redis connection (`REDIS_HOST`, `REDIS_PORT`, `REDIS_DB`)
  - `CORS_ALLOWED_ORIGINS=http://localhost:8080`

Make sure `DB_HOST=postgres` and `REDIS_HOST=redis` – these match the Docker Compose service names.

### 3. Start the stack locally

Build and start all services:

```bash
docker compose up -d --build
```

This will start:

- `f1_postgres` on `5432`
- `f1_redis` on `6379`
- `f1_backend_rds` on `8000`
- `f1_backend_redis` on `8001`
- `f1_frontend` on `8080`

Apply Django migrations for the RDS backend:

```bash
docker compose run --rm backend_rds python manage.py migrate
```

### 4. Verify services

- Frontend (API Tester):  
  `http://localhost:8080/`

- Backend RDS connectivity endpoint:  
  `http://localhost:8000/test_connection/`

  Expected response:

  ```json
  { "message": "Connection to Backend RDS is successful!" }
  ```

- Backend Redis connectivity endpoint:  
  `http://localhost:8001/test_connection/`

  Expected response:

  ```json
  { "message": "Connection to Backend Redis is successful!" }
  ```

To follow logs:

```bash
docker compose logs -f
# or specific services:
docker compose logs -f backend_rds backend_redis frontend
```

---

## AWS Deployment

### 1. ECR Repositories

Three ECR repositories are used:

- `f1-backend-rds`
- `f1-backend-redis`
- `f1-frontend`

Each repository stores a `latest` tag for the corresponding Docker image.

### 2. EC2 Instance

An Ubuntu-based EC2 instance is used to run the Docker Compose stack:

- Instance type: `t3.micro`
- Region: `eu-north-1` (Stockholm)
- Security group:
  - `22/tcp` – SSH (opened for GitHub Actions and manual access)
  - `80/tcp` – HTTP (optional, depending on usage)
  - `8000/tcp` – RDS backend
  - `8001/tcp` – Redis backend
  - `8080/tcp` – frontend

On the EC2 instance:

```bash
# Clone repository on EC2
git clone https://github.com/AndriiDorohov/f1-store-devops-andrii.git
cd f1-store-devops-andrii

# Log in to ECR (example)
aws ecr get-login-password --region eu-north-1   | docker login     --username AWS     --password-stdin 071338670555.dkr.ecr.eu-north-1.amazonaws.com

# Pull and start the stack
docker compose pull
docker compose up -d
```

The frontend becomes available at:

```text
http://<EC2_PUBLIC_IP>:8080/
```

For this project, an example EC2 public IP was:

```text
http://13.51.193.36:8080/
```

(Your IP address will differ.)

### 3. S3 + CloudFront

The static frontend can also be deployed to S3 and served via CloudFront:

- S3 bucket (example):  
  `f1-store-frontend-andrii-eu-north-1`
- Static website endpoint:  
  `http://f1-store-frontend-andrii-eu-north-1.s3-website.eu-north-1.amazonaws.com`
- CloudFront distribution:
  - Domain: something like `d1ezffl7bmi5kv.cloudfront.net`
  - Origin: S3 website endpoint above

This demonstrates:

- basic static site hosting on S3;
- using CloudFront as a global CDN in front of the bucket.

---

## CI/CD – GitHub Actions

The project includes a GitHub Actions workflow:

- Location: `.github/workflows/ecr-build-and-push.yml`
- Triggers:
  - on push to `main`
  - manual `workflow_dispatch`

### Job 1 – Build and push to ECR

Steps:

1. Checkout the repository.
2. Configure AWS credentials using repository secrets:
   - `AWS_ACCESS_KEY_ID`
   - `AWS_SECRET_ACCESS_KEY`
   - `AWS_REGION`
   - `AWS_ACCOUNT_ID`
   - `ECR_REGISTRY`
3. Login to ECR.
4. Build Docker images:
   - `backend_rds`
   - `backend_redis`
   - `frontend`
5. Tag and push images to ECR:
   - `$ECR_REGISTRY/f1-backend-rds:latest`
   - `$ECR_REGISTRY/f1-backend-redis:latest`
   - `$ECR_REGISTRY/f1-frontend:latest`

### Job 2 – Deploy to EC2

Steps:

1. Use `appleboy/ssh-action` to connect to the EC2 instance via SSH using secrets:
   - `EC2_HOST`
   - `EC2_USER`
   - `EC2_SSH_KEY`
2. On EC2, run:
   - `git pull origin main`
   - `aws ecr get-login-password | docker login ...`
   - `docker compose pull`
   - `docker compose up -d`
   - `docker image prune -f`

This delivers a full CI/CD pipeline:

- Any push to `main` triggers:
  - new image build and push to ECR;
  - remote update of running containers on EC2 via Docker Compose.

---

## Screenshots (Suggested)

For portfolio and documentation, the following screenshots are recommended:

1. **API Connection Tester UI**
   - Browser window showing `http://localhost:8080/` or `http://<EC2_PUBLIC_IP>:8080/`.
   - Both RDS and Redis tests returning `200 OK`.

2. **AWS S3 Bucket**
   - S3 console page showing the bucket:
     - e.g. `f1-store-frontend-andrii-eu-north-1`
   - Static website hosting settings.

3. **CloudFront Distribution**
   - CloudFront console showing the distribution with:
     - Origin pointing to the S3 website endpoint.
     - Distribution domain name (e.g. `d1ezffl7bmi5kv.cloudfront.net`).

4. **EC2 + Docker**
   - EC2 console with the running instance `f1-store-devops-andrii-ec2`.
   - Terminal screenshot with:
     - `docker ps` output showing:
       - `f1_frontend`
       - `f1_backend_rds`
       - `f1_backend_redis`
       - `f1_postgres`
       - `f1_redis`

5. **GitHub Actions**
   - Actions tab with a successful run of:
     - `Build, push Docker images to ECR and deploy to EC2`
   - Logs showing:
     - build and push steps,
     - deploy steps (SSH → `docker compose pull` → `docker compose up -d`).

You can store screenshots under a `screenshots/` folder and reference them in this README for a more visual portfolio.

---

## What I Learned / Key DevOps Highlights

This project captures an end-to-end DevOps flow:

1. **Containerization of multi-service apps**
   - Learned to structure Dockerfiles for multiple services (two Django backends + Nginx frontend).
   - Practiced isolating configuration and secrets via `.env` and `.env.example`.

2. **Local orchestration with Docker Compose**
   - Designed a clean `docker-compose.yml` with service dependencies.
   - Connected services by name (`postgres`, `redis`) and validated health with `/test_connection/` endpoints.

3. **Cloud deployment on AWS**
   - Gained hands-on experience with:
     - EC2 for compute,
     - ECR for container registry,
     - S3 + CloudFront for static hosting.
   - Worked with security groups (opening ports only as needed).

4. **CI/CD with GitHub Actions**
   - Implemented a two-stage pipeline:
     - build & publish images to ECR,
     - deploy updated containers on EC2.
   - Used GitHub Secrets and IAM to manage credentials securely in workflows.

5. **Production-like mindset**
   - Treated a simple test-application as a real-world micro-stack:
     - reproducible local setup,
     - cloud deployment,
     - automated updates,
     - clear documentation and architecture diagrams.

---

## How to Reuse This Project

You can reuse this repository as a template for:

- small microservice demos;
- DevOps learning projects;
- interview or portfolio showcases for:
  - Docker,
  - AWS (EC2, ECR, S3, CloudFront),
  - GitHub Actions.

Clone it, adapt the services, and plug in your own application logic, while keeping the deployment and CI/CD patterns.

---

## License

This project is intended as an educational and portfolio-oriented example.  
Adapt and extend it for your own learning and demonstrations as needed.
# F1 Store DevOps – Multi-Container AWS Deployment

A production-style DevOps project that demonstrates how to take a small multi-service application from local Docker Compose setup to a full AWS deployment with CI/CD.

The application simulates a simple “F1 Store” platform with:
- Two Django-based backends:
  - `backend_rds` – uses PostgreSQL (RDS-style) for persistence.
  - `backend_redis` – uses Redis for fast in-memory access.
- A static frontend served via Nginx that provides an **API Connection Tester** UI to verify connectivity to both backends.

This repository focuses on **infrastructure, automation, and deployment patterns**, not on complex business logic.

---

## Architecture Overview

The project follows a classic multi-container architecture:

- **Frontend (Nginx)**
  - Serves a static HTML/CSS/JS application.
  - Exposes the API Connection Tester UI on port `8080` (local) or `:8080` on EC2.
- **Backend RDS (Django)**
  - Connects to PostgreSQL.
  - Exposes a `/test_connection/` endpoint to validate DB connectivity.
- **Backend Redis (Django)**
  - Connects to Redis.
  - Exposes a `/test_connection/` endpoint to validate Redis connectivity.
- **PostgreSQL**
  - Stores relational data for the RDS backend.
- **Redis**
  - Used as an in-memory data store/cache for the Redis backend.

### AWS Architecture

On AWS, the stack looks like this:

- **ECR (Elastic Container Registry)**
  - Stores container images:
    - `f1-backend-rds`
    - `f1-backend-redis`
    - `f1-frontend`
- **EC2**
  - Hosts the runtime Docker Compose stack:
    - pulls images from ECR;
    - runs all services (`postgres`, `redis`, `backend_rds`, `backend_redis`, `frontend`).
- **S3 + CloudFront**
  - S3 bucket (static website hosting) used for storing a built frontend variant.
  - CloudFront distribution in front of the S3 bucket for global caching and delivery.

The diagram below is captured in the repository as `diagram.png`:

- It shows local Docker Compose topology.
- It shows AWS components: ECR, EC2, S3, CloudFront, GitHub Actions.

> See `diagram.png` in the root of the repository for a visual architecture overview.

---

## Tech Stack

**Languages & Frameworks**

- Python 3
- Django 3.2
- JavaScript / HTML / CSS (static frontend)

**Databases & Caches**

- PostgreSQL 16
- Redis 7 (Alpine-based)

**Containers & Orchestration**

- Docker
- Docker Compose

**Web & Networking**

- Nginx (for static frontend)
- HTTP-based health and connection test endpoints

**Cloud & DevOps**

- AWS EC2 (compute)
- AWS ECR (container registry)
- AWS S3 (static website hosting)
- AWS CloudFront (CDN)
- AWS IAM (access control)
- GitHub Actions (CI/CD)
- Git + GitHub (version control)

---

## Local Development

### Prerequisites

- Docker
- Docker Compose plugin (`docker compose` command)
- Git

### 1. Clone the repository

```bash
git clone https://github.com/AndriiDorohov/f1-store-devops-andrii.git
cd f1-store-devops-andrii
```

### 2. Environment configuration

Copy example environment files and adjust values if needed:

```bash
cp backend_rds/.env.example backend_rds/.env
cp backend_redis/.env.example backend_redis/.env
```

Default `.env.example` values are configured for local Docker Compose usage:

- `backend_rds/.env.example`:
  - PostgreSQL connection (`DB_NAME`, `DB_USER`, `DB_PASSWORD`, `DB_HOST`, `DB_PORT`)
  - `CORS_ALLOWED_ORIGINS=http://localhost:8080`
- `backend_redis/.env.example`:
  - Redis connection (`REDIS_HOST`, `REDIS_PORT`, `REDIS_DB`)
  - `CORS_ALLOWED_ORIGINS=http://localhost:8080`

Make sure `DB_HOST=postgres` and `REDIS_HOST=redis` – these match the Docker Compose service names.

### 3. Start the stack locally

Build and start all services:

```bash
docker compose up -d --build
```

This will start:

- `f1_postgres` on `5432`
- `f1_redis` on `6379`
- `f1_backend_rds` on `8000`
- `f1_backend_redis` on `8001`
- `f1_frontend` on `8080`

Apply Django migrations for the RDS backend:

```bash
docker compose run --rm backend_rds python manage.py migrate
```

### 4. Verify services

- Frontend (API Tester):  
  `http://localhost:8080/`

- Backend RDS connectivity endpoint:  
  `http://localhost:8000/test_connection/`

  Expected response:

  ```json
  { "message": "Connection to Backend RDS is successful!" }
  ```

- Backend Redis connectivity endpoint:  
  `http://localhost:8001/test_connection/`

  Expected response:

  ```json
  { "message": "Connection to Backend Redis is successful!" }
  ```

To follow logs:

```bash
docker compose logs -f
# or specific services:
docker compose logs -f backend_rds backend_redis frontend
```

---

## AWS Deployment

### 1. ECR Repositories

Three ECR repositories are used:

- `f1-backend-rds`
- `f1-backend-redis`
- `f1-frontend`

Each repository stores a `latest` tag for the corresponding Docker image.

### 2. EC2 Instance

An Ubuntu-based EC2 instance is used to run the Docker Compose stack:

- Instance type: `t3.micro`
- Region: `eu-north-1` (Stockholm)
- Security group:
  - `22/tcp` – SSH (opened for GitHub Actions and manual access)
  - `80/tcp` – HTTP (optional, depending on usage)
  - `8000/tcp` – RDS backend
  - `8001/tcp` – Redis backend
  - `8080/tcp` – frontend

On the EC2 instance:

```bash
# Clone repository on EC2
git clone https://github.com/AndriiDorohov/f1-store-devops-andrii.git
cd f1-store-devops-andrii

# Log in to ECR (example)
aws ecr get-login-password --region eu-north-1   | docker login     --username AWS     --password-stdin 071338670555.dkr.ecr.eu-north-1.amazonaws.com

# Pull and start the stack
docker compose pull
docker compose up -d
```

The frontend becomes available at:

```text
http://<EC2_PUBLIC_IP>:8080/
```

For this project, an example EC2 public IP was:

```text
http://13.51.193.36:8080/
```

(Your IP address will differ.)

### 3. S3 + CloudFront

The static frontend can also be deployed to S3 and served via CloudFront:

- S3 bucket (example):  
  `f1-store-frontend-andrii-eu-north-1`
- Static website endpoint:  
  `http://f1-store-frontend-andrii-eu-north-1.s3-website.eu-north-1.amazonaws.com`
- CloudFront distribution:
  - Domain: something like `d1ezffl7bmi5kv.cloudfront.net`
  - Origin: S3 website endpoint above

This demonstrates:

- basic static site hosting on S3;
- using CloudFront as a global CDN in front of the bucket.

---

## CI/CD – GitHub Actions

The project includes a GitHub Actions workflow:

- Location: `.github/workflows/ecr-build-and-push.yml`
- Triggers:
  - on push to `main`
  - manual `workflow_dispatch`

### Job 1 – Build and push to ECR

Steps:

1. Checkout the repository.
2. Configure AWS credentials using repository secrets:
   - `AWS_ACCESS_KEY_ID`
   - `AWS_SECRET_ACCESS_KEY`
   - `AWS_REGION`
   - `AWS_ACCOUNT_ID`
   - `ECR_REGISTRY`
3. Login to ECR.
4. Build Docker images:
   - `backend_rds`
   - `backend_redis`
   - `frontend`
5. Tag and push images to ECR:
   - `$ECR_REGISTRY/f1-backend-rds:latest`
   - `$ECR_REGISTRY/f1-backend-redis:latest`
   - `$ECR_REGISTRY/f1-frontend:latest`

### Job 2 – Deploy to EC2

Steps:

1. Use `appleboy/ssh-action` to connect to the EC2 instance via SSH using secrets:
   - `EC2_HOST`
   - `EC2_USER`
   - `EC2_SSH_KEY`
2. On EC2, run:
   - `git pull origin main`
   - `aws ecr get-login-password | docker login ...`
   - `docker compose pull`
   - `docker compose up -d`
   - `docker image prune -f`

This delivers a full CI/CD pipeline:

- Any push to `main` triggers:
  - new image build and push to ECR;
  - remote update of running containers on EC2 via Docker Compose.

---

## Screenshots (Suggested)

For portfolio and documentation, the following screenshots are recommended:

1. **API Connection Tester UI**
   - Browser window showing `http://localhost:8080/` or `http://<EC2_PUBLIC_IP>:8080/`.
   - Both RDS and Redis tests returning `200 OK`.

2. **AWS S3 Bucket**
   - S3 console page showing the bucket:
     - e.g. `f1-store-frontend-andrii-eu-north-1`
   - Static website hosting settings.

3. **CloudFront Distribution**
   - CloudFront console showing the distribution with:
     - Origin pointing to the S3 website endpoint.
     - Distribution domain name (e.g. `d1ezffl7bmi5kv.cloudfront.net`).

4. **EC2 + Docker**
   - EC2 console with the running instance `f1-store-devops-andrii-ec2`.
   - Terminal screenshot with:
     - `docker ps` output showing:
       - `f1_frontend`
       - `f1_backend_rds`
       - `f1_backend_redis`
       - `f1_postgres`
       - `f1_redis`

5. **GitHub Actions**
   - Actions tab with a successful run of:
     - `Build, push Docker images to ECR and deploy to EC2`
   - Logs showing:
     - build and push steps,
     - deploy steps (SSH → `docker compose pull` → `docker compose up -d`).

You can store screenshots under a `screenshots/` folder and reference them in this README for a more visual portfolio.

---

## What I Learned / Key DevOps Highlights

This project captures an end-to-end DevOps flow:

1. **Containerization of multi-service apps**
   - Learned to structure Dockerfiles for multiple services (two Django backends + Nginx frontend).
   - Practiced isolating configuration and secrets via `.env` and `.env.example`.

2. **Local orchestration with Docker Compose**
   - Designed a clean `docker-compose.yml` with service dependencies.
   - Connected services by name (`postgres`, `redis`) and validated health with `/test_connection/` endpoints.

3. **Cloud deployment on AWS**
   - Gained hands-on experience with:
     - EC2 for compute,
     - ECR for container registry,
     - S3 + CloudFront for static hosting.
   - Worked with security groups (opening ports only as needed).

4. **CI/CD with GitHub Actions**
   - Implemented a two-stage pipeline:
     - build & publish images to ECR,
     - deploy updated containers on EC2.
   - Used GitHub Secrets and IAM to manage credentials securely in workflows.

5. **Production-like mindset**
   - Treated a simple test-application as a real-world micro-stack:
     - reproducible local setup,
     - cloud deployment,
     - automated updates,
     - clear documentation and architecture diagrams.

---

## How to Reuse This Project

You can reuse this repository as a template for:

- small microservice demos;
- DevOps learning projects;
- interview or portfolio showcases for:
  - Docker,
  - AWS (EC2, ECR, S3, CloudFront),
  - GitHub Actions.

Clone it, adapt the services, and plug in your own application logic, while keeping the deployment and CI/CD patterns.

---

## License

This project is intended as an educational and portfolio-oriented example.  
Adapt and extend it for your own learning and demonstrations as needed.
