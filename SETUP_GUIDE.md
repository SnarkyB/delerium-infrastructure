# Setup Guide

Complete guide for setting up Delirium infrastructure.

## Quick Start

### Automated Setup (Recommended)

```bash
# Clone the repository
git clone https://github.com/marcusb333/delerium-infrastructure.git
cd delerium-infrastructure

# Run interactive setup wizard
./scripts/setup.sh
```

The setup wizard will:
1. ✅ Check prerequisites (Docker, Docker Compose)
2. ✅ Detect and resolve port conflicts
3. ✅ Generate secure secrets
4. ✅ Create configuration files
5. ✅ Build and deploy containers
6. ✅ Verify services are healthy

## Prerequisites

### System Requirements

- **Operating System**: Linux, macOS, or Windows with WSL2
- **RAM**: Minimum 2GB, recommended 4GB+
- **Disk Space**: Minimum 5GB free
- **Network**: Internet connection for pulling Docker images

### Required Software

1. **Docker** (20.10+)
   ```bash
   # Install Docker
   curl -fsSL https://get.docker.com -o get-docker.sh
   sudo sh get-docker.sh
   ```

2. **Docker Compose** (v2+)
   ```bash
   # Usually included with Docker Desktop
   # Or install separately:
   sudo apt install docker-compose-plugin
   ```

3. **Git**
   ```bash
   # Ubuntu/Debian
   sudo apt install git
   
   # macOS
   brew install git
   ```

### Optional Tools

- **curl**: For health checks
- **openssl**: For generating secrets
- **lsof**: For port conflict detection

## Setup Methods

### Method 1: Interactive Setup Wizard

**Best for**: First-time users, development environments

```bash
./scripts/setup.sh
```

**Features:**
- Guided step-by-step process
- Automatic port conflict detection
- Secret generation
- Service deployment
- Health verification

**Example Session:**

```
╔════════════════════════════════════════════════════════════╗
║                                                            ║
║      🔐 Delirium Setup - Zero-Knowledge Paste System      ║
║                                                            ║
╚════════════════════════════════════════════════════════════╝

▸ Checking prerequisites...
✓ Docker found and running: Docker version 24.0.6
✓ Docker Compose found: v2.23.0
✓ curl found
✓ git found

▸ Setting up environment configuration...
ℹ Creating .env file from template...
ℹ Generating secure DELETION_TOKEN_PEPPER...
✓ Generated secure pepper
✓ Environment configuration created

▸ Checking repository structure...
✓ Client repository found
✓ Server repository found

▸ Setting up Docker services...
✓ Created data and log directories

ℹ Select deployment type:
  1) Development (local, port 8080)
  2) Production (HTTPS, ports 80/443)
  3) Production without SSL (HTTP, port 80)

Enter choice [1-3]: 1

ℹ Using development configuration

ℹ Checking for port conflicts...
⚠ Port conflicts detected!

  Port 8080 is in use:
  com.docker 81298 marcusb  264u  IPv6  *:8080 (LISTEN)

ℹ The following Docker containers are using these ports:
  - delirium-server-old (0.0.0.0:8080->8080/tcp)

Would you like to stop these containers? [Y/n]: y

ℹ Stopping conflicting containers...
✓ Stopped container: delirium-server-old
✓ All port conflicts resolved

ℹ Pulling/building Docker images (this may take a few minutes)...
✓ Images pulled/checked

ℹ Starting services...
✓ Services started

▸ Waiting for services to be healthy...
✓ Services are healthy!

╔════════════════════════════════════════════════════════════╗
║                                                            ║
║                  ✓ Setup Complete!                        ║
║                                                            ║
╚════════════════════════════════════════════════════════════╝

🌐 Access Delirium:
   http://localhost:8081

📊 Useful Commands:
   docker compose logs -f    - View logs
   docker compose ps         - Service status
   ./scripts/health-check.sh - Health check
```

### Method 2: Quick Start (Automated)

**Best for**: CI/CD, automated deployments

```bash
# Headless mode (no prompts)
HEADLESS=1 ./scripts/quick-start.sh

# Or with Makefile
make quick-start-headless
```

### Method 3: Manual Setup

**Best for**: Custom configurations, advanced users

```bash
# 1. Create environment file
cp docker-compose/.env.example .env

# 2. Generate secrets
openssl rand -hex 32  # Use for DELETION_TOKEN_PEPPER

# 3. Edit configuration
nano .env

# 4. Build code
cd ../delerium-client && npm install && npm run build
cd ../delerium-server && ./gradlew clean build

# 5. Deploy
cd ../delerium-infrastructure/docker-compose
WEB_PORT=8081 docker compose -f docker-compose.yml -f docker-compose.dev.yml up -d --build
```

## Configuration

### Environment Variables

Create a `.env` file in the root directory:

```bash
# Required: Secret pepper for deletion token hashing
# Generate with: openssl rand -hex 32
DELETION_TOKEN_PEPPER=your-64-character-hex-string-here

# Optional: Logging level
LOG_LEVEL=INFO

# Optional: Port configuration
WEB_PORT=8080

# Optional: GitHub username
GITHUB_USERNAME=your-username
```

### Port Configuration

#### Development Mode
- **Backend**: 8080 (direct API access)
- **Frontend**: 8081 (Nginx)

#### Production Mode
- **HTTP**: 80
- **HTTPS**: 443 (with SSL)

#### Custom Ports

Set the `WEB_PORT` environment variable:

```bash
# Use port 3000
WEB_PORT=3000 docker compose up -d

# Or in .env file
echo "WEB_PORT=3000" >> .env
```

