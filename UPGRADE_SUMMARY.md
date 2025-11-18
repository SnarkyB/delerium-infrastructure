# Upgrade Summary: Multi-Architecture & JDK 21

## Overview

This upgrade brings **multi-architecture support** and updates all dependencies to their latest versions, including a major upgrade to **JDK 21**.

## 🎯 Key Changes

### 1. Multi-Architecture Support ✅

Your Docker images now support multiple CPU architectures:

| Architecture | Platform | Devices |
|--------------|----------|---------|
| **AMD64** | `linux/amd64` | Intel/AMD processors, most cloud VMs |
| **ARM64** | `linux/arm64` | Apple Silicon (M1/M2/M3), AWS Graviton, Raspberry Pi 4+ |
| **ARMv7** | `linux/arm/v7` | Raspberry Pi 3, older ARM devices |

**Benefits:**
- Deploy on Raspberry Pi, Apple Silicon Macs, AWS Graviton instances
- Automatic platform detection - Docker pulls the right image for your system
- Cost savings with ARM-based cloud instances (AWS Graviton is ~40% cheaper)

### 2. JDK 21 Upgrade ⚡

**Before:** JDK 17  
**After:** JDK 21 (LTS)

**Changes:**
- Builder: `gradle:8.11.1-jdk21` (was `gradle:8.10.2-jdk17`)
- Runtime: `eclipse-temurin:21-jre-jammy` (consistent across all platforms)

**Benefits:**
- Latest LTS Java version with performance improvements
- Virtual threads support (Project Loom)
- Pattern matching and modern language features
- Better garbage collection

### 3. Updated Dependencies 📦

#### GitHub Actions
- `actions/checkout`: v4 → **v4.2.2**
- `actions/upload-artifact`: v4 → **v4.5.0**
- `actions/setup-node`: **v4.1.0** (new)
- `docker/setup-qemu-action`: v3 → **v3.2.0**
- `docker/setup-buildx-action`: v3 → **v3.7.1**
- `docker/login-action`: v3 → **v3.3.0**
- `docker/build-push-action`: v5 → **v6.9.0**

#### Base Images
- **Nginx**: `1.27-alpine` → `1.27.3-alpine`
- **Node.js**: `20-alpine` → `22-alpine`
- **Gradle**: `8.10.2-jdk17` → `8.11.1-jdk21`

### 4. Enhanced Docker Build Script 🔨

The `docker-build.sh` script now supports:
- Multi-platform builds with a single command
- Automatic buildx builder setup
- Custom platform selection
- Push to Docker Hub or GitHub Container Registry
- Comprehensive build summaries

**Example:**
```bash
# Build for all platforms and push
./docker-build.sh 1.0.0 ghcr your-username "linux/amd64,linux/arm64,linux/arm/v7" push
```

### 5. New GitHub Actions Workflow 🤖

**File:** `.github/workflows/docker-multiarch.yml`

**Features:**
- Automatic multi-arch builds on version tags
- Pushes to both Docker Hub and GHCR
- Tests images on different architectures
- Manual trigger support
- Comprehensive build summaries

### 6. Security Enhancements 🔒

- Non-root user execution in containers
- Proper file permissions and ownership
- Health checks integrated into Dockerfile
- OCI image labels for better traceability

## 📁 Files Modified

### Infrastructure Repository
```
delerium-infrastructure/
├── .github/workflows/
│   ├── docker-multiarch.yml          [NEW] Multi-arch build workflow
│   └── integration-tests.yml         [UPDATED] Latest action versions, Node 22
├── docker-compose/
│   └── docker-compose.yml            [UPDATED] Platform support, latest images
├── CHANGELOG.md                      [NEW] Version history
├── MULTI_ARCH.md                     [NEW] Multi-arch documentation
├── UPGRADE_SUMMARY.md                [NEW] This file
└── README.md                         [UPDATED] Prerequisites, multi-arch info
```

### Server Repository
```
delerium-server/
├── Dockerfile                        [UPDATED] JDK 21, multi-arch, security
└── docker-build.sh                   [UPDATED] Multi-platform support
```

## 🚀 How to Use

### Option 1: Use Pre-Built Images (Easiest)

```bash
# Pull automatically detects your architecture
docker pull ghcr.io/marcusb333/delerium-server:latest

# Run on any platform
docker compose up -d
```

### Option 2: Build Locally

