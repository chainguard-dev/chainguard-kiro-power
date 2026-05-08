# Chainguard Power for Kiro

Bring Chainguard's supply chain security capabilities directly into your Kiro IDE. This power gives you an AI agent that knows how to work with Chainguard Images, Wolfi APK packages, Chainguard Libraries, and the Chainguard platform — activated automatically when you mention relevant topics in conversation.

---

## What this power does

| You ask about... | What happens |
|---|---|
| Migrating a Dockerfile to Chainguard Images | Agent rewrites your `FROM` line, translates packages to Wolfi APKs, handles multi-stage builds |
| Securing Java/JavaScript/Python dependencies | Agent configures Maven/Gradle/npm/pip/poetry/uv to pull from `libraries.cgr.dev` |
| Finding a Chainguard image or tag | Agent queries the live `cgr.dev` registry via MCP |
| Looking up an APK package | Agent searches `apk.cgr.dev` for Wolfi equivalents |
| Managing your org, policies, or clusters | Agent guides you through `chainctl` and the Chainguard platform API |

---

## Installation

1. Open Kiro and go to the **Powers** panel in the sidebar
2. Search for **Chainguard**
3. Click **Install** — Kiro registers all four MCP servers automatically

That's it. No JSON editing, no CLI setup for the power itself.

---

## Before you start

Make sure `chainctl` is installed and you're logged in:

```bash
# Install chainctl (macOS)
brew install chainguard-dev/tap/chainctl

# Install chainctl (Linux)
curl -s "https://dl.enforce.dev/chainctl/latest/chainctl_$(uname -s | tr '[:upper:]' '[:lower:]')_$(uname -m).tar.gz" | tar xz -C /usr/local/bin

# Log in
chainctl auth login

# Verify
chainctl auth status
```

The four MCP servers in this power (`cg-api`, `cg-apk`, `cg-oci`, `cg-versions`) use your existing Chainguard session — no separate credentials needed for image and package lookups.

> **Chainguard Libraries** (Java/JavaScript/Python) require a separate subscription. See [chainguard.dev/libraries](https://www.chainguard.dev/libraries).

---

## Walkthrough: Migrating a Dockerfile

Here's exactly what a migration conversation looks like end-to-end.

### Step 1 — Open a chat and paste your Dockerfile

```
You: Can you help me migrate this Dockerfile to use Chainguard Images?

FROM python:3.11-slim
WORKDIR /app
RUN apt-get update && apt-get install -y curl git libpq-dev
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
CMD ["python", "app.py"]
```

### Step 2 — Kiro activates the power

When Kiro sees keywords like `migrate` and `Dockerfile`, it automatically loads the Chainguard power and the `migrate` workflow. You don't need to invoke any command.

### Step 3 — The agent works through the migration

Kiro will:
1. **Identify the base image** — `python:3.11-slim`
2. **Query `cg-oci`** to find `cgr.dev/chainguard/python:3.11`
3. **Query `cg-apk`** to translate each package:
   - `curl` → `curl` (same name in Wolfi)
   - `git` → `git` (same)
   - `libpq-dev` → `libpq-dev` (same)
   - `apt-get update` and the `apt-get install` pattern → replaced with `apk add --no-cache`
4. **Rewrite the Dockerfile** and explain the changes

### Step 4 — Review the output

```dockerfile
FROM cgr.dev/chainguard/python:3.11-dev
WORKDIR /app
RUN apk add --no-cache curl git libpq-dev
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
CMD ["python", "app.py"]
```

Kiro will also note:
- The `:3.11-dev` tag is used because `pip install` requires the package manager (use `:3.11` for a fully distroless runtime image)
- The image runs as a non-root user by default

### Step 5 — Also secure your Python dependencies (optional)

Because the Dockerfile runs `pip install -r requirements.txt`, Kiro will suggest also routing your pip installs through Chainguard Libraries:

```
Kiro: I notice your Dockerfile uses pip install. Your base image is now secured,
but the packages in requirements.txt still resolve from public PyPI. Would you
like me to configure pip to use Chainguard Libraries instead?
```

Just say yes and Kiro will walk you through the `pip.conf` or `pyproject.toml` configuration.

---

## Other things you can ask

```
"Do you have a Chainguard image for Nginx?"
"What's the Wolfi equivalent of build-essential?"
"Show me the latest tag for cgr.dev/chainguard/node"
"How do I set up my Maven project to use Chainguard Libraries?"
"Help me configure .npmrc to pull from libraries.cgr.dev"
"How do I register a Kubernetes cluster with Chainguard Enforce?"
"Show me how to verify the SBOM for a Chainguard image"
"Set up a GitHub Actions identity for my org"
```

---

## MCP servers included

| Server | What it connects to | What the agent uses it for |
|--------|--------------------|-----------------------------|
| `cg-api` | console-api.enforce.dev | Org management, policies, clusters, IAM, attestations |
| `cg-apk` | apk.cgr.dev | APK package search and metadata |
| `cg-oci` | cgr.dev | Image discovery, tags, manifests |
| `cg-versions` | versions.cgr.dev | Latest versions and upgrade paths |

> Chainguard Libraries (`libraries.cgr.dev`) is documentation-driven — the agent provides configuration guidance without a dedicated MCP server.

---

## Resources

- [Chainguard Images catalog](https://images.chainguard.dev)
- [Chainguard Academy docs](https://edu.chainguard.dev)
- [Chainguard Libraries](https://www.chainguard.dev/libraries)
- [chainctl CLI reference](https://edu.chainguard.dev/chainguard/chainctl-usage/)
- [Wolfi OS](https://edu.chainguard.dev/open-source/wolfi/overview/)
