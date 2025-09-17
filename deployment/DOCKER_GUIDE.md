# Docker Deployment Guide

This guide covers containerization and Docker deployment for the iServe Protocol, following containerization best practices with security hardening and production optimization.

## 📋 Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Container Architecture](#container-architecture)
4. [Building Images](#building-images)
5. [Running Containers](#running-containers)
6. [Docker Compose](#docker-compose)
7. [Security Best Practices](#security-best-practices)
8. [Production Deployment](#production-deployment)
9. [Troubleshooting](#troubleshooting)

## Overview

The iServe Protocol uses multi-stage Docker builds with security hardening for all services:

- **Frontend Services**: React DApps with nginx serving
- **Backend Services**: Go microservices with distroless runtime
- **Database**: PostgreSQL with custom configuration
- **Blockchain**: Ethereum node and IPFS integration

## Prerequisites

### System Requirements
- Docker 20.10+
- Docker Compose 2.0+
- 8GB RAM (development)
- 16GB RAM (production)

### Development Tools
```bash
# Install Docker and Docker Compose
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh

# Install Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

## Container Architecture

### Service Overview

```mermaid
graph TB
    subgraph "Frontend Tier"
        DApp[React DApp<br/>Port: 3000]
        Portal[Governance Portal<br/>Port: 3001]
    end
    
    subgraph "API Tier"
        GoAPI[Go API Service<br/>Port: 8080]
        GovAPI[Governance API<br/>Port: 8081]
    end
    
    subgraph "Data Tier"
        Postgres[(PostgreSQL<br/>Port: 5432)]
        Redis[(Redis<br/>Port: 6379)]
    end
    
    subgraph "Blockchain Tier"
        IPFS[IPFS Node<br/>Port: 5001]
        Ethereum[Ethereum Node<br/>Port: 8545]
    end
    
    DApp --> GoAPI
    Portal --> GovAPI
    GoAPI --> Postgres
    GovAPI --> Postgres
    GoAPI --> Redis
    GoAPI --> IPFS
    GoAPI --> Ethereum
```

### Image Strategy

- **Base Images**: Distroless for runtime, Alpine for build
- **Multi-stage**: Separate build and runtime stages
- **Security**: Non-root users, minimal attack surface
- **Size**: Optimized for minimal image size

## Building Images

### Frontend Image (React DApp)

```dockerfile
# Dockerfile.frontend
# Build stage
FROM node:18-alpine AS builder

WORKDIR /app

# Copy package files
COPY package*.json ./
COPY tsconfig*.json ./

# Install dependencies
RUN npm ci --only=production

# Copy source code
COPY src/ ./src/
COPY public/ ./public/

# Build application
RUN npm run build

# Runtime stage
FROM nginx:alpine

# Create non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nextjs -u 1001

# Copy built application
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/nginx.conf

# Set permissions
RUN chown -R nextjs:nodejs /usr/share/nginx/html
RUN chown -R nextjs:nodejs /var/cache/nginx
RUN chown -R nextjs:nodejs /var/log/nginx

# Switch to non-root user
USER nextjs

EXPOSE 3000
CMD ["nginx", "-g", "daemon off;"]
```

### Backend Image (Go API)

```dockerfile
# Dockerfile.backend
# Build stage
FROM golang:1.21-alpine AS builder

WORKDIR /app

# Install build dependencies
RUN apk add --no-cache git ca-certificates tzdata

# Copy go mod files
COPY go.mod go.sum ./

# Download dependencies
RUN go mod download

# Copy source code
COPY . .

# Build binary
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build \
    -ldflags='-w -s -extldflags "-static"' \
    -a -installsuffix cgo \
    -o main ./cmd/server

# Runtime stage
FROM gcr.io/distroless/static:nonroot

WORKDIR /

# Copy binary and certificates
COPY --from=builder /app/main .
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

# Use non-root user
USER nonroot:nonroot

EXPOSE 8080

ENTRYPOINT ["/main"]
```

### Build Commands

```bash
# Build frontend image
docker build -f Dockerfile.frontend -t iserve/dapp:latest ./dapp

# Build backend image
docker build -f Dockerfile.backend -t iserve/api:latest ./api

# Build governance portal
docker build -f Dockerfile.governance -t iserve/governance:latest ./governance

# Build all images
make docker-build-all
```

## Running Containers

### Individual Containers

```bash
# Run PostgreSQL
docker run -d \
  --name iserve-postgres \
  -e POSTGRES_DB=iserve \
  -e POSTGRES_USER=iserve \
  -e POSTGRES_PASSWORD=secure_password \
  -p 5432:5432 \
  -v postgres_data:/var/lib/postgresql/data \
  postgres:15-alpine

# Run Redis
docker run -d \
  --name iserve-redis \
  -p 6379:6379 \
  -v redis_data:/data \
  redis:7-alpine

# Run API service
docker run -d \
  --name iserve-api \
  -p 8080:8080 \
  -e DATABASE_URL=postgresql://iserve:secure_password@postgres:5432/iserve \
  -e REDIS_URL=redis://redis:6379 \
  --link iserve-postgres:postgres \
  --link iserve-redis:redis \
  iserve/api:latest

# Run frontend
docker run -d \
  --name iserve-dapp \
  -p 3000:3000 \
  iserve/dapp:latest
```

### Container Health Checks

```bash
# Add health checks to Dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1
```

## Docker Compose

### Development Configuration

```yaml
# docker-compose.dev.yml
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: iserve
      POSTGRES_USER: iserve
      POSTGRES_PASSWORD: dev_password
    ports:
      - "5432:5432"
    volumes:
      - postgres_dev:/var/lib/postgresql/data
      - ./scripts/init.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U iserve"]
      interval: 30s
      timeout: 10s
      retries: 3

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_dev:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 30s
      timeout: 10s
      retries: 3

  ipfs:
    image: ipfs/go-ipfs:latest
    ports:
      - "4001:4001"
      - "5001:5001"
      - "8080:8080"
    volumes:
      - ipfs_data:/data/ipfs
    environment:
      - IPFS_PROFILE=server

  api:
    build:
      context: ./api
      dockerfile: Dockerfile.backend
    ports:
      - "8080:8080"
    environment:
      - DATABASE_URL=postgresql://iserve:dev_password@postgres:5432/iserve
      - REDIS_URL=redis://redis:6379
      - IPFS_URL=http://ipfs:5001
      - LOG_LEVEL=debug
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    volumes:
      - ./configs:/app/configs:ro
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  governance-api:
    build:
      context: ./governance
      dockerfile: Dockerfile.governance
    ports:
      - "8081:8080"
    environment:
      - DATABASE_URL=postgresql://iserve:dev_password@postgres:5432/iserve
      - REDIS_URL=redis://redis:6379
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy

  dapp:
    build:
      context: ./dapp
      dockerfile: Dockerfile.frontend
    ports:
      - "3000:3000"
    environment:
      - REACT_APP_API_URL=http://localhost:8080
      - REACT_APP_CHAIN_ID=1337
    depends_on:
      - api

  governance-portal:
    build:
      context: ./governance-portal
      dockerfile: Dockerfile.frontend
    ports:
      - "3001:3000"
    environment:
      - REACT_APP_API_URL=http://localhost:8081
      - REACT_APP_CHAIN_ID=1337
    depends_on:
      - governance-api

volumes:
  postgres_dev:
  redis_dev:
  ipfs_data:

networks:
  default:
    name: iserve-dev
```

### Production Configuration

```yaml
# docker-compose.prod.yml
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD_FILE: /run/secrets/postgres_password
    volumes:
      - postgres_prod:/var/lib/postgresql/data
      - ./configs/postgres.conf:/etc/postgresql/postgresql.conf
    secrets:
      - postgres_password
    deploy:
      replicas: 1
      resources:
        limits:
          memory: 2G
          cpus: '1.0'
        reservations:
          memory: 1G
          cpus: '0.5'

  api:
    image: iserve/api:${IMAGE_TAG:-latest}
    environment:
      - DATABASE_URL=postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@postgres:5432/${POSTGRES_DB}
      - LOG_LEVEL=info
    secrets:
      - postgres_password
      - api_private_key
    deploy:
      replicas: 3
      resources:
        limits:
          memory: 1G
          cpus: '0.5'
        reservations:
          memory: 512M
          cpus: '0.25'
      update_config:
        parallelism: 1
        delay: 10s
        order: start-first

secrets:
  postgres_password:
    external: true
  api_private_key:
    external: true

volumes:
  postgres_prod:
    external: true

networks:
  default:
    external: true
    name: iserve-prod
```

### Running with Docker Compose

```bash
# Development environment
docker-compose -f docker-compose.dev.yml up -d

# Production environment
docker-compose -f docker-compose.prod.yml up -d

# View logs
docker-compose logs -f api

# Stop services
docker-compose down

# Rebuild and restart
docker-compose up --build -d
```

## Security Best Practices

### Image Security

```dockerfile
# Use specific tags, not 'latest'
FROM node:18.17.0-alpine

# Scan for vulnerabilities
RUN apk add --no-cache --upgrade apk-tools
RUN apk audit --no-cache

# Create non-root user
RUN addgroup -g 1001 -S appuser && \
    adduser -S appuser -u 1001 -G appuser

# Don't run as root
USER appuser

# Set read-only filesystem
# In docker run: --read-only --tmpfs /tmp
```

### Runtime Security

```bash
# Run with security options
docker run \
  --security-opt=no-new-privileges:true \
  --cap-drop=ALL \
  --cap-add=NET_BIND_SERVICE \
  --read-only \
  --tmpfs /tmp \
  --user 1001:1001 \
  iserve/api:latest

# Use secrets for sensitive data
docker secret create postgres_password password.txt
docker service create \
  --secret postgres_password \
  postgres:15-alpine
```

### Network Security

```yaml
# docker-compose.yml with network isolation
version: '3.8'

services:
  api:
    networks:
      - backend
      - frontend

  postgres:
    networks:
      - backend

  dapp:
    networks:
      - frontend

networks:
  backend:
    driver: bridge
    internal: true
  frontend:
    driver: bridge
```

## Production Deployment

### Container Registry

```bash
# Tag images for production
docker tag iserve/api:latest registry.iserveprotocol.com/api:v1.0.0

# Push to registry
docker push registry.iserveprotocol.com/api:v1.0.0

# Pull in production
docker pull registry.iserveprotocol.com/api:v1.0.0
```

### Environment Configuration

```bash
# Production environment file
cat > .env.prod <<EOF
POSTGRES_DB=iserve_prod
POSTGRES_USER=iserve_user
IMAGE_TAG=v1.0.0
LOG_LEVEL=info
METRICS_ENABLED=true
EOF
```

### Health Monitoring

```yaml
# Add comprehensive health checks
services:
  api:
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
```

## Troubleshooting

### Common Issues

```bash
# Check container logs
docker logs iserve-api

# Debug container
docker exec -it iserve-api sh

# Check resource usage
docker stats

# Inspect container
docker inspect iserve-api

# Check networks
docker network ls
docker network inspect iserve-dev
```

### Performance Tuning

```yaml
# Optimize container resources
deploy:
  resources:
    limits:
      memory: 1G
      cpus: '0.5'
    reservations:
      memory: 512M
      cpus: '0.25'
```

### Log Management

```yaml
# Configure logging
logging:
  driver: "json-file"
  options:
    max-size: "10m"
    max-file: "3"
```

---

*This Docker guide follows security best practices and production deployment standards.*