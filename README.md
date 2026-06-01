# gitflow-actions-playground

**GitLab Flow on GitHub Actions** for multi-team, multi-environment delivery. Demonstrates branching strategy, environment promotion, reusable workflows, and composite actions.

> All jobs are **echo only** — the goal is to demonstrate **flow and structure**, not to execute real work.

---

## Team

| Role | Count | Responsibility |
|---|---|---|
| Developers | 4 | Code, PRs into `Xdev`; own `lint` & `test` |
| QA Engineers | 2 | Validate `Xqa` and `Xpreprod`; own `e2e-test` |
| DevOps Engineers | 2 | Own pipeline, composite actions, environments, prod approvals |

Two parallel streams: **Stream 1** (`1*`) and **Stream 2** (`2*`), each with 2 devs + 1 QA + 1 DevOps.

---

## Branching Model

```
Stream 1:  1dev ──► 1qa ──► 1preprod ──► 1prod
Stream 2:  2dev ──► 2qa ──► 2preprod ──► 2prod
```

| Branch / Env | Trigger | Pipeline |
|---|---|---|
| `Xdev`, `Xqa` | `push`, `pull_request` | `ci.yml` + `deploy.yml` |
| `Xpreprod` | `push`, `pull_request` | `ci.yml` + `cd.yml` (full promotion to prod) |
| `Xprod` | hotfix only | restricted via branch protection |

> **Standard path to prod:** `Xpreprod` → manual approval → `Xprod`. Direct push to `Xprod` is hotfix-only.

---

## Reusable Workflows vs Composite Actions

|  | Reusable Workflow | Composite Action |
|---|---|---|
| Location | `.github/workflows/*.yml` | `.github/actions/<name>/action.yml` |
| Invocation | `uses: ./.github/workflows/x.yml` at `jobs:` | `uses: ./.github/actions/x` at `steps:` |
| Owns jobs | yes | no |
| Supports `matrix` | yes | no (caller side only) |
| Own `secrets:` block | yes | no (via inputs) |
| Best for | full stage (CI, CD, deploy) | atomic step (lint, build, deploy step) |

---

## Repository Layout

```
gitflow-actions-playground/
├── README.md
├── .github/
│   ├── CODEOWNERS
│   ├── pull_request_template.md
│   │
│   ├── actions/                          # Composite actions (echo only)
│   │   ├── setup-cache/action.yml
│   │   ├── lint/action.yml
│   │   ├── test/action.yml
│   │   ├── build/action.yml
│   │   ├── docker/action.yml
│   │   ├── deploy/action.yml
│   │   ├── smoke-test/action.yml
│   │   ├── e2e-test/action.yml
│   │   └── performance-test/action.yml
│   │
│   └── workflows/
│       ├── ci.yml                        # Reusable: lint → test → build → docker
│       ├── cd.yml                        # Reusable: stage → smoke → e2e → perf → approval → prod
│       ├── deploy.yml                    # Reusable: simple deploy for non-prod envs
│       │
│       ├── env-1dev.yml                  # Trigger for branch 1dev
│       ├── env-1qa.yml                   # Trigger for branch 1qa
│       ├── env-1preprod.yml              # Trigger for branch 1preprod (calls cd.yml)
│       ├── env-1prod.yml                 # Hotfix-only trigger for branch 1prod
│       │
│       ├── env-2dev.yml
│       ├── env-2qa.yml
│       ├── env-2preprod.yml
│       └── env-2prod.yml
```

---

## Pipeline Stages

**CI:** `lint → test → build → docker`

**CD (preprod → prod):** `deploy-stage → smoke-test → e2e-test → performance-test → [APPROVAL] → deploy-prod`

```
   PR / push to Xdev|Xqa|Xpreprod|Xprod
              │
              ▼
   ┌─────────────────────────┐
   │ env-X<env>.yml          │   per-env trigger
   └─────────────────────────┘
              │
              ▼
   ┌─────────────────────────┐
   │ ci.yml (reusable)       │   lint → test → build → docker
   └─────────────────────────┘
              │  (push only, not PR)
              ▼
   ┌─────────────────────────┐
   │ deploy.yml  (Xdev/Xqa)  │   simple deploy
   │ cd.yml      (Xpreprod)  │   stage → smoke → e2e → perf → approval → prod
   └─────────────────────────┘
```

---

## Reusable Workflows

| File | Purpose | Inputs | Outputs |
|---|---|---|---|
| `ci.yml` | CI pipeline | `environment` | `image-tag` (git SHA) |
| `deploy.yml` | Simple deploy (dev/qa) | `environment`, `image-tag` | — |
| `cd.yml` | Full CD with preprod → prod promotion | `stage_env`, `prod_env`, `image-tag` | — |

**Principles:**
- **Single immutable artifact** (`image-tag = git SHA`) is promoted across environments — no rebuilds between stages.
- `cd.yml` is used **only from `Xpreprod`**; other envs use `deploy.yml`.
- Approval is a **native GitHub Environment protection rule** on `Xprod`, not a fake `echo "approved"` job.

---

## Composite Actions

| Action | Inputs | Outputs | Demo behavior |
|---|---|---|---|
| `setup-cache` | `cache-key-prefix` | — | Mock cache restore |
| `lint` | — | — | Mock static analysis |
| `test` | — | — | Mock unit tests |
| `build` | `image-tag` | `artifact-path`, `image-tag` | Creates `dist/app.bundle` |
| `docker` | `image-tag` | — | Mock `docker build` + `push` |
| `deploy` | `environment`, `image-tag` | — | Mock deploy |
| `smoke-test` | — | — | Mock health checks |
| `e2e-test` | — | — | Mock E2E scenarios |
| `performance-test` | — | — | Mock load test |

**Why composite actions:**
1. **DRY** — same step (`lint`, `deploy`) reused across multiple workflows.
2. **Stable contract** via `inputs` / `outputs` — workflows don't depend on internal implementation.
3. **Easy upgrade path** — replace one `action.yml` with real logic; all workflows pick it up.

---

## Promotion Flow

```
┌────────┐ merge ┌────────┐ merge ┌──────────┐ approval ┌────────┐
│ 1dev   │ ────► │ 1qa    │ ────► │ 1preprod │ ───────► │ 1prod  │
└────────┘       └────────┘       └──────────┘          └────────┘
 ci+deploy        ci+deploy        ci + full CD          (via cd.yml)
                                   + smoke/e2e/perf
```