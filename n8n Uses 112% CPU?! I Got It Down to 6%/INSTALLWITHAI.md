# n8n High-Performance Installation - AI-Assisted Guide

Use these prompts with an AI assistant (Claude, ChatGPT, etc.) to install n8n step by step.

---

## How to Use This Guide

1. Copy each prompt below
2. Paste it to your AI assistant
3. Follow the AI's instructions
4. Move to the next step

---

## Step 1: Create Project Directory

### Prompt:

```
I want to set up a high-performance n8n installation. Please help me create the project directory.

Create a directory called "n8n-compose" and navigate into it. Show me the command to run.
```

### Expected AI Response:
The AI should provide:
```bash
mkdir -p n8n-compose && cd n8n-compose
```

---

## Step 2: Create Directory Structure

### Prompt:

```
I'm setting up n8n with nginx, prometheus monitoring, and grafana dashboards.

Create the following subdirectories in my n8n-compose folder:
- nginx (for load balancer config)
- prometheus (for metrics config)
- grafana/dashboards (for dashboard JSON files)
- local-files (for workflow file access)

Show me the command.
```

### Expected AI Response:
```bash
mkdir -p nginx prometheus grafana/dashboards local-files
```

---

## Step 3: Generate Encryption Key

### Prompt:

```
I need to generate a secure encryption key for n8n. This key will encrypt all credentials stored in n8n.

Requirements:
- Use openssl to generate a base64-encoded random key
- The key should be 24 bytes (192 bits)
- Show me the command and explain that I need to save the output

Generate the command for me.
```

### Expected AI Response:
```bash
openssl rand -base64 24
```

---

## Step 4: Create Environment File

### Prompt:

```
Create a .env file for my n8n production setup with the following configuration:

n8n Settings:
- Version: 2.4.3
- Host: localhost:5678
- Protocol: http
- Timezone: America/Los_Angeles
- Metrics enabled

PostgreSQL Settings:
- User: n8n
- Password: placeholder (I'll change it)
- Database: n8n

Performance Settings:
- 5 worker replicas
- 20 concurrent executions per worker
- 3 webhook processors (load balanced)
- 60 second worker timeout
- 14 days (336 hours) execution data retention

Logging:
- Log level: info
- Output: console

Create the complete .env file content using a heredoc (cat > .env << 'EOF') so I can paste it directly. Include comments explaining each section. Add a placeholder for the encryption key that I'll replace.
```

---

## Step 5: Create Nginx Configuration

### Prompt:

```
Create an nginx configuration file for load balancing n8n with these requirements:

Architecture:
- 4 worker processes
- Load balance webhook traffic across 3 webhook processors (n8n-webhook-1, n8n-webhook-2, n8n-webhook-3)
- Route all other traffic to main n8n instance
- Use least_conn algorithm for webhook load balancing

Performance Tuning:
- 4096 worker connections
- Enable epoll and multi_accept
- Keepalive timeout 65s
- Keepalive requests 10000
- TCP nodelay and nopush enabled

Proxy Settings:
- HTTP 1.1 with keepalive
- 100MB max body size for large webhook payloads
- 128k buffer sizes
- Forward real IP headers
- WebSocket upgrade support for the main n8n route
- 86400s (24 hour) timeout for WebSocket connections

Health Endpoints:
- /nginx-health - returns 200 "healthy"
- /nginx-status - stub_status for prometheus (restrict to private IPs)

Routing:
- /webhook/* and /webhook-test/* -> n8n_webhooks upstream
- Everything else -> n8n_main upstream

Create the complete nginx.conf using a heredoc (cat > nginx/nginx.conf << 'EOF') so I can paste it directly.
```

---

## Step 6: Create Prometheus Configuration

### Prompt:

```
Create a Prometheus configuration file to monitor my n8n stack.

Scrape Targets:
1. n8n-production - scrape n8n:5678 with label instance="production"
2. postgresql - scrape postgres-exporter:9187
3. redis - scrape redis-exporter:9121
4. nginx - scrape nginx-exporter:9113

Settings:
- Scrape interval: 15 seconds
- Evaluation interval: 15 seconds

Create the complete prometheus.yml using a heredoc (cat > prometheus/prometheus.yml << 'EOF') so I can paste it directly.
```

---

## Step 7: Create Docker Compose File

### Prompt:

