# Deployment Guide

Complete guide for deploying Delirium in various environments.

## Table of Contents

- [Local Development](#local-development)
- [Production Deployment](#production-deployment)
- [VPS Deployment](#vps-deployment)
- [Docker Hub Deployment](#docker-hub-deployment)
- [Kubernetes Deployment](#kubernetes-deployment)
- [Environment Variables](#environment-variables)
- [SSL/TLS Configuration](#ssltls-configuration)
- [Backup & Recovery](#backup--recovery)

## 🖥️ Local Development

### Quick Setup

```bash
# 1. Clone repositories
mkdir -p ~/delerium-dev && cd ~/delerium-dev
git clone https://github.com/marcusb333/delerium-infrastructure.git
git clone https://github.com/marcusb333/delerium-client.git
git clone https://github.com/marcusb333/delerium-server.git

# 2. Run setup wizard
cd delerium-infrastructure
./scripts/setup.sh
```

### Manual Development Setup

```bash
# 1. Build client
cd delerium-client
npm install
npm run build

# 2. Build server
cd ../delerium-server
./gradlew clean build

# 3. Deploy with Docker
cd ../delerium-infrastructure/docker-compose
WEB_PORT=8081 docker compose -f docker-compose.yml -f docker-compose.dev.yml up -d --build
```

### Development Ports

- **Backend API**: http://localhost:8080
- **Frontend**: http://localhost:8081

### Hot Reload Development

```bash
# Terminal 1: Start backend in Docker
cd docker-compose
docker compose -f docker-compose.yml -f docker-compose.dev.yml up server

# Terminal 2: Watch client changes
cd delerium-client
npm run watch

# Terminal 3: Serve frontend
cd docker-compose
docker compose -f docker-compose.yml -f docker-compose.dev.yml up web
```

## 🚀 Production Deployment

### Prerequisites

- VPS or cloud server (Ubuntu 20.04+ recommended)
- Domain name pointed to your server
- SSH access to the server
- Docker and Docker Compose installed

### Step 1: Server Preparation

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Install Docker Compose
sudo apt install docker-compose-plugin

# Create deployment user
sudo useradd -m -s /bin/bash delirium
sudo usermod -aG docker delirium
```

### Step 2: Clone and Configure

```bash
# Switch to deployment user
sudo su - delirium

# Clone repository
git clone https://github.com/marcusb333/delerium-infrastructure.git
cd delerium-infrastructure

# Run setup
./scripts/setup.sh
# Select option 2 (Production)
# Enter your domain name
# Enter email for Let's Encrypt
```

### Step 3: SSL Certificate Setup

```bash
# Generate SSL certificates with Let's Encrypt
./scripts/setup-ssl.sh

# Or use existing certificates
mkdir -p ssl/certs ssl/private
cp /path/to/cert.pem ssl/certs/
cp /path/to/key.pem ssl/private/
```

### Step 4: Deploy

```bash
# Deploy with SSL
cd docker-compose
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d

# Check status
docker compose ps
docker compose logs -f
```

### Step 5: Verify Deployment

```bash
# Run health check
./scripts/health-check.sh

# Test endpoints
curl https://your-domain.com
curl https://your-domain.com/api/pow
```

## 🌐 VPS Deployment

### Automated VPS Setup

```bash
# From your local machine
./scripts/setup-vps-from-local.sh

# Follow prompts:
# - Enter VPS IP address
# - Enter SSH user
# - Enter domain name
# - Script will configure everything automatically
```

### Manual VPS Setup

```bash
# 1. SSH into VPS
ssh user@your-vps-ip

# 2. Install dependencies
sudo apt update
sudo apt install -y docker.io docker-compose git curl

# 3. Clone and setup
git clone https://github.com/marcusb333/delerium-infrastructure.git
cd delerium-infrastructure
HEADLESS=1 ./scripts/quick-start.sh

# 4. Configure firewall
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 22/tcp
sudo ufw enable
```

### VPS Security Hardening

```bash
# Run security setup
./scripts/security-setup.sh

# This configures:
# - Firewall rules
# - Fail2ban
# - Automatic security updates
# - Docker security options
# - Nginx security headers
```

## 🐳 Docker Hub Deployment

### Using Pre-built Images

```bash
# Pull images from Docker Hub
docker pull marcusb333/delerium-server:latest
docker pull nginx:1.27-alpine

# Create docker-compose.yml
cat > docker-compose.yml << 'EOF'
version: '3.8'
services:
  server:
    image: marcusb333/delerium-server:latest
    environment:
      - DELETION_TOKEN_PEPPER=${DELETION_TOKEN_PEPPER}
    ports:
      - "8080:8080"
    volumes:
      - server-data:/data
    restart: unless-stopped

  web:
    image: nginx:1.27-alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./client:/usr/share/nginx/html:ro
    depends_on:
      - server
    restart: unless-stopped

volumes:
  server-data:
EOF

# Deploy
docker compose up -d
```

### Building and Pushing Images

```bash
# Build images
docker build -t marcusb333/delerium-server:latest ../delerium-server
docker build -t marcusb333/delerium-client:latest ../delerium-client

# Login to Docker Hub
docker login

# Push images
docker push marcusb333/delerium-server:latest
docker push marcusb333/delerium-client:latest
```

## ☸️ Kubernetes Deployment

### Prerequisites

- Kubernetes cluster (1.19+)
- kubectl configured
- Helm 3+ (optional)

### Basic Kubernetes Deployment

```yaml
# delirium-deployment.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: delirium

---
apiVersion: v1
kind: Secret
metadata:
  name: delirium-secrets
  namespace: delirium
type: Opaque
stringData:
  deletion-token-pepper: "your-secure-random-string"

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: delirium-server
  namespace: delirium
spec:
  replicas: 2
  selector:
    matchLabels:
      app: delirium-server
  template:
    metadata:
      labels:
        app: delirium-server
    spec:
      containers:
      - name: server
        image: marcusb333/delerium-server:latest
        ports:
        - containerPort: 8080
        env:
        - name: DELETION_TOKEN_PEPPER
          valueFrom:
            secretKeyRef:
              name: delirium-secrets
              key: deletion-token-pepper
        volumeMounts:
        - name: data
          mountPath: /data
        livenessProbe:
          httpGet:
            path: /api/pow
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /api/pow
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: delirium-data

---
apiVersion: v1
kind: Service
metadata:
  name: delirium-server
  namespace: delirium
spec:
  selector:
    app: delirium-server
  ports:
  - port: 8080
    targetPort: 8080

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: delirium-web
  namespace: delirium
spec:
  replicas: 2
  selector:
    matchLabels:
      app: delirium-web
  template:
    metadata:
      labels:
        app: delirium-web
    spec:
      containers:
      - name: nginx
        image: nginx:1.27-alpine
        ports:
        - containerPort: 80
        volumeMounts:
        - name: nginx-config
          mountPath: /etc/nginx/nginx.conf
          subPath: nginx.conf
        - name: client-files
          mountPath: /usr/share/nginx/html
      volumes:
      - name: nginx-config
        configMap:
          name: nginx-config
      - name: client-files
        configMap:
          name: client-files

---
apiVersion: v1
kind: Service
metadata:
  name: delirium-web
  namespace: delirium
spec:
  type: LoadBalancer
  selector:
    app: delirium-web
  ports:
  - port: 80
    targetPort: 80

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: delirium-data
  namespace: delirium
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

### Deploy to Kubernetes

```bash
# Apply configuration
kubectl apply -f delirium-deployment.yaml

# Check status
kubectl get pods -n delirium
kubectl get services -n delirium

# View logs
kubectl logs -f deployment/delirium-server -n delirium

# Get external IP
kubectl get service delirium-web -n delirium
```

## 🔐 Environment Variables

### Required Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `DELETION_TOKEN_PEPPER` | Secret for hashing deletion tokens | `openssl rand -hex 32` |

### Optional Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `LOG_LEVEL` | Logging level | `INFO` |
| `SERVER_PORT` | Backend port | `8080` |
| `WEB_PORT` | Frontend port | `8080` |
| `DATA_DIR` | Data directory | `/data` |
| `GITHUB_USERNAME` | GitHub username | `marcusb333` |

### Production Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `DOMAIN` | Your domain name | `paste.example.com` |
| `LETSENCRYPT_EMAIL` | Email for SSL certs | `admin@example.com` |
| `CORS_ALLOW_ALL` | Allow all CORS | `false` |
| `DEV_MODE` | Development mode | `false` |

## 🔒 SSL/TLS Configuration

### Let's Encrypt (Automated)

```bash
# Run SSL setup script
./scripts/setup-ssl.sh

# Enter your domain and email
# Certificates will be generated automatically
```

### Manual Certificate Installation

```bash
# 1. Create SSL directories
mkdir -p ssl/certs ssl/private

# 2. Copy certificates
cp /path/to/fullchain.pem ssl/certs/cert.pem
cp /path/to/privkey.pem ssl/private/key.pem

# 3. Set permissions
chmod 644 ssl/certs/cert.pem
chmod 600 ssl/private/key.pem

# 4. Deploy with SSL
cd docker-compose
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

### Certificate Renewal

```bash
# Automatic renewal (Let's Encrypt)
# Certificates auto-renew via certbot

# Manual renewal
certbot renew

# Reload nginx
docker compose exec web nginx -s reload
```

## 💾 Backup & Recovery

### Automated Backups

```bash
# Run backup script
./scripts/backup.sh

# Backups are saved to: backups/backup-YYYY-MM-DD-HHMMSS.tar.gz
```

### Manual Backup

```bash
# Backup database and configuration
tar -czf backup-$(date +%Y%m%d).tar.gz \
  data/ \
  .env \
  docker-compose/

# Copy to remote server
scp backup-*.tar.gz user@backup-server:/backups/
```

### Restore from Backup

```bash
# 1. Stop services
docker compose down

# 2. Extract backup
tar -xzf backup-20250101.tar.gz

# 3. Restart services
docker compose up -d

# 4. Verify
./scripts/health-check.sh
```

### Scheduled Backups

```bash
# Add to crontab
crontab -e

# Daily backup at 2 AM
0 2 * * * /path/to/delerium-infrastructure/scripts/backup.sh

# Weekly backup on Sunday at 3 AM
0 3 * * 0 /path/to/delerium-infrastructure/scripts/backup.sh
```

## 📊 Monitoring

### Health Checks

```bash
# Manual health check
./scripts/health-check.sh

# Automated monitoring
./scripts/monitor.sh
```

### Log Monitoring

```bash
# View all logs
docker compose logs -f

# View specific service
docker compose logs -f server

# Search logs
docker compose logs | grep ERROR
```

### Resource Monitoring

```bash
# Container stats
docker stats

# Disk usage
docker system df

# Network usage
docker network inspect delirium-network
```

## 🔄 Updates & Maintenance

### Update Application

```bash
# Pull latest changes
git pull origin main

# Rebuild and redeploy
docker compose down
docker compose up -d --build

# Verify
./scripts/health-check.sh
```

### Update Docker Images

```bash
# Pull latest images
docker compose pull

# Restart with new images
docker compose up -d

# Clean old images
docker image prune -a
```

### Database Migration

```bash
# Backup before migration
./scripts/backup.sh

# Run migration
docker compose exec server ./migrate.sh

# Verify
docker compose logs server
```

## 🆘 Troubleshooting

### Common Issues

See [README.md#troubleshooting](README.md#troubleshooting) for detailed troubleshooting guide.

### Emergency Procedures

```bash
# Stop all services
docker compose down

# Remove all containers and volumes
docker compose down -v

# Clean Docker system
docker system prune -a --volumes

# Restore from backup
tar -xzf backup-latest.tar.gz
docker compose up -d
```

## 📞 Support

- **Documentation**: [GitHub Wiki](https://github.com/marcusb333/delerium/wiki)
- **Issues**: [GitHub Issues](https://github.com/marcusb333/delerium-infrastructure/issues)
- **Discussions**: [GitHub Discussions](https://github.com/marcusb333/delerium/discussions)

---

Last updated: 2025-11-18
