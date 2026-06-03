# gitflow-actions-playground

Promotion flow `dev → qa → preprod → prod` on GitHub Actions.

## Idea

**Build once on `dev`. Promote the same version everywhere else.**

Version = commit SHA (`github.sha`). Each env branch deploys independently — no rebuild.

```
dev       qa        preprod      prod
 │         │           │           │
 BUILD     deploy      deploy      deploy
 deploy    smoke       smoke       smoke
 smoke     e2e         e2e
```

`preprod` and `prod` are **separate envs**. Prod is reached by merge `preprod → prod`, not auto-deploy from preprod workflow.

## Workflows

| Branch | What runs |
|---|---|
| `dev` | build → deploy → smoke |
| `qa` | deploy → smoke + e2e |
| `preprod` | deploy → smoke + e2e |
| `prod` | deploy → smoke (+ approval via GitHub Environment) |
| `perf` | separate env, manual |

## Key files

| File | Role |
|---|---|
| `ci.yml` | Build artifact + push image (dev only) |
| `deploy.yml` | Deploy version to env. Outputs `app_url` |
| `smoke.yml` / `e2e.yml` | Tests using `base_url` from deploy output |
| `perf.yml` | Isolated perf env |

## GitHub Environments

Create `dev`, `qa`, `preprod`, `prod`, `perf` in repo settings. Add **required reviewers** on `prod`.
test
x
