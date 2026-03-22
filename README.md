# proxmox-playbook

Ansible IaC for Proxmox VE and Proxmox Backup Server. Secrets managed via AWS SSM Parameter Store, accessed through AWS IAM Identity Center SSO.

---

## Prerequisites

### Dependencies
```bash
ansible-galaxy collection install community.general amazon.aws ansible.posix
pipx inject ansible-core boto3 botocore
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
| `/infra/common/public_key` | String | Admin user SSH public key — provisioned by bootstrap |
| `/infra/common/ansible_public_key` | String | `ansible` service account SSH public key |
| `/infra/common/ansible_private_key` | SecureString | `ansible` service account SSH private key |
| `/infra/common/cloudflare/api_token` | SecureString | Cloudflare API Token with DNS Edit permission — required for ACME |
| `/infra/common/cloudflare/zone_id` | String | Cloudflare zone ID |
| `/infra/common/smtp/user` | String | SMTP username |
| `/infra/common/smtp/pass` | SecureString | SMTP password |
| `/infra/oidc/aws/issuer_url` | String | Cognito OIDC issuer URL — `https://cognito-idp.<region>.amazonaws.com/<pool-id>` |
| `/infra/oidc/aws/pve/client_id` | String | Cognito app client ID for Proxmox PVE |
| `/infra/oidc/aws/pve/client_secret` | SecureString | Cognito app client secret for Proxmox PVE |
| `/infra/oidc/aws/pbs/client_id` | String | Cognito app client ID for PBS |
| `/infra/oidc/aws/pbs/client_secret` | SecureString | Cognito app client secret for PBS |
| `/infra/oidc/teleport/client_id` | String | Teleport OIDC application client ID |
| `/infra/oidc/teleport/client_secret` | SecureString | Teleport OIDC application client secret |
| `/infra/proxmox/terraform_tmp_pass` | SecureString | Initial password for `terraform-prov@pve` — used only at user creation |
| `/infra/proxmox/packer_api_password` | SecureString | Initial password for `packer-builder@pve` — used only at user creation |
| `/infra/proxmox/terraform_token` | SecureString | Proxmox API token for Terraform — written automatically by `site.yml` |
| `/infra/proxmox/packer_token` | SecureString | Proxmox API token for Packer — written automatically by `site.yml` |
| `/infra/pbs/backup_user_password` | SecureString | Password for `backup-user@pbs` |
| `/infra/pbs/fingerprint` | String | PBS TLS fingerprint — written automatically by `pbs.yml` |

---

## Cognito OIDC Setup Notes

Proxmox and PBS use AWS Cognito as a fallback OIDC realm (`aws-sso`) independent of Teleport.

**Cognito app client requirements:**
- App type: Confidential client
- Grant type: Authorization code
- Scopes: `openid`, `profile`, `email`
- Allowed callback URLs:
  - PVE: `https://<pve-hostname>:<gui-port>`
  - PBS: `https://<pbs-hostname>:8007`

**Username mapping:** Set `preferred_username` on each Cognito user to match their desired Proxmox username. The realm uses `--username-claim preferred_username --query-userinfo 0` to read claims from the ID token directly.

**First-login flow:** The realm uses `autocreate 1` so users are created on first login. After logging in for the first time, re-run `--tags oidc_aws` to assign the user to the `oidc-admins` group and gain Administrator permissions.

---

## Deployment Order

### 1. Bootstrap — Proxmox first-time setup
Runs as `root` on port 22. Creates the admin and ansible OS accounts, transitions SSH to port 2222.
```bash
ansible-playbook -i inventory/hosts.yml playbooks/bootstrap.yml
```

**Post-run:**
- SSH in as root and set passwords: `passwd <admin-user> && passwd root`
- Log into Proxmox UI and configure TOTP for the admin user

### 2. Proxmox — post-install configuration
```bash
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --skip-tags pbs
```
Configures repos, ZFS storage, firewall, ACME cert, Terraform/Packer API users, and Cognito OIDC realm.

### 3. Proxmox — first Cognito login
Log into the Proxmox UI using the `aws-sso` realm, then assign group membership:
```bash
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --tags oidc_aws
```

### 4. PBS — provision and configure
```bash
ansible-playbook -i inventory/hosts.yml playbooks/pbs.yml
```
Configures repos, NFS mount, datastore, prune/GC schedules, UFW firewall, users, and Cognito OIDC realm.

### 5. PBS — first Cognito login
Log into the PBS UI using the `aws-sso` realm, then assign group membership:
```bash
ansible-playbook -i inventory/hosts.yml playbooks/pbs.yml --tags oidc_aws
```

### 6. Proxmox — register PBS storage and backup job
```bash
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --tags pbs
```

### 7. Teleport — configure OIDC
Once Teleport is deployed and the OIDC app is created, seed the Teleport SSM parameters then run:
```bash
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --tags oidc_teleport
ansible-playbook -i inventory/hosts.yml playbooks/pbs.yml --tags oidc_teleport
```

---

## Day-2 Operations

All playbooks are idempotent and safe to re-run at any time.

```bash
# ACME certificate renewal
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --tags acme

# Firewall rules
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --tags firewall

# Notification configuration
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --tags notifications
ansible-playbook -i inventory/hosts.yml playbooks/pbs.yml --tags notifications

# OIDC realms
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --tags oidc_aws
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --tags oidc_teleport
ansible-playbook -i inventory/hosts.yml playbooks/pbs.yml --tags oidc_aws
ansible-playbook -i inventory/hosts.yml playbooks/pbs.yml --tags oidc_teleport

# PBS storage registration
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --tags pbs

# PBS datastore and users
ansible-playbook -i inventory/hosts.yml playbooks/pbs.yml --tags datastore
ansible-playbook -i inventory/hosts.yml playbooks/pbs.yml --tags users
```
