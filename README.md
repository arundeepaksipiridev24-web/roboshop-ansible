# RoboShop Ansible

This project follows the flat layout in `../roboshop-ansible-v1` and the
installation steps and service order in `../roboshop-documentation`.

## Layout

- `01-mongodb.yaml` through `11-frontend.yaml`: one playbook per service,
  numbered in documentation order. Payment and Dispatch run before Frontend.
- `*.service.j2`, `roboshop.conf.j2`, `mongo.repo`, `rabbitmq.repo`: assets
  beside the playbooks, as in the reference project.
- `group_vars/all/main.yaml`: shared infrastructure settings, service endpoints,
  artifact URL and version. The `all/` directory also keeps the existing
  `credentials.yaml`; this avoids splitting the all-group defaults between a file
  and a directory. All passwords are defined only in
  `group_vars/all/credentials.yaml`.
- `roboshop.yaml`: existing EC2 and Route 53 provisioning entry point.
- `services.yaml`: databases, backends, then frontend; uses existing hosts.
- `main.yaml`: provisioning, SSH setup, then all services.

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
`group_vars/all/main.yaml`, and the existing VPC, subnet and hosted-zone
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
`group_vars/all/credentials.yaml`. Change them there if the AMI login changes.
No `-k`, `-K`, or Vault password flag is needed. The first SSH connection accepts
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
