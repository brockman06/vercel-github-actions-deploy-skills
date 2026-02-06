# Deploy Vercel via GitHub Actions — Complete Setup Guide

A comprehensive guide for setting up the Git Author Override deployment method for Vercel on the free Hobby plan.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Step-by-Step Setup](#step-by-step-setup)
- [How It Works Under the Hood](#how-it-works-under-the-hood)
- [Advanced Configuration](#advanced-configuration)
- [Troubleshooting](#troubleshooting)

---

## Prerequisites

- A GitHub repository with a Vercel-compatible project (Next.js, React, Svelte, etc.)
- A Vercel account (free Hobby plan works)
- Node.js installed locally (for `vercel link`)

---

## Step-by-Step Setup

### 1. Install Vercel CLI and Link Project

```bash
npm install -g vercel
vercel link
```

Follow the prompts to link your repo to a Vercel project. This creates `.vercel/project.json`:

```json
{
  "orgId": "org_xxxxxxxxxxxx",
  "projectId": "prj_xxxxxxxxxxxx"
}
```

Add `.vercel` to your `.gitignore`:

```bash
echo ".vercel" >> .gitignore
```

### 2. Create a Vercel Deploy Token

1. Go to [vercel.com/account/tokens](https://vercel.com/account/tokens)
2. Click **Create Token**
3. Name it `github-actions` (or any name you like)
4. Set scope to your account
5. Copy the token (you won't see it again)

### 3. Add GitHub Secrets

Go to your repository on GitHub:

1. **Settings** → **Secrets and variables** → **Actions**
2. Click **New repository secret** for each:

| Secret Name | Value | Where to Find It |
|-------------|-------|-------------------|
| `VERCEL_TOKEN` | Your deploy token | Step 2 above |
| `VERCEL_ORG_ID` | `orgId` value | `.vercel/project.json` |
| `VERCEL_PROJECT_ID` | `projectId` value | `.vercel/project.json` |
| `DEPLOY_EMAIL` | Vercel account owner's email | Your Vercel account settings |
| `DEPLOY_NAME` | Vercel account owner's name | Your Vercel account settings |

### 4. Create the Workflow File

Create `.github/workflows/deploy.yml` in your repo.

**For Bun projects:**

```yaml
name: Deploy to Vercel

on:
  push:
    branches:
      - main
  workflow_dispatch:

env:
  VERCEL_ORG_ID: ${{ secrets.VERCEL_ORG_ID }}
  VERCEL_PROJECT_ID: ${{ secrets.VERCEL_PROJECT_ID }}

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Override git author to repo owner
        run: |
          git config user.email "${{ secrets.DEPLOY_EMAIL }}"
          git config user.name "${{ secrets.DEPLOY_NAME }}"
          git commit --amend --reset-author --no-edit

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: Install Bun
        uses: oven-sh/setup-bun@v2

      - name: Install Vercel CLI
        run: npm install -g vercel

      - name: Pull Vercel environment
        run: vercel pull --yes --environment=production --token=${{ secrets.VERCEL_TOKEN }}

      - name: Build project
        run: vercel build --prod --token=${{ secrets.VERCEL_TOKEN }}

      - name: Deploy to Vercel
        run: vercel deploy --prebuilt --prod --token=${{ secrets.VERCEL_TOKEN }}
```

**For npm projects:** Remove the "Install Bun" step entirely.

**For pnpm projects:** Replace the "Install Bun" step with:

```yaml
      - name: Install pnpm
        uses: pnpm/action-setup@v4
```

### 5. Commit and Push

```bash
git add .github/workflows/deploy.yml
git commit -m "ci: add Vercel deploy workflow"
git push origin main
```

Go to your repo's **Actions** tab to watch the first deployment.

---

## How It Works Under the Hood

### The Git Author Override

The key step is:

```yaml
- name: Override git author to repo owner
  run: |
    git config user.email "${{ secrets.DEPLOY_EMAIL }}"
    git config user.name "${{ secrets.DEPLOY_NAME }}"
    git commit --amend --reset-author --no-edit
```

This does three things:
1. **Sets the git config** to the Vercel account owner's identity
2. **Amends the latest commit** to change its author to the account owner
3. **`--no-edit`** keeps the original commit message

**Important:** This only happens on the disposable GitHub Actions runner. Your actual repository history is never modified. The runner is destroyed after the job completes.

### The Deploy Pipeline

```
vercel pull    → Fetches project config + environment variables from Vercel
vercel build   → Builds the project locally (on the runner)
vercel deploy  → Uploads the pre-built output to Vercel's edge network
```

The `--prebuilt` flag on deploy tells Vercel to skip building again and use the output from the previous build step.

---

## Advanced Configuration

### Prevent Double Deploys

If the Vercel account owner pushes, both Vercel's Git integration and GitHub Actions will deploy. To prevent this:

1. Go to **Vercel** → **Project Settings** → **Git**
2. Set **Ignored Build Step** to: `exit 0`

This disables Vercel's built-in Git-triggered deployments entirely.

### Preview Deploys on Pull Requests

```yaml
name: Preview Deploy

on:
  pull_request:
    branches:
      - main

env:
  VERCEL_ORG_ID: ${{ secrets.VERCEL_ORG_ID }}
  VERCEL_PROJECT_ID: ${{ secrets.VERCEL_PROJECT_ID }}

jobs:
  deploy-preview:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Override git author to repo owner
        run: |
          git config user.email "${{ secrets.DEPLOY_EMAIL }}"
          git config user.name "${{ secrets.DEPLOY_NAME }}"
          git commit --amend --reset-author --no-edit

      - uses: actions/setup-node@v4
        with:
          node-version: 20

      - run: npm install -g vercel

      - run: vercel pull --yes --environment=preview --token=${{ secrets.VERCEL_TOKEN }}

      - run: vercel build --token=${{ secrets.VERCEL_TOKEN }}

      - name: Deploy preview
        id: deploy
        run: |
          URL=$(vercel deploy --prebuilt --token=${{ secrets.VERCEL_TOKEN }})
          echo "url=$URL" >> $GITHUB_OUTPUT

      - name: Comment preview URL on PR
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `Preview deployed to: ${{ steps.deploy.outputs.url }}`
            })
```

### Deploy Only When Specific Files Change

```yaml
on:
  push:
    branches:
      - main
    paths:
      - 'src/**'
      - 'public/**'
      - 'package.json'
      - 'next.config.*'
```

### Add Build Notifications (Slack)

Add this step at the end of your workflow:

```yaml
      - name: Notify Slack
        if: always()
        uses: 8398a7/action-slack@v3
        with:
          status: ${{ job.status }}
          fields: repo,message,commit,author,action,eventName,ref
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

### Monorepo Support

If you have multiple Vercel projects in one repo, create separate workflow files for each:

```yaml
# .github/workflows/deploy-frontend.yml
on:
  push:
    branches: [main]
    paths: ['apps/frontend/**']

env:
  VERCEL_ORG_ID: ${{ secrets.VERCEL_ORG_ID }}
  VERCEL_PROJECT_ID: ${{ secrets.VERCEL_FRONTEND_PROJECT_ID }}
```

```yaml
# .github/workflows/deploy-docs.yml
on:
  push:
    branches: [main]
    paths: ['apps/docs/**']

env:
  VERCEL_ORG_ID: ${{ secrets.VERCEL_ORG_ID }}
  VERCEL_PROJECT_ID: ${{ secrets.VERCEL_DOCS_PROJECT_ID }}
```

---

## Troubleshooting

### "Error: No commits found"

The `git commit --amend` step fails if there are no commits. This should not happen in a normal workflow since the checkout step fetches the repo with its history.

**Fix:** Ensure the checkout step uses the default `fetch-depth` (1 is fine).

### "Error: VERCEL_TOKEN is not set"

GitHub Secrets are case-sensitive. Verify the secret name matches exactly: `VERCEL_TOKEN`, not `vercel_token`.

### Build works locally but fails in CI

- Check that all required environment variables are set in your Vercel project dashboard
- `vercel pull` fetches env vars automatically, but they must exist in Vercel first
- Ensure Node.js version matches your local setup

### Deploy succeeds but site doesn't update

- Check if Vercel's Git integration is also deploying (causing a race condition)
- Set **Ignored Build Step** to `exit 0` in Vercel project settings

### "Error: Rate limited"

Vercel has API rate limits. If deploying frequently:
- Add concurrency control to prevent parallel deploys:

```yaml
concurrency:
  group: vercel-deploy
  cancel-in-progress: true
```
