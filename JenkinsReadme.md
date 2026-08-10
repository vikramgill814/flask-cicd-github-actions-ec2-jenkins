# Jenkins CI/CD Pipeline for Flask Application

This file documents the Jenkins part of the assignment. The pipeline file is [Jenkinsfile](Jenkinsfile), and the Jenkins screenshots are stored in [screenshots/jenkins](screenshots/jenkins).

## Objective

Set up a Jenkins pipeline that automatically builds, tests, packages, and deploys a simple Flask application to a staging environment.

## Prerequisites

- Jenkins installed on a VM or cloud Jenkins server.
- Java installed for Jenkins.
- Python 3, `python3-venv`, `pip`, `zip`, and `curl` installed on the Jenkins server.
- Jenkins plugins installed:
  - Pipeline
  - Git
  - GitHub
  - Mailer or Email Extension
- GitHub repository access from Jenkins.

## Repository

Repository URL:

```text
https://github.com/vikramgill814/flask-cicd-github-actions-ec2-jenkins
```

Important files:

| File | Purpose |
| --- | --- |
| `app.py` | Flask application. |
| `test_app.py` | Unit tests using pytest. |
| `requirements.txt` | Python dependencies. |
| `Jenkinsfile` | Jenkins declarative pipeline. |

## Jenkins Job Configuration

1. Open Jenkins and complete the first-time setup wizard.
2. Create a new item named `flask-jenkins-cicd`.
3. Select **Pipeline** as the job type.
4. In the job configuration, select **Pipeline script from SCM**.
5. Select **Git** as SCM.
6. Add the repository URL.
7. Use branch `main`.
8. Set Script Path to `Jenkinsfile`.
9. Enable **GitHub hook trigger for GITScm polling**.
10. Save the job.

## GitHub Webhook

In GitHub, go to **Repository > Settings > Webhooks > Add webhook** and use:

```text
Payload URL: http://<jenkins-url>/github-webhook/
Content type: application/json
Events: push events
```

This triggers a new Jenkins build whenever code is pushed to the main branch.

## Pipeline Stages

| Stage | What it does |
| --- | --- |
| Checkout SCM | Jenkins pulls the latest code from GitHub. |
| Build | Creates a clean virtual environment and installs dependencies from `requirements.txt`. |
| Test | Runs the Flask unit tests using `pytest -v`. |
| Build Artifact | Creates `app-build.zip` and stores it as a Jenkins build artifact. |
| Deploy to Staging | Copies the app into `/var/lib/jenkins/flask-staging`, starts it on port `5000`, and runs a health check. |
| Post Actions | Reports success/failure and sends email if Jenkins SMTP and `NOTIFY_EMAIL` are configured. |

## Email Notifications

The `post` section in [Jenkinsfile](Jenkinsfile) supports success and failure notifications.

To enable email:

1. Go to **Manage Jenkins > System**.
2. Configure SMTP in Mailer or Extended E-mail Notification.
3. Add `NOTIFY_EMAIL` as a Jenkins global or job environment variable.
4. Run the pipeline. Jenkins will send a build status email after success or failure.

## Staging Deployment

The Jenkins pipeline deploys the Flask app to:

```text
/var/lib/jenkins/flask-staging
```

The app runs on:

```text
http://<jenkins-server-ip>:5000/
http://<jenkins-server-ip>:5000/health
```

Expected health check response:

```json
{"status":"ok"}
```

## Jenkins Screenshots

| Evidence | Screenshot |
| --- | --- |
| Jenkins setup step 1 | ![Jenkins setup step 1](<screenshots/jenkins/Setup-Step1-Wizard-Jenkins-08-10-2026_08_48_PM.png>) |
| Jenkins setup step 2 | ![Jenkins setup step 2](<screenshots/jenkins/Setup-Step2-Wizard-Jenkins-08-10-2026_08_49_PM.png>) |
| Jenkins setup step 3 | ![Jenkins setup step 3](<screenshots/jenkins/Setup-step3-Wizard-Jenkins-08-10-2026_08_50_PM.png>) |
| Jenkins ready | ![Jenkins ready](<screenshots/jenkins/jenkins-ready-Wizard-Jenkins-08-10-2026_08_51_PM.png>) |
| New pipeline item | ![New pipeline item](<screenshots/jenkins/new_item_pipeline/New-Item-Jenkins-08-10-2026_08_55_PM.png>) |
| Pipeline SCM configuration | ![Pipeline SCM configuration](<screenshots/jenkins/new_item_pipeline/flask-jenkins-cicd-Config-Jenkins-08-10-2026_08_56_PM.png>) |
| GitHub trigger enabled | ![GitHub trigger enabled](<screenshots/jenkins/new_item_pipeline/fgithub-triggers-lask-jenkins-cicd-Config-Jenkins-08-10-2026_08_58_PM.png>) |
| GitHub webhook configured | ![GitHub webhook configured](<screenshots/jenkins/new_item_pipeline/Github-Webhooks-·-Settings-·-vikramgill814-setup-jenkins2-35-linux-08-10-2026_09_00_PM.png>) |
| Successful build with artifact | ![Successful build with artifact](<screenshots/jenkins/new_item_pipeline/build-success-flask-jenkins-cicd-2-Jenkins-08-10-2026_08_56_PM.png>) |
| Pipeline stages overview | ![Pipeline stages overview](<screenshots/jenkins/new_item_pipeline/pipeline-overview-flask-jenkins-cicd-Jenkins-08-10-2026_08_57_PM.png>) |

## Deliverables Covered

- Jenkinsfile present in repository root.
- Build stage installs dependencies with pip.
- Test stage runs pytest.
- Deploy stage deploys to staging.
- GitHub webhook trigger configured.
- Email notification support documented.
- Jenkins screenshots included.
