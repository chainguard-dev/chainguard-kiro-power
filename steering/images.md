---
name: images
description: Workflow for discovering, inspecting, and working with Chainguard container images from cgr.dev
---

# Working with Chainguard Images

## Goal
Help users find, evaluate, and use Chainguard container images from `cgr.dev`.

## MCP Servers Used
- **`cg-oci`** — primary source for image discovery, tag listing, and manifest inspection
- **`cg-versions`** — version history and latest version lookups
- **`cg-api`** — attestation and SBOM verification for images (requires authentication)

## Workflows

### Discovering available images
Use `cg-oci` to browse the Chainguard image catalog. When a user asks "what images does Chainguard have?" or "do you have an image for X?":
- Search by runtime or use case (e.g., `python`, `nginx`, `postgres`, `redis`)
- Retrieve the image directory URL: `cgr.dev/chainguard/<image-name>`

### Listing tags
For a given image, use `cg-oci` to list available tags. Key tag conventions:
- `latest` — most recent stable version
- `latest-dev` — includes shell (`/bin/sh`) and `apk` package manager; for debugging and CI
- `<version>` — e.g., `3.12`, `20`, `1.21` — version-pinned tags
- `<version>-dev` — version-pinned with dev tooling
- Digests (`sha256:...`) — immutable references; recommend for production

### Inspecting image metadata
Use `cg-oci` to pull image manifests. Look for:
- Architecture support (`amd64`, `arm64`)
- Image size — Chainguard images are minimal; if a user asks why the image seems small, that's by design
- Labels and annotations — Chainguard images include OCI annotations with source and build info

### Checking CVE count / image freshness
Use `cg-versions` to check:
- When the image was last rebuilt
- Current version vs. available version
- CVE count comparisons between Chainguard and other base images are documented at `images.chainguard.dev`

### Verifying supply chain artifacts
Use `cg-api` to verify SBOM and Sigstore attestations:
- SBOM attestations are attached as OCI artifacts to every Chainguard image
- To verify: `cosign verify-attestation --type spdxjson cgr.dev/chainguard/<image>:<tag>`
- Users can also use `chainctl images history` to review build provenance

## Image Variants

Every Chainguard image has two primary variants:

| Variant | Tag suffix | Contents | Use case |
|---------|-----------|----------|----------|
| Production | `latest` or `<version>` | Runtime only, no shell | Production workloads |
| Development | `latest-dev` or `<version>-dev` | Shell + apk package manager | Debugging, CI, local dev |

Always recommend the non-dev tag for production deployments.

## Pulling Images

```bash
# Public/free images (no auth required)
docker pull cgr.dev/chainguard/python:latest

# Private/enterprise images (requires Chainguard auth)
chainctl auth configure-docker
docker pull cgr.dev/<org>/<image>:<tag>
```

## Useful References
- Image catalog: `https://images.chainguard.dev`
- Image directory API: `https://cgr.dev/chainguard/<image-name>`
- CVE comparison dashboard: `https://images.chainguard.dev/security`