```bash
# Build for your current platform
docker compose build

# Or build for specific platforms
cd delerium-server
./docker-build.sh latest dockerhub your-username
```

### Option 3: Build Multi-Arch (Advanced)

```bash
cd delerium-server

# Build for all platforms
./docker-build.sh 1.0.0 ghcr your-username "linux/amd64,linux/arm64,linux/arm/v7" push
```

## 🔧 Migration Guide

### For Existing Deployments

**No action required!** Just pull the latest images:

```bash
cd delerium-infrastructure/docker-compose
docker compose pull
docker compose up -d
```

### For Local Development

1. **Update Docker** (if needed):
   ```bash
   docker --version  # Should be 24.0+
   ```

2. **Rebuild images**:
   ```bash
   docker compose build --no-cache
   ```

3. **Restart services**:
   ```bash
   docker compose up -d
   ```

### For CI/CD Pipelines

GitHub Actions workflows are automatically updated. No changes needed.

## 📊 Testing Multi-Architecture

### Verify Image Platforms

```bash
# Check what platforms are available
docker buildx imagetools inspect ghcr.io/marcusb333/delerium-server:latest

# Expected output:
# Name:      ghcr.io/marcusb333/delerium-server:latest
# MediaType: application/vnd.docker.distribution.manifest.list.v2+json
# Digest:    sha256:...
# Manifests:
#   Name:      ghcr.io/marcusb333/delerium-server:latest@sha256:...
#   MediaType: application/vnd.docker.distribution.manifest.v2+json
#   Platform:  linux/amd64
#   
#   Name:      ghcr.io/marcusb333/delerium-server:latest@sha256:...
#   MediaType: application/vnd.docker.distribution.manifest.v2+json
#   Platform:  linux/arm64
#   
#   Name:      ghcr.io/marcusb333/delerium-server:latest@sha256:...
#   MediaType: application/vnd.docker.distribution.manifest.v2+json
#   Platform:  linux/arm/v7
```

### Test on Different Platforms

```bash
# Force ARM64 (even on AMD64 host)
docker run --platform linux/arm64 -d -p 8080:8080 \
  -e DELETION_TOKEN_PEPPER=test-secret \
  ghcr.io/marcusb333/delerium-server:latest

# Test health
curl http://localhost:8080/api/pow
```

## 🐛 Troubleshooting

### Issue: "exec format error"

**Cause:** Wrong architecture image.

**Solution:**
```bash
# Let Docker auto-detect
docker pull ghcr.io/marcusb333/delerium-server:latest

# Or specify explicitly
docker pull --platform linux/$(uname -m) ghcr.io/marcusb333/delerium-server:latest
```

### Issue: Slow builds on non-native architectures

**Cause:** QEMU emulation is slower.

**Solution:**
- Use GitHub Actions for multi-arch builds (native speed)
- Build only for your target platform locally
- Use pre-built images

### Issue: Buildx not available

**Solution:**
```bash
# Check Docker version
docker --version  # Need 24.0+

# Enable buildx
docker buildx install

# Create builder
docker buildx create --name multiarch --use
```

## 📚 Documentation

- **Multi-Architecture Guide**: [`MULTI_ARCH.md`](MULTI_ARCH.md)
- **Changelog**: [`CHANGELOG.md`](CHANGELOG.md)
- **Main README**: [`README.md`](README.md)
- **Deployment Guide**: [`DEPLOYMENT.md`](DEPLOYMENT.md)

## 🎉 Benefits Summary

✅ **Deploy anywhere**: AMD64, ARM64, ARMv7  
✅ **Latest tech**: JDK 21, Node 22, Nginx 1.27.3  
✅ **Automated builds**: GitHub Actions workflow  
✅ **Better security**: Non-root containers, health checks  
✅ **Cost savings**: Use cheaper ARM instances  
✅ **Future-proof**: Latest LTS versions  
✅ **Easy to use**: Automatic platform detection  

## 🤝 Need Help?

- **Documentation**: See `MULTI_ARCH.md` for detailed guide
- **Issues**: [GitHub Issues](https://github.com/marcusb333/delerium-infrastructure/issues)
- **Discussions**: [GitHub Discussions](https://github.com/marcusb333/delerium/discussions)

---

**Upgrade completed on:** 2025-11-18  
**Version:** 2.0.0  
**Supported platforms:** linux/amd64, linux/arm64, linux/arm/v7
