<div align="center">

<h1>🐳 Docura — Deployment</h1>
<h3><em>Intelligence for Your Documents</em></h3>
<p>Docker Compose · AWS EC2 · Nginx · Prometheus · Grafana</p>

[![Docker](https://img.shields.io/badge/Docker_Compose-Stack-2496ED?style=for-the-badge&logo=docker&logoColor=white)](https://www.docker.com/)
[![AWS](https://img.shields.io/badge/AWS_EC2-t3.small-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white)](https://aws.amazon.com/)
[![Nginx](https://img.shields.io/badge/Nginx-Reverse_Proxy-009639?style=for-the-badge&logo=nginx&logoColor=white)](https://nginx.org/)
[![Prometheus](https://img.shields.io/badge/Prometheus-Monitoring-E6522C?style=for-the-badge&logo=prometheus&logoColor=white)](https://prometheus.io/)
[![Grafana](https://img.shields.io/badge/Grafana-Dashboards-F46800?style=for-the-badge&logo=grafana&logoColor=white)](https://grafana.com/)

*Infrastructure-as-code for Docura — "Intelligence for Your Documents". Spins up the complete production stack — backend, frontend, database, and monitoring — with a single command.*

**[🔗 Backend Repo](https://github.com/YasasBanuka/document-insight-backend) · [⚡ Frontend Repo](https://github.com/YasasBanuka/document-insight-frontend)**

</div>

---

## 🗂️ Contents

```
docura-deployment/
├── docker-compose.yml   # Orchestrates all 5 services
├── nginx.conf           # Reverse proxy + SSE streaming config
├── prometheus.yml       # Prometheus scrape targets
├── .env.example         # Environment variable template
└── README.md            # This file
```

---

## 🚀 Quick Start

**Prerequisites:** Docker, Docker Compose, and a free [Groq API Key](https://console.groq.com/).

```bash
# 1. Clone this repository
git clone https://github.com/YasasBanuka/docura-deployment.git
cd docura-deployment

# 2. Configure environment variables
cp .env.example .env
nano .env   # Fill in your GROQ_API_KEY and JWT_SECRET

# 3. Launch the full stack
docker compose up -d

# 4. Enable pgvector (first launch only — one time setup)
docker exec -it docura-postgres psql -U postgres -d docura \
  -c "CREATE EXTENSION IF NOT EXISTS vector;"

# 5. Verify everything is running
docker compose ps
curl http://localhost/api/actuator/health
```

**Access:**
- 🌐 **Web App:** `http://localhost` (or `http://<your-ec2-ip>`)
- 📊 **Grafana:** `http://localhost:3000` (admin / admin)
- 🔍 **Prometheus:** `http://localhost:9090`

---

## 🏗️ Stack Architecture

```
┌─────────────────────────────────────────────┐
│             Docker Bridge Network            │
│                                             │
│  ┌─────────────────────────────────────┐   │
│  │     Nginx (docura-frontend :80)     │   │
│  │  /api/* → proxy → backend:8080     │   │
│  │  /*     → React SPA (index.html)   │   │
│  └─────────────────┬───────────────────┘   │
│                    │                         │
│  ┌─────────────────▼───────────────────┐   │
│  │   Spring Boot (docura-backend :8080) │   │
│  │   Groq LLaMA + ONNX Embeddings      │   │
│  └───────────────┬─────────────────────┘   │
│                  │                           │
│  ┌───────────────▼────────────────────┐    │
│  │  PostgreSQL 16 + pgvector (:5432)  │    │
│  │  Named Volume: postgres-data        │    │
│  └────────────────────────────────────┘    │
│                                             │
│  Prometheus (:9090) ──scrapes──► backend   │
│  Grafana (:3000) ──reads──► Prometheus     │
└─────────────────────────────────────────────┘
```

All internal ports are on the Docker bridge network — only **port 80 is exposed** to the host. Port 3000 and 9090 are bound to the host for admin monitoring access.

---

## ⚙️ Environment Variables (`.env`)

```env
# Database
DB_USERNAME=postgres
DB_PASSWORD=<strong-random-password>

# Spring Boot
JWT_SECRET=<run: openssl rand -hex 64>
GROQ_API_KEY=gsk_<your-groq-api-key>
SPRING_PROFILES_ACTIVE=prod

# Optional
UPLOAD_DIR=/app/uploads
```

Generate a secure JWT secret:
```bash
openssl rand -hex 64
```

---

## 🔧 Services Reference

### `postgres` — Database
- **Image:** `pgvector/pgvector:pg16`
- **Data Volume:** `postgres-data` (persistent across restarts)
- **Internal port:** `5432` (not exposed to host)

### `backend` — Spring Boot API
- **Image:** `ybanuka/docura-backend:latest` (pulled from Docker Hub)
- **Port:** `8080` (internal only — accessed via Nginx proxy)
- **Depends on:** `postgres`
- **Profile:** `prod` (uses Groq Chat + ONNX local embeddings)
- **Upload Volume:** `docura-uploads` (persists documents across restarts)

### `frontend` — Nginx + React SPA  
- **Image:** `ybanuka/docura-frontend:latest` (pulled from Docker Hub)
- **Port:** `80` (exposed to host)
- **Config:** Mounts `./nginx.conf` into container
- **Depends on:** `backend`

### `prometheus` — Metrics Scraper
- **Image:** `prom/prometheus:latest`
- **Port:** `9090` (exposed to host for admin UI)
- **Config:** Mounts `./prometheus.yml`; scrapes backend every 15s
- **Data Volume:** `prometheus-data`

### `grafana` — Dashboard
- **Image:** `grafana/grafana:latest`
- **Port:** `3000` (exposed to host)
- **Default credentials:** `admin / admin` (change on first login)
- **Data Volume:** `grafana-data`

---

## 🔐 AWS Security Group Rules

For production on AWS EC2, configure Inbound Rules:

| Port | Protocol | Source | Purpose |
|---|---|---|---|
| `22` | TCP | **My IP only** | SSH access |
| `80` | TCP | `0.0.0.0/0` | Public web traffic |
| `9090` | TCP | **My IP only** | Prometheus (open only when needed) |
| `3000` | TCP | **My IP only** | Grafana (open only when needed) |

> **Never expose port 5432 (PostgreSQL) or 8080 (backend) externally.**

---

## 📋 Operational Runbook

### Update to Latest Images
```bash
docker compose pull          # Download latest images from Docker Hub
docker compose up -d         # Restart containers (rolling update)
docker compose logs -f       # Watch logs
```

### View Logs
```bash
docker compose logs -f backend --tail=100    # Backend API logs
docker compose logs -f frontend              # Nginx access logs
docker compose ps                            # Container health status
```

### Database Backup
```bash
# Create timestamped backup
docker exec docura-postgres pg_dump -U postgres docura > backup_$(date +%Y%m%d_%H%M%S).sql

# Restore
cat backup_YYYYMMDD.sql | docker exec -i docura-postgres psql -U postgres -d docura
```

### Reset Everything (⚠️ Destructive)
```bash
docker compose down -v    # Removes ALL containers AND volumes (data loss!)
docker compose up -d      # Fresh start
```

---

## 📊 Grafana Setup

After accessing Grafana at `:3000`:

1. **Add Prometheus data source:**
   - Go to **Connections → Data Sources → Add new data source**
   - Select **Prometheus**
   - URL: `http://prometheus:9090` *(use Docker internal hostname, not `localhost`)*
   - Click **Save & Test**

2. **Useful Prometheus queries:**
   ```promql
   # Request rate per endpoint
   rate(http_server_requests_seconds_count[1m])
   
   # 99th percentile latency
   histogram_quantile(0.99, rate(http_server_requests_seconds_bucket[5m]))
   
   # Active DB connections
   hikaricp_connections_active
   ```

---

## 📚 Further Documentation

| Document | Description |
|---|---|
| [docs/infrastructure.md](docs/infrastructure.md) | AWS EC2 spec, all 5 Docker services explained, Nginx SSE config, volume strategy, TLS roadmap |
| [docs/monitoring.md](docs/monitoring.md) | Prometheus setup, 25+ PromQL query reference, Grafana dashboard guide, recommended alerts |

---

## 🔗 Related Repositories

- **[docura-backend](https://github.com/YasasBanuka/document-insight-backend)** — Spring Boot 3 backend with full docs in `docs/`
- **[docura-frontend](https://github.com/YasasBanuka/document-insight-frontend)** — React 18 TypeScript SPA

---

<div align="center">
  <sub>Built by <strong>Yasas Banuka</strong> · Docker · Nginx · AWS EC2 · Prometheus · Grafana</sub>
</div>
