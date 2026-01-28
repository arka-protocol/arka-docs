# Installation Guide

This guide covers all installation methods for ARKA Engine, from local development to production deployments.

## Table of Contents

- [System Requirements](#system-requirements)
- [Installation Methods](#installation-methods)
  - [Docker Compose (Recommended)](#docker-compose-recommended)
  - [Kubernetes/Helm](#kuberneteshelm)
  - [Binary Installation](#binary-installation)
  - [Building from Source](#building-from-source)
- [Configuration](#configuration)
- [Verification](#verification)
- [Upgrading](#upgrading)

---

## System Requirements

### Minimum Requirements

| Component | Requirement |
|-----------|-------------|
| CPU | 4 cores |
| RAM | 8 GB |
| Storage | 50 GB SSD |
| OS | Linux (Ubuntu 20.04+), macOS 12+, Windows Server 2019+ |

### Recommended Production Requirements

| Component | Requirement |
|-----------|-------------|
| CPU | 16+ cores |
| RAM | 64 GB |
| Storage | 500 GB NVMe SSD |
| Network | 1 Gbps |
| OS | Linux (Ubuntu 22.04 LTS) |

### Software Dependencies

| Software | Version | Required |
|----------|---------|----------|
| Docker | 20.10+ | For containerized deployment |
| Docker Compose | 2.0+ | For local development |
| Kubernetes | 1.25+ | For K8s deployment |
| Helm | 3.10+ | For K8s deployment |
| PostgreSQL | 14+ | Database |
| Redis | 7.0+ | Cache and message broker |

---

## Installation Methods

### Docker Compose (Recommended)

Best for: Local development, testing, small deployments

#### 1. Prerequisites

```bash
# Install Docker
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER

# Install Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# Verify installation
docker --version
docker-compose --version
```

#### 2. Download ARKA Engine

```bash
# Clone repository
git clone https://github.com/arka-engine/pact.git
cd pact

# Or download release archive
curl -LO https://github.com/arka-engine/pact/releases/latest/downlo../arka-1.0.0.tar.gz
tar xzf arka-1.0.0.tar.gz
cd arka-1.0.0
```

#### 3. Configure Environment

```bash
# Copy example environment file
cp .env.example .env

# Edit configuration
nano .env
```

Essential environment variables:

```bash
# .env file
ARKA_ENVIRONMENT=development
ARKA_LOG_LEVEL=info

# Database
POSTGRES_HOST=arka-postgres
POSTGRES_PORT=5432
POSTGRES_DB=pact
POSTGRES_USER=pact
POSTGRES_PASSWORD=your-secure-password

# Redis
REDIS_HOST=arka-redis
REDIS_PORT=6379

# API
ARKA_API_PORT=8080
ARKA_API_SECRET=your-api-secret-key

# RTVM
RTVM_PORT=8081
RTVM_WORKERS=4
```

#### 4. Start Services

```bash
# Start all services
docker-compose up -d

# View logs
docker-compose logs -f

# Check status
docker-compose ps
```

#### 5. Initialize Database

```bash
# Run database migrations
docker-compose exec arka-core arka-cli db migrate

# Seed initial data (optional)
docker-compose exec arka-core arka-cli db seed
```

---

### Kubernetes/Helm

Best for: Production deployments, high availability, auto-scaling

#### 1. Prerequisites

```bash
# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Install Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Verify
kubectl version --client
helm version
```

#### 2. Add ARKA Helm Repository

```bash
# Add repository
helm repo add pact https://charts.arka-engine.io
helm repo update

# Search available charts
helm search repo pact
```

#### 3. Create Namespace

```bash
kubectl create namespace arka-system
```

#### 4. Configure Values

Create a `values.yaml` file:

```yaml
# values.yaml
global:
  environment: production
  imageTag: "1.0.0"

core:
  replicas: 3
  resources:
    requests:
      cpu: "2"
      memory: "4Gi"
    limits:
      cpu: "4"
      memory: "8Gi"

rtvm:
  replicas: 5
  resources:
    requests:
      cpu: "4"
      memory: "8Gi"
    limits:
      cpu: "8"
      memory: "16Gi"

postgresql:
  enabled: true
  auth:
    postgresPassword: "your-secure-password"
    database: pact
  primary:
    persistence:
      size: 100Gi
  readReplicas:
    replicaCount: 2

redis:
  enabled: true
  architecture: replication
  auth:
    enabled: true
    password: "your-redis-password"

ingress:
  enabled: true
  className: nginx
  hosts:
    - host: pact.your-domain.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: arka-tls
      hosts:
        - pact.your-domain.com

monitoring:
  enabled: true
  prometheus:
    enabled: true
  grafana:
    enabled: true
```

#### 5. Install ARKA Engine

```bash
# Install with custom values
helm install pact pact/arka-engine \
  --namespace arka-system \
  --values values.yaml \
  --wait

# Verify deployment
kubectl get pods -n arka-system
kubectl get services -n arka-system
```

#### 6. Verify Installation

```bash
# Port forward for testing
kubectl port-forward svc/arka-core 8080:8080 -n arka-system

# Test health endpoint
curl http://localhost:8080/health
```

---

### Binary Installation

Best for: Air-gapped environments, custom deployments

#### 1. Download Binaries

```bash
# Download for Linux AMD64
curl -LO https://github.com/arka-engine/pact/releases/latest/downlo../arka-linux-amd64.tar.gz
tar xzf arka-linux-amd64.tar.gz

# Download for Linux ARM64
curl -LO https://github.com/arka-engine/pact/releases/latest/downlo../arka-linux-arm64.tar.gz

# Download for macOS
curl -LO https://github.com/arka-engine/pact/releases/latest/downlo../arka-darwin-amd64.tar.gz
```

#### 2. Install Binaries

```bash
# Install binaries
sudo mv arka-core /usr/local/bin/
sudo mv arka-rtvm /usr/local/bin/
sudo mv arka-cli /usr/local/bin/

# Verify installation
arka-cli version
```

#### 3. Create Configuration

```bash
# Create config directory
sudo mkdir -p /etc/pact

# Create configuration file
sudo tee /etc/pact/config.yaml << EOF
environment: production
log_level: info

database:
  host: localhost
  port: 5432
  name: pact
  user: pact
  password_file: /etc/pact/secrets/db-password

redis:
  host: localhost
  port: 6379
  password_file: /etc/pact/secrets/redis-password

api:
  host: 0.0.0.0
  port: 8080
  tls:
    enabled: true
    cert_file: /etc/pact/certs/server.crt
    key_file: /etc/pact/certs/server.key

rtvm:
  host: 0.0.0.0
  port: 8081
  workers: 8
EOF
```

#### 4. Create Systemd Services

```bash
# ARKA Core service
sudo tee /etc/systemd/system/arka-core.service << EOF
[Unit]
Description=ARKA Engine Core Service
After=network.target postgresql.service redis.service

[Service]
Type=simple
User=pact
Group=pact
ExecStart=/usr/local/bin/arka-core --config /etc/pact/config.yaml
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

# ARKA RTVM service
sudo tee /etc/systemd/syst../arka-rtvm.service << EOF
[Unit]
Description=ARKA Engine RTVM Service
After=network.target arka-core.service

[Service]
Type=simple
User=pact
Group=pact
ExecStart=/usr/local/b../arka-rtvm --config /etc/pact/config.yaml
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

# Enable and start services
sudo systemctl daemon-reload
sudo systemctl enable arka-core arka-rtvm
sudo systemctl start arka-core arka-rtvm
```

---

### Building from Source

Best for: Development, customization

#### 1. Prerequisites

```bash
# Install Go 1.21+
curl -LO https://go.dev/dl/go1.21.5.linux-amd64.tar.gz
sudo tar -C /usr/local -xzf go1.21.5.linux-amd64.tar.gz
export PATH=$PATH:/usr/local/go/bin

# Install Node.js 20+ (for dashboard)
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs

# Install Rust (for RTVM)
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source ~/.cargo/env
```

#### 2. Clone Repository

```bash
git clone https://github.com/arka-engine/pact.git
cd pact
```

#### 3. Build Components

```bash
# Build all components
make build

# Or build individually
make build-core      # Go-based core service
make build-rtvm      # Rust-based RTVM
make build-cli       # CLI tool
make build-dashboard # React dashboard

# Run tests
make test

# Run linting
make lint
```

#### 4. Install

```bash
# Install to /usr/local/bin
sudo make install

# Or install to custom location
make install PREFIX=/opt/pact
```

---

## Configuration

### Configuration Files

ARKA Engine uses a hierarchical configuration system:

1. **Default configuration** - Built-in defaults
2. **Configuration file** - `/etc/pact/config.yaml`
3. **Environment variables** - `ARKA_*` prefix
4. **Command-line flags** - Highest priority

### Configuration Reference

See [Configuration Reference](../deployment/configuration.md) for complete documentation.

### Secrets Management

```bash
# Using environment variables
export ARKA_DATABASE_PASSWORD=secret

# Using files
echo "secret" > /etc/pact/secrets/db-password
chmod 600 /etc/pact/secrets/db-password

# Using external secret managers (Vault, AWS Secrets Manager)
# See deployment guide for integration details
```

---

## Verification

### Health Checks

```bash
# Core service health
curl http://localhost:8080/health

# RTVM health
curl http://localhost:8081/health

# Detailed component status
curl http://localhost:8080/health/detailed
```

### Smoke Tests

```bash
# Run built-in smoke tests
arka-cli test smoke

# Expected output:
# ✓ Database connection
# ✓ Redis connection
# ✓ RTVM communication
# ✓ Rule compilation
# ✓ Rule evaluation
# ✓ Proof generation
# All smoke tests passed!
```

### Version Verification

```bash
# Check versions
arka-cli version

# Output:
# ARKA CLI: 1.0.0
# ARKA Core: 1.0.0
# ARKA RTVM: 1.0.0
# Go: 1.21.5
# Rust: 1.74.0
```

---

## Upgrading

### Docker Compose

```bash
# Pull latest images
docker-compose pull

# Stop services
docker-compose down

# Backup database
docker-compose exec arka-postgres pg_dump -U pact pact > backup.sql

# Start with new version
docker-compose up -d

# Run migrations
docker-compose exec arka-core arka-cli db migrate
```

### Kubernetes/Helm

```bash
# Update repository
helm repo update

# Check available versions
helm search repo pact --versions

# Upgrade
helm upgrade arka pact/arka-engine \
  --namespace arka-system \
  --values values.yaml \
  --wait
```

### Binary

```bash
# Stop services
sudo systemctl stop arka-core arka-rtvm

# Backup binaries
sudo mv /usr/local/b../arka-* /usr/local/b../arka-backup/

# Download new version
curl -LO https://github.com/arka-engine/pact/releases/latest/downlo../arka-linux-amd64.tar.gz
tar xzf arka-linux-amd64.tar.gz

# Install new binaries
sudo mv arka-* /usr/local/bin/

# Run migrations
arka-cli db migrate

# Start services
sudo systemctl start arka-core arka-rtvm
```

---

## Next Steps

- [Quick Start Guide](./quickstart.md) - Get started quickly
- [Concepts & Terminology](./concepts.md) - Understand ARKA concepts
- [System Overview](../architecture/system-overview.md) - Learn the architecture
