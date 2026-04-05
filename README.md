# infra-playbook

Ansible IaC for Proxmox VE, Proxmox Backup Server, and services VM (Traefik + Authentik). Secrets managed via AWS SSM Parameter Store, accessed through AWS IAM Identity Center SSO.

---

## Prerequisites

### Dependencies
```bash
ansible-galaxy collection install community.general community.aws amazon.aws ansible.posix
pipx inject ansible-core boto3 botocore passlib
aws sso login --profile <your-aws-profile>
```

### Inventory
```bash
cp inventory/hosts.yml.example inventory/hosts.yml
cp inventory/group_vars/all.yml.example inventory/group_vars/all.yml
```

---

## SSM Parameters

Seed these parameters before running any playbook.

| Path | Type | Description |
|---|---|---|
| `/infra/common/public_key` | String | Admin user SSH public key |
| `/infra/common/admin_password` | SecureString | Admin user login password |
| `/infra/common/root_password` | SecureString | Root user login password |
| `/infra/common/ansible_public_key` | String | `ansible` service account SSH public key |
| `/infra/common/ansible_private_key` | SecureString | `ansible` service account SSH private key |
| `/infra/common/cloudflare/api_token` | SecureString | Cloudflare API token with DNS Edit permission — required for Traefik ACME |
| `/infra/common/cloudflare/zone_id` | String | Cloudflare zone ID |
| `/infra/common/smtp/user` | String | SMTP username |
| `/infra/common/smtp/pass` | SecureString | SMTP password |
| `/infra/proxmox/terraform_tmp_pass` | SecureString | Initial password for `terraform-prov@pve` — used only at user creation |
| `/infra/proxmox/packer_api_password` | SecureString | Initial password for `packer-builder@pve` — used only at user creation |
| `/infra/proxmox/terraform_token` | SecureString | Proxmox API token for Terraform — written automatically by `site.yml` |
| `/infra/proxmox/packer_token` | SecureString | Proxmox API token for Packer — written automatically by `site.yml` |
| `/infra/pbs/backup_user_password` | SecureString | Password for `backup-user@pbs` |
| `/infra/pbs/fingerprint` | String | PBS TLS fingerprint — written automatically by `pbs.yml` |
| `/infra/svc/authentik/secret_key` | SecureString | Authentik secret key |
| `/infra/svc/authentik/postgres_password` | SecureString | Authentik PostgreSQL password |

---

## Deployment Order

### 1. Bootstrap
Runs as `root` on port 22. Creates the `ansible` service account and transitions SSH to port 2222.
```bash
ansible-playbook -i inventory/hosts.yml playbooks/bootstrap.yml
```

### 2. Create admin user
Runs the `base` role to create the admin OS user, set passwords from SSM, install the SSH key, and configure sudoers.
```bash
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --tags base
ansible-playbook -i inventory/hosts.yml playbooks/pbs.yml --tags base
```

### 3. Proxmox — post-install configuration
```bash
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --skip-tags pbs
```
Configures repos, ZFS storage, firewall, notifications, and Terraform/Packer API users. PBS registration is skipped — PBS isn't deployed yet.

**Post-run:**
- Log into the PVE web UI and set up TOTP under your username → Two Factor → Add TOTP.

### 4. PBS — install and configure
```bash
ansible-playbook -i inventory/hosts.yml playbooks/pbs.yml
```
Installs the PBS package, configures repos, NFS mount, datastore, prune/GC schedules, firewall, and users.

**Post-run:**
- Log into the PBS web UI and set up TOTP manually:
  Administration → User Management → `sysop@pam` → TFA → Add TOTP

### 5. Proxmox — register PBS storage and backup job
```bash
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --tags pbs
```
Registers PBS as a PVE storage backend and creates a nightly backup job (`--all 1`). Templates and the PBS VM are automatically excluded on every run — new Packer templates are picked up dynamically. All other VMs are backed up. Notifications fire on errors and warnings only.

### 6. Services VM — deploy Traefik and Authentik
```bash
ansible-playbook -i inventory/hosts.yml playbooks/svc.yml
```
Installs Docker, mounts NFS storage, configures UFW, and deploys Traefik (reverse proxy with Cloudflare DNS-01 wildcard cert) and Authentik (identity provider) as Docker Compose stacks.

---

## Day-2 Operations

All playbooks are idempotent and safe to re-run at any time.

```bash
# Password rotation — update SSM then run:
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --tags base
ansible-playbook -i inventory/hosts.yml playbooks/pbs.yml --tags base

# System (repos, PVE users, vzdump config)
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --tags system

# Storage (ZFS provisioning, local-lvm)
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --tags storage

# Firewall rules
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --tags firewall
ansible-playbook -i inventory/hosts.yml playbooks/pbs.yml --tags network
ansible-playbook -i inventory/hosts.yml playbooks/svc.yml --tags firewall

# Notification configuration
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --tags notifications
ansible-playbook -i inventory/hosts.yml playbooks/pbs.yml --tags notifications

# PBS storage registration
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --tags pbs

# PBS datastore and users
ansible-playbook -i inventory/hosts.yml playbooks/pbs.yml --tags datastore
ansible-playbook -i inventory/hosts.yml playbooks/pbs.yml --tags users

# Redeploy Traefik (config or version change)
ansible-playbook -i inventory/hosts.yml playbooks/svc.yml --tags traefik

# Redeploy Authentik (config or version change)
ansible-playbook -i inventory/hosts.yml playbooks/svc.yml --tags authentik
```
