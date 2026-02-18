# Ansible Quiz App

**Infrastructure as Code** for the [quiz-app repo here] – a certification‑style quiz platform.

This repository contains **Ansible playbooks and roles** to configure and deploy the entire stack on a **single Hetzner VPS**, with a clear vertical scaling path and CI/CD via GitHub Actions.

> **Provisioning vs Configuration:** Infrastructure (VPS, firewall, DNS) is provisioned via **Terraform**. Everything that runs on the server — OS hardening, Docker, app deployment — is handled by **Ansible**. The two tools are complementary: Terraform creates the machine, Ansible configures it.

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
| Infrastructure      | **Terraform** (VPS, firewall, DNS)   |
| Configuration       | **Ansible** (OS, Docker, app deploy) |
| Hosting             | **Hetzner VPS** (CX22 / CX32 / CX42) |

**Traffic pattern:** Read‑heavy, low writes.  
**Philosophy:** Simplicity, portability, vertical scalability, no over‑engineering.

---

## Terraform — Infrastructure Provisioning

Terraform manages all cloud infrastructure as code. If the VPS dies, `terraform apply` recreates it and Ansible handles the rest.

**What Terraform covers:**
- Hetzner VPS (server type, image, region, SSH key)
- Hetzner firewall rules
- Cloudflare DNS records

### Vertical Scaling via Terraform

Scaling is a one-liner — no clicking in the Hetzner panel:

```bash
terraform apply -var="server_type=cx42"   # scale up
terraform apply -var="server_type=cx22"   # scale down
```

Terraform will show a plan before applying. The VPS stops, resizes, and restarts automatically.

### Terraform Repo Structure

```
terraform/
├── main.tf          # Hetzner server, SSH key, firewall
├── dns.tf           # Cloudflare DNS records
├── variables.tf     # server_type, region, etc.
├── outputs.tf       # VPS IP (fed into Ansible inventory)
└── terraform.tfvars # Local values — not committed
```

> ⚠️ `terraform.tfvars` contains API tokens and is gitignored. Store state remotely (Terraform Cloud free tier or Hetzner Object Storage).

### Basic Terraform Workflow

```bash
terraform init      # download providers, initialize
terraform plan      # preview changes — always run this first
terraform apply     # provision / update infrastructure
terraform destroy   # full teardown — use with caution
```

---

## Cost & Scaling Strategy

| Plan   | vCPU | RAM  | Storage | Approx. monthly |
|--------|------|------|---------|-----------------|
| CX22   | 2    | 4GB  | 40GB    | €5              |
| CX32   | 4    | 8GB  | 80GB    | €9–12           |
| CX42   | 8    | 16GB | 160GB   | €18–22          |

**Launch recommendation:** Start with **CX32** (or CX42 if uncertain), monitor load, then **downscale** to CX22 if sustained traffic is low.

**Future options** (when needed):
- Managed database (Postgres/MongoDB) – €15–50/month  
- CDN + Object Storage – €5–20/month  
- Load balancer – €5–15/month  

**Current cost estimate (CX22 + domain + Cloudflare Free):** **~€6–7/month**.

---

## Ansible — Configuration & Deployment

Once Terraform has provisioned the VPS and output its IP, Ansible takes over — OS hardening, Docker installation, firewall config, and app deployment.

### Repo Structure

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

## Getting Started

### 1. Prerequisites

- Terraform installed locally  
- Ansible installed locally (or rely on GitHub Actions)  
- Hetzner API token and Cloudflare API token  
- SSH key pair  

### 2. Clone this repository

```bash
git clone https://github.com/raduhhr/ansible-quiz-app.git
cd ansible-quiz-app
```

### 3. Provision infrastructure with Terraform

```bash
cd terraform
cp terraform.tfvars.example terraform.tfvars
# Fill in your Hetzner + Cloudflare tokens and SSH key path
terraform init
terraform plan
terraform apply
# Outputs the VPS IP on completion
```

### 4. Configure Ansible inventory

Create `inventory/production/hosts.yml` using the IP from Terraform output:

```yaml
all:
  hosts:
    quiz-vps:
      ansible_host: 1.2.3.4          # <– from terraform output
      ansible_user: admin
```

> ⚠️ **This file is ignored by Git** – IP/credentials stay local.

### 5. (Optional) Encrypt secrets with Ansible Vault

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
# To also destroy infrastructure: cd terraform && terraform destroy
```

---

## CI/CD with GitHub Actions

Every push to the `main` branch automatically runs the `deploy.yml` playbook on the production VPS.

**Secrets** must be configured in the GitHub repository:

| Secret name      | Value                                |
|------------------|--------------------------------------|
| `VPS_HOST`       | Your VPS IP address                 |
| `VPS_USER`       | SSH username (e.g. `admin`)         |
| `VPS_SSH_KEY`    | **Private SSH key** (PEM format)    |
| `DISCORD_WEBHOOK`| (Optional) for deployment notifications |

The workflow uses these secrets to connect and deploy **without exposing any credentials** in the code.

---

## Integration with the quiz-app Repository

When Paul pushes code to the **quiz-app repository**, it will trigger this Ansible pipeline automatically.

How to:
1. In the app repo, add a **repository dispatch** GitHub Action.
2. It calls this repo's `deploy.yml` workflow via `repository_dispatch`.
3. Everything stays decoupled – app developers never touch infrastructure.

*(Setup instructions will be added HERE once the app repo is ready.)*

---

## License

MIT © Radu Circei
