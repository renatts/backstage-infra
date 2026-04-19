# Contributing

## Branching strategy

- `main` is protected. No direct pushes. No force-push. No deletion.
- Every change lands via a pull request targeting `main`.
- One logical change per branch. Keep PRs small and reviewable.

### Branch naming

Use a type prefix + short kebab-case slug:

| Prefix      | Use for                              | Example                          |
|-------------|--------------------------------------|----------------------------------|
| `feat/`     | New infrastructure capability        | `feat/networkpolicy-prd`         |
| `fix/`      | Bug fix                              | `fix/dev-ingress-tls-mismatch`   |
| `chore/`    | Deps, cleanup, bumps                 | `chore/bump-backstage-chart-2.7` |
| `docs/`     | Docs-only                            | `docs/kind-bootstrap`            |
| `refactor/` | Restructure without behaviour change | `refactor/slim-base-values`      |
| `ci/`       | CI / workflow changes                | `ci/add-kustomize-render-check`  |

### Commit messages

[Conventional Commits](https://www.conventionalcommits.org/):

```
<type>(<scope>): <subject>

<body — optional, explains *why*>
```

Scope should hint at the area (`dev`, `prd`, `argocd`, `base`, `kind`).

### Pull request flow

1. Branch off `main`: `git switch -c feat/<slug>`.
2. Commit and push the branch.
3. Open a PR targeting `main`. Fill the PR body with **what** changed and **why**.
4. Render both overlays locally before pushing:
   ```bash
   kustomize build --enable-helm kustomize/overlays/dev
   kustomize build --enable-helm kustomize/overlays/prd
   ```
5. Resolve all review conversations before merge.
6. **Squash merge** only. The branch is deleted automatically on merge.
7. Rebase on `main` if the branch is behind (linear history required).

### Argo CD sync

Argo CD auto-syncs `main` to every cluster defined in the ApplicationSet. A
merge to `main` is a deploy. Review carefully; prefer `selfHeal: false`
temporarily if a risky change needs a manual sync step.
