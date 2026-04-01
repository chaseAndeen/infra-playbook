# infra-playbook

Ansible IaC for Proxmox VE, Proxmox Backup Server, and Teleport. Secrets managed via AWS SSM Parameter Store, accessed through AWS IAM Identity Center SSO.

---

## Architecture

Teleport is the single access point for all infrastructure:

- **SSH** — all VM access goes through Teleport's SSH CA. No static keys needed after initial bootstrap.
- **Web UIs** — PVE, PBS, TrueNAS, and future services are registered as Teleport app proxy applications. One login at `teleport.infra.kernelstack.dev` gives access to everything.
- **Break-glass** — local PAM admin user exists on PVE and PBS for direct access if Teleport is unavailable. TOTP is not enforced at the realm level — configure it per-user via the web UI after first login.

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
| `/infra/common/cloudflare/api_token` | SecureString | Cloudflare API token with DNS Edit permission — required for ACME |
| `/infra/common/cloudflare/zone_id` | String | Cloudflare zone ID |
| `/infra/common/smtp/user` | String | SMTP username |
| `/infra/common/smtp/pass` | SecureString | SMTP password |
| `/infra/proxmox/terraform_tmp_pass` | SecureString | Initial password for `terraform-prov@pve` — used only at user creation |
| `/infra/proxmox/packer_api_password` | SecureString | Initial password for `packer-builder@pve` — used only at user creation |
| `/infra/proxmox/terraform_token` | SecureString | Proxmox API token for Terraform — written automatically by `site.yml` |
| `/infra/proxmox/packer_token` | SecureString | Proxmox API token for Packer — written automatically by `site.yml` |
| `/infra/pbs/backup_user_password` | SecureString | Password for `backup-user@pbs` |
| `/infra/pbs/fingerprint` | String | PBS TLS fingerprint — written automatically by `pbs.yml` |
| `/infra/teleport/node_token` | SecureString | Teleport node enrollment token — written automatically by `teleport.yml` |

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
ansible-playbook -i inventory/hosts.yml playbooks/teleport.yml --tags base
```

### 3. Proxmox — post-install configuration
```bash
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --skip-tags pbs,teleport_agent
```
Configures repos, ZFS storage, firewall, notifications, and Terraform/Packer API users. PBS registration and Teleport agent enrollment are skipped — PBS isn't deployed yet and the enrollment token doesn't exist in SSM until step 6.

**Post-run:**
- Log into the PVE web UI and set up TOTP under your username → Two Factor → Add TOTP.

### 4. PBS — provision and configure
```bash
ansible-playbook -i inventory/hosts.yml playbooks/pbs.yml
```
Configures repos, NFS mount, datastore, prune/GC schedules, firewall, and users.

**Post-run:**
- Log into the PBS web UI and set up TOTP manually:
  Administration → User Management → `sysop@pam` → TFA → Add TOTP

### 5. Proxmox — register PBS storage and backup job
```bash
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --tags pbs
```
Registers PBS as a PVE storage backend and creates a nightly backup job (`--all 1`). Templates and the PBS VM are automatically excluded on every run — new Packer templates are picked up dynamically. All other VMs are backed up. Notifications fire on errors and warnings only.

### 6. Teleport — configure
```bash
ansible-playbook -i inventory/hosts.yml playbooks/teleport.yml
```
Obtains a TLS certificate via Cloudflare DNS-01 challenge (no inbound port 443 required), writes production config, configures firewall, creates admin user, registers PVE/PBS/TrueNAS as app proxy applications, and generates a node enrollment token stored in SSM.

**Post-run:**
- Use the invite link printed by the `users` task to complete Teleport admin account setup
- Configure the following OPNsense Unbound host overrides, all pointing at the Teleport VM IP:
  - `teleport` → `infra.kernelstack.dev`
  - `*.teleport` → `infra.kernelstack.dev` (wildcard for app subdomains)
  - `datavault` → `infra.kernelstack.dev` (TrueNAS served via Teleport `public_addr`)

### 7. Enroll nodes
Node enrollment is automated. Each app entry in `teleport_apps` has an `install_agent` bool that controls whether the Teleport node agent is installed on the corresponding host. Set `install_agent: true` for any host you want enrolled.

```bash
# Enroll PVE nodes (runs as part of site.yml, or standalone):
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --tags teleport_agent

# Enroll PBS nodes:
ansible-playbook -i inventory/hosts.yml playbooks/pbs.yml --tags teleport_agent
```

The playbook downloads the Teleport `.deb` directly (Debian Trixie is not in the Teleport APT repo), configures the node using the token from SSM, and enables the service. Re-runs are idempotent — the binary and config are only written if absent.

---

## Day-2 Operations

All playbooks are idempotent and safe to re-run at any time.

```bash
# Password rotation — update SSM then run:
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --tags base
ansible-playbook -i inventory/hosts.yml playbooks/pbs.yml --tags base
ansible-playbook -i inventory/hosts.yml playbooks/teleport.yml --tags base

# TLS certificate renewal (Teleport — certbot runs automatically via systemd timer)
# To force a manual renewal:
ansible-playbook -i inventory/hosts.yml playbooks/teleport.yml --tags acme

# System (repos, PVE users, vzdump config)
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --tags system

# Storage (ZFS provisioning, local-lvm)
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --tags storage

# Firewall rules
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --tags firewall
ansible-playbook -i inventory/hosts.yml playbooks/pbs.yml --tags network
ansible-playbook -i inventory/hosts.yml playbooks/teleport.yml --tags firewall

# Notification configuration
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --tags notifications
ansible-playbook -i inventory/hosts.yml playbooks/pbs.yml --tags notifications

# Teleport app registrations (add/update entries in teleport_apps in group_vars/all.yml, then:)
ansible-playbook -i inventory/hosts.yml playbooks/teleport.yml --tags config

# Teleport node agent upgrade (update teleport_agent_version in group_vars/all.yml, then:)
# Note: only re-installs if the binary is absent. To force upgrade, remove /usr/local/bin/teleport first.
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --tags teleport_agent
ansible-playbook -i inventory/hosts.yml playbooks/pbs.yml --tags teleport_agent

# PBS storage registration
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --tags pbs

# PBS datastore and users
ansible-playbook -i inventory/hosts.yml playbooks/pbs.yml --tags datastore
ansible-playbook -i inventory/hosts.yml playbooks/pbs.yml --tags users
```
