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

Seed these parameters before running any playbook. Parameters marked *auto* are written by the playbook on first run and do not need to be seeded manually.

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
| `/infra/proxmox/terraform_token` | SecureString | *auto* — Proxmox API token for Terraform |
| `/infra/proxmox/packer_token` | SecureString | *auto* — Proxmox API token for Packer |
| `/infra/pbs/backup_user_password` | SecureString | Password for `backup-user@pbs` |
| `/infra/pbs/fingerprint` | String | *auto* — PBS TLS fingerprint |
| `/infra/svc/authentik/secret_key` | SecureString | *auto* — Authentik secret key |
| `/infra/svc/authentik/postgres_password` | SecureString | *auto* — Authentik PostgreSQL password |
| `/infra/svc/authentik/bootstrap_token` | SecureString | *auto* — Authentik bootstrap API token |
| `/infra/svc/oidc/<name>/client_secret` | SecureString | *auto* — OIDC client secret per `proxy_services` entry (e.g. `/infra/svc/oidc/pve/client_secret`) |
| `/infra/svc/unifi/mongo_root_password` | SecureString | *auto* — MongoDB root password for UniFi |
| `/infra/svc/unifi/mongo_password` | SecureString | *auto* — MongoDB application user password for UniFi |

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
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --skip-tags pbs,oidc
```
Configures repos, ZFS storage, firewall, notifications, and Terraform/Packer API users. PBS registration and OIDC are skipped — neither is deployed yet.

**Post-run:**
- Log into the PVE web UI and set up TOTP for `sysop@pam`: username → Two Factor → Add TOTP.

### 4. PBS — install and configure
```bash
ansible-playbook -i inventory/hosts.yml playbooks/pbs.yml --skip-tags oidc
```
Installs the PBS package, configures repos, NFS mount, datastore, prune/GC schedules, firewall, and users.

**Post-run:**
- Log into the PBS web UI and set up TOTP for `sysop@pam`: Administration → User Management → `sysop@pam` → TFA → Add TOTP.

### 5. Proxmox — register PBS storage and backup job
```bash
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --tags pbs
```
Registers PBS as a PVE storage backend and creates a nightly backup job (`--all 1`). Templates and the PBS VM are automatically excluded. Notifications fire on errors and warnings only.

### 6. Services VM — deploy Traefik, Authentik, UniFi, and Homepage
```bash
ansible-playbook -i inventory/hosts.yml playbooks/svc.yml
```
Installs Docker, mounts NFS storage, configures UFW, and deploys Traefik, Authentik, UniFi Network Application, and Homepage as Docker Compose stacks. Also:
- Generates OIDC client secrets for each `proxy_services` entry and stores them in SSM
- Auto-generates MongoDB credentials for UniFi and stores them in SSM
- Templates the Authentik blueprint (`/blueprints/custom/proxy-services.yml`) and applies it on container start — creates OAuth2/OIDC providers and applications for each service
- Creates Traefik file-provider routes for each service in `proxy_services`

**Post-run:**
- Complete Authentik initial setup at `https://authentik.<your-domain>/if/flow/initial-setup/`
- Enable TOTP for your admin account under User Settings → MFA Devices
- Access UniFi at `https://unifi.<your-domain>/` — on first run the setup wizard allows restoring a `.unf` backup
- Homepage is available at `https://homepage.<your-domain>/` — edit `roles/svc/templates/homepage/config/services.yaml.j2` to add or remove service tiles

### 7. Configure OIDC realms on PVE and PBS
```bash
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --tags oidc
ansible-playbook -i inventory/hosts.yml playbooks/pbs.yml --tags oidc
```
Registers the Authentik OpenID realm on each system, syncs client credentials from SSM, and grants `Administrator`/`Admin` to all users listed in `pve_oidc_admins` / `pbs_oidc_admins`.

Users in `pve_oidc_admins` and `pbs_oidc_admins` can now log in via **Sign in with Authentik** on the PVE and PBS login screens.

---

## Adding a New Proxied Service

All proxy routing and Authentik integration is driven from the `proxy_services` list in `inventory/group_vars/all.yml`. Adding a service is a one-file change:

```yaml
proxy_services:
  - name: my-app           # identifier — used for route/provider names
    label: "My App"        # displayed in Authentik UI
    domain: "my-app.svc.example.com"
    backend_url: "http://192.168.x.x:3000"
    backend_tls_skip_verify: false
    # auth: none | forward_auth | oidc
    auth: forward_auth     # gate behind Authentik without native SSO
```

Then re-run:
```bash
ansible-playbook -i inventory/hosts.yml playbooks/svc.yml --tags traefik,authentik
```

For services with native OIDC support (e.g. Grafana), set `auth: oidc` and add an `oidc:` block. See `all.yml.example` for the full field reference and commented examples.

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

# OIDC realm config and admin ACLs (run after any proxy_services or pve/pbs_oidc_admins change)
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --tags oidc
ansible-playbook -i inventory/hosts.yml playbooks/pbs.yml --tags oidc

# Redeploy Traefik (config or version change)
ansible-playbook -i inventory/hosts.yml playbooks/svc.yml --tags traefik

# Redeploy Authentik (config or version change, re-applies blueprint)
ansible-playbook -i inventory/hosts.yml playbooks/svc.yml --tags authentik

# Redeploy UniFi (config or version change)
ansible-playbook -i inventory/hosts.yml playbooks/svc.yml --tags unifi

# Redeploy Homepage (config or version change)
ansible-playbook -i inventory/hosts.yml playbooks/svc.yml --tags homepage
```
