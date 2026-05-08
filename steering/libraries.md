---
name: libraries
description: Workflow for migrating Java, JavaScript, and Python projects to use Chainguard Libraries — hardened, SLSA-built packages from libraries.cgr.dev
---

# Chainguard Libraries Migration

## Goal
Replace or front standard package registries (Maven Central, npm, PyPI) with Chainguard Libraries — rebuilt, signed, SLSA L3-compliant packages that eliminate 99.7% of supply chain malware by design.

## No MCP backing — documentation-driven
The current Chainguard MCP servers do not cover Libraries. This workflow is based on official Chainguard documentation. Always direct users to `https://edu.chainguard.dev/chainguard/libraries/` for the latest configuration details.

## Prerequisites

1. **Chainguard Libraries subscription** — required for access to `libraries.cgr.dev`. Direct users to `https://www.chainguard.dev/libraries` to sign up or start a trial.

2. **Pull token** — obtain credentials via chainctl:
   ```bash
   # Get identity ID and token for your specific ecosystem
   chainctl auth configure-docker   # also sets up library credentials
   # Or generate a pull token directly:
   chainctl tokens create --parent <org-id>
   ```
   The pull token provides two values used across all ecosystems:
   - **Identity ID** — used as the username
   - **Token** — used as the password

3. **Approach: direct vs. repo manager** — ask the user which they use:
   - **Direct**: configure the build tool to hit `libraries.cgr.dev` directly (simpler, good for individuals/small teams)
   - **Repository manager** (JFrog Artifactory, Sonatype Nexus, Cloudsmith, Google Artifact Registry): configure as an upstream proxy (recommended for enterprises)

---

## Java

### Registry URL
`https://libraries.cgr.dev/java/`

### Authentication environment variables
```bash
export CHAINGUARD_JAVA_IDENTITY_ID="<your-identity-id>"
export CHAINGUARD_JAVA_TOKEN="<your-token>"
```

### Maven — direct access (settings.xml)
```xml
<settings>
  <activeProfiles>
    <activeProfile>chainguard</activeProfile>
  </activeProfiles>
  <profiles>
    <profile>
      <id>chainguard</id>
      <repositories>
        <repository>
          <id>chainguard</id>
          <url>https://libraries.cgr.dev/java/</url>
          <releases><enabled>true</enabled></releases>
          <snapshots><enabled>false</enabled></snapshots>
        </repository>
        <repository>
          <id>central</id>
          <url>https://repo1.maven.org/maven2/</url>
          <releases><enabled>true</enabled></releases>
          <snapshots><enabled>false</enabled></snapshots>
        </repository>
      </repositories>
      <pluginRepositories>
        <pluginRepository>
          <id>chainguard</id>
          <url>https://libraries.cgr.dev/java/</url>
          <releases><enabled>true</enabled></releases>
          <snapshots><enabled>false</enabled></snapshots>
        </pluginRepository>
        <pluginRepository>
          <id>central</id>
          <url>https://repo1.maven.org/maven2/</url>
        </pluginRepository>
      </pluginRepositories>
    </profile>
  </profiles>
  <servers>
    <server>
      <id>chainguard</id>
      <username>${env.CHAINGUARD_JAVA_IDENTITY_ID}</username>
      <password>${env.CHAINGUARD_JAVA_TOKEN}</password>
    </server>
  </servers>
</settings>
```

> **Important**: Chainguard must be listed before Maven Central so artifacts are resolved from the secured registry first.

### Gradle — direct access (build.gradle)
```gradle
repositories {
  maven {
    url = uri("https://libraries.cgr.dev/java/")
    credentials {
      username = providers.environmentVariable("CHAINGUARD_JAVA_IDENTITY_ID").orNull
      password = providers.environmentVariable("CHAINGUARD_JAVA_TOKEN").orNull
    }
  }
  mavenCentral()
}
```

### Via repository manager (Maven settings.xml)
Replace the Chainguard URL with your repo manager's virtual/group URL:
```xml
<mirror>
  <id>repo-manager</id>
  <mirrorOf>*</mirrorOf>
  <url>https://<your-repo-manager>/repository/java-all/</url>
</mirror>
```

### Cache cleanup (run before first use)
```bash
rm -rf ~/.m2/repository   # Maven
rm -rf ~/.gradle/caches/  # Gradle
```

---

## JavaScript

### Registry URL
`https://libraries.cgr.dev/javascript/`

### Authentication environment variables
```bash
export CHAINGUARD_JAVASCRIPT_IDENTITY_ID="<your-identity-id>"
export CHAINGUARD_JAVASCRIPT_TOKEN="<your-token>"
```

### npm — direct access (.npmrc)
```bash
# Generate base64-encoded credentials (no line wrapping)
export token=$(echo -n "${CHAINGUARD_JAVASCRIPT_IDENTITY_ID}:${CHAINGUARD_JAVASCRIPT_TOKEN}" | base64 -w 0)

npm config set registry https://libraries.cgr.dev/javascript/ --location=project
npm config set //libraries.cgr.dev/javascript/:_auth "${token}" --location=project
```
Resulting `.npmrc`:
```
registry=https://libraries.cgr.dev/javascript/
//libraries.cgr.dev/javascript/:_auth=<base64-credentials>
```

