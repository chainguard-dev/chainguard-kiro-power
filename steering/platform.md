---
name: platform
description: Workflow for managing the Chainguard platform including organizations, policies, cluster registration, IAM, and supply chain artifact verification
---

# Chainguard Platform Management

## Goal
Help users manage their Chainguard organization, configure policies, register clusters, manage IAM, and work with supply chain security artifacts (SBOM, attestations, signatures).

## MCP Servers Used
- **`cg-api`** — primary server for all platform operations
- **`cg-oci`** — fetching image metadata for policy decisions
- **`cg-versions`** — validating image versions against policy requirements

## Prerequisites
All platform operations require authentication:
```bash
chainctl auth login
chainctl auth status   # verify
```

## Workflows

### Organization and IAM management

**List organizations:**
Use `cg-api` to list the user's Chainguard organizations and their IDs.

**Manage IAM groups:**
- Organizations contain groups (projects/environments)
- Use `chainctl iam groups list` to see the hierarchy
- Use `cg-api` to create or modify groups programmatically

**Role bindings:**
```bash
# Grant access to a user
chainctl iam role-bindings create \
  --group <group-id> \
  --identity <identity-id> \
  --role viewer   # viewer | editor | owner
```

Common roles:
| Role | Capabilities |
|------|-------------|
| `viewer` | Read-only access to images and policies |
| `editor` | Can create/modify policies, pull images |
| `owner` | Full admin including IAM management |

### Policy management

Chainguard Enforce uses Rego/OPA policies to govern what images can run in registered clusters.

**List policies:**
Use `cg-api` to list policies in an organization or group.

**Create a policy:**
Policies are defined as YAML. A basic image allowlist policy:

```yaml
apiVersion: policy.sigstore.dev/v1beta1
kind: ClusterImagePolicy
metadata:
  name: require-chainguard-images
spec:
  images:
    - glob: "cgr.dev/**"
  authorities:
    - keyless:
        url: https://fulcio.sigstore.dev
        identities:
          - issuer: https://token.actions.githubusercontent.com
            subjectRegExp: "https://github.com/chainguard-images/.*"
```

**Apply a policy:**
```bash
chainctl policies apply -f policy.yaml --group <group-id>
```

### Cluster registration

Chainguard Enforce can monitor Kubernetes clusters for policy compliance.

**Register a cluster:**
```bash
# Generate install manifest
chainctl clusters install --group <group-id> > chainguard-enforce.yaml

# Apply to cluster
kubectl apply -f chainguard-enforce.yaml
```

**List registered clusters:**
Use `cg-api` to list clusters and their compliance status.

**Check cluster compliance:**
```bash
chainctl clusters records list --cluster <cluster-id>
```

### SBOM and attestation workflows

Every Chainguard image includes a Software Bill of Materials (SBOM) and Sigstore attestations.

**Inspect SBOM:**
```bash
# View SBOM for an image
cosign download attestation \
  --predicate-type https://spdx.dev/Document \
  cgr.dev/chainguard/python:latest | jq -r '.payload' | base64 -d | jq
```

Use `cg-api` to query SBOM data programmatically via the platform API.

**Verify image signature:**
```bash
cosign verify \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  --certificate-identity-regexp "https://github.com/chainguard-images/.*" \
  cgr.dev/chainguard/python:latest
```

**Check image provenance:**
```bash
chainctl images history cgr.dev/chainguard/python:latest
```

### Vulnerability management

**Check CVEs for an image:**
Use `cg-api` or visit `images.chainguard.dev` to see:
- CVE count for a specific image tag
- CVE comparison vs. upstream equivalents
- Time-to-fix metrics

**Trivy scan for verification:**
```bash
trivy image cgr.dev/chainguard/python:latest
```

### Private image registry access

**Configure Docker credential helper:**
```bash
chainctl auth configure-docker
```
This sets up `~/.docker/config.json` with a credential helper that automatically provides valid tokens for `cgr.dev/<your-org>/`.

**Pull a private image:**
```bash
docker pull cgr.dev/<your-org>/<image>:<tag>
```

**CI/CD: OIDC authentication (GitHub Actions example):**
```yaml
- name: Authenticate to Chainguard
  uses: chainguard-dev/actions/setup-chainctl@main
  with:
    identity: <your-identity-id>

- name: Pull private image
  run: docker pull cgr.dev/${{ env.ORG }}/${{ env.IMAGE }}:latest
```

## Common chainctl Commands Reference

```bash
# Auth
chainctl auth login
chainctl auth status
chainctl auth configure-docker

# Orgs and groups
chainctl iam organizations list
chainctl iam groups list

# Images
chainctl images list --group <group-id>
chainctl images history <image-ref>

# Policies
chainctl policies list
chainctl policies apply -f policy.yaml
chainctl policies delete <policy-id>

# Clusters
chainctl clusters list
chainctl clusters records list

# Identity (for CI/CD)
chainctl iam identities list
chainctl iam identities create github \
  --github-repo <org/repo> \
  --role editor \
  --group <group-id>
```
