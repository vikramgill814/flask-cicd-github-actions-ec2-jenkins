# Flask CI/CD Pipeline with GitHub Actions

A minimal Flask application used to demonstrate a complete CI/CD pipeline using **GitHub Actions**.

## Application

- `app.py` — Flask app with two routes: `/` and `/health`
- `test_app.py` — pytest unit tests for both routes
- `requirements.txt` — Flask + pytest

Run locally:
```bash
pip install -r requirements.txt
python app.py
```

Run tests locally:
```bash
pytest -v
```

## CI/CD Workflow (`.github/workflows/ci-cd.yml`)

The workflow triggers on:
- **Push to `main`** — runs install, test, and build stages
- **Push to `staging`** — runs install, test, build, then **deploys to staging**
- **Release published** (i.e. a version tag is created and published as a GitHub Release) — runs install, test, build, then **deploys to production**

### Jobs

1. **install-and-test** — installs dependencies with `pip` and runs the pytest suite
2. **build** — packages the app into a zip artifact and uploads it as a build artifact
3. **deploy-staging** — runs only when the triggering branch is `staging`; downloads the build artifact and deploys it (see note below)
4. **deploy-production** — runs only when a GitHub Release is published; downloads the build artifact and deploys it (see note below)

> **Note on deployment steps:** The actual deploy commands in this workflow are placeholders (`echo` statements) since there is no live staging/production server wired up in this demo. In a real deployment, the commented-out `scp`/`ssh` lines would be uncommented and pointed at a real target server (for example, an EC2 instance), using the SSH key and any API token stored in GitHub Secrets.

## Branch Setup

This workflow expects two branches:
- `main` — primary development branch
- `staging` — triggers a staging deployment on every push

Create the staging branch if it doesn't exist:
```bash
git checkout -b staging
git push -u origin staging
```

## GitHub Secrets Configuration

Go to **Repository → Settings → Secrets and variables → Actions → New repository secret** and add:

| Secret Name          | Purpose                                    |
|-----------------------|---------------------------------------------|
| `STAGING_DEPLOY_KEY` | SSH/deploy key for the staging server        |
| `STAGING_API_TOKEN`  | API token used during staging deployment     |
| `PROD_DEPLOY_KEY`    | SSH/deploy key for the production server     |
| `PROD_API_TOKEN`     | API token used during production deployment  |

These are referenced in the workflow via `${{ secrets.SECRET_NAME }}` and are never printed in logs.

## Triggering a Deployment

- **Staging:** push any commit to the `staging` branch
  ```bash
  git checkout staging
  git merge main
  git push origin staging
  ```
- **Production:** create and publish a GitHub Release (tag a version, e.g. `v1.0.0`) from the repository's **Releases** page

## Viewing Pipeline Runs

Go to the **Actions** tab of the repository to see each workflow run, with per-job logs for install/test, build, and deploy stages.
