# n8n High-Performance Installation Guide

Step-by-step commands to set up n8n with performance optimizations. Run each step in order.

---

## Step 1: Create Project Directory

```bash
mkdir -p n8n-compose && cd n8n-compose
```

---

## Step 2: Create Directory Structure

```bash
mkdir -p nginx prometheus grafana/dashboards local-files
```

---

## Step 3: Generate Encryption Key

```bash
# Generate and save this key - you'll need it in Step 4
openssl rand -base64 24
```

**Save the output** - you'll add it to the `.env` file in the next step.

---

## Step 4: Create Environment File

```bash
cat > .env << 'EOF'
# ===========================================
# n8n Production Configuration
# ===========================================

# n8n Version
N8N_VERSION=2.4.3

# ===========================================
# PostgreSQL Database Configuration
# ===========================================
POSTGRES_USER=n8n
POSTGRES_PASSWORD=CHANGE_ME_TO_SECURE_PASSWORD
POSTGRES_DB=n8n

# ===========================================
# n8n General Configuration
# ===========================================
N8N_HOST=localhost:5678
N8N_PORT=5678
N8N_PROTOCOL=http
WEBHOOK_URL=http://localhost:5678/
N8N_EDITOR_BASE_URL=http://localhost:5678
N8N_PUSH_BACKEND=websocket
N8N_SECURE_COOKIE=false
GENERIC_TIMEZONE=America/Los_Angeles

# IMPORTANT: Replace with your key from Step 3
N8N_ENCRYPTION_KEY=PASTE_YOUR_KEY_FROM_STEP_3_HERE

# ===========================================
# Queue Mode & Worker Configuration
# ===========================================
WORKER_REPLICAS=5
WORKER_CONCURRENCY=20
WEBHOOK_REPLICAS=3
WORKER_TIMEOUT=60000

# ===========================================
# Execution Data Management
# ===========================================
EXECUTIONS_DATA_MAX_AGE=336

# ===========================================
# Logging Configuration
# ===========================================
N8N_LOG_LEVEL=info
N8N_LOG_OUTPUT=console
EOF
```

**Important:** Edit the file to:
1. Change `POSTGRES_PASSWORD` to a secure password
2. Replace `N8N_ENCRYPTION_KEY` with the key from Step 3

```bash
# Edit the file
nano .env
```

---

## Step 5: Create Nginx Configuration

```bash
cat > nginx/nginx.conf << 'EOF'
worker_processes 4;
error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections 4096;
    use epoll;
    multi_accept on;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    map $http_upgrade $connection_upgrade {
        default upgrade;
        '' close;
    }

    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for" '
                    'rt=$request_time uct="$upstream_connect_time" '
                    'uht="$upstream_header_time" urt="$upstream_response_time"';

    access_log /var/log/nginx/access.log main;

    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    keepalive_requests 10000;

    upstream n8n_webhooks {
        least_conn;
        keepalive 32;
        server n8n-webhook-1:5678 max_fails=3 fail_timeout=30s;
        server n8n-webhook-2:5678 max_fails=3 fail_timeout=30s;
        server n8n-webhook-3:5678 max_fails=3 fail_timeout=30s;
    }

    upstream n8n_main {
        server n8n:5678;
        keepalive 16;
    }

    server {
        listen 80;
        server_name _;

        client_max_body_size 100M;
        client_body_buffer_size 128k;
        proxy_buffer_size 128k;
        proxy_buffers 4 256k;
        proxy_busy_buffers_size 256k;

        proxy_http_version 1.1;
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Connection "";
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;

        location ~ ^/webhook/ {
            proxy_pass http://n8n_webhooks;
            proxy_set_header Host $http_host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        location ~ ^/webhook-test/ {
            proxy_pass http://n8n_webhooks;
            proxy_set_header Host $http_host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        location / {
            proxy_pass http://n8n_main;
            proxy_set_header Host $http_host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection $connection_upgrade;
            proxy_set_header Origin $http_origin;
            proxy_read_timeout 86400s;
            proxy_send_timeout 86400s;
        }

        location /nginx-health {
            access_log off;
            return 200 "healthy\n";
            add_header Content-Type text/plain;
        }

        location /nginx-status {
            stub_status on;
            access_log off;
            allow 127.0.0.1;
            allow 10.0.0.0/8;
            allow 172.16.0.0/12;
            allow 192.168.0.0/16;
            deny all;
        }
    }
}
EOF
```

