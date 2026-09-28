# Taxi Mobility Platform (v19.0) - Containerization & Deployment Architecture

This document provides a comprehensive specification and operational guide for containerizing, configuring, deploying, and troubleshooting the **Taxi Mobility Platform (v19.0)** across local development clusters, staging environments, and production Kubernetes/Docker infrastructures.

---

## 1. Containerization Overview

The platform uses a multi-surface microservices architecture tailored for the Saudi Arabian market (SAMA, ZATCA Phase-2, and Nafath compliant). The core components containerized include:

* **API Gateway (`api-gateway`)**: Node.js 22 Alpine runtime handling HTTP routing, WebSocket telemetry, ZATCA e-invoicing webhooks, and SAMA payment callbacks.
* **PostGIS Database (`postgis`)**: PostgreSQL 16 with PostGIS 3.4 spatial extensions for KNN ($<->$) nearest-driver geospatial matching.
* **Prometheus & Grafana (`monitoring`)**: Real-time observability stack tracking HTTP latency, active trips, and fleet telemetry.

---

## 2. Multi-Stage Dockerfile (`Dockerfile`)

The production Dockerfile employs multi-stage builds to minimize image footprint and enforce security best practices (non-root execution).

```dockerfile
# ==========================================
# Stage 1: Build & Dependencies Builder
# ==========================================
FROM node:22-alpine AS builder

WORKDIR /app

# Copy package descriptors
COPY package*.json ./
RUN npm ci --include=dev

# Copy application source code
COPY . .

# Build application assets (Vite / TypeScript bundle)
RUN npm run build

# ==========================================
# Stage 2: Production Minimal Runtime
# ==========================================
FROM node:22-alpine AS runner

WORKDIR /app

# Install curl for healthcheck probes
RUN apk add --no-cache curl

# Create non-root system user for security compliance
USER node

# Copy dependencies and build outputs from builder
COPY --chown=node:node package*.json ./
COPY --chown=node:node --from=builder /app/node_modules ./node_modules
COPY --chown=node:node --from=builder /app/dist ./dist
COPY --chown=node:node --from=builder /app/services ./services
COPY --chown=node:node --from=builder /app/database ./database
COPY --chown=node:node --from=builder /app/server.ts ./server.ts
COPY --chown=node:node --from=builder /app/server.js ./server.js
COPY --chown=node:node --from=builder /app/migrate.js ./migrate.js

# Expose container application port
EXPOSE 3000

# Container Health Check probe complying with K8s/Docker standards
HEALTHCHECK --interval=30s --timeout=5s --start-period=15s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1

# Start enterprise master server
CMD ["node", "services/api-gateway/src/server.js"]
```

---

## 3. Docker Compose Stack Configuration (`docker-compose.yml`)

The local and staging development stack orchestrates PostGIS, the API Gateway, Prometheus, and Grafana on an internal bridge network (`taxi-internal-net`).

```yaml
version: '3.8'

services:
  postgis:
    image: postgis/postgis:16-3.4
    container_name: taxi-ksa-postgis
    restart: always
    environment:
      POSTGRES_USER: taxi_admin
      POSTGRES_PASSWORD: taxi_secure_password_2026
      POSTGRES_DB: taxi_mobility_db
    ports:
      - "5432:5432"
    volumes:
      - postgis_data:/var/lib/postgresql/data
      - ./database/migrations/001_initial.sql:/docker-entrypoint-initdb.d/001_initial.sql:ro
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U taxi_admin -d taxi_mobility_db"]
      interval: 5s
      timeout: 5s
      retries: 5
    networks:
      - taxi-internal-net

  api-gateway:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: taxi-ksa-api-gateway
    restart: unless-stopped
    depends_on:
      postgis:
        condition: service_healthy
    environment:
      PORT: 3000
      NODE_ENV: production
      DATABASE_URL: postgresql://taxi_admin:taxi_secure_password_2026@postgis:5432/taxi_mobility_db
      SAMA_WEBHOOK_SECRET: sama-payment-webhook-secret
      ZATCA_SELLER_VAT: "310123456700003"
    ports:
      - "3000:3000"
    volumes:
      - ./services/api-gateway:/app/services/api-gateway
    networks:
      - taxi-internal-net

  prometheus:
    image: prom/prometheus:latest
    container_name: taxi-ksa-prometheus
    restart: unless-stopped
    ports:
      - "9090:9090"
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
    networks:
      - taxi-internal-net

  grafana:
    image: grafana/grafana:latest
    container_name: taxi-ksa-grafana
    restart: unless-stopped
    ports:
      - "3001:3000"
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin_secure_2026
    volumes:
      - ./monitoring/grafana/provisioning:/etc/grafana/provisioning
      - ./monitoring/grafana/dashboards:/var/lib/grafana/dashboards
    depends_on:
      - prometheus
    networks:
      - taxi-internal-net

volumes:
  postgis_data:

networks:
  taxi-internal-net:     driver: bridge
```

