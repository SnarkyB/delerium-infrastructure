# Delirium Infrastructure

> Docker-based infrastructure for deploying the Delirium zero-knowledge paste system

[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Docker](https://img.shields.io/badge/docker-ready-blue.svg)](https://www.docker.com/)
[![Compose](https://img.shields.io/badge/compose-v2-blue.svg)](https://docs.docker.com/compose/)

## 📋 Table of Contents

- [Overview](#overview)
- [Quick Start](#quick-start)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Configuration](#configuration)
- [Deployment Modes](#deployment-modes)
- [Port Configuration](#port-configuration)
- [Scripts Reference](#scripts-reference)
- [Troubleshooting](#troubleshooting)
- [Contributing](#contributing)

## 🎯 Overview

This repository contains the Docker infrastructure for deploying Delirium, a zero-knowledge encrypted paste service. It provides:

- **Multi-environment support**: Development, Production, and Production with SSL
- **Automated setup**: Interactive wizard with port conflict detection
- **Security-first**: Automatic secret generation and validation
- **Health monitoring**: Built-in health checks and monitoring scripts
- **Easy deployment**: One-command deployment with Docker Compose

## 🚀 Quick Start

### Option 1: Interactive Setup (Recommended)

```bash
# Clone the repository
git clone https://github.com/marcusb333/delerium-infrastructure.git
cd delerium-infrastructure

# Run the interactive setup wizard
./scripts/setup.sh
```

The setup wizard will:
- ✅ Check prerequisites (Docker, Docker Compose)
- ✅ Detect and resolve port conflicts
- ✅ Generate secure secrets automatically
- ✅ Create `.env` configuration file
- ✅ Build and deploy containers
- ✅ Verify services are healthy

### Option 2: Quick Start (Automated)

```bash
# For headless/automated environments
make quick-start-headless

# Or with the Makefile
make quick-start
```

### Option 3: Manual Setup

```bash
# 1. Create environment file
cp docker-compose/.env.example .env

# 2. Generate secure secrets
openssl rand -hex 32  # Use this for DELETION_TOKEN_PEPPER

# 3. Edit .env file with your secrets
nano .env

# 4. Build and deploy
cd docker-compose
docker compose -f docker-compose.yml -f docker-compose.dev.yml up -d --build
```

## 📦 Prerequisites

### Required

- **Docker** 20.10+ ([Install Docker](https://docs.docker.com/get-docker/))
- **Docker Compose** v2+ (included with Docker Desktop)
- **Git** (for cloning repositories)

### Optional

- **curl** (for health checks)
- **openssl** (for generating secrets)
- **lsof** (for port conflict detection)

### Verify Prerequisites

```bash
docker --version
docker compose version
git --version
```

## 🔧 Installation

### 1. Clone Required Repositories

The infrastructure requires the client and server repositories:

```bash
# Create project directory
mkdir -p ~/delerium-dev
cd ~/delerium-dev

# Clone all repositories
git clone https://github.com/marcusb333/delerium-infrastructure.git
git clone https://github.com/marcusb333/delerium-client.git
git clone https://github.com/marcusb333/delerium-server.git
```

Expected directory structure:
```
delerium-dev/
├── delerium-infrastructure/  (this repo)
├── delerium-client/           (frontend)
└── delerium-server/           (backend)
```

### 2. Build Local Code

```bash
# Build client (TypeScript)
cd delerium-client
npm install
npm run build

# Build server (Kotlin/Gradle)
cd ../delerium-server
./gradlew clean build

# Return to infrastructure
cd ../delerium-infrastructure
```

### 3. Deploy with Docker

```bash
# Run the setup script
./scripts/setup.sh

# Or manually
cd docker-compose
WEB_PORT=8081 docker compose -f docker-compose.yml -f docker-compose.dev.yml up -d --build
```

## ⚙️ Configuration

### Environment Variables

Create a `.env` file in the root directory:

```bash
# Required: Secret pepper for deletion token hashing
DELETION_TOKEN_PEPPER=your-secure-random-string-here

# Optional: Logging configuration
LOG_LEVEL=INFO

# Optional: Port configuration
WEB_PORT=8080

# Optional: GitHub username for repository cloning
GITHUB_USERNAME=your-username
```

### Generate Secure Secrets

```bash
# Generate a secure random pepper (32 bytes = 64 hex characters)
openssl rand -hex 32

# Or use the setup script (recommended)
./scripts/setup.sh
```

### Security Best Practices

- ✅ **Never commit `.env` to version control** (already in `.gitignore`)
- ✅ **Use strong random secrets** (minimum 32 characters)
- ✅ **Rotate secrets periodically** (every 90 days recommended)
- ✅ **Store secrets securely** (use a password manager)
- ✅ **Use HTTPS in production** (setup SSL with `./scripts/setup-ssl.sh`)

## 🌍 Deployment Modes

### Development Mode

For local development with hot-reload and debugging:

```bash
cd docker-compose
WEB_PORT=8081 docker compose -f docker-compose.yml -f docker-compose.dev.yml up -d
```

**Features:**
- Port 8080: Backend API (direct access)
- Port 8081: Frontend (Nginx)
- Debug logging enabled
- Source code mounted for live updates
- No resource limits

**Access:**
- Frontend: http://localhost:8081
- Backend API: http://localhost:8080

### Production Mode (HTTP)

For production deployment without SSL:

```bash
cd docker-compose
docker compose -f docker-compose.yml up -d
```

**Features:**
- Port 80: Frontend (Nginx)
- Backend not exposed directly
- Production logging
- Resource limits enforced
- Optimized for performance

**Access:**
- Application: http://your-domain.com

### Production Mode (HTTPS)

For production deployment with SSL certificates:

```bash
# Generate SSL certificates first
./scripts/setup-ssl.sh

# Deploy with SSL
cd docker-compose
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

**Features:**
- Port 80: HTTP (redirects to HTTPS)
- Port 443: HTTPS (SSL/TLS)
- Automatic certificate renewal
- Security headers enabled
- HSTS enabled

**Access:**
- Application: https://your-domain.com

## 🔌 Port Configuration

### Default Ports

| Mode | Service | Port | Description |
|------|---------|------|-------------|
| Dev | Backend | 8080 | Direct API access for debugging |
| Dev | Frontend | 8081 | Nginx serving client files |
| Prod | HTTP | 80 | Public web access |
| Prod | HTTPS | 443 | Secure web access |

### Port Conflict Resolution

The setup script automatically detects and resolves port conflicts:

1. **Detects processes** using required ports
2. **Identifies Docker containers** blocking ports
3. **Prompts to stop** conflicting containers
4. **Verifies ports are freed** before continuing

Manual port conflict resolution:

```bash
# Find what's using a port
lsof -i :8080

# Stop a specific Docker container
docker stop container-name

# Stop all Docker containers
docker stop $(docker ps -q)
```

### Custom Ports

To use custom ports, set the `WEB_PORT` environment variable:

```bash
# Use port 3000 instead of 8080
WEB_PORT=3000 docker compose -f docker-compose.yml -f docker-compose.dev.yml up -d
```

Or add to your `.env` file:
```bash
WEB_PORT=3000
```

## 📜 Scripts Reference

### Setup & Deployment

| Script | Description | Usage |
|--------|-------------|-------|
| `setup.sh` | Interactive setup wizard | `./scripts/setup.sh` |
| `quick-start.sh` | Automated quick start | `./scripts/quick-start.sh` |
| `deploy.sh` | Production deployment | `./scripts/deploy.sh` |
| `dev.sh` | Development mode | `./scripts/dev.sh` |

### Security

| Script | Description | Usage |
|--------|-------------|-------|
| `security-setup.sh` | Security hardening | `./scripts/security-setup.sh` |
| `security-check.sh` | Security verification | `./scripts/security-check.sh` |
| `security-scan.sh` | Vulnerability scanning | `./scripts/security-scan.sh` |
| `setup-ssl.sh` | SSL certificate setup | `./scripts/setup-ssl.sh` |

### Maintenance

| Script | Description | Usage |
|--------|-------------|-------|
| `health-check.sh` | Service health check | `./scripts/health-check.sh` |
| `monitor.sh` | Service monitoring | `./scripts/monitor.sh` |
| `backup.sh` | Data backup | `./scripts/backup.sh` |

### CI/CD

| Script | Description | Usage |
|--------|-------------|-------|
| `ci-verify-all.sh` | Full CI verification | `./scripts/ci-verify-all.sh` |
| `ci-verify-backend.sh` | Backend tests | `./scripts/ci-verify-backend.sh` |
| `ci-verify-frontend.sh` | Frontend tests | `./scripts/ci-verify-frontend.sh` |
| `pre-pr-check.sh` | Pre-PR validation | `./scripts/pre-pr-check.sh` |

### Makefile Targets

```bash
make help              # Show all available commands
make setup             # Interactive setup wizard
make start             # Build and start services
make stop              # Stop all services
make restart           # Restart services
make logs              # Follow service logs
make dev               # Start development mode
make clean             # Clean up everything
make health-check      # Check service health
make security-check    # Run security checks
make backup            # Create data backup
```

## 🔍 Troubleshooting

### Port Already in Use

**Problem:** Error: "Bind for 0.0.0.0:8080 failed: port is already allocated"

**Solution:**
```bash
# Option 1: Use the setup script (handles this automatically)
./scripts/setup.sh

# Option 2: Manually stop conflicting containers
docker ps  # Find the container
docker stop container-name

# Option 3: Use a different port
WEB_PORT=8081 docker compose up -d
```

### Services Not Healthy

**Problem:** Health checks failing or services not responding

**Solution:**
```bash
# Check container status
docker ps

# View logs
docker compose logs -f

# Restart services
docker compose restart

# Full rebuild
docker compose down
docker compose up -d --build
```

### Docker Daemon Not Running

**Problem:** "Cannot connect to the Docker daemon"

**Solution:**
```bash
# macOS/Windows: Start Docker Desktop
open -a Docker

# Linux: Start Docker service
sudo systemctl start docker

# Verify Docker is running
docker ps
```

### Permission Denied Errors

**Problem:** Permission errors when running scripts

**Solution:**
```bash
# Make scripts executable
chmod +x scripts/*.sh

# Or run with bash
bash scripts/setup.sh
```

### Build Failures

**Problem:** Docker build fails or takes too long

**Solution:**
```bash
# Clean Docker cache
docker system prune -a

# Rebuild without cache
docker compose build --no-cache

# Check disk space
df -h
```

### Missing Dependencies

**Problem:** npm or Gradle build failures

**Solution:**
```bash
# Client dependencies
cd delerium-client
rm -rf node_modules package-lock.json
npm install

# Server dependencies
cd delerium-server
./gradlew clean build --refresh-dependencies
```

## 📊 Monitoring & Logs

### View Logs

```bash
# All services
docker compose logs -f

# Specific service
docker compose logs -f server
docker compose logs -f web

# Last 100 lines
docker compose logs --tail=100

# With timestamps
docker compose logs -f -t
```

### Container Stats

```bash
# Real-time resource usage
docker stats

# Container status
docker ps

# Detailed inspection
docker inspect container-name
```

### Health Checks

```bash
# Run health check script
./scripts/health-check.sh

# Manual health check
curl http://localhost:8080/api/pow
curl http://localhost:8081/
```

## 🛡️ Security

### Security Checklist

- [ ] Generated secure random secrets (32+ characters)
- [ ] Configured `.env` file with proper values
- [ ] Verified `.env` is in `.gitignore`
- [ ] Enabled HTTPS for production (if applicable)
- [ ] Set up firewall rules (production)
- [ ] Configured automatic backups
- [ ] Enabled security headers (production)
- [ ] Set up monitoring and alerts

### Run Security Checks

```bash
# Full security scan
./scripts/security-scan.sh

# Security verification
./scripts/security-check.sh

# Apply security hardening
./scripts/security-setup.sh
```

## 🤝 Contributing

Contributions are welcome! Please see the main [Delirium repository](https://github.com/marcusb333/delerium) for contribution guidelines.

### Development Workflow

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Run tests: `make test`
5. Submit a pull request

## 📝 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## 🔗 Related Repositories

- [delerium](https://github.com/marcusb333/delerium) - Main repository
- [delerium-client](https://github.com/marcusb333/delerium-client) - Frontend (TypeScript)
- [delerium-server](https://github.com/marcusb333/delerium-server) - Backend (Kotlin)

## 📧 Support

- **Issues**: [GitHub Issues](https://github.com/marcusb333/delerium-infrastructure/issues)
- **Documentation**: [Wiki](https://github.com/marcusb333/delerium/wiki)
- **Discussions**: [GitHub Discussions](https://github.com/marcusb333/delerium/discussions)

---

Made with ❤️ by the Delirium team
