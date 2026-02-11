# Ansible Quiz App

**Infrastructure as Code** for the [Quiz Application] – a certification‑style quiz platform.

This repository contains **Ansible playbooks and roles** to provision and deploy the entire stack on a **single Hetzner VPS**, with a clear vertical scaling path and CI/CD via GitHub Actions.

---

## Architecture Overview

<img width="391" height="587" alt="image" src="https://github.com/user-attachments/assets/29ab7060-04a9-4e85-a5d3-5504d614d755" />

| Component           | Technology                             |
|---------------------|----------------------------------------|
| Frontend            | Angular (SPA, static files)           |
| Backend API         | Node.js                               |
| Database            | PostgreSQL or MongoDB                |
| Reverse Proxy       | Caddy / Traefik / Nginx              |
| Static images       | ~2000–3000 images, served locally    |
| Orchestration       | Docker Compose                       |
| Provisioning        | Ansible                              |
| Hosting             | **Hetzner VPS** (CX22 / CX32 / CX42) |

**Traffic pattern:** Read‑heavy, low writes.  
**Philosophy:** Simplicity, portability, vertical scalability, no over‑engineering.

---
## Cost & Scaling Strategy

| Plan   | vCPU | RAM  | Storage | Approx. monthly |
|--------|------|------|---------|-----------------|
| CX22   | 2    | 4GB  | 40GB    | €5              |
| CX32   | 4    | 8GB  | 80GB    | €9–12           |
| CX42   | 8    | 16GB | 160GB   | €18–22          |

**Launch recommendation:** Start with **CX32** (or CX42 if uncertain), monitor load, then **downscale** to CX22 if sustained traffic is low.  
Scaling is **vertical** – stop VPS, resize in Hetzner panel, restart, optional filesystem resize.

**Future options** (when needed):
- Managed database (Postgres/MongoDB) – €15–50/month  
- CDN + Object Storage – €5–20/month  
- Load balancer – €5–15/month  

**Current cost estimate (CX22 + domain + Cloudflare Free):** **~€6–7/month**.

---

## Repo Structure

```
.
├── .github/workflows/       # GitHub Actions CI/CD
│   └── deploy.yml          # Auto‑deploy on push
├── group_vars/             # Global variables
│   └── all/
├── host_vars/              # Per‑host variables
├── inventory/
│   └── production/         # Production inventory
│       └── hosts.yml      # (not committed – contains IP)
├── playbooks/
│   ├── provision.yml      # Full server setup
│   ├── deploy.yml         # Deploy / update app only
│   ├── restart.yml        # Restart containers
│   ├── stop.yml           # Stop application
│   └── nuke.yml           # Full teardown
└── roles/
    ├── base/              # OS hardening, Docker, UFW, etc.
    └── quiz-app/          # Clone repo, docker‑compose up
```

---

## Setting Started

### 1. Prerequisites

- Ansible installed locally (or rely on GitHub Actions)  
- A **Hetzner VPS** (or any Ubuntu 22.04/24.04 VM)  
- SSH key pair – **public key** added to `~/.ssh/authorized_keys` on the VPS  
- Your friend’s **quiz app repository** (Dockerised)

### 2. Clone this repository

```bash
git clone https://github.com/raduhhr/ansible-quiz-app.git
cd ansible-quiz-app
```

### 3. Configure inventory

Create `inventory/production/hosts.yml`:

```yaml
all:
  hosts:
    quiz-vps:
      ansible_host: 1.2.3.4          # <– VPS IP
      ansible_user: admin            # <– SSH user
```

> ⚠️ **This file is ignored by Git** – IP/credentials stay local.

### 4. (Optional) Encrypt secrets with Ansible Vault

```bash
ansible-vault create group_vars/all/vault.yml
# Add e.g. vault_db_password: "secret"
```

Reference vaulted variables in `group_vars/all/vars.yml`.

---

## Usage

### Provision a fresh VPS (base + app)

```bash
ansible-playbook playbooks/provision.yml
```

### Deploy a new version of the app

```bash
ansible-playbook playbooks/deploy.yml
```

### Restart containers

```bash
ansible-playbook playbooks/restart.yml
```

### Stop the application

```bash
ansible-playbook playbooks/stop.yml
```

### Full teardown (remove everything)

```bash
ansible-playbook playbooks/nuke.yml   # use with caution!
```

---

## CI/CD with GitHub Actions

Every push to the `main` branch of **this repository** automatically runs the `deploy.yml` playbook on the production VPS.

**Secrets** must be configured in the GitHub repository:

| Secret name      | Value                                |
|------------------|--------------------------------------|
| `VPS_HOST`       | Your VPS IP address                 |
| `VPS_USER`       | SSH username (e.g. `admin`)         |
| `VPS_SSH_KEY`    | **Private SSH key** (PEM format)    |
| `DISCORD_WEBHOOK`| (Optional)for deployment notifications |

The workflow uses these secrets to connect and deploy **without exposing any credentials** in the code.

---

## Integration with the quiz-app Repository

When Paul pushes code to the **quiz-app repository**, it will trigger this Ansible pipeline automatically.

How to: 
1. In the app repo, add a **repository dispatch** GitHub Action.
2. It calls this repo’s `deploy.yml` workflow via `repository_dispatch`.
3. Everything stays decoupled – app developers never touch infrastructure.

*(Setup instructions will be added HERE once the app repo is ready.)*

---

## License

MIT © Radu Circei
