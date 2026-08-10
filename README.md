# Flask CI/CD Assignment

This repository contains a simple Flask application with two CI/CD implementations:

- Jenkins Pipeline for build, test, artifact creation, and staging deployment.
- GitHub Actions workflow for install, test, build, staging deployment, and production deployment.

Repository URL: `https://github.com/vikramgill814/flask-cicd-github-actions-ec2-jenkins`

## Application

- `app.py` - Flask app with `/` and `/health` routes.
- `test_app.py` - pytest test cases for both routes.
- `requirements.txt` - Python dependencies.
- `Jenkinsfile` - Jenkins CI/CD pipeline definition.
- `.github/workflows/ci-cd.yml` - GitHub Actions CI/CD workflow.

Run locally:

```bash
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
python app.py
```

Run tests:

```bash
pytest -v
```

## Jenkins CI/CD Pipeline

The Jenkins pipeline is defined in [Jenkinsfile](Jenkinsfile). It runs on the Jenkins server and deploys the Flask app to a staging directory on the same Jenkins host.

### Jenkins Prerequisites

- Jenkins installed on a VM or cloud Jenkins server.
- Java installed for Jenkins.
- Python 3, `python3-venv`, `pip`, `zip`, and `curl` installed on the Jenkins server.
- Jenkins plugins: Pipeline, Git, GitHub, and Mailer or Email Extension for email notification.
- GitHub repository access configured in Jenkins.

### Jenkins Job Setup

1. Open Jenkins and finish the setup wizard.
2. Create a new Pipeline job named `flask-jenkins-cicd`.
3. Select **Pipeline script from SCM**.
4. Select **Git** as SCM and enter this repository URL.
5. Use branch `main`.
6. Set Script Path to `Jenkinsfile`.
7. Enable **GitHub hook trigger for GITScm polling**.
8. In GitHub, add a webhook:
   - Payload URL: `http://<jenkins-url>/github-webhook/`
   - Content type: `application/json`
   - Event: push events

### Jenkins Pipeline Stages

| Stage | Purpose |
| --- | --- |
| Checkout SCM | Jenkins checks out the repository from GitHub. |
| Build | Creates a virtual environment and installs dependencies with pip. |
| Test | Runs unit tests using `pytest -v`. |
| Build Artifact | Creates `app-build.zip` and archives it in Jenkins. |
| Deploy to Staging | Copies the app to `/var/lib/jenkins/flask-staging`, starts Flask on port `5000`, and verifies `/health`. |
| Post Actions | Prints success/failure status and sends email if `NOTIFY_EMAIL` is configured in Jenkins. |

### Jenkins Notifications

Email notification is handled from the `post` block in [Jenkinsfile](Jenkinsfile). To enable actual emails:

1. Go to **Manage Jenkins > System**.
2. Configure SMTP under Mailer or Extended E-mail Notification.
3. Add `NOTIFY_EMAIL` as a Jenkins global/job environment variable with the recipient email address.
4. Run the pipeline again. Jenkins sends success or failure mail after the build.

### Jenkins Screenshots

| Evidence | Screenshot |
| --- | --- |
| Jenkins setup step 1 | ![Jenkins setup step 1](<screenshots/jenkins/Setup-Step1-Wizard-Jenkins-08-10-2026_08_48_PM.png>) |
| Jenkins setup step 2 | ![Jenkins setup step 2](<screenshots/jenkins/Setup-Step2-Wizard-Jenkins-08-10-2026_08_49_PM.png>) |
| Jenkins ready | ![Jenkins ready](<screenshots/jenkins/jenkins-ready-Wizard-Jenkins-08-10-2026_08_51_PM.png>) |
| Pipeline job created | ![Pipeline job created](<screenshots/jenkins/new_item_pipeline/New-Item-Jenkins-08-10-2026_08_55_PM.png>) |
| Pipeline configured from SCM | ![Pipeline configured from SCM](<screenshots/jenkins/new_item_pipeline/flask-jenkins-cicd-Config-Jenkins-08-10-2026_08_56_PM.png>) |
| GitHub trigger enabled | ![GitHub trigger enabled](<screenshots/jenkins/new_item_pipeline/fgithub-triggers-lask-jenkins-cicd-Config-Jenkins-08-10-2026_08_58_PM.png>) |
| GitHub webhook configured | ![GitHub webhook configured](<screenshots/jenkins/new_item_pipeline/Github-Webhooks-·-Settings-·-vikramgill814-setup-jenkins2-35-linux-08-10-2026_09_00_PM.png>) |
| Successful Jenkins build | ![Successful Jenkins build](<screenshots/jenkins/new_item_pipeline/build-success-flask-jenkins-cicd-2-Jenkins-08-10-2026_08_56_PM.png>) |
| Jenkins pipeline stages | ![Jenkins pipeline stages](<screenshots/jenkins/new_item_pipeline/pipeline-overview-flask-jenkins-cicd-Jenkins-08-10-2026_08_57_PM.png>) |

