from pathlib import Path

content = """# Data-Platform Workspace

_A unified mono-repo for ingestion pipelines, shared business libraries and deployable services – powered by **[uv](https://docs.astral.sh/uv)**._

---

## 1. What is this repo?

* **Mono-repo**: All Python code – three shared libraries and two production apps – lives here so we evolve them together.
* **uv workspace**: One lock-file & one virtual-env for everyone; each package stays editable.  ([docs.astral.sh](https://docs.astral.sh/uv/concepts/projects/workspaces/?utm_source=chatgpt.com))
* **GitLab CI**: A slim pipeline lint-tests-builds everything using the same `uv.lock`.

> **New to mono-repos?** Think of the repo root as a _meta-project_ that only tracks members & dependencies; the real packages live one folder deeper.

---

## 2. Quick-start (first 5 minutes)

```bash
# 1. Grab uv (fast installer)
curl -Ls https://astral.sh/uv/install | bash    # or `pipx install uv`

# 2. Clone and enter the repo
 git clone git@gitlab.com:myco/data-platform.git
 cd data-platform

# 3. Sync *everything* into .venv (1-time ~30 s)
 uv sync --all-packages                          # installs Python if missing  ([docs.astral.sh](https://docs.astral.sh/uv/guides/install-python/?utm_source=chatgpt.com))

# 4. Run the smoke tests
 uv run pytest -q
```

That’s it – your editor will auto-discover the `.venv` in `repo-root/.venv`.

---

## 3. Repository layout

```
repo-root/
│
├── pyproject.toml      # workspace root (not published)
├── uv.lock             # deterministic deps for *all* members
│
├── packages/
│   ├── common-utils/       # logging & helpers
│   ├── data-models/        # Pydantic schemas
│   └── analytics-core/     # business logic
│
├── apps/
│   ├── ingestion-svc/      # ETL & batch jobs
│   └── reporting-dash/     # FastAPI + React UI
└── .gitlab-ci.yml
```

Every library/app contains its own `pyproject.toml` and `src/<import_name>/…` tree.

---

## 4. Daily developer workflow

| Task | Command |
|------|---------|
| **Install / update deps** | `uv sync` (`--all-packages` for entire workspace) |
| **Run a tool** | `uv run ruff check .` or `uv run --package ingestion-svc my-cli --help` ([docs.astral.sh](https://docs.astral.sh/uv/reference/cli/?utm_source=chatgpt.com)) |
| **Add a runtime dep** | `uv add --package data-models pydantic[email]` |
| **Add a dev-only dep** | `uv add --package data-models --dev pytest` |
| **Start an app** | `uv run ingestion` or `uv run --package reporting-dash reporting` |
| **Run all tests** | `uv run pytest -q` |
| **Regenerate lock-file** | `uv lock` |

### 4.1 `--package` explained

`--package <member>` lets you target a specific library or app from the repo root.  Without it, commands act on the folder you are _inside_.

### 4.2 Dependency groups

* **Runtime** deps live under `[project].dependencies` → ship with your wheel.
* **Dev / lint / docs** deps live under `[[dependency-groups]]` and are installed only when requested. Use `--dev` (alias for `--group dev`) during `uv add`. citeturn21view0

To install extra groups in your env:

```bash
uv sync --group docs --group lint    # adds docs+lint on top of defaults
```

---

## 5. Coding guidelines

* **src-layout** only: put code in `src/<import_name>/`, never in the package root.
* **Type-safe libraries** (`ruff`, `mypy`, `deptry`) – enforced in CI.
* **One feature branch per MR**; squash-merge with Conventional Commits (`feat: ingest csv`).
* After adding/removing deps, commit both the member `pyproject.toml` **and** `uv.lock`.

---

## 6. Testing & Linting

```bash
# run locally (all members)
uv run ruff check .            # style & unused import guard
uv run deptry .                # missing/unused deps  ([docs.astral.sh](https://docs.astral.sh/uv/concepts/projects/dependencies/?utm_source=chatgpt.com))
uv run pytest -q               # unit tests

# narrow to one package
uv run --package common-utils pytest -q
```

Coverage reports are generated in CI and surfaced in GitLab.

---

## 7. Building & running apps

```bash
# wheel / sdist for one app (local)
uv build --package ingestion-svc --out dist/

# docker image (example)
docker build -t myco/ingestion -f apps/ingestion-svc/Dockerfile .
```

For production we use Docker images built in a dedicated `release` pipeline.

---

## 8. Release process (TL;DR)

1. Merge feature branches into `main`.
2. Bump versions with `hatch version patch` in each changed member **or** run the helper script `scripts/bump-all.sh`.
3. Tag once: `git tag ingestion-svc/1.1.0 && git push --tags`.
4. GitLab release job builds wheels & pushes them to the internal Package Registry.

---

## 9. Troubleshooting FAQ

| Symptom | Fix |
|---------|-----|
| **Module not found** after `git pull` | Run `uv sync --all-packages` (new deps added by a teammate). |
| `uv run …` says *No such package* | Spell the member name exactly as in `packages/` or `apps/`. |
| Accidentally added wrong dep | `uv remove <name>` then `uv lock`. |
| Double `src/src/…` after `uv init` | You ran `uv init` twice – delete the inner `src` folder, keep a single level. |

---

## 10. Further reading

* **uv docs** – <https://docs.astral.sh/uv> (workspaces, CLI, dependency groups) ([docs.astral.sh](https://docs.astral.sh/uv/concepts/projects/workspaces/?utm_source=chatgpt.com), [docs.astral.sh](https://docs.astral.sh/uv/concepts/projects/dependencies/?utm_source=chatgpt.com))
* **Monorepo patterns** – <https://federico.is/posts/2024/12/18/managing-python-workspaces-with-uv/> ([federico.is](https://federico.is/posts/2024/12/18/managing-python-workspaces-with-uv/?utm_source=chatgpt.com))
* **Lock-file rationale** – DataCamp tutorial <https://www.datacamp.com/tutorial/python-uv> ([datacamp.com](https://www.datacamp.com/tutorial/python-uv?utm_source=chatgpt.com))

---

Happy hacking – and ping @dev-infra if anything here is unclear!
"""

Path("/mnt/data/README.md").write_text(content)
