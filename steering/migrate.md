---
name: migrate
description: Workflow for migrating Dockerfiles and applications to use Chainguard Images
---

# Dockerfile Migration to Chainguard Images

## Goal
Replace a `FROM` base image with the appropriate Chainguard equivalent, minimizing CVEs and image size while maintaining application functionality.

## Workflow

### 1. Identify the base image
Read the user's Dockerfile or ask them to share it. Extract:
- The current base image name and tag (e.g., `python:3.11-slim`, `node:20-alpine`, `ubuntu:22.04`)
- Any `RUN apt-get install`, `RUN apk add`, or `RUN yum install` lines — these are packages that may need Wolfi APK equivalents

### 2. Find the Chainguard image equivalent
Use `cg-oci` to find the right image:
- Query for images matching the runtime (e.g., `python`, `node`, `go`, `java`, `nginx`)
- If the user's tag implies a specific version, use `cg-versions` to confirm the available version tags
- Prefer `cgr.dev/chainguard/<image>:<version>` for production, `cgr.dev/chainguard/<image>:latest-dev` if shell/tools are needed during migration

### 3. Resolve package dependencies
For every `apt-get install` / `apk add` / `yum install` package in the original Dockerfile:
- Use `cg-apk` to find the Wolfi/Chainguard APK equivalent
- Common translations:
  - `libssl-dev` → `openssl-dev`
  - `ca-certificates` → already included in all Chainguard images
  - `curl` → `curl`
  - `git` → `git`
  - `build-essential` → `build-base`
  - Package not found → suggest using the `:latest-dev` variant or building a custom image with `apko`

### 4. Produce the migrated Dockerfile
Rewrite the Dockerfile with:
- Updated `FROM` line pointing to `cgr.dev/chainguard/<image>:<tag>`
- `RUN apk add --no-cache <packages>` replacing distro-specific package commands (only in `:latest-dev` or if packages exist in Wolfi)
- Remove `apt-get update && apt-get install -y` patterns entirely
- If the image is non-dev and fully distroless, remove all `RUN apk add` lines and note that those packages must be bundled at build time

### 5. Handle non-root user
- Chainguard images default to a non-root user. If the app needs port < 1024 or writes to root-owned paths, add a note
- Avoid `USER root` unless strictly necessary
- If the user's app had `USER appuser` or similar, it likely already works as-is

### 6. Multi-stage Dockerfile pattern (recommended)
For compiled languages (Go, Java, Rust, C), suggest a multi-stage build:

```dockerfile
# Build stage — use a full builder image
FROM cgr.dev/chainguard/go:latest AS builder
WORKDIR /app
COPY . .
RUN go build -o /app/server ./cmd/server

# Runtime stage — minimal distroless
FROM cgr.dev/chainguard/static:latest
COPY --from=builder /app/server /server
ENTRYPOINT ["/server"]
```

### 7. Check for language-level dependency installs
If the Dockerfile contains `RUN pip install`, `RUN npm install`, `RUN mvn`, or `RUN gradle`, the base image migration alone does not secure those dependencies — they still resolve from public PyPI, npm, or Maven Central. In this case, also load `libraries.md` and walk the user through pointing those package managers at `libraries.cgr.dev` so the full supply chain is hardened, not just the base image.

Common patterns to watch for:
```dockerfile
RUN pip install -r requirements.txt          # → suggest libraries.md (Python)
RUN npm install                              # → suggest libraries.md (JavaScript)
RUN mvn dependency:resolve                  # → suggest libraries.md (Java)
RUN gradle dependencies                     # → suggest libraries.md (Java)
```

### 8. Validate
After generating the migrated Dockerfile, remind the user to:
- Build locally: `docker build -t myapp:test .`
- Check for missing shared libraries: `docker run --rm myapp:test ldd /path/to/binary` (use `:latest-dev` tag for this)
- Verify the image runs as non-root: `docker run --rm myapp:test whoami`

## Common Base Image Mappings

| Original image | Chainguard equivalent |
|---|---|
| `python:3.X-slim` | `cgr.dev/chainguard/python:3.X` |
| `python:3.X` | `cgr.dev/chainguard/python:3.X-dev` |
| `node:XX-alpine` | `cgr.dev/chainguard/node:XX` |
| `node:XX` | `cgr.dev/chainguard/node:XX-dev` |
| `golang:1.XX` | `cgr.dev/chainguard/go:1.XX` |
| `ubuntu:22.04` / `debian:bookworm-slim` | `cgr.dev/chainguard/wolfi-base:latest` |
| `alpine:3.X` | `cgr.dev/chainguard/wolfi-base:latest` |
| `scratch` | `cgr.dev/chainguard/static:latest` |
| `nginx:alpine` | `cgr.dev/chainguard/nginx:latest` |
| `openjdk:XX-slim` | `cgr.dev/chainguard/jre:XX` |
| `maven:3.X-eclipse-temurin-XX` | `cgr.dev/chainguard/maven:3.X-jdk-XX` |
| `rust:1.X` | `cgr.dev/chainguard/rust:latest` |
