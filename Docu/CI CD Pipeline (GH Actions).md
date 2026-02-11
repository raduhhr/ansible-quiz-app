```markdown
# CI/CD Pipeline Documentation

This document describes the continuous integration and deployment setup for the quiz application infrastructure. The pipeline uses **GitHub Actions** to automatically provision and deploy the application to a Hetzner VPS whenever changes are pushed to this repository, and optionally when the upstream `quiz-app` repository receives updates.

---

## Overview

The CI/CD pipeline is defined in `.github/workflows/deploy.yml`. It performs the following tasks:

- Checks out the Ansible playbooks.
- Installs Ansible on the runner.
- Creates an inventory file using GitHub Secrets (no credentials are stored in the repository).
- Runs the `deploy.yml` playbook to update the application on the production VPS.

Two trigger modes are supported:

1. **Push to `main`** – Deploys the latest infrastructure code.
2. **Repository dispatch** – Allows external repositories (e.g., the `quiz-app` code repository) to trigger a deployment.

---

## Workflow Configuration

### File Location
`.github/workflows/deploy.yml`

### Trigger Definitions
```yaml
on:
  push:
    branches: [ main ]
  repository_dispatch:
    types: [ deploy-production ]
  workflow_dispatch:    # manual trigger
```

- `push`: deploys on every commit to `main`.
- `repository_dispatch`: listens for a custom event named `deploy-production`.
- `workflow_dispatch`: enables manual runs from the GitHub Actions tab.

---

## Required Secrets

The following secrets must be configured in the repository **Settings → Secrets and variables → Actions**:

| Secret Name       | Description                                                                 |
|-------------------|-----------------------------------------------------------------------------|
| `VPS_HOST`        | Public IP address or hostname of the target VPS.                           |
| `VPS_USER`        | SSH username for the VPS (e.g., `admin`, `ubuntu`).                        |
| `VPS_SSH_KEY`     | Private SSH key (PEM format) that matches the public key installed on the VPS. |
| `DISCORD_WEBHOOK` | (Optional) Webhook URL for Discord deployment notifications.               |

**Important:** The SSH key must be in PEM format. If you are using an OpenSSH format key, convert it with:
```bash
ssh-keygen -p -m PEM -f ~/.ssh/id_ed25519
```

---

## Deployment Job Steps

1. **Checkout** – Fetches the Ansible code.
2. **Install Ansible** – Installs the latest Ansible via `pip`.
3. **Create Inventory** – Writes a `hosts.yml` file from secrets.
4. **Fix SSH Key Permissions** – Ensures the private key file has `600` permissions.
5. **Run Ansible Playbook** – Executes `playbooks/deploy.yml` with the generated inventory and key.
6. **Notify** – Sends success/failure messages to Discord (if webhook is configured).

---

## Integration with the `quiz-app` Repository

The pipeline can be triggered automatically whenever your colleague pushes code to the `quiz-app` repository. This is achieved using **GitHub repository dispatch events** – no cross-repository collaborator access is required.

### Step 1: Create a Personal Access Token (PAT)

You need a token that can dispatch events to **this** repository. Two approaches:

- **Personal token (simple)** – Create a classic PAT with `repo` scope from your GitHub account. Share it securely with the app repository owner.
- **Bot account (cleanest)** – Create a dedicated GitHub account (e.g., `quiz-deploy-bot`), add it as a **collaborator with Write access** to this repository, and generate a PAT from that account.

The token must be stored as a secret named `ANSIBLE_REPO_TOKEN` in the **`quiz-app` repository**.

### Step 2: Add Dispatch Workflow to `quiz-app`

Your colleague should add the following workflow file to the `quiz-app` repository (`.github/workflows/trigger-deploy.yml`):

```yaml
name: Trigger Production Deploy

on:
  push:
    branches: [ main ]

jobs:
  trigger:
    runs-on: ubuntu-latest
    steps:
      - name: Repository Dispatch
        uses: peter-evans/repository-dispatch@v2
        with:
          token: ${{ secrets.ANSIBLE_REPO_TOKEN }}
          repository: raduhhr/ansible-quiz-app   # Change to your repo name
          event-type: deploy-production
```

Now, every push to the `main` branch of `quiz-app` will send a dispatch event to this repository, triggering the deployment workflow.

---

## Manual Deployment

You can also trigger a deployment manually:

1. Go to the **Actions** tab of this repository.
2. Select the **Deploy to Production** workflow.
3. Click **Run workflow** and choose the branch.

This is useful for testing or one-off updates.

---

## Verifying Deployments

- **GitHub Actions logs** – Each run provides a detailed log of the Ansible output.
- **Discord notifications** – If configured, success/failure messages are sent instantly.
- **Application health** – After deployment, the application endpoints should respond as expected.

---

## Security Considerations

- **No credentials in code** – IPs, usernames, and SSH keys are stored as GitHub Secrets and never appear in the repository.
- **Least privilege** – The token used for repository dispatch has only the necessary permissions (`repo` scope) and is stored encrypted in the app repository.
- **Private repository** – This repository is private, limiting access to invited collaborators only.

---

## Troubleshooting

| Issue                          | Likely Cause                                | Solution                                                                 |
|--------------------------------|---------------------------------------------|--------------------------------------------------------------------------|
| `Permissions 0644 for '/tmp/ssh_key' are too open` | SSH key file permissions too permissive.    | Add a step: `chmod 600 /tmp/ssh_key`.                                   |
| `Host key verification failed` | VPS host key not trusted.                  | Set `ANSIBLE_HOST_KEY_CHECKING: "false"` (acceptable for ephemeral runners). |
| `DEPLOY_FAILED` notification   | Playbook error.                            | Check the Action logs for Ansible error messages.                        |
| Dispatch event not triggering  | Token missing or incorrect repository name.| Verify the token scope and the repository name in the dispatch action.   |

---

This CI/CD pipeline provides a robust, secure, and fully automated deployment path for the quiz application. No manual SSH or direct access is required after initial setup.
