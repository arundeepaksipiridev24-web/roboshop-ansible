# RoboShop Ansible

Configure usernames in `group_vars/all/users.yaml`, AWS/DNS settings in
`infrastructure.yaml`, and endpoints/artifact versions in `services.yaml`.
Create encrypted `group_vars/all/vault.yaml` using [VAULT-PRACTICE.md](VAULT-PRACTICE.md).
The vault unlock password stays at `~/.ansible/roboshop-vault-pass` on EC2.

## Deploy

From the repository directory on your Linux control node:

```bash
ansible-galaxy collection install -r requirements.yaml
ansible-playbook main.yaml
```

Install Ansible, AWS CLI, boto3/botocore in the Python used by Ansible, and
sshpass on the control node first. Configure AWS credentials or an instance role.
Use compatible Ansible/collection versions; the previously reported unsupported
Ansible 2.14 combination is not validated by these static checks.

`main.yaml` provisions instances, updates DNS, bootstraps SSH and deploys all
services. The control node must reach every target private IP on TCP 22.
Security groups, routes and internet/NAT access for package downloads must
already exist. Public DNS points to the frontend public IP.

For an existing service:

```bash
ansible-playbook 08-shipping.yaml
```

For service deployment without provisioning:

```bash
ansible-playbook services.yaml
```

Individual runs use inventory.ini: set `ansible_host` to the current private IP
for publicly resolving hostnames, especially the frontend. Full provisioning
sets private addresses dynamically for that run.

## Code layout

- Numbered playbooks contain service-specific steps and short shared-task imports.
- `tasks/application.yaml`: application account, directory, archive download/extraction.
- `tasks/nodejs.yaml`: Node.js runtime and dependencies, with a successful-install marker.
- `tasks/service.yaml`: systemd template, start, pending restart and HTTP health check.
- `tasks/rabbitmq-environment.yaml`: private connection password files.
- `handlers/application.yaml`: common application restart.
- `templates/` and `files/`: service configurations and package repository definitions.
- `group_vars/all/`: separate infrastructure, account, service and encrypted-password settings.

Shared tasks retain the existing /app layout and only extract when artifacts
change or the source entry point is missing. Shipping keeps Java build/database
steps in its own playbook; Dispatch checks systemd rather than an HTTP endpoint.
Initial MySQL root password setup uses ALTER USER only when passwordless login
works and the configured password does not. Existing unknown passwords cause
an explicit failure rather than resetting data.

## Teardown

```bash
ansible-playbook delete-instances.yaml --check
ansible-playbook delete-instances.yaml
```

Teardown removes configured application A records and the frontend root A
record, then terminates matching managed/tagged instances. It retains the zone,
unrelated DNS records, security groups and VPC. Termination is permanent.

## Checks before running

```bash
ansible-playbook main.yaml --syntax-check
ansible-playbook services.yaml --syntax-check
ansible-inventory --graph
```

Review [CONFIGURATION.md](CONFIGURATION.md) for account-change limitations.
