# Flask CI/CD Pipeline with GitHub Actions

A complete CI/CD pipeline for a Flask application, built with **GitHub Actions**, deploying to real AWS EC2 instances via SSH. This covers install → test → build → deploy-to-staging → deploy-to-production, with staging and production pointing at two separate EC2 servers.

## Application

- `app.py` — Flask app with two routes: `/` and `/health`
- `test_app.py` — pytest unit tests for both routes
- `requirements.txt` — Flask + pytest

Run locally:
```bash
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
python app.py
```

Run tests locally:
```bash
pytest -v
```

> **macOS note:** if `pip install` fails with `externally-managed-environment`, it means you're trying to install into the system Python. Always use a virtual environment (as above) — never `--break-system-packages`.
> If `python app.py` fails with "Address already in use" on port 5000, that's macOS's AirPlay Receiver squatting on the port. Either disable AirPlay Receiver in System Settings → General → AirDrop & Handoff, or run on a different port locally for testing.

## Pipeline Architecture

```
Push to main/staging, or Release published
        │
        ▼
┌─────────────────────┐
│ Install & Run Tests  │  pip install -r requirements.txt, pytest -v
└─────────┬────────────┘
          ▼
┌─────────────────────┐
│ Build Application     │  zip the app (excluding .git, venv, __pycache__)
└─────────┬────────────┘
          │
   ┌──────┴──────┐
   ▼             ▼
Deploy to      Deploy to
Staging        Production
(push to        (release
 staging         published)
 branch only)
```

### Deployment Targets
- **Staging** → EC2 instance `vikramjeet-ec2`
- **Production** → EC2 instance `vikramjeet-ec1`

Deployment uses [`appleboy/scp-action`](https://github.com/appleboy/scp-action) to copy `app.py` and `requirements.txt` to the instance, followed by [`appleboy/ssh-action`](https://github.com/appleboy/ssh-action) to set up a virtual environment, install dependencies, and (re)start the Flask app in the background.

## GitHub Secrets Configuration

Go to **Repository → Settings → Secrets and variables → Actions → New repository secret** and add:

| Secret Name       | Purpose                                                              |
|--------------------|------------------------------------------------------------------------|
| `STAGING_HOST`    | Current public IP address of the staging EC2 instance (`vikramjeet-ec2`) |
| `STAGING_SSH_KEY` | Full contents of the `.pem` private key used to SSH into staging      |
| `PROD_HOST`       | Current public IP address of the production EC2 instance (`vikramjeet-ec1`) |
| `PROD_SSH_KEY`    | Full contents of the `.pem` private key used to SSH into production   |

The SSH user for both instances is `ec2-user` (default for Amazon Linux AMIs), hardcoded in the workflow file.

> **Important — public IPs change on restart.** Unless an Elastic IP is attached, an EC2 instance's public IPv4 address changes every time it stops and starts. If a deployment step suddenly fails with `dial tcp ***:22: i/o timeout` even though the instance is running, the most likely cause is a stale `STAGING_HOST`/`PROD_HOST` secret pointing at an old IP. Update the secret with the current IP and re-run.

## Branch and Trigger Setup

- `main` — primary development branch; push here runs install, test, and build (no deploy)
- `staging` — push here runs install, test, build, **and deploys to staging**
- **GitHub Release** (published) — runs install, test, build, **and deploys to production**

Create the staging branch if it doesn't exist:
```bash
git checkout -b staging
git push -u origin staging
```

Trigger a production deploy by creating a new Release (with a version tag, e.g. `v1.0.0`) from the repository's **Releases** page.

## Security Group Requirements

Both EC2 instances need inbound rules for:
- **Port 22 (SSH)** from `0.0.0.0/0` — GitHub Actions runners use dynamic IPs, so a fixed IP allowlist isn't practical here
- **Port 5000 (Flask app)** from `0.0.0.0/0` — to view/verify the running app in a browser

## Troubleshooting Journey (Real Issues Hit During This Assignment)

This pipeline didn't work on the first try — here's every real failure encountered and how it was diagnosed and fixed, since understanding *why* each fix works is the actual point of the assignment.

### 1. YAML syntax error (line 49)
**Symptom:** Workflow failed instantly with `Invalid workflow file... error in your yaml syntax`.
**Cause:** A step name (`- name: "Build" application package`) had a quote character mid-string, which is technically valid YAML but risky to hand-type/copy-paste.
**Fix:** Removed the ambiguous quotes; always paste exact file content into GitHub's editor rather than retyping, to avoid invisible whitespace/character issues.

### 2. Self-killing the deploy script (`Process exited with status 143 from signal TERM`)
**Symptom:** The deploy step died abruptly right after printing "Stopping existing application...", with no further output.
**Cause:** The stop command used `pgrep -f "python3 app.py"` / `pkill -f "python3 app.py"` to find and kill the previously-running Flask process. But the **entire deploy script itself** is sent to the remote server as a single command whose text also contains the literal string `"python3 app.py"` — so the pattern match caught the currently-running deployment script and killed it, not the old Flask process.
**Fix:** Replaced text-matching (`pgrep`/`pkill`) with a **PID file**. On each deploy, the script writes the newly-started process's PID to `flask.pid`. On the next deploy, it reads that file, checks with `kill -0` whether that PID is still alive, and only then sends it `kill -15`. This never depends on matching text, so it can't self-match.

### 3. Broken virtual environment (`venv/bin/activate: No such file or directory`)
**Symptom:** `source venv/bin/activate` failed even though a `venv` folder existed on the server.
**Cause:** An earlier failed run (before `set -e` was added to the script) had left a **partial, broken venv** — the directory existed, but `python3 -m venv venv` never got to finish creating `bin/activate`. Since the script only checked `if [ ! -d "venv" ]`, it saw the (broken) folder and skipped recreating it.
**Fix:** Changed the check to look for the actual file that matters — `if [ ! -f "venv/bin/activate" ]` — and if it's missing, `rm -rf venv` before recreating it from scratch.

### 4. Git object permission errors during file copy
**Symptom:** `tar: ./.git/objects/...: Cannot open: Permission denied` during the SCP step.
**Cause:** The SCP step was configured with `source: "."`, copying the entire repository **including the `.git` folder**. Git stores its internal object files as read-only (mode 444) for integrity — when the copy tried to overwrite an existing read-only `.git` object already on the server, the write was denied.
**Fix:** Changed `source: "."` to `source: "app.py,requirements.txt"` — only copy the files actually needed on the server. This also avoids shipping unnecessary Git internals to a production machine.

### 5. Stale IP after instance restart (`dial tcp ***:22: i/o timeout`)
**Symptom:** Production deploy couldn't even establish an SSH connection, despite the instance being confirmed "running".
**Cause:** The EC2 instance's public IP had changed after a stop/start cycle (from earlier testing in a different assignment), but the `PROD_HOST` GitHub Secret still held the old IP.
**Fix:** Updated `PROD_HOST` with the instance's current public IP and re-ran the job. (A permanent fix would be attaching an **Elastic IP** to both instances so their public IP never changes.)

## Verifying a Deployment

After a successful run, visit (note: use `http://`, not `https://` — the app has no SSL cert on port 5000):
```
http://<instance-public-ip>:5000/
http://<instance-public-ip>:5000/health
```
Expected response from `/health`: `{"status": "ok"}`

## Viewing Pipeline Runs

Go to the **Actions** tab of the repository to see each workflow run, with per-job logs for install/test, build, and deploy stages.