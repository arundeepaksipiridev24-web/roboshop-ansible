# Configuration map

Edit these files before deployment:

| File | Purpose |
|---|---|
| group_vars/all/infrastructure.yaml | Region, AMI, VPC, subnet, DNS zone, instance list |
| group_vars/all/users.yaml | SSH login, OS service account, RabbitMQ and database usernames |
| group_vars/all/services.yaml | Service endpoints and artifact version |
| group_vars/all/vault.yaml | Encrypted passwords only; create on EC2 using VAULT-PRACTICE.md |
| inventory.ini | Target hostnames for individual service runs |
| templates/ | Systemd units and Nginx configuration |
| files/ | Package repository definitions |

Numbered service playbooks remain at the project root, following the reference.
Run everything with `ansible-playbook main.yaml`, or only Shipping with
`ansible-playbook 08-shipping.yaml`. Run from this repository directory.
Ansible automatically loads the files under group_vars/all for both commands.

The private/public IP behavior remains: provisioning registers private IPs for
Ansible, while public DNS points to the frontend public IP. Individual service
runs use inventory.ini, so set ansible_host to the current private IP there for
any publicly resolving hostname (especially the frontend).

Keep mysql_admin_user as root for a fresh MySQL installation; changing an
administrator name does not create or rename an administrator. Changing ssh_user
requires an existing login account in the AMI. Changing app_user creates the new
OS account; changing RabbitMQ/Shipping usernames creates the configured account
and updates clients, but does not remove previous accounts or migrate data.
Shipping usernames must use letters, numbers and underscores.
The local .local/credentials.yaml backup remains ignored and is never loaded.
