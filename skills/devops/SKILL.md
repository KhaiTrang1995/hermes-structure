# Skill: DevOps & Infrastructure Automation

This skill empowers the agent to build, containerize, orchestrate, deploy, and monitor scalable applications, microservices, and AI infrastructures. It covers Docker operations, CI/CD pipeline automation, and system diagnostics.

## System Prompt & Agent Instruction

```text
You are an expert DevOps engineer and Site Reliability Engineer (SRE).
When the user requests infrastructure automation, container deployment, or CI/CD debugging, you will activate this skill to:
1. Write, validate, and build Dockerfiles and Docker Compose files.
2. Analyze and fix CI/CD workflows (GitHub Actions, GitLab CI, etc.).
3. Monitor system performance, disk space, and networking metrics.
4. Diagnose and resolve service crashes, port conflicts, and permission issues.
```

---

## Trigger Conditions

Activate this skill when the conversation involves:
- "Build a Docker image for the project"
- "Deploy with Docker Compose"
- "Fix GitHub Actions pipeline failure"
- "Inspect server resource utilization / disk space"
- "Set up a reverse proxy / NGINX config"
- "Debug application deployment crash"

---

## Configuration Templates

### 1. Dockerfile Template (Python/AI App)
```dockerfile
# Use official lightweight Python image
FROM python:3.10-slim-buster

# Prevent Python from writing .pyc files and buffering stdout
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

WORKDIR /app

# Install system dependencies (build-essential, curl, etc.)
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    curl \
    && rm -rf /var/lib/apt/lists/*

# Install python dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy source code
COPY . .

# Expose port and configure healthcheck
EXPOSE 8000
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
  CMD curl -f http://localhost:8000/health || exit 1

# Launch application
CMD ["python", "app.py"]
```

### 2. Docker Compose File Template
```yaml
version: '3.8'

services:
  app:
    build: .
    container_name: hermes-app
    restart: unless-stopped
    ports:
      - "8000:8000"
    environment:
      - ENV=production
      - DB_URL=postgresql://user:pass@db:5432/hermes
    depends_on:
      db:
        condition: service_healthy
    volumes:
      - ./data:/app/data

  db:
    image: postgres:15-alpine
    container_name: hermes-postgres
    restart: unless-stopped
    environment:
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=pass
      - POSTGRES_DB=hermes
    volumes:
      - pgdata:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user -d hermes"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  pgdata:
```

---

## Step-by-Step Execution Protocol

### Step 1: Pre-flight Verification & Auditing
- Diagnose host specifications:
  - **Linux**: Check disk space (`df -h`), memory usage (`free -m`), and active ports (`ss -tulpn` or `netstat -tulpn`).
  - **Windows**: Check port utilization via `Get-NetTCPConnection` or `netstat -ano`.

### Step 2: Building & Deploying Containers
- Compile images securely using build arguments and multi-stage builds.
- Boot stacks in detached mode to ensure they continue running in the background:
  ```bash
  docker compose up -d --build
  ```

### Step 3: Checking Live Health Metrics
- Audit container logs and resource metrics:
  ```bash
  docker ps
  docker compose logs -f --tail=100
  docker stats --no-stream
  ```

---

## Troubleshooting Guide

- **Port Already in Use / Binding Conflicts**:
  1. Detect which PID is locking the target port (e.g. port 8000):
     - **Linux**: `lsof -i :8000` or `netstat -nlp | grep 8000`
     - **Windows (PowerShell)**: `Get-Process -Id (Get-NetTCPConnection -LocalPort 8000).OwningProcess`
  2. Kill the offending process or bind the container to a different external port (e.g. `"8080:8000"`).
- **Out of Disk Space on Docker Host**:
  1. Run Docker system prune to free up orphaned blocks:
     ```bash
     docker system prune -a --volumes -f
     ```
  2. Investigate largest host directories: `du -sh /* 2>/dev/null | sort -hr`.
- **Permission Denied inside Containers (Volume Mounts)**:
  1. Ensure the user inside the container matches the UID/GID of the host mount directory.
  2. Run `chmod -R 755 ./data` or configure target mount ownership appropriately.
