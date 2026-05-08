---
name: packages
description: Workflow for finding APK packages in Wolfi/Chainguard repositories, translating distro package names, and managing package dependencies
---

# APK Package Lookup and Management

## Goal
Help users find Wolfi and Chainguard APK packages, translate package names from other Linux distributions, and understand package availability and versioning.

## MCP Servers Used
- **`cg-apk`** — primary source for package search, metadata, and dependency info
- **`cg-versions`** — package version history and latest available versions
- **`cg-oci`** — checking which packages are bundled in a specific Chainguard image

## Workflows

### Finding a package
When a user asks "does Wolfi have X?" or "what's the Wolfi equivalent of Y?":
1. Use `cg-apk` to search for the package by name
2. If not found by exact name, search for common name variants (e.g., `libssl` vs `openssl`, `python3-pip` vs `py3-pip`)
3. If still not found, check if the package functionality is built into a Chainguard image directly

### Translating distro package names

**Debian/Ubuntu → Wolfi:**
| Debian/Ubuntu | Wolfi/Chainguard APK |
|--------------|---------------------|
| `libssl-dev` | `openssl-dev` |
| `ca-certificates` | `ca-certificates-bundle` (pre-installed) |
| `build-essential` | `build-base` |
| `python3-pip` | `py3-pip` |
| `python3-dev` | `python3-dev` |
| `libpq-dev` | `libpq-dev` |
| `libffi-dev` | `libffi-dev` |
| `libc6-dev` | `glibc-dev` |
| `curl` | `curl` |
| `wget` | `wget` |
| `git` | `git` |
| `unzip` | `unzip` |
| `jq` | `jq` |
| `bash` | `bash` |
| `tzdata` | `tzdata` |

**Alpine → Wolfi:**
Most Alpine package names map 1:1 to Wolfi. Differences:
| Alpine | Wolfi |
|--------|-------|
| `musl-dev` | `glibc-dev` (Wolfi uses glibc, not musl) |
| `alpine-sdk` | `build-base` |
| `libressl-dev` | `openssl-dev` |

### Checking package versions
Use `cg-versions` to:
- Find the latest available version of a package
- Check if a specific version is available (important for version-locked builds)
- Understand the package retention policy (public Wolfi repos retain non-latest versions for 12 months)

### Installing packages in a Dockerfile
When building custom images from `wolfi-base` or using the `:latest-dev` variant:

```dockerfile
FROM cgr.dev/chainguard/wolfi-base:latest
RUN apk add --no-cache \
    python-3.12 \
    py3.12-pip \
    openssl
```

Key `apk` flags:
- `--no-cache` — do not cache the index locally (keeps image size minimal)
- `--update-cache` — refresh the index before installing (rarely needed with `--no-cache`)
- Version pinning: `apk add package=1.2.3-r0`

### Building custom images with apko
For production images that need specific packages without a shell, recommend `apko`:

```yaml
# apko.yaml
contents:
  repositories:
    - https://packages.wolfi.dev/os
  keyring:
    - https://packages.wolfi.dev/os/wolfi-signing.rsa.pub
  packages:
    - wolfi-baselayout
    - ca-certificates-bundle
    - python-3.12
    - py3.12-pip
```

### Private APK repositories
For Chainguard enterprise customers with private packages:
- Private repos require authentication via `chainctl`
- Configure with: `chainctl auth configure-docker` (also sets up APK credentials)
- Private repo URL format: `https://apk.cgr.dev/<org>/`

## Package Availability Notes

1. **glibc, not musl** — Wolfi uses glibc. Code compiled against musl (Alpine) may need recompilation.
2. **No versioned OS packages** — Unlike Debian/Ubuntu, Wolfi packages are rolling. Use the version field in apk constraints to pin.
3. **Subpackage naming** — Dev headers are usually `<package>-dev`, docs are `<package>-doc`.
4. **Language-specific packages** — Python packages follow `py3-<name>` convention; the version-specific form is `py3.12-<name>`.