> **Note**: The trailing slash in the registry URL is required.

### yarn classic (v1) — direct access
```
registry=https://libraries.cgr.dev/javascript/
//libraries.cgr.dev/javascript/:_auth="<base64-token>"
//libraries.cgr.dev/javascript/:always-auth=true
```

### yarn berry (v2+) — direct access
```bash
export authInfo="${CHAINGUARD_JAVASCRIPT_IDENTITY_ID}:${CHAINGUARD_JAVASCRIPT_TOKEN}"
yarn config set npmRegistryServer https://libraries.cgr.dev/javascript
yarn config set 'npmRegistries["//libraries.cgr.dev/javascript"].npmAuthIdent' "${authInfo}"
yarn config set 'npmRegistries["//libraries.cgr.dev/javascript"].npmAlwaysAuth' "true"
```
> Yarn Berry uses plain text (not base64) for `npmAuthIdent`.

### pnpm — direct access (.npmrc)
Same format as npm — add to `.npmrc`:
```
registry=https://libraries.cgr.dev/javascript/
//libraries.cgr.dev/javascript/:_auth=<base64-credentials>
```

### bun — direct access (bunfig.toml)
```toml
[install.registry]
url = "https://libraries.cgr.dev/javascript/"
username = "$CHAINGUARD_JAVASCRIPT_IDENTITY_ID"
password = "$CHAINGUARD_JAVASCRIPT_TOKEN"
```

### Via repository manager
Point `registry=` at your repo manager's virtual npm URL (Chainguard should be Priority 1):
```
registry=https://<your-repo-manager>/repository/javascript-all/
```

---

## Python

### Registry URLs
| Purpose | URL |
|---------|-----|
| All packages (full catalog) | `https://libraries.cgr.dev/python/simple/` |
| CVE-remediated packages only | `https://libraries.cgr.dev/python-remediated/simple/` |

Chainguard recommends using both: remediated first, full catalog as fallback, PyPI as last resort.

### Authentication environment variables
```bash
export CHAINGUARD_PYTHON_IDENTITY_ID="<your-identity-id>"
export CHAINGUARD_PYTHON_TOKEN="<your-token>"
```
> **Note**: Replace any `/` in the identity ID with `_` when using it in URLs.

### Shared .netrc authentication (works with pip, uv, poetry)
```
machine libraries.cgr.dev
login <CHAINGUARD_PYTHON_IDENTITY_ID>
password <CHAINGUARD_PYTHON_TOKEN>
```
```bash
chmod 600 ~/.netrc
```

### pip — direct access (pip.conf)
```ini
[global]
index-url = https://<CHAINGUARD_PYTHON_IDENTITY_ID>:<CHAINGUARD_PYTHON_TOKEN>@libraries.cgr.dev/python/simple/
```

### requirements.txt — direct access
```
--index-url https://<CHAINGUARD_PYTHON_IDENTITY_ID>:<CHAINGUARD_PYTHON_TOKEN>@libraries.cgr.dev/python/simple/
requests==2.32.3
flask==3.1.0
```

### poetry — direct access (pyproject.toml + auth)
```bash
poetry config http-basic.chainguard <CHAINGUARD_PYTHON_IDENTITY_ID> <CHAINGUARD_PYTHON_TOKEN>
```
```toml
[[tool.poetry.source]]
name = "chainguard-remediated"
url = "https://libraries.cgr.dev/python-remediated/simple/"
priority = "primary"

[[tool.poetry.source]]
name = "chainguard"
url = "https://libraries.cgr.dev/python/simple/"
priority = "supplemental"

[[tool.poetry.source]]
name = "PyPI"
priority = "supplemental"
```

### uv — direct access (pyproject.toml with .netrc)
```toml
[[tool.uv.index]]
name = "cgr-remediated"
url = "https://libraries.cgr.dev/python-remediated/simple"
authenticate = "always"

[[tool.uv.index]]
name = "cgr"
url = "https://libraries.cgr.dev/python/simple"
authenticate = "always"

[[tool.uv.index]]
name = "pypi"
url = "https://pypi.org/simple/"
default = true

[tool.uv]
index-strategy = "unsafe-best-match"
```

### Via repository manager
Configure your repo manager with Chainguard at Priority 1 and PyPI as fallback, then point pip/uv/poetry at the repo manager's simple index URL.

---

## Key principles

1. **Chainguard Libraries are drop-in replacements** — package names and versions stay the same. No code changes required; only registry configuration changes.

2. **Prioritize Chainguard over public registries** — always list `libraries.cgr.dev` first so packages are resolved from the secured source before falling back to public registries.

3. **Fallback caveat** — packages resolved from the public fallback (Maven Central, npm, PyPI) are NOT covered by Chainguard's malware-resistance guarantees. For maximum security, use Chainguard Repository as the exclusive source.

4. **SLSA L3 provenance** — every package is built in Chainguard's controlled factory and signed. Users can verify attestations for any artifact.

5. **Remediated vs. standard** — for Python, the `-remediated` index contains packages with Chainguard-applied CVE patches on top of upstream. Use it as the primary source with the standard index as fallback.
