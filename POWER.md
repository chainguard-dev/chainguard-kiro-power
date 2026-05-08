---
name: chainguard
displayName: Chainguard
description: Secure container images, language libraries, supply chain security, and Wolfi APK package workflows using Chainguard's platform. Covers Dockerfile migration, image discovery, package lookup, Chainguard platform management, and securing Java/JavaScript/Python dependencies via Chainguard Libraries.
keywords: chainguard, cgr.dev, wolfi, distroless, container security, supply chain, apk, chainguard images, enforce, chainctl, sbom, attestation, cve, hardened, migration, libraries, libraries.cgr.dev, maven, gradle, npm, yarn, pnpm, pip, pypi, poetry, uv, java library, python package, javascript package, dependency, secure dependencies
author: Chainguard
---

## Onboarding

Before using Chainguard tools, verify the user is authenticated:

1. Check if `chainctl` is installed: run `chainctl version`. If missing, direct the user to install it:
   - macOS: `brew install chainguard-dev/tap/chainctl`
   - Linux/other: `curl -s "https://dl.enforce.dev/chainctl/latest/chainctl_$(uname -s | tr '[:upper:]' '[:lower:]')_$(uname -m).tar.gz" | tar xz -C /usr/local/bin`

2. Check authentication: run `chainctl auth status`. If not logged in, prompt the user to run `chainctl auth login`.

3. Confirm access: run `chainctl iam organizations list` to verify the user can reach the Chainguard platform.

Once authenticated, the MCP servers (`cg-api`, `cg-apk`, `cg-oci`, `cg-versions`) will use the same session credentials.

---

## MCP Tools Reference

This power provides four MCP servers. Use the right server for each task:

| Server | URL | Use for |
|--------|-----|---------|
| `cg-api` | console-api.enforce.dev | Organization management, policies, cluster registration, IAM, attestation verification |
| `cg-apk` | apk.cgr.dev | APK package search, package metadata, dependency info, package equivalents |
| `cg-oci` | cgr.dev | Image discovery, tag listing, manifest inspection, image metadata |
| `cg-versions` | versions.cgr.dev | Latest version lookups, version history, upgrade paths |

---

## Steering Instructions

Load the appropriate steering file based on the user's task:

- Migrating a Dockerfile or app to use Chainguard images → `migrate.md`
- Browsing, discovering, or inspecting Chainguard container images → `images.md`
- Looking up APK packages, finding equivalents, or managing Wolfi packages → `packages.md`
- Managing the Chainguard platform (org, policies, clusters, IAM, SBOMs) → `platform.md`
- Securing Java, JavaScript, or Python dependencies via Chainguard Libraries → `libraries.md`

### Quick decision guide

- Keywords like "migrate", "convert", "FROM", "Dockerfile", "base image", "replace my base" → load `migrate.md`
- Keywords like "image", "tag", "cgr.dev", "distroless", "container", "pull", "digest" → load `images.md`
- Keywords like "package", "apk", "wolfi", "install", "apt", "yum", "equivalent" → load `packages.md`
- Keywords like "policy", "cluster", "enforce", "organization", "SBOM", "attestation", "IAM", "sigstore", "cosign" → load `platform.md`
- Keywords like "libraries", "maven", "gradle", "npm", "pip", "pypi", "poetry", "uv", "yarn", "pnpm", "java dependency", "python dependency", "javascript dependency", "libraries.cgr.dev", "secure my dependencies" → load `libraries.md`

---

## Core Principles

When helping with Chainguard tasks, always keep these in mind:

1. **Minimal by default** — Chainguard images contain only what is necessary. When suggesting additions, prefer APK packages over alternative approaches that increase image size.

2. **Use versioned tags for production** — Recommend `:latest` only for development. For production Dockerfiles, use a pinned digest or dated tag (e.g., `cgr.dev/chainguard/python:3.12`).

3. **Non-root by default** — Chainguard images run as non-root. If a user's workload requires root, flag it and suggest `cgr.dev/chainguard/<image>:latest-dev` or a custom image built with `apko`.

4. **dev vs. non-dev variants** — The `:latest-dev` tag includes a shell and package manager for debugging. Production workloads should use the non-dev tag.

5. **Supply chain verification** — Chainguard images ship with SBOM and Sigstore attestations. When security is a concern, use `cg-api` tools to verify attestations and check provenance.
