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

> **Deployment targets:** This workflow deploys to real AWS EC2 instances via SSH.
> - **Staging** → `vikramjeet-ec1`
> - **Production** → `vikramjeet-ec2`
>
> Deployment uses [`appleboy/scp-action`](https://github.com/appleboy/scp-action) to copy `app.py` and `requirements.txt` to the instance, followed by [`appleboy/ssh-action`](https://github.com/appleboy/ssh-action) to install dependencies in a virtual environment and (re)start the Flask app in the background.

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

| Secret Name       | Purpose                                                      |
|-------------------|----------------------------------------------------------------|
| `STAGING_HOST`    | Public IP address of the staging EC2 instance (`vikramjeet-ec1`) |
| `STAGING_SSH_KEY` | Full contents of the `.pem` private key used to SSH into staging |
| `PROD_HOST`       | Public IP address of the production EC2 instance (`vikramjeet-ec2`) |
| `PROD_SSH_KEY`    | Full contents of the `.pem` private key used to SSH into production |

The SSH user for both instances is `ec2-user` (default for Amazon Linux AMIs), hardcoded in the workflow file.

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