# Building Bookinfo Sample on Windows

This guide provides step-by-step instructions for building the Istio Bookinfo sample application on Windows using PowerShell.

## Prerequisites

Before you begin, ensure you have the following installed:

- **Docker Desktop for Windows** (running and configured)
  - Download from: https://www.docker.com/products/docker-desktop
  - Ensure Docker Desktop is running before building
- **Git Bash or WSL** (for running bash scripts)
- **PowerShell** (Windows PowerShell or PowerShell Core)

## Environment Setup

### 1. Set Environment Variables

Open PowerShell and set the required environment variables:

```powershell
# Set your Docker Hub username (or registry)
$env:BOOKINFO_HUB = "docker.io/YOUR_USERNAME"

# Set the version tag for your images
$env:BOOKINFO_TAG = "1.0"
```

Example:
```powershell
$env:BOOKINFO_HUB = "docker.io/qiang"
$env:BOOKINFO_TAG = "1.0"
```

### 2. Navigate to Istio Root Directory

```powershell
cd c:\Users\YOUR_USERNAME\source\repos\istio
```

## Building the Bookinfo Services

### Option 1: Build for Your Platform Only (Recommended)

This builds images for `linux/amd64` only, which is faster and suitable for most Windows development:

```powershell
$env:TAGS = "1.0"
$env:HUB = "docker.io/YOUR_USERNAME"
docker buildx bake -f samples/bookinfo/src/docker-bake.hcl --set '*.platform=linux/amd64' --load
```

### Option 2: Build for Multiple Platforms

To build for both `linux/amd64` and `linux/arm64`:

```powershell
$env:TAGS = "1.0"
$env:HUB = "docker.io/YOUR_USERNAME"
docker buildx bake -f samples/bookinfo/src/docker-bake.hcl --load
```

**Note:** Multi-platform builds require Docker BuildKit and may take significantly longer.

### Option 3: Using the Build Script (Alternative)

If you have Git Bash or WSL installed, you can use the provided bash script:

```bash
# From Git Bash or WSL
cd /mnt/c/Users/YOUR_USERNAME/source/repos/istio/samples/bookinfo

BOOKINFO_TAG=1.0 BOOKINFO_HUB=docker.io/YOUR_USERNAME src/build-services.sh --load
```

**Windows Line Endings Fix:** If you encounter errors about `\r` (carriage return), convert the line endings:

```powershell
(Get-Content src/build-services.sh -Raw) -replace "`r`n", "`n" | Set-Content -NoNewline src/build-services.sh
```

## Build Output

The build process creates the following Docker images:

### Core Services
- **Productpage** (Python/Flask)
  - `examples-bookinfo-productpage-v1`
  - `examples-bookinfo-productpage-v-flooding`

- **Details** (Ruby)
  - `examples-bookinfo-details-v1`
  - `examples-bookinfo-details-v2`

- **Reviews** (Java/Open Liberty)
  - `examples-bookinfo-reviews-v1` (no ratings)
  - `examples-bookinfo-reviews-v2` (ratings with black stars)
  - `examples-bookinfo-reviews-v3` (ratings with red stars)

- **Ratings** (Node.js)
  - `examples-bookinfo-ratings-v1`
  - `examples-bookinfo-ratings-v2`
  - `examples-bookinfo-ratings-v-delayed`
  - `examples-bookinfo-ratings-v-faulty`
  - `examples-bookinfo-ratings-v-unavailable`
  - `examples-bookinfo-ratings-v-unhealthy`

### Databases
- `examples-bookinfo-mongodb`
- `examples-bookinfo-mysqldb`

## Verify Build

List all built images:

```powershell
docker images | Select-String "examples-bookinfo"
```

Or for more details:

```powershell
docker images --filter "reference=*/examples-bookinfo-*"
```

## Troubleshooting

### Docker Desktop Not Running

**Error:** `error during connect: ... dockerDesktopLinuxEngine: The system cannot find the file specified`

**Solution:** Start Docker Desktop and wait for it to fully initialize before running the build.

### Line Ending Issues in WSL/Git Bash

**Error:** `$'\r': command not found`

**Solution:** Convert the bash script to Unix line endings:

```powershell
(Get-Content src/build-services.sh -Raw) -replace "`r`n", "`n" | Set-Content -NoNewline src/build-services.sh
```

### Platform Architecture Errors

**Error:** `exec /bin/sh: exec format error`

**Solution:** Build for your specific platform only using the `--set '*.platform=linux/amd64'` flag.

### Permission Errors

**Error:** Permission denied errors when running bash scripts

**Solution:** Ensure you have the necessary permissions or run PowerShell as Administrator.

## Pushing Images to Registry (Optional)

To push the built images to your Docker registry:

```powershell
$env:TAGS = "1.0"
$env:HUB = "docker.io/YOUR_USERNAME"
docker buildx bake -f samples/bookinfo/src/docker-bake.hcl --set '*.platform=linux/amd64' --push
```

**Note:** You must be logged in to your Docker registry:

```powershell
docker login
```

## Build Time Estimates

Approximate build times (on modern hardware):

- **First build:** 5-10 minutes (downloads base images)
- **Subsequent builds:** 2-5 minutes (uses cached layers)
- **Multi-platform build:** 10-20 minutes

## Next Steps

After building the images:

1. **Deploy to Kubernetes:** Use the YAML files in `platform/kube/` to deploy
2. **Configure Istio:** Apply the networking configurations from `networking/`
3. **Test the Application:** Access the productpage service to verify deployment

## Additional Resources

- [Istio Bookinfo Documentation](https://istio.io/docs/examples/bookinfo/)
- [Docker Buildx Documentation](https://docs.docker.com/buildx/working-with-buildx/)
- [Istio Getting Started](https://istio.io/latest/docs/setup/getting-started/)

## Build Configuration

The build configuration is defined in:
- `src/docker-bake.hcl` - Docker Bake configuration file
- `src/build-services.sh` - Build wrapper script
- Individual `Dockerfile`s in each service directory

To customize the build (e.g., change base images, add dependencies), modify these files accordingly.