```
Create a docker-compose.yml for a high-performance n8n production setup.

Services needed:

1. nginx (load balancer)
   - Image: nginx:alpine
   - Port: 5678:80
   - Mount nginx.conf
   - Health check on /nginx-health
   - Depends on: n8n, n8n-webhook-1, n8n-webhook-2, n8n-webhook-3

2. postgres (database)
   - Image: postgres:16-alpine
   - Performance tuning via command flags:
     - shared_buffers=512MB
     - effective_cache_size=1GB
     - work_mem=32MB
     - max_connections=500
     - synchronous_commit=off
     - checkpoint_completion_target=0.9
     - random_page_cost=1.1 (SSD optimized)
     - effective_io_concurrency=200
   - Health check with pg_isready
   - Volume for data persistence

3. redis (queue)
   - Image: redis:7-alpine
   - Command flags: appendonly yes, maxmemory 512mb, noeviction policy, tcp-backlog 2048
   - Health check with redis-cli ping
   - Volume for data persistence

4. pgbouncer (connection pooling - CRITICAL for sustained load)
   - Image: edoburu/pgbouncer:latest
   - Transaction pooling mode
   - Listen on port 6432
   - Auth type: scram-sha-256
   - Pool settings: DEFAULT_POOL_SIZE=75, MAX_CLIENT_CONN=1000, MAX_DB_CONNECTIONS=150
   - Health check with pg_isready on port 6432
   - Depends on: postgres (healthy)

5. n8n (main process)
   - Image: docker.n8n.io/n8nio/n8n:${N8N_VERSION}
   - Queue mode with Redis
   - Connect to pgbouncer:6432 (NOT postgres directly)
   - Pool size 25
   - Metrics enabled
   - Execution pruning enabled
   - Environment variables from .env
   - Health check on /healthz
   - Depends on: pgbouncer (healthy), redis (healthy)

6. n8n-webhook-1, n8n-webhook-2, and n8n-webhook-3 (webhook processors)
   - Same image as n8n
   - Command: webhook
   - Connect to pgbouncer:6432
   - Pool size 25
   - Memory limit: 512M (CRITICAL - prevents memory leak degradation)
   - Memory reservation: 256M
   - Depends on: pgbouncer, redis, n8n (healthy)

7. n8n-worker (workflow executors)
   - Same image as n8n
   - Command: worker
   - Connect to pgbouncer:6432
   - Replicas from WORKER_REPLICAS env var
   - Concurrency from WORKER_CONCURRENCY env var
   - Memory limit: 512M
   - Memory reservation: 256M
   - Depends on: pgbouncer, redis, n8n (healthy)

7. prometheus (metrics)
   - Image: prom/prometheus:latest
   - Port: 9091:9090
   - Mount prometheus.yml
   - 30 day retention
   - Volume for data

8. grafana (dashboards)
   - Image: grafana/grafana:latest
   - Port: 3000:3000
   - Admin user/password: admin/admin
   - Depends on: prometheus
   - Volume for data

9. postgres-exporter
   - Image: prometheuscommunity/postgres-exporter:latest
   - Connect to postgres database
   - Depends on: postgres (healthy)

10. redis-exporter
    - Image: oliver006/redis_exporter:latest
    - Connect to redis
    - Depends on: redis (healthy)

11. nginx-exporter
    - Image: nginx/nginx-prometheus-exporter:latest
    - Scrape nginx /nginx-status
    - Depends on: nginx

Volumes: postgres_data, redis_data, n8n_data, prometheus_data, grafana_data

Network: n8n-network (bridge driver)

Create the complete docker-compose.yml using a heredoc (cat > docker-compose.yml << 'EOF') so I can paste it directly.
```

---

## Step 8: Start the Stack

### Prompt:

```
I've created all the configuration files for my n8n setup. Now I need to start all the services.

Show me the Docker Compose command to:
1. Start all services in detached mode
2. Wait for them to be healthy

Also explain what order the services will start in based on their dependencies.
```

### Expected AI Response:
```bash
docker compose up -d
```

---

## Step 9: Verify Installation

### Prompt:

```
I just started my n8n Docker Compose stack. Help me verify everything is working correctly.

Show me commands to:
1. Check that all containers are running and healthy
2. Wait 30 seconds for health checks to complete
3. Test the n8n health endpoint at localhost:5678/healthz
4. Test the nginx health endpoint at localhost:5678/nginx-health

Provide all commands I can run in sequence.
```

### Expected AI Response:
```bash
docker compose ps
sleep 30
curl http://localhost:5678/healthz
curl http://localhost:5678/nginx-health
```

---

## Step 10: Configure Grafana Data Source

### Prompt:

```
I need to configure Grafana to use Prometheus as a data source via the API.

Grafana is running at localhost:3000 with credentials admin/admin.
Prometheus is accessible from within Docker at http://prometheus:9090.

Show me the curl command to:
1. Add Prometheus as the default data source in Grafana

Then show me how to verify Prometheus is scraping all targets correctly by querying the Prometheus API.
```

### Expected AI Response:
```bash
# Add Prometheus data source
curl -X POST http://admin:admin@localhost:3000/api/datasources \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Prometheus",
    "type": "prometheus",
    "url": "http://prometheus:9090",
    "access": "proxy",
    "isDefault": true
  }'

# Verify Prometheus targets
curl -s http://localhost:9091/api/v1/targets | jq '.data.activeTargets[] | {job: .labels.job, health: .health}'
```

---

## Step 11: Access Services

### Prompt:

