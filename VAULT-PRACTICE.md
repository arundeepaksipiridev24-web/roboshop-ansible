# Vault practice on EC2 and VS Code

The project keeps ordinary settings in group_vars/all/main.yaml and encrypted
passwords in group_vars/all/vault.yaml. The encryption password stays only on
the EC2 control node at ~/.ansible/roboshop-vault-pass. It is a separate password
from the five application/login passwords stored inside the vault.

## 1. Create the vault password file on EC2

Run as ec2-user:

```bash
mkdir -p ~/.ansible
chmod 700 ~/.ansible
umask 077
vi -n ~/.ansible/roboshop-vault-pass
```

Press i, enter one strong vault password on a single line, press Esc, then :wq.
Use your own value; this is the password that encrypts/decrypts the YAML file.

```bash
chmod 600 ~/.ansible/roboshop-vault-pass
cd ~/roboshop-ansible
```

## 2. Create the service password YAML using vi

If using the newly restructured local project, the previous passwords are
preserved in ignored .local/credentials.yaml. You can use those values.
If your EC2 checkout still has group_vars/all/credentials.yaml, migrate it first:

```bash
mkdir -p .local
chmod 700 .local
mv group_vars/all/credentials.yaml .local/credentials.yaml
chmod 600 .local/credentials.yaml
```

Run that migration only if the old file exists. It must not remain inside
group_vars/all, where Ansible would load the plaintext values too.

```bash
cp examples/vault.example.yaml group_vars/all/vault.yaml
vi -n group_vars/all/vault.yaml
```

Replace all five placeholders with your real passwords, keeping the variable
names unchanged. Use the existing instance SSH/sudo passwords and existing
MySQL root password. Save with Esc, :wq.

## 3. Encrypt on EC2

```bash
ansible-vault encrypt group_vars/all/vault.yaml
head -n 1 group_vars/all/vault.yaml
```

The first line should be $ANSIBLE_VAULT;1.1;AES256. ansible.cfg already contains:

```ini
[defaults]
vault_password_file = ~/.ansible/roboshop-vault-pass
```

The configured file supplies the encryption password automatically.

## 4. Copy encrypted data into VS Code

```bash
cat group_vars/all/vault.yaml
```

Copy the entire encrypted output, including the $ANSIBLE_VAULT header, into
E:\devops-shell-script\roboshop-ansible\group_vars\all\vault.yaml in VS Code.
Preserve all lines. Only this encrypted file belongs in Git; never copy the
vault password file or the .local plaintext file into Git.
You can alternatively download the encrypted file using your SSH file-transfer tool.

## 5. Run everything without a vault prompt

```bash
ansible-playbook main.yaml
```

For just Shipping:

```bash
ansible-playbook 08-shipping.yaml
```

## 6. Edit passwords later

```bash
EDITOR=vi ansible-vault edit group_vars/all/vault.yaml
```

It decrypts temporarily for editing and re-encrypts when you save and exit.
To view decrypted values privately on EC2, use ansible-vault view; do not paste
that decrypted output into Git or chat.

For interactive practice instead of the saved password file:

```bash
ANSIBLE_VAULT_PASSWORD_FILE= ansible-playbook main.yaml --ask-vault-pass
```

A new control node needs the same vault password recreated in its password file.
The local project is not encrypted yet: you choose the vault password and perform
the encryption on EC2. Keep the ignored old plaintext copy only until you verify
the encrypted vault works, then remove that backup.

Usernames are configured separately in group_vars/all/users.yaml. Only passwords go in vault.yaml. Infrastructure is in infrastructure.yaml and endpoints/artifact settings in services.yaml.
