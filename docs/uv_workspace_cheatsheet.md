
# UV Workspace Monorepo & Deployment **Cheat Sheet**
*(pure‑Python, no‑Docker, GitLab + Nexus 2025 edition)*  

---

## 1  Project layout

```text
your-repo/
├─ pyproject.toml            ← workspace root (not published)
├─ uv.lock                   ← single lockfile for *all* members
├─ .python-version           ← runtime pin, e.g. `3.12.3`
├─ packages/
│  └─ awesome_lib/
│     ├─ pyproject.toml
│     └─ src/awesome_lib/…
├─ apps/
│  └─ web_api/
│     ├─ pyproject.toml
│     └─ src/web_api/…
└─ scripts/
   └─ deploy.sh              ← atomic rollout script (see §6)
```

*Use `uv init` + `[tool.uv.workspace]` to generate the scaffold.*

---

## 2  Root `pyproject.toml`

```toml
[project]
name            = "monorepo-root"
version         = "0.0.0"
requires-python = ">=3.12"
description     = "Workspace aggregator – never published"

[tool.uv.workspace]
members = ["packages/*", "apps/*"]

[[tool.uv.index]]
name        = "corp-release"
url         = "https://nexus.example.com/repository/pypi-release/simple/"
publish-url = "https://nexus.example.com/repository/pypi-release/legacy/"
default     = true

[[tool.uv.index]]
name        = "corp-snapshot"
url         = "https://nexus.example.com/repository/pypi-snapshot/simple/"
publish-url = "https://nexus.example.com/repository/pypi-snapshot/legacy/"
explicit    = true

[tool.uv]
native-tls          = true
allow-insecure-host = ["nexus.example.com"]

[project.optional-dependencies]
dev = ["ruff>=0.4", "pytest>=8.2", "pytest-cov>=5", "black>=24.4"]

[build-system]
requires      = ["hatchling"]
build-backend = "hatchling.build"
```

---

## 3  Member packages

### 3.1 Library `packages/awesome_lib/pyproject.toml`

```toml
[project]
name            = "awesome-lib"
version         = "0.1.0"
requires-python = ">=3.12"
description     = "Reusable helpers"
dependencies    = ["requests>=2.32"]

[build-system]
requires      = ["hatchling"]
build-backend = "hatchling.build"
```

### 3.2 Application `apps/web_api/pyproject.toml`

```toml
[project]
name            = "web-api"
version         = "0.4.2.dev0"                 # dev build → snapshot
requires-python = ">=3.12"
dependencies    = [
  "fastapi>=0.111",
  "uvicorn[standard]>=0.30",
  "awesome-lib==0.1.0",
]

[tool.uv.sources]
awesome-lib = { workspace = true }

[project.scripts]
web-api = "web_api.main:cli"

[build-system]
requires      = ["hatchling"]
build-backend = "hatchling.build"
```

---

## 4  Daily commands (developer)

| Task | Command |
|------|---------|
| Sync all + dev tools | `uv sync --dev --all-extras` |
| Lint & format | `uvx ruff check .` / `uvx ruff format .` |
| Run tests | `uv run pytest -q` |
| Add dependency | `uv add requests` |
| Bump patch | `hatch version patch` |
| Build wheels + sdist | `uv build --no-sources` |
| Publish snapshot | `uv publish --index corp-snapshot` |

---

## 5  GitLab CI pipeline (`.gitlab-ci.yml`)

```yaml
image: ghcr.io/astral-sh/uv:0.7-python3.12-bookworm-slim
variables:
  UV_CACHE_DIR: .uv-cache
  UV_PUBLISH_TOKEN: $NEXUS_TOKEN
  UV_NATIVE_TLS: "1"
  UV_INSECURE_HOST: "nexus.example.com"
cache:
  key: { files: [uv.lock] }
  paths: [ $UV_CACHE_DIR ]
stages: [lint, test, build, prepare, publish]

lint:
  stage: lint
  script:
    - uv sync --dev --locked --all-extras
    - uv run ruff check .

test:
  stage: test
  needs: [lint]
  script:
    - uv sync --dev --locked --all-extras
    - uv run pytest -q --cov=src
  artifacts:
    reports: { junit: pytest.xml }

build:
  stage: build
  needs: [test]
  script: |
    uv run uv build --no-sources
  artifacts:
    paths: [dist/**]

prepare_bundle:
  stage: prepare
  needs: [build]
  script: |
    mkdir deploy
    cp uv.lock deploy/
    cp dist/*.whl deploy/
    tar -czf web-api-${CI_COMMIT_TAG}.tgz -C deploy .
  artifacts:
    paths: [web-api-${CI_COMMIT_TAG}.tgz]
  rules:
    - if: $CI_COMMIT_TAG

publish_snapshot:
  stage: publish
  needs: [build]
  script: |
    uv run uv publish --index corp-snapshot
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
```

---

## 6  `scripts/deploy.sh` (run on the server)

```bash
#!/usr/bin/env bash
set -euo pipefail
APP="web_api"
APP_ROOT="/opt/${APP}"
SERVICE="web-api.service"
REV="${1:?version tag}"
BUNDLE="${2:-}"
WORKDIR=$(mktemp -d)
echo "Deploying $REV → $WORKDIR"

if [[ -n "$BUNDLE" ]]; then
  cp "$BUNDLE" "$WORKDIR/bundle.tgz"
else
  curl -sfL "https://gitlab.example.com/api/v4/projects/123/artifacts/${REV}/download?job=prepare_bundle" \
       -o "$WORKDIR/bundle.tgz"
fi

DEST="${APP_ROOT}/${REV}"
sudo mkdir -p "$DEST"
sudo tar -xzf "$WORKDIR/bundle.tgz" -C "$DEST"

pushd "$DEST"
uv python install
uv venv .venv
uv sync --locked --no-editable --index corp-release
popd

sudo ln -sfn "$DEST" "${APP_ROOT}/current"
sudo systemctl restart "$SERVICE"
echo "✅ Done"
```

*Rollback:*  
```bash
sudo ln -sfn /opt/web_api/<old-tag> /opt/web_api/current
sudo systemctl restart web-api.service
```

---

## 7  Direct‑from‑Nexus alternative (skip bundle)

```bash
git checkout v0.4.3
uv python install
uv venv .venv
uv sync --package web-api --locked --no-editable --index corp-release
systemctl restart web-api.service
```

---

## 8  Best‑practice recap

1. **One lockfile** governs the entire workspace.  
2. Use `uv sync --locked --no-editable` for every production install.  
3. Publish **both** wheel (`py3-none-any`) and sdist.  
4. Separate **snapshot** vs. **release** Nexus repos.  
5. Keep each release in its own dir; promote via symlink → zero‑downtime rollback.  
6. Never call `uv pip install` in CI or prod—use `uv sync`.  
7. Maintain TLS / proxy opts in `tool.uv` or `~/.config/uv/uv.toml`.  

---

### Glossary

| Term | Meaning |
|------|---------|
| **uv** | Fast Python packaging tool (venv, lock, build, publish). |
| **Workspace** | Multiple packages managed by one lockfile. |
| **Snapshot repo** | Nexus repo for `*.dev*` or `*.rc*` versions. |
| **Release repo** | Nexus repo for final, immutable versions. |
| **`--no-sources`** | Ignore `[tool.uv.sources]` overrides during build. |
| **`--no-editable`** | Install wheels, never editable links, during sync. |

---

*Cheat sheet compiled 2025‑06‑23 (Europe/London).*  
