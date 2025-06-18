# Mono‑Repo Guide – **UV Workspace × GitLab CI**

> **Audience:** All engineers – even if you’ve never touched a mono‑repo, wheels, or UV before.  
> **Repo:** `my-monorepo/` (root of this document)

---

## 1  Why this repo exists

| Aspect | Benefit |
|--------|---------|
| **Single repo, many packages** | One PR pipeline, shared conventions, easy refactors |
| **UV workspace** | One lock‑file (`uv.lock`) & one virtual‑env (`.venv`) for *every* member → no version drift |
| **GitLab CI + internal PyPI** | Deterministic wheels, no Docker required |

---

## 2  Repo layout

```text
my-monorepo/
├─ src/my_monorepo/              # optional root library
├─ packages/package_a/src/package_a
├─ apps/app_a/src/app_a
├─ configs/      resources/      tests/
├─ pyproject.toml  uv.lock  .gitlab-ci.yml
```

*(If `src/my_monorepo/` is unused, set `tool.uv.package=false` and delete the folder.)*

---

## 3  First‑time setup

```bash
curl -Ls https://astral.sh/uv/install | bash   # 1) install uv
git clone git@gitlab.com:myco/my-monorepo.git  # 2) clone
cd my-monorepo
uv sync --all-packages                         # 3) install deps
uv run pytest -q                               # 4) smoke test
```

---

## 4  Daily commands

| Need | Command |
|------|---------|
| Sync deps (dev) | `uv sync` |
| Runtime‑only sync (prod) | `uv sync --no-default-groups` |
| Add dep | `uv add --package package-a rich` |
| Add dev dep | `uv add --package package-a --dev pytest` |
| Lint | `uv run ruff check .` |
| Test | `uv run pytest -q` |
| Build wheel | `uv build --package app-a` |
| Run app | `uv run --package app-a app-a --help` |

---

## 5  GitLab CI

See `.gitlab-ci.yml` snippet included earlier – stages: **lint → test → build → publish → deploy**. Wheels upload to `${PRIVATE_PYPI}` and servers pull from there.

---

## 6  Server deployment (no Docker)

1. CI calls `scripts/deploy-app-a.sh "$CI_COMMIT_TAG"` on success.  
2. Script SSHes to host, makes new `venv`, installs `app-a==$VERSION` from private PyPI, flips `/opt/app-a/current` symlink, restarts systemd service.  
3. Rollback = point symlink to previous release and restart.

---

## 7  Internal PyPI config

`~/.config/pip/pip.conf` on every host:

```ini
[global]
index-url = https://gitlab.example.com/api/v4/projects/<id>/packages/pypi/simple
extra-index-url = https://pypi.org/simple
trusted-host = gitlab.example.com
```

---

## 8  Cheat‑sheet

```bash
uv sync --all-packages          # full dev install
uv lock                         # regenerate lock‑file
uv add --group docs mkdocs      # docs extra
uv build --package package-a
twine upload -r "$PRIVATE_PYPI" dist/*
```

---

## 9  Troubleshooting

| Problem | Fix |
|---------|-----|
| Missing module after pull | `uv sync --all-packages` |
| Lock mismatch in CI | `uv lock` then commit |
| Double `src/src` | Delete inner `src` (ran `uv init` twice) |
| Wheel not found on host | Ensure publish job succeeded and pip.conf points at proxy |

---

*Questions?* Ping **#dev‑infra** or open an infra ticket.