---

## Step 6: Create Prometheus Configuration

```bash
cat > prometheus/prometheus.yml << 'EOF'
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'n8n-production'
    static_configs:
      - targets: ['n8n:5678']
        labels:
          instance: 'production'

  - job_name: 'postgresql'
    static_configs:
      - targets: ['postgres-exporter:9187']

  - job_name: 'redis'
    static_configs:
      - targets: ['redis-exporter:9121']

  - job_name: 'nginx'
    static_configs:
      - targets: ['nginx-exporter:9113']
EOF
```

---

## Step 7: Create Docker Compose File

```bash
cat > docker-compose.yml << 'EOF'
services:
  # Nginx Load Balancer
  nginx:
    image: nginx:alpine
    restart: unless-stopped
    ports:
      - "5678:80"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - n8n
      - n8n-webhook-1
      - n8n-webhook-2
      - n8n-webhook-3
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://127.0.0.1/nginx-health"]
      interval: 10s
      timeout: 5s
      retries: 3
    networks:
      - n8n-network

  # PostgreSQL Database
  postgres:
    image: postgres:16-alpine
    restart: unless-stopped
    command: >
      postgres
      -c shared_buffers=512MB
      -c effective_cache_size=1GB
      -c work_mem=32MB
      -c max_connections=500
      -c synchronous_commit=off
      -c checkpoint_completion_target=0.9
      -c random_page_cost=1.1
      -c effective_io_concurrency=200
    environment:
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - POSTGRES_DB=${POSTGRES_DB}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 5s
      timeout: 5s
      retries: 10
      start_period: 30s
    networks:
      - n8n-network

  # Redis Queue
  redis:
    image: redis:7-alpine
    restart: unless-stopped
    command: redis-server --appendonly yes --maxmemory 512mb --maxmemory-policy noeviction --tcp-backlog 2048 --tcp-keepalive 300 --timeout 0
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - n8n-network

  # PgBouncer - Connection Pooling (CRITICAL for sustained load)
  pgbouncer:
    image: edoburu/pgbouncer:latest
    restart: unless-stopped
    environment:
      - DATABASE_URL=postgres://${POSTGRES_USER}:${POSTGRES_PASSWORD}@postgres:5432/${POSTGRES_DB}
      - PGBOUNCER_USER=${POSTGRES_USER}
      - PGBOUNCER_PASSWORD=${POSTGRES_PASSWORD}
      - LISTEN_PORT=6432
      - AUTH_TYPE=scram-sha-256
      - POOL_MODE=transaction
      - DEFAULT_POOL_SIZE=75
      - MIN_POOL_SIZE=25
      - RESERVE_POOL_SIZE=25
      - MAX_CLIENT_CONN=1000
      - MAX_DB_CONNECTIONS=150
      - SERVER_IDLE_TIMEOUT=600
      - QUERY_WAIT_TIMEOUT=120
    depends_on:
      postgres:
        condition: service_healthy
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -h localhost -p 6432 -U ${POSTGRES_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - n8n-network

  # n8n Main Process
  n8n:
    image: docker.n8n.io/n8nio/n8n:${N8N_VERSION:-latest}
    restart: unless-stopped
    environment:
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=pgbouncer
      - DB_POSTGRESDB_PORT=6432
      - DB_POSTGRESDB_DATABASE=${POSTGRES_DB}
      - DB_POSTGRESDB_USER=${POSTGRES_USER}
      - DB_POSTGRESDB_PASSWORD=${POSTGRES_PASSWORD}
      - DB_POSTGRESDB_POOL_SIZE=25
      - EXECUTIONS_MODE=queue
      - QUEUE_BULL_REDIS_HOST=redis
      - QUEUE_BULL_REDIS_PORT=6379
      - QUEUE_HEALTH_CHECK_ACTIVE=true
      - N8N_HOST=${N8N_HOST}
      - N8N_PORT=5678
      - N8N_PROTOCOL=${N8N_PROTOCOL:-http}
      - N8N_EDITOR_BASE_URL=${N8N_EDITOR_BASE_URL}
      - N8N_PUSH_BACKEND=${N8N_PUSH_BACKEND:-websocket}
      - N8N_SECURE_COOKIE=${N8N_SECURE_COOKIE:-true}
      - NODE_ENV=production
      - WEBHOOK_URL=${WEBHOOK_URL}
      - GENERIC_TIMEZONE=${GENERIC_TIMEZONE}
      - N8N_METRICS=true
      - N8N_DIAGNOSTICS_ENABLED=false
      - EXECUTIONS_DATA_PRUNE=true
      - EXECUTIONS_DATA_MAX_AGE=${EXECUTIONS_DATA_MAX_AGE:-336}
      - EXECUTIONS_DATA_SAVE_ON_ERROR=all
      - EXECUTIONS_DATA_SAVE_ON_SUCCESS=all
      - EXECUTIONS_DATA_SAVE_ON_PROGRESS=false
      - EXECUTIONS_DATA_SAVE_MANUAL_EXECUTIONS=true
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}
      - N8N_LOG_LEVEL=${N8N_LOG_LEVEL:-info}
      - N8N_LOG_OUTPUT=${N8N_LOG_OUTPUT:-console}
    volumes:
      - n8n_data:/home/node/.n8n
      - ./local-files:/home/node/.n8n-files
    depends_on:
      pgbouncer:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD-SHELL", "wget -qO- http://localhost:5678/healthz || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 3
    networks:
      - n8n-network

  # Webhook Processor 1
  n8n-webhook-1:
    image: docker.n8n.io/n8nio/n8n:${N8N_VERSION:-latest}
    restart: unless-stopped
    command: webhook
    deploy:
      resources:
        limits:
          memory: 512M
        reservations:
          memory: 256M
    environment:
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=pgbouncer
      - DB_POSTGRESDB_PORT=6432
      - DB_POSTGRESDB_DATABASE=${POSTGRES_DB}
      - DB_POSTGRESDB_USER=${POSTGRES_USER}
      - DB_POSTGRESDB_PASSWORD=${POSTGRES_PASSWORD}
      - DB_POSTGRESDB_POOL_SIZE=25
      - EXECUTIONS_MODE=queue
      - QUEUE_BULL_REDIS_HOST=redis
      - QUEUE_BULL_REDIS_PORT=6379
      - N8N_HOST=${N8N_HOST}
      - N8N_PORT=5678
      - N8N_PROTOCOL=${N8N_PROTOCOL:-http}
      - NODE_ENV=production
      - WEBHOOK_URL=${WEBHOOK_URL}
      - GENERIC_TIMEZONE=${GENERIC_TIMEZONE}
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}
      - N8N_LOG_LEVEL=${N8N_LOG_LEVEL:-info}
    volumes:
      - n8n_data:/home/node/.n8n
      - ./local-files:/home/node/.n8n-files
    depends_on:
      pgbouncer:
        condition: service_healthy
      redis:
        condition: service_healthy
      n8n:
        condition: service_healthy
    networks:
      - n8n-network

  # Webhook Processor 2
  n8n-webhook-2:
    image: docker.n8n.io/n8nio/n8n:${N8N_VERSION:-latest}
    restart: unless-stopped
    command: webhook
    deploy:
      resources:
        limits:
          memory: 512M
        reservations:
          memory: 256M
    environment:
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=pgbouncer
      - DB_POSTGRESDB_PORT=6432
      - DB_POSTGRESDB_DATABASE=${POSTGRES_DB}
      - DB_POSTGRESDB_USER=${POSTGRES_USER}
      - DB_POSTGRESDB_PASSWORD=${POSTGRES_PASSWORD}
      - DB_POSTGRESDB_POOL_SIZE=25
      - EXECUTIONS_MODE=queue
      - QUEUE_BULL_REDIS_HOST=redis
      - QUEUE_BULL_REDIS_PORT=6379
      - N8N_HOST=${N8N_HOST}
      - N8N_PORT=5678
      - N8N_PROTOCOL=${N8N_PROTOCOL:-http}
      - NODE_ENV=production
      - WEBHOOK_URL=${WEBHOOK_URL}
      - GENERIC_TIMEZONE=${GENERIC_TIMEZONE}
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}
      - N8N_LOG_LEVEL=${N8N_LOG_LEVEL:-info}
    volumes:
      - n8n_data:/home/node/.n8n
      - ./local-files:/home/node/.n8n-files
    depends_on:
      pgbouncer:
        condition: service_healthy
      redis:
        condition: service_healthy
      n8n:
        condition: service_healthy
    networks:
      - n8n-network

  # Webhook Processor 3
  n8n-webhook-3:
    image: docker.n8n.io/n8nio/n8n:${N8N_VERSION:-latest}
    restart: unless-stopped
    command: webhook
    deploy:
      resources:
        limits:
          memory: 512M
        reservations:
          memory: 256M
    environment:
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=pgbouncer
      - DB_POSTGRESDB_PORT=6432
      - DB_POSTGRESDB_DATABASE=${POSTGRES_DB}
      - DB_POSTGRESDB_USER=${POSTGRES_USER}
      - DB_POSTGRESDB_PASSWORD=${POSTGRES_PASSWORD}
      - DB_POSTGRESDB_POOL_SIZE=25
      - EXECUTIONS_MODE=queue
      - QUEUE_BULL_REDIS_HOST=redis
      - QUEUE_BULL_REDIS_PORT=6379
      - N8N_HOST=${N8N_HOST}
      - N8N_PORT=5678
      - N8N_PROTOCOL=${N8N_PROTOCOL:-http}
      - NODE_ENV=production
      - WEBHOOK_URL=${WEBHOOK_URL}
      - GENERIC_TIMEZONE=${GENERIC_TIMEZONE}
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}
      - N8N_LOG_LEVEL=${N8N_LOG_LEVEL:-info}
    volumes:
      - n8n_data:/home/node/.n8n
      - ./local-files:/home/node/.n8n-files
    depends_on:
      pgbouncer:
        condition: service_healthy
      redis:
        condition: service_healthy
      n8n:
        condition: service_healthy
    networks:
      - n8n-network

  # Workers
  n8n-worker:
    image: docker.n8n.io/n8nio/n8n:${N8N_VERSION:-latest}
    restart: unless-stopped
    command: worker
    environment:
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=pgbouncer
      - DB_POSTGRESDB_PORT=6432
      - DB_POSTGRESDB_DATABASE=${POSTGRES_DB}
      - DB_POSTGRESDB_USER=${POSTGRES_USER}
      - DB_POSTGRESDB_PASSWORD=${POSTGRES_PASSWORD}
      - DB_POSTGRESDB_POOL_SIZE=25
      - EXECUTIONS_MODE=queue
      - QUEUE_BULL_REDIS_HOST=redis
      - QUEUE_BULL_REDIS_PORT=6379
      - QUEUE_HEALTH_CHECK_ACTIVE=true
      - QUEUE_BULL_REDIS_TIMEOUT_THRESHOLD=${WORKER_TIMEOUT:-60000}
      - N8N_CONCURRENCY_PRODUCTION_LIMIT=${WORKER_CONCURRENCY:-10}
      - NODE_ENV=production
      - GENERIC_TIMEZONE=${GENERIC_TIMEZONE}
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}
      - N8N_LOG_LEVEL=${N8N_LOG_LEVEL:-info}
      - N8N_LOG_OUTPUT=${N8N_LOG_OUTPUT:-console}
    volumes:
      - n8n_data:/home/node/.n8n
      - ./local-files:/home/node/.n8n-files
    depends_on:
      pgbouncer:
        condition: service_healthy
      redis:
        condition: service_healthy
      n8n:
        condition: service_healthy
    deploy:
      replicas: ${WORKER_REPLICAS:-2}
      resources:
        limits:
          memory: 512M
        reservations:
          memory: 256M
    networks:
      - n8n-network

  # Prometheus
  prometheus:
    image: prom/prometheus:latest
    restart: unless-stopped
    ports:
      - "9091:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=30d'
      - '--web.enable-lifecycle'
    networks:
      - n8n-network

  # Grafana
  grafana:
    image: grafana/grafana:latest
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_USERS_ALLOW_SIGN_UP=false
    volumes:
      - grafana_data:/var/lib/grafana
    depends_on:
      - prometheus
    networks:
      - n8n-network

  # PostgreSQL Exporter
  postgres-exporter:
    image: prometheuscommunity/postgres-exporter:latest
    restart: unless-stopped
    environment:
      - DATA_SOURCE_NAME=postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@postgres:5432/${POSTGRES_DB}?sslmode=disable
    depends_on:
      postgres:
        condition: service_healthy
    networks:
      - n8n-network

  # Redis Exporter
  redis-exporter:
    image: oliver006/redis_exporter:latest
    restart: unless-stopped
    environment:
      - REDIS_ADDR=redis://redis:6379
    depends_on:
      redis:
        condition: service_healthy
    networks:
      - n8n-network

  # Nginx Exporter
  nginx-exporter:
    image: nginx/nginx-prometheus-exporter:latest
    restart: unless-stopped
    command:
      - '-nginx.scrape-uri=http://nginx:80/nginx-status'
    depends_on:
      - nginx
    networks:
      - n8n-network

volumes:
  postgres_data:
  redis_data:
  n8n_data:
  prometheus_data:
  grafana_data:

networks:
  n8n-network:
    driver: bridge
EOF
```

