---
name: hermes-build
description: "Bump prebuilt images and tool versions in the coolify-stacks repo."
version: 1.0.0
platforms: [linux, macos]
metadata:
  hermes:
    tags: [deploy, coolify, docker, bump, version, firecrawl]
    related_skills: []
---

# Hermes Build: Coolify Stack Version Bumps

## Overview

This repository (`coolify-stacks`) packages Hermes Agent and Hermes WebUI as a deployable Docker Compose stack for Coolify. The agent image is custom-built with pre-installed tool packages so the gateway can use them without runtime PyPI access.

Three files define the versions the stack runs:

| File | What it pins |
|------|-------------|
| `apps/hermes/Dockerfile.agent` | Agent base image SHA + pre-installed tool versions |
| `apps/hermes/docker-compose.yml` | WebUI image tag + agent-src volume tag |
| `apps/hermes/docker-compose.yml` | Agent image reference (`ghcr.io/.../hermes-agent:latest`) |

The upstream counterparts:

| Upstream source | What to check |
|----------------|--------------|
| `hermes-agent/tools/lazy_deps.py` | Pinned versions of lazy-installable tools (firecrawl, exa, etc.) |
| `hermes-agent/pyproject.toml` | Core dependency versions |
| `hermes-webui/CHANGELOG.md` | Latest webui release version |
| `hub.docker.com/r/nousresearch/hermes-agent/tags` | Available agent image SHAs |

---

## Bumping the Agent Base Image

The agent base image is pinned by SHA so weekly rebuilds don't break when `latest` moves.

**1. Find the new SHA:**

Pull the latest image and read its digest:

```sh
docker pull nousresearch/hermes-agent:latest
docker inspect nousresearch/hermes-agent:latest --format '{{.RepoDigests}}'
```

Or find a specific version tag on Docker Hub / ghcr.io.

**2. Update `Dockerfile.agent`:**

The `FROM` line at the top of the file:

```
FROM nousresearch/hermes-agent@sha256:<OLD_SHA>
→
FROM nousresearch/hermes-agent@sha256:<NEW_SHA>
```

**3. Rebuild and push:**

Push the change to the `main` branch of `coolify-stacks`. The GitHub Actions workflow at `.github/workflows/hermes-agent.yml` builds and pushes the derived image to `ghcr.io/sangoisanga/coolify-stacks/hermes-agent:latest`.

Or trigger manually: GitHub → Actions → "Build & push hermes-agent" → "Run workflow".

---

## Bumping the WebUI Version

**1. Find the new version:**

Read `hermes-webui/CHANGELOG.md` — the latest release is the topmost `## [vX.Y.Z]` entry under `## [Unreleased]`.

**2. Update `docker-compose.yml`:**

Two places hold the webui version tag — the image reference and the agent-src volume name (both must match):

```yaml
  hermes-webui:
    image: "ghcr.io/nesquena/hermes-webui:<OLD_VERSION>"
    volumes:
      - "hermes-agent-src-<OLD_VERSION>:/opt/hermes"
      - "hermes-agent-src-<OLD_VERSION>:/home/hermeswebui/.hermes/hermes-agent:ro"
```

Change `<OLD_VERSION>` to `<NEW_VERSION>` in all three places.

**WARNING:** The `hermes-agent-src-<VERSION>` volume stores a snapshot of the agent's source tree at image build time. If you update the webui version without also rebuilding/pushing the agent image, the webui container will try to install agent deps from a stale or missing source snapshot. Always bump the agent image first, then the webui.

---

## Bumping Pre-installed Tool Versions

Pre-installed tools are baked into the agent venv in `Dockerfile.agent`. Their versions must match what the agent's `lazy_deps.py` expects so the runtime lazy-install finds them already present.

**1. Find the pinned version in the agent source:**

```
hermes-agent/tools/lazy_deps.py
→ "search.firecrawl": ("firecrawl-py==4.17.0",)
→ "search.exa": ("exa-py==2.10.2",)
→ "search.parallel": ("parallel-web==0.4.2",)
→ "provider.anthropic": ("anthropic==0.87.0",)
→ ... and all other lazy deps
```

**2. Update the corresponding `RUN` line in `Dockerfile.agent`:**

```dockerfile
RUN /usr/local/bin/uv pip install --python /opt/hermes/.venv/bin/python \
    firecrawl-py==<NEW_VERSION> \
    --no-cache-dir
```

**3. Push to `main`** — the CI workflow builds and pushes the new image.

---

## Expanding the Pre-installed Tool Set

To add another tool from `lazy_deps.py` into the baked image, add it to the `uv pip install` command in `Dockerfile.agent`:

```dockerfile
RUN /usr/local/bin/uv pip install --python /opt/hermes/.venv/bin/python \
    firecrawl-py==4.17.0 \
    exa-py==2.10.2 \
    --no-cache-dir
```

Add the `FIRECRAWL_API_KEY` (or whatever API key the tool needs) to the `environment:` block in `docker-compose.yml` so Coolify can inject it.

---

## Current Pinned Versions

| Component | Source | Pinned at |
|-----------|--------|-----------|
| Agent base image | `Dockerfile.agent` `FROM` | `sha256:7f71a4183817c3ac803f29d0b2aedc9ebbab4ec6be9c1d02e1cbda6f5d7ea52b` |
| WebUI image tag | `docker-compose.yml` `hermes-webui.image` | `0.51.671` |
| firecrawl-py | `Dockerfile.agent` `uv pip install` | `4.17.0` |