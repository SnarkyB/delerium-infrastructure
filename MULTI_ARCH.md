# Multi-Architecture Support Guide

This guide explains how to build, deploy, and use Delirium Docker images across multiple CPU architectures.

## 📋 Table of Contents

- [Supported Architectures](#supported-architectures)
- [Quick Start](#quick-start)
- [Building Multi-Arch Images](#building-multi-arch-images)
- [Using Pre-Built Images](#using-pre-built-images)
- [GitHub Actions CI/CD](#github-actions-cicd)
- [Platform-Specific Deployment](#platform-specific-deployment)
- [Troubleshooting](#troubleshooting)
- [Performance Considerations](#performance-considerations)

## 🏗️ Supported Architectures

Delirium supports the following CPU architectures:

| Architecture | Platform ID | Common Devices | Status |
|--------------|-------------|----------------|--------|
| **AMD64** | `linux/amd64` | Intel/AMD x86_64 processors, most cloud VMs | ✅ Fully Supported |
| **ARM64** | `linux/arm64` | Apple Silicon (M1/M2/M3), AWS Graviton, Raspberry Pi 4+ | ✅ Fully Supported |
| **ARMv7** | `linux/arm/v7` | Raspberry Pi 3, older ARM devices | ✅ Fully Supported |

## 🚀 Quick Start

### Using Pre-Built Multi-Arch Images

The easiest way to use Delirium is with our pre-built multi-architecture images. Docker will automatically pull the correct image for your platform:

```bash
# Pull from Docker Hub (automatically selects correct architecture)
docker pull marcusb333/delerium-server:latest

# Or from GitHub Container Registry
docker pull ghcr.io/marcusb333/delerium-server:latest

# Run on any architecture
docker run -d -p 8080:8080 \
  -e DELETION_TOKEN_PEPPER=your-secret-here \
  -v delirium-data:/data \
  marcusb333/delerium-server:latest
```

### Deploy with Docker Compose

The docker-compose configuration automatically detects and uses the correct architecture:

```bash
cd docker-compose
docker compose up -d
```

## 🔨 Building Multi-Arch Images

### Prerequisites

1. **Docker with Buildx support** (Docker 19.03+)
   ```bash
   docker buildx version
   ```

2. **QEMU for cross-platform emulation** (for building non-native architectures)
   ```bash
   docker run --rm --privileged multiarch/qemu-user-static --reset -p yes
   ```

### Method 1: Using the Build Script (Recommended)

The server repository includes a convenient build script:

```bash
cd delerium-server

# Build for all architectures (local only)
./docker-build.sh latest dockerhub your-username

# Build and push to Docker Hub
./docker-build.sh 1.0.0 dockerhub your-username "linux/amd64,linux/arm64,linux/arm/v7" push

# Build and push to GitHub Container Registry
./docker-build.sh 1.0.0 ghcr your-username "linux/amd64,linux/arm64" push

# Build for specific platforms only
./docker-build.sh latest dockerhub your-username "linux/amd64,linux/arm64"
```

**Script Parameters:**
- `version`: Image version tag (e.g., `1.0.0`, `latest`)
- `registry`: Target registry (`dockerhub` or `ghcr`)
- `username`: Your Docker Hub or GitHub username
- `platforms`: Comma-separated list of target platforms (optional)
- `push`: Add this flag to push to registry after building (optional)

### Method 2: Using Docker Buildx Directly

```bash
cd delerium-server

# Create a new builder instance (one-time setup)
docker buildx create --name multiarch-builder --driver docker-container --bootstrap --use

# Build for multiple architectures
docker buildx build \
  --platform linux/amd64,linux/arm64,linux/arm/v7 \
  --tag your-username/delerium-server:latest \
  --push \
  .

# Build without pushing (local only - single platform)
docker buildx build \
  --platform linux/amd64 \
  --tag your-username/delerium-server:latest \
  --load \
  .
```

**Note:** Multi-platform builds with `--load` are not supported. To test locally, build for your current platform only.

### Method 3: Using Docker Compose with Buildx

```bash
cd docker-compose

# Build for current platform
docker compose build

# Build for specific platform
docker compose build --build-arg BUILDPLATFORM=linux/arm64
```

## 📦 Using Pre-Built Images

### From Docker Hub

```bash
# Latest version
docker pull marcusb333/delerium-server:latest

# Specific version
docker pull marcusb333/delerium-server:v1.0.0

# Force specific architecture
docker pull --platform linux/arm64 marcusb333/delerium-server:latest
```

### From GitHub Container Registry

```bash
# Latest version
docker pull ghcr.io/marcusb333/delerium-server:latest

# Specific version
docker pull ghcr.io/marcusb333/delerium-server:v1.0.0

# Force specific architecture
docker pull --platform linux/amd64 ghcr.io/marcusb333/delerium-server:latest
```

### Verify Image Architecture

```bash
# Inspect image manifest
docker buildx imagetools inspect marcusb333/delerium-server:latest

# Check local image
docker image inspect marcusb333/delerium-server:latest | grep Architecture
```

## 🤖 GitHub Actions CI/CD

### Automated Multi-Arch Builds

The repository includes a GitHub Actions workflow that automatically builds and publishes multi-architecture images:

**Workflow File:** `.github/workflows/docker-multiarch.yml`

**Triggers:**
- Version tags (e.g., `v1.0.0`, `server-v1.0.0`)
- Pushes to `main` branch
- Manual workflow dispatch

**What it does:**
1. Sets up QEMU for cross-platform emulation
2. Configures Docker Buildx
3. Builds images for AMD64, ARM64, and ARMv7
4. Pushes to both Docker Hub and GHCR
5. Tests images on different architectures
6. Generates build summary

### Manual Workflow Trigger

You can manually trigger a multi-arch build from GitHub:

1. Go to **Actions** tab in your repository
2. Select **Build Multi-Arch Docker Images** workflow
3. Click **Run workflow**
4. Configure options:
   - **Version tag**: e.g., `v1.0.0` or `latest`
   - **Target platforms**: e.g., `linux/amd64,linux/arm64`
   - **Push to registry**: Check to publish images

### Required Secrets

Configure these secrets in your GitHub repository settings:

- `DOCKERHUB_USERNAME`: Your Docker Hub username
- `DOCKERHUB_TOKEN`: Docker Hub access token ([create one](https://hub.docker.com/settings/security))
- `GITHUB_TOKEN`: Automatically provided by GitHub Actions

## 🎯 Platform-Specific Deployment

### Force Specific Platform

Sometimes you may want to force a specific platform (e.g., for testing):

**Docker Run:**
```bash
docker run --platform linux/arm64 -d -p 8080:8080 \
  -e DELETION_TOKEN_PEPPER=your-secret \
  marcusb333/delerium-server:latest
```

**Docker Compose:**

Edit `docker-compose.yml` and uncomment the platform line:

```yaml
services:
  server:
    image: marcusb333/delerium-server:latest
    platform: linux/arm64  # Force ARM64
    # ... rest of config
```

### Platform-Specific Optimization

The Dockerfile is optimized for each architecture:

- **AMD64**: Uses native x86_64 JVM
- **ARM64**: Uses native AArch64 JVM (optimized for Apple Silicon, AWS Graviton)
- **ARMv7**: Uses 32-bit ARM JVM (compatible with older devices)

## 🔧 Troubleshooting

### Issue: "exec format error"

**Cause:** Trying to run an image built for a different architecture.

**Solution:**
```bash
# Check your system architecture
uname -m

# Pull the correct architecture explicitly
docker pull --platform linux/$(uname -m) marcusb333/delerium-server:latest

# Or let Docker auto-detect
docker pull marcusb333/delerium-server:latest
```

### Issue: Slow Build Times on Non-Native Architectures

**Cause:** Building for ARM on AMD64 (or vice versa) uses QEMU emulation, which is slower.

**Solution:**
- Use GitHub Actions for multi-arch builds (runs on native hardware)
- Build only for your target platform locally
- Use pre-built images from Docker Hub/GHCR

### Issue: "no matching manifest for linux/arm64/v8"

**Cause:** The image doesn't support your architecture.

**Solution:**
```bash
# Check available architectures
docker buildx imagetools inspect marcusb333/delerium-server:latest

# Use a different architecture
docker pull --platform linux/amd64 marcusb333/delerium-server:latest
```

### Issue: Buildx Not Available

**Cause:** Docker version is too old or buildx is not enabled.

**Solution:**
```bash
# Check Docker version (need 19.03+)
docker --version

# Enable buildx (if not enabled)
docker buildx install

# Update Docker to latest version
# macOS: Update Docker Desktop
# Linux: Follow https://docs.docker.com/engine/install/
```

### Issue: Cannot Push Multi-Arch Manifest

**Cause:** Not logged into registry or insufficient permissions.

**Solution:**
```bash
# Login to Docker Hub
docker login

# Login to GHCR
echo $GITHUB_TOKEN | docker login ghcr.io -u your-username --password-stdin

# Verify permissions
docker buildx imagetools inspect your-username/delerium-server:latest
```

## ⚡ Performance Considerations

### Build Performance

| Architecture | Build Method | Relative Speed | Notes |
|--------------|--------------|----------------|-------|
| Native | Direct build | 1x (baseline) | Fastest |
| Cross-compile | Buildx + QEMU | 2-5x slower | Varies by complexity |
| CI/CD | GitHub Actions | 1x per platform | Parallel builds |

**Recommendations:**
- Use GitHub Actions for production builds (parallel native builds)
- Build locally only for your target platform during development
- Use pre-built images for deployment

### Runtime Performance

All architectures run at native speed once deployed:

| Architecture | Performance | Notes |
|--------------|-------------|-------|
| AMD64 | Excellent | Mature JVM optimization |
| ARM64 | Excellent | Modern ARM CPUs, optimized JVM |
| ARMv7 | Good | Suitable for low-traffic deployments |

**Recommendations:**
- **Production (high traffic)**: AMD64 or ARM64
- **Edge/IoT**: ARM64 or ARMv7
- **Development**: Any architecture
- **Cost optimization**: ARM64 (AWS Graviton, cheaper than AMD64)

### Memory Usage

Memory usage is consistent across architectures:

- **Minimum**: 256MB RAM
- **Recommended**: 512MB RAM
- **High traffic**: 1GB+ RAM

Adjust in `docker-compose.yml`:

```yaml
deploy:
  resources:
    limits:
      memory: 512M
    reservations:
      memory: 256M
```

## 📚 Additional Resources

### Documentation
- [Docker Buildx Documentation](https://docs.docker.com/buildx/working-with-buildx/)
- [Multi-platform Images](https://docs.docker.com/build/building/multi-platform/)
- [QEMU User Emulation](https://www.qemu.org/docs/master/user/main.html)

### Delirium Documentation
- [Main README](README.md)
- [Deployment Guide](DEPLOYMENT.md)
- [Setup Guide](SETUP_GUIDE.md)

### Registry Links
- [Docker Hub Repository](https://hub.docker.com/r/marcusb333/delerium-server)
- [GitHub Container Registry](https://github.com/marcusb333/delerium-server/pkgs/container/delerium-server)

## 🤝 Contributing

Found an issue with multi-arch support? Please [open an issue](https://github.com/marcusb333/delerium-infrastructure/issues) or submit a pull request.

---

**Last Updated:** 2025-11-18  
**Supported Platforms:** linux/amd64, linux/arm64, linux/arm/v7