---

## 4. Environment Variables Specification (`.env.example`)

Ensure all required production secrets are supplied in your `.env` file prior to deployment:

```env
# GEMINI_API_KEY: Required for Gemini AI API calls.
GEMINI_API_KEY="MY_GEMINI_API_KEY"

# APP_URL: The URL where this applet is hosted.
APP_URL="MY_APP_URL"

# Server Configuration
NODE_ENV=production
PORT=3000

# Database Configuration (PostgreSQL + PostGIS in KSA Datacenter)
DATABASE_URL=postgresql://app_user:SecureDatabasePassword123@ksa-db-cluster.internal:5432/taxidb?sslmode=verify-full

# Security Secrets
JWT_SECRET=super_secure_jwt_secret_key_v19_change_in_production
ENCRYPTION_KEY=32_byte_hex_string_for_sensitive_data_encryption

# KSA Regulatory Integrations (ZATCA & Nafath)
ZATCA_CSID=your_zatca_cryptographic_stamp_identifier
NAFATH_API_KEY=your_official_nafath_gateway_api_key

# Payment Gateway (SAMA-Compliant: Moyasar / HyperPay)
PAYMENT_GATEWAY_SECRET_KEY=sk_test_YourMoyasarSecretKeyHere
PAYMENT_GATEWAY_WEBHOOK_SECRET=whsec_YourWebhookVerificationSecret

# Push Notifications (Firebase Admin SDK)
FIREBASE_PROJECT_ID=taxi-platform-ksa-prod
FIREBASE_CLIENT_EMAIL=firebase-adminsdk-xyz@taxi-platform-ksa-prod.iam.gserviceaccount.com
FIREBASE_PRIVATE_KEY="-----BEGIN PRIVATE KEY-----\nMIIEvgIBADA...YourFirebasePrivateKey...\n-----END PRIVATE KEY-----\n"
```

---

## 5. PostGIS Spatial Extensions & Database Initialization

The platform relies on PostgreSQL 16 with PostGIS 3.4+ for spatial dispatching. During startup, migrations automatically execute:

```sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "postgis";

-- Spatial Indexing for Driver Geographies
CREATE INDEX IF NOT EXISTS idx_drivers_location_gist ON drivers USING GIST (current_location);
CREATE INDEX IF NOT EXISTS idx_trips_pickup_gist ON trips USING GIST (pickup_location);
```

---

## 6. Troubleshooting Container Deployments

1. **Database Healthcheck Fails:**
   * *Symptom:* `api-gateway` container restarts with connection refusal.
   * *Resolution:* Verify PostgreSQL container health status using `docker compose ps`. Ensure `POSTGRES_USER` and `POSTGRES_PASSWORD` match your `DATABASE_URL`.

2. **Port 3000 Already in Use:**
   * *Symptom:* `Error starting userland proxy: listen tcp4 0.0.0.0:3000: bind: address already in use`.
   * *Resolution:* Stop conflicting processes or override the port via `PORT=3002 docker compose up -d`.

3. **Memory Limits Exceeded in K8s:**
   * *Symptom:* Pod killed with `OOMKilled`.
   * *Resolution:* Adjust resource requests and limits in `k8s/deployment.yaml` (recommended: request `512Mi`, limit `1024Mi`).