```
My n8n stack is now running. Summarize the URLs and credentials I need to access each service:

1. n8n (main application)
2. Grafana (monitoring dashboards)
3. Prometheus (metrics)

Format as a table.
```

### Expected AI Response:

| Service | URL | Credentials |
|---------|-----|-------------|
| n8n | http://localhost:5678 | Create on first access |
| Grafana | http://localhost:3000 | admin / admin |
| Prometheus | http://localhost:9091 | No auth required |

---

## Troubleshooting Prompts

### If containers won't start:

```
My n8n Docker Compose containers are not starting properly.

Show me commands to:
1. View logs for all services
2. View logs for a specific service (n8n, postgres, redis)
3. Check which containers are running vs stopped
4. See detailed container status including health checks
```

### If database connection fails:

```
My n8n is showing database connection errors.

Show me commands to:
1. Check if PostgreSQL container is healthy
2. View PostgreSQL logs
3. Test database connection from inside the n8n container
4. Check how many active database connections exist
```

### If workers aren't processing:

```
My n8n workers are not processing jobs from the queue.

Show me commands to:
1. Check worker container logs
2. Verify Redis is running and accessible
3. Check the Redis queue length for waiting jobs
4. Restart the worker containers
```

### If performance is slow:

```
My n8n is running slowly under load.

Show me commands to:
1. Check CPU and memory usage of all containers
2. Check PostgreSQL active connections and their states
3. Check Redis memory usage
4. Check if any containers are being OOM killed
```

---

## Scaling Prompts

### To add more workers:

```
I need to scale my n8n workers from 5 to 10 replicas.

Show me:
1. How to update the .env file
2. The command to apply the changes
3. How to verify the new workers are running
```

### To add another webhook processor:

```
I need to add a third webhook processor (n8n-webhook-3) to my n8n stack.

Show me:
1. The docker-compose.yml service definition to add
2. The nginx.conf upstream changes needed
3. Commands to apply the changes without downtime
```

---

## Backup Prompt

```
I need to backup my n8n installation including:
1. PostgreSQL database
2. n8n data volume (encryption keys, settings)
3. Redis data

Show me commands to:
1. Create a backup directory with today's date
2. Dump the PostgreSQL database
3. Copy the n8n data volume contents
4. Export Redis data

Format as a complete backup script I can save and run.
```

---

## Complete Installation - Single Prompt

If you want to do everything in one prompt:

```
Help me install a high-performance n8n production setup with Docker Compose.

Requirements:
- nginx load balancer (4 workers, least_conn algorithm)
- PostgreSQL 16 with performance tuning (512MB shared buffers, SSD optimized)
- PgBouncer for connection pooling (transaction mode, 150 max DB connections) - CRITICAL for sustained load
- Redis 7 for queue (512MB, noeviction)
- 5 n8n workers with 20 concurrency each (100 total concurrent executions)
- 3 webhook processors (load balanced) with 512MB memory limits - prevents memory leak degradation
- All n8n services connect to pgbouncer:6432, NOT postgres directly
- Prometheus + Grafana monitoring with exporters for postgres, redis, nginx
- DB connection pool size of 25 per service

Please provide:
1. Directory structure to create
2. Complete .env file
3. Complete nginx.conf
4. Complete prometheus.yml
5. Complete docker-compose.yml (with PgBouncer and memory limits)
6. Commands to start everything
7. Commands to verify it's working
8. URLs to access each service

Use heredocs (cat > file << 'EOF') for all file creation so I can copy-paste directly.
```

---

## Summary

| Step | Prompt Purpose |
|------|----------------|
| 1 | Create project directory |
| 2 | Create folder structure |
| 3 | Generate encryption key |
| 4 | Create .env configuration |
| 5 | Create nginx load balancer config |
| 6 | Create Prometheus monitoring config |
| 7 | Create Docker Compose stack (with PgBouncer + memory limits) |
| 8 | Start all services |
| 9 | Verify installation |
| 10 | Configure Grafana |
| 11 | Access services |

Each prompt is self-contained and provides enough context for the AI to generate the correct response.

---

## Performance Optimizations Included

This configuration includes critical performance optimizations discovered through extensive load testing:

### PgBouncer (Connection Pooling)
- **Without PgBouncer**: System crashes after ~5 minutes at 300 VUs due to database connection exhaustion
- **With PgBouncer**: 99.98% success rate over 1 hour at 300 VUs (716,714 successful requests)

### Webhook Memory Limits (512MB)
- **Without limits**: Webhook processors leak memory, growing to 700MB+ and causing degradation
- **With limits**: Containers auto-restart when hitting limit, maintaining consistent performance

### Benchmark Results
| Test | Duration | VUs | Throughput | Success Rate |
|------|----------|-----|------------|--------------|
| Short burst | 60s | 500 | 312 req/s | 100% |
| Sustained load | 1 hour | 300 | 199 req/s | 99.98% |