---

## Step 8: Start the Stack

```bash
docker compose up -d
```

---

## Step 9: Verify Installation

```bash
# Check all containers are running
docker compose ps

# Wait for health checks to pass (about 30-60 seconds)
sleep 30

# Test n8n health endpoint
curl http://localhost:5678/healthz

# Test nginx health endpoint
curl http://localhost:5678/nginx-health
```

---

## Step 10: Configure Grafana

### 10a: Add Prometheus Data Source

```bash
curl -X POST http://admin:admin@localhost:3000/api/datasources \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Prometheus",
    "type": "prometheus",
    "url": "http://prometheus:9090",
    "access": "proxy",
    "isDefault": true
  }'
```

### 10b: Verify Prometheus Targets

```bash
curl -s http://localhost:9091/api/v1/targets | jq '.data.activeTargets[] | {job: .labels.job, health: .health}'
```

Expected output:
```json
{"job": "n8n-production", "health": "up"}
{"job": "postgresql", "health": "up"}
{"job": "redis", "health": "up"}
{"job": "nginx", "health": "up"}
```

---

## Step 11: Access Your Services

| Service | URL | Credentials |
|---------|-----|-------------|
| **n8n** | http://localhost:5678 | Create on first access |
| **Grafana** | http://localhost:3000 | admin / admin |
| **Prometheus** | http://localhost:9091 | No auth |