More Jenkins-specific notes are available in [JenkinsReadme.md](JenkinsReadme.md).

## GitHub Actions CI/CD Pipeline

The GitHub Actions workflow is defined in [.github/workflows/ci-cd.yml](.github/workflows/ci-cd.yml).

### GitHub Actions Triggers

- Push to `main`: installs dependencies, runs tests, and builds the application.
- Push to `staging`: installs dependencies, runs tests, builds, and deploys to staging.
- Published GitHub release: installs dependencies, runs tests, builds, and deploys to production.

### GitHub Actions Jobs

| Job | Purpose |
| --- | --- |
| Install Dependencies & Run Tests | Checks out code, installs Python 3.11 dependencies, and runs `pytest -v`. |
| Build Application | Creates `app-build.zip` and uploads it as an artifact. |
| Deploy to Staging | Uses SSH/SCP to deploy to the staging EC2 instance when `staging` branch is updated. |
| Deploy to Production | Uses SSH/SCP to deploy to the production EC2 instance when a release is published. |

### GitHub Secrets

Add these in **Repository > Settings > Secrets and variables > Actions**:

| Secret Name | Purpose |
| --- | --- |
| `STAGING_HOST` | Public IP or DNS of the staging EC2 instance. |
| `STAGING_SSH_KEY` | Private SSH key for staging deployment. |
| `PROD_HOST` | Public IP or DNS of the production EC2 instance. |
| `PROD_SSH_KEY` | Private SSH key for production deployment. |

The workflow uses `ec2-user` as the SSH username for both EC2 instances.

### GitHub Actions Screenshots

| Evidence | Screenshot |
| --- | --- |
| Successful staging workflow | ![Successful staging workflow](<screenshots/ci-cd-fix-staging-deploy-·-vikramgill814-flask-cicd-github-actions-ec2-jenkins-dcebd34-08-08-2026_11_18_AM.png>) |
| Release workflow run | ![Release workflow run](<screenshots/v1-0-0-Initial-Release-·-vikramgill814-flask-cicd-github-actions-ec2-jenkins-dcebd34-08-08-2026_11_18_AM.png>) |
| New release created | ![New release created](<screenshots/New-release-·-vikramgill814-flask-cicd-github-actions-ec2-jenkins-08-08-2026_11_20_AM.png>) |
| Release published | ![Release published](<screenshots/Release-v1-0-0-Initial-Release_create-·-vikramgill814-flask-cicd-github-actions-ec2-jenkins-08-08-2026_11_20_AM.png>) |

## Deployment Verification

After a successful deployment, open:

```text
http://<server-public-ip>:5000/
http://<server-public-ip>:5000/health
```

Expected `/health` response:

```json
{"status":"ok"}
```

## Submission Checklist

- Forked or created GitHub repository with Flask source code.
- `Jenkinsfile` added at the repository root.
- `.github/workflows/ci-cd.yml` added for GitHub Actions.
- Jenkins pipeline configured with GitHub webhook trigger.
- GitHub Actions workflow configured with staging and production deployment rules.
- Required deployment secrets configured in GitHub.
- Jenkins and GitHub Actions screenshots added under `screenshots/`.
- Repository URL included for Vlearn submission.
