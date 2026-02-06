# Example Workflow Files

Ready-to-use GitHub Actions workflow files for deploying Vercel projects using the Git Author Override method.

## Pick Your File

| File | Package Manager | When to Use |
|------|----------------|-------------|
| `deploy-bun.yml` | Bun | Your project has a `bun.lock` file |
| `deploy-npm.yml` | npm | Your project has a `package-lock.json` file |
| `deploy-pnpm.yml` | pnpm | Your project has a `pnpm-lock.yaml` file |

## Usage

1. Choose the file matching your package manager
2. Copy it to `.github/workflows/deploy.yml` in your repo
3. Set up the required GitHub Secrets (see below)
4. Push to `main`

## Required GitHub Secrets

All three workflows require the same 5 secrets:

| Secret | Value |
|--------|-------|
| `VERCEL_TOKEN` | Token from [vercel.com/account/tokens](https://vercel.com/account/tokens) |
| `VERCEL_ORG_ID` | `orgId` from `.vercel/project.json` |
| `VERCEL_PROJECT_ID` | `projectId` from `.vercel/project.json` |
| `DEPLOY_EMAIL` | Email of the Vercel account owner |
| `DEPLOY_NAME` | Name of the Vercel account owner |

## How to Get Org ID and Project ID

```bash
npx vercel link
```

This creates `.vercel/project.json` containing both values. Make sure `.vercel` is in your `.gitignore`.

## Customization

### Change the deploy branch

Replace `main` with your branch name:

```yaml
on:
  push:
    branches:
      - production  # or any branch
```

### Add preview deploys for PRs

```yaml
on:
  pull_request:
    branches:
      - main
```

And remove `--prod` from the build and deploy steps.

### Add build caching

For faster builds, add a cache step before the build:

```yaml
- name: Cache Vercel build
  uses: actions/cache@v4
  with:
    path: .vercel/output
    key: vercel-build-${{ hashFiles('**/package-lock.json') }}
```
