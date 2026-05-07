# GitHub Environments and approval gates

The CD workflow in `Fifa_API/.github/workflows/cd.yml` deploys to dev automatically and to prod only after a human approves. The approval gate is implemented via GitHub Environments.

## What you need to do, once, in the GitHub UI

These are repository-level settings on the `Fifa_API` repo (the workflow's home), NOT on this platform repo.

1. Open `https://github.com/ajugulum123/Fifa_API/settings/environments`
2. Click **New environment**
3. Name: `dev`
   - No protection rules. dev deploys are auto-approved.
   - Optional: add deployment branch rule allowing only `main`.
4. **New environment** again. Name: `production`
   - Under **Deployment protection rules**, enable **Required reviewers** and add yourself (or whoever holds the prod gate).
   - Optional: enable **Wait timer** for, say, 5 minutes so you can hit cancel after reading the diff.
   - Under **Deployment branches and tags**, restrict to `main` only.

## What the workflow does

The `deploy-dev` job has:

```yaml
environment:
  name: dev
  url: https://fifa-api.dev.local:8443
```

The `deploy-prod` job has:

```yaml
environment:
  name: production
  url: https://fifa-api.example.com
```

When `deploy-prod` is queued, GitHub holds it and emails the listed reviewers. The job stays `pending` until someone clicks **Review deployments** on the run page and approves.

## Per-environment secrets

You can store environment-scoped secrets that are visible only to jobs targeting that environment. Useful for prod-only credentials like a real cluster's kubeconfig if you ever move off in-cluster runners.

```bash
gh secret set CACHE_ACCESS_KEY --env production --body "<value>" --repo ajugulum123/Fifa_API
```

## Smoke testing the approval flow

```bash
# Push a trivial change to main (docs/comment) to fire the workflow
echo "# triggered $(date -u +%FT%TZ)" >> README.md
git add README.md
git commit -m "trigger CD smoke"
git push origin main
```

Watch:
- `deploy-dev` runs and applies to `myapp-dev`. Auto.
- `deploy-prod` queues and waits.
- Approve via the GitHub UI or `gh run approve <run-id> --repo ajugulum123/Fifa_API`.
- `deploy-prod` runs and applies to `myapp-prod`.