## Port Conflict Resolution

The setup script automatically detects and resolves port conflicts.

### Automatic Resolution

When running `./scripts/setup.sh`, the script will:

1. **Detect** processes using required ports
2. **Identify** Docker containers blocking ports
3. **Prompt** to stop conflicting containers
4. **Verify** ports are freed before continuing

### Manual Resolution

If you encounter port conflicts:

```bash
# Find what's using a port
lsof -i :8080

# Stop a specific container
docker stop container-name

# Stop all containers
docker stop $(docker ps -q)

# Use a different port
WEB_PORT=8081 docker compose up -d
```

## Deployment Modes

### Development

```bash
cd docker-compose
WEB_PORT=8081 docker compose -f docker-compose.yml -f docker-compose.dev.yml up -d
```

**Access:**
- Frontend: http://localhost:8081
- Backend: http://localhost:8080

### Production (HTTP)

```bash
cd docker-compose
docker compose -f docker-compose.yml up -d
```

**Access:**
- Application: http://your-domain.com

### Production (HTTPS)

```bash
# Setup SSL first
./scripts/setup-ssl.sh

# Deploy
cd docker-compose
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

**Access:**
- Application: https://your-domain.com

## Verification

### Health Checks

```bash
# Run health check script
./scripts/health-check.sh

# Manual checks
curl http://localhost:8080/api/pow  # Backend
curl http://localhost:8081/         # Frontend
```

### View Logs

```bash
# All services
docker compose logs -f

# Specific service
docker compose logs -f server
docker compose logs -f web

# Last 100 lines
docker compose logs --tail=100
```

### Container Status

```bash
# List running containers
docker ps

# Detailed status
docker compose ps

# Resource usage
docker stats
```

## Troubleshooting

### Port Already in Use

**Error:** `Bind for 0.0.0.0:8080 failed: port is already allocated`

**Solution:**
```bash
# Option 1: Use setup script (handles automatically)
./scripts/setup.sh

# Option 2: Stop conflicting containers
docker stop $(docker ps -q)

# Option 3: Use different port
WEB_PORT=8081 docker compose up -d
```

### Docker Daemon Not Running

**Error:** `Cannot connect to the Docker daemon`

**Solution:**
```bash
# macOS/Windows
open -a Docker

# Linux
sudo systemctl start docker

# Verify
docker ps
```

### Build Failures

**Error:** Build fails or takes too long

**Solution:**
```bash
# Clean cache
docker system prune -a

# Rebuild without cache
docker compose build --no-cache

# Check disk space
df -h
```

### Services Not Healthy

**Error:** Health checks failing

**Solution:**
```bash
# Check logs
docker compose logs

# Restart services
docker compose restart

# Full rebuild
docker compose down
docker compose up -d --build
```

### Permission Errors

**Error:** Permission denied on scripts

**Solution:**
```bash
# Make scripts executable
chmod +x scripts/*.sh

# Or run with bash
bash scripts/setup.sh
```

## Advanced Configuration

### Custom Nginx Configuration

Edit `nginx/dev.conf` or `nginx/nginx.conf`:

```nginx
server {
    listen 80;
    server_name localhost;
    
    # Custom settings
    client_max_body_size 10M;
    
    location / {
        root /usr/share/nginx/html;
        try_files $uri $uri/ /index.html;
    }
    
    location /api {
        proxy_pass http://server:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

### Resource Limits

Edit `docker-compose.yml`:

```yaml
services:
  server:
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 1G
        reservations:
          cpus: '1.0'
          memory: 512M
```

### Volume Configuration

```yaml
volumes:
  server-data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /path/to/data
```

## Security Best Practices

### Secrets Management

- ✅ Never commit `.env` to version control
- ✅ Use strong random secrets (32+ characters)
- ✅ Rotate secrets periodically (every 90 days)
- ✅ Store secrets in a password manager
- ✅ Use environment-specific secrets

### Production Hardening

```bash
# Run security setup
./scripts/security-setup.sh

# This configures:
# - Firewall rules
# - Security headers
# - SSL/TLS
# - Rate limiting
# - Fail2ban
```

### SSL/TLS

```bash
# Generate certificates
./scripts/setup-ssl.sh

# Or use existing certificates
mkdir -p ssl/certs ssl/private
cp cert.pem ssl/certs/
cp key.pem ssl/private/
chmod 600 ssl/private/key.pem
```

## Maintenance

### Updates

```bash
# Pull latest code
git pull origin main

# Rebuild and redeploy
docker compose down
docker compose up -d --build
```

### Backups

```bash
# Run backup script
./scripts/backup.sh

# Manual backup
tar -czf backup-$(date +%Y%m%d).tar.gz data/ .env
```

### Monitoring

```bash
# Start monitoring
./scripts/monitor.sh

# View metrics
docker stats

# Check logs
docker compose logs -f
```

## Next Steps

After successful setup:

1. **Read the documentation**: [README.md](README.md)
2. **Configure security**: [DEPLOYMENT.md](DEPLOYMENT.md#security)
3. **Set up backups**: `./scripts/backup.sh`
4. **Enable monitoring**: `./scripts/monitor.sh`
5. **Review security checklist**: Run `./scripts/security-check.sh`

## Getting Help

- **Documentation**: [GitHub Wiki](https://github.com/marcusb333/delerium/wiki)
- **Issues**: [GitHub Issues](https://github.com/marcusb333/delerium-infrastructure/issues)
- **Discussions**: [GitHub Discussions](https://github.com/marcusb333/delerium/discussions)

---

Last updated: 2025-11-18
