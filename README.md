# RoboShop Ansible

See [CONFIGURATION.md](CONFIGURATION.md) for the configuration file structure.
See [VAULT-PRACTICE.md](VAULT-PRACTICE.md) for creating encrypted passwords on EC2.

## Requirements and configuration

Run Ansible from a Linux control node with access to the RHEL 9 servers.
The documentation requires RHEL 9.8 or newer for Python 3.14. Service versions:
MongoDB 7.0, Redis 7, MySQL 8.4, RabbitMQ 4.x, Node.js 24,
Java 25 with Maven 3.9, Python 3.14, Go >=1.25 and Nginx 1.26.
Application artifacts default to `v4`.

```shell
ansible-galaxy collection install -r requirements.yaml
```

Review `inventory.ini`, SSH settings in `ansible.cfg`, shared settings in
`group_vars/all/infrastructure.yaml`, and the existing VPC, subnet and hosted-zone
settings in the provisioning playbooks before provisioning. Inventory uses
`sodhichebutha.online`; update its entries too if changing `domain_name`.
The provisioning workflow also requires AWS CLI, boto3/botocore and AWS credentials
on the control node, plus the security groups described in the documentation.

## Run everything with one command

From this repository on the Linux control node, run as `ec2-user`:

```shell
ansible-playbook main.yaml
```

This creates the instances, updates DNS, waits for connections, prepares the
control node SSH key, installs it on all servers, configures passwordless sudo,
and deploys all eleven services in documentation order.

SSH, sudo, RabbitMQ, MySQL root and Shipping database passwords are set in
`group_vars/all/vault.yaml`. Change them there if the AMI login changes.
Complete VAULT-PRACTICE.md first. The configured EC2 vault password file supplies the decryption password automatically. No `-k` or `-K` is needed. The first SSH connection accepts
new host keys automatically; changed host keys are still rejected. An existing
control node private key must have no passphrase for this unattended workflow.
Shipping's database user is configured from `shipping_db_password`; the
password embedded in the downloaded `db/app-user.sql` is not used.

One-time control node prerequisites: Ansible, `sshpass`, OpenSSH tools,
AWS CLI, boto3/botocore, the collections in `requirements.yaml`, and configured
AWS credentials. The AMI must allow the SSH password configured in YAML or an
existing authorized control node key. The Linux control node needs network
access to all instance private IPs.

Before deployment, validate on the Linux control node:

```shell
ansible-inventory --graph
ansible-playbook services.yaml --syntax-check
ansible-playbook main.yaml --syntax-check
```

The playbooks check service health after deployment. Open the frontend and use
**System status** to verify backend connectivity, then place an order to verify
Payment and Dispatch. The Payment proxy route is included in the default Nginx
configuration, as documented.

## Delete application instances

Preview the matched instances:

```shell
ansible-playbook delete-instances.yaml --check
```

Terminate all matched RoboShop application instances:

```shell
ansible-playbook delete-instances.yaml
```

This is permanent termination, including EBS volumes configured to delete on
termination. The playbook targets only the configured service names in the
configured VPC with both `Project=roboshop` and `ManagedBy=ansible` tags, as set
by `01-create-ec2.yaml`. It does not select instances by name alone. Untagged
older instances are excluded; check the preview if no instances match.
The control node is excluded unless it has itself been given all those matching
application tags and a configured service name. No password prompts are added.
DNS records, security groups, the VPC, retained EBS volumes and Elastic IPs are
left in place. `main.yaml` never imports the deletion playbook.

Shipping account setup uses the MySQL CLI with SQL on standard input and the
central credentials. No community.mysql collection or Python MySQL driver is
required. Passwords and raw database errors remain hidden; a failure reports
only the client exit status and MySQL error numbers.

## Encrypted credentials

Follow [VAULT-PRACTICE.md](VAULT-PRACTICE.md) to create and encrypt passwords on EC2 and copy only the encrypted YAML into VS Code. Previous local plaintext credentials are preserved under ignored `.local/credentials.yaml`; they are no longer loaded automatically.
