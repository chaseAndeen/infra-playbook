# infra-playbook

Ansible IaC for Proxmox VE, Proxmox Backup Server, and Teleport. Secrets managed via AWS SSM Parameter Store, accessed through AWS IAM Identity Center SSO.

---

## Architecture

Teleport is the single access point for all infrastructure:

- **SSH** — all VM access goes through Teleport's SSH CA. No static keys needed after initial bootstrap.
- **Web UIs** — PVE, PBS, TrueNAS, and future services are registered as Teleport app proxy applications. One login at `teleport.infra.kernelstack.dev` gives access to everything.
- **Break-glass** — local PAM admin user exists on PVE and PBS for direct access if Teleport is unavailable. TOTP is enforced on PVE PAM; PBS requires manual TOTP setup via the web UI.

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
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --skip-tags pbs
```
Configures repos, ZFS storage, firewall, ACME cert, Terraform/Packer API users, and enforces TOTP on the PAM realm.

**Post-run:**
- Log into the PVE web UI — you will be prompted to enrol an authenticator app before proceeding.

### 4. PBS — provision and configure
```bash
ansible-playbook -i inventory/hosts.yml playbooks/pbs.yml
```
Configures repos, NFS mount, datastore, prune/GC schedules, firewall, and users.

**Post-run:**
- PBS does not support realm-level TFA enforcement. Log into the PBS web UI and set up TOTP manually:
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
- Ensure `teleport.infra.kernelstack.dev` has a Cloudflare DNS A record pointing at the Teleport VM

### 7. Enroll nodes
After Teleport is configured, enroll PVE, PBS, and any other VMs using the token stored in SSM.

Retrieve the token:
```bash
NODE_TOKEN=$(aws ssm get-parameter \
  --name /infra/teleport/node_token \
  --with-decryption \
  --query Parameter.Value \
  --output text \
  --profile InfraProvisioner)
```

On each VM, generate a config and enable the service:
```bash
sudo teleport configure \
  --roles=node \
  --token=$NODE_TOKEN \
  --proxy=teleport.infra.kernelstack.dev:443 \
  --output=/etc/teleport.yaml

sudo systemctl enable --now teleport
```

---

## Day-2 Operations

All playbooks are idempotent and safe to re-run at any time.

```bash
# Password rotation — update SSM then run:
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --tags base
ansible-playbook -i inventory/hosts.yml playbooks/pbs.yml --tags base
ansible-playbook -i inventory/hosts.yml playbooks/teleport.yml --tags base

# ACME certificate renewal (PVE — Cloudflare DNS-01)
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --tags acme

# TLS certificate renewal (Teleport — certbot runs automatically via systemd timer)
# To force a manual renewal:
ansible-playbook -i inventory/hosts.yml playbooks/teleport.yml --tags acme

# TFA enforcement check
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --tags tfa
ansible-playbook -i inventory/hosts.yml playbooks/pbs.yml --tags tfa

# Firewall rules
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --tags firewall
ansible-playbook -i inventory/hosts.yml playbooks/pbs.yml --tags firewall
ansible-playbook -i inventory/hosts.yml playbooks/teleport.yml --tags firewall

# Notification configuration
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --tags notifications
ansible-playbook -i inventory/hosts.yml playbooks/pbs.yml --tags notifications

# Teleport app registrations
ansible-playbook -i inventory/hosts.yml playbooks/teleport.yml --tags apps

# PBS storage registration
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --tags pbs

# PBS datastore and users
ansible-playbook -i inventory/hosts.yml playbooks/pbs.yml --tags datastore
ansible-playbook -i inventory/hosts.yml playbooks/pbs.yml --tags users
```
