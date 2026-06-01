# gitlabflow-actions
**GitLab Flow on GitHub Actions** for multi-team, multi-environment delivery. Demonstrates branching strategy, environment promotion, reusable workflows, composite actions.

## Reusable Workflows vs Composite Actions

|  | Reusable Workflow | Composite Action |
|---|---|---|
| Location | `.github/workflows/_*.yml` | `.github/actions/<name>/action.yml` |
| Invocation | `uses: ./.github/workflows/_x.yml` at `jobs:` | `uses: ./.github/actions/x` at `steps:` |
| Owns jobs | yes | no |
| Supports `matrix` | yes | no (caller side only) |
| Own `secrets:` block | yes | no (via inputs) |
| Best for | full stage (CI, deploy) | atomic step (scan, build, push) |

---

## Repository Layout

```
gitflow-actions-playground/
├── README.md
├── docs/                                 
├── .github/
│   ├── CODEOWNERS                        
│   ├── pull_request_template.md
│   ├── actions/                  
│   └── workflows/
│       ├── ci.yml                       
│       ├── build-push.yml               
│       ├── deploy.yml                  
│       ├── qa-suite.yml                 
│       ├── on-feature.yml
│       ├── on-pr.yml
│       ├── on-develop.yml
│       ├── on-release.yml
│       ├── on-main.yml
│       ├── on-tag.yml
│       ├── on-hotfix.yml
│       └── manual-deploy.yml