---

## Verification Commands

### Check Container Status

```bash
docker compose ps
```

### Check Resource Usage

```bash
docker stats --no-stream --filter "name=n8n-compose"
```

### Check PostgreSQL Connections

```bash
docker exec n8n-compose-postgres-1 psql -U n8n -d n8n -c \
  "SELECT count(*) FROM pg_stat_activity WHERE datname='n8n';"
```

### Check Redis Queue

```bash
docker exec n8n-compose-redis-1 redis-cli LLEN bull:jobs:wait
```

### View Logs

```bash
# All services
docker compose logs -f

# Specific service
docker compose logs -f n8n
docker compose logs -f n8n-worker
```

---

## Stopping and Starting

```bash
# Stop all services
docker compose down

# Start all services
docker compose up -d

# Restart a specific service
docker compose restart n8n
```

---

## Uninstall (Remove Everything)

```bash
# Stop and remove containers, networks
docker compose down

# Also remove volumes (THIS DELETES ALL DATA)
docker compose down -v
```

---

## Quick Reference

| What | Command |
|------|---------|
| Start | `docker compose up -d` |
| Stop | `docker compose down` |
| Status | `docker compose ps` |
| Logs | `docker compose logs -f` |
| Restart | `docker compose restart` |
| Scale workers | Edit `.env` then `docker compose up -d` |
