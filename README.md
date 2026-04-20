# aap-install

Installs Ansible Automation Platform 2.6 (containerized bundle) on a fresh RHEL 9 system using ansible-core.

---

## Prerequisites

On the RHEL 9 target server (as root):

```bash
dnf install -y ansible-core git
```

---

## Setup

### 1. Get your Red Hat offline token

Log in to https://access.redhat.com/management/api and generate an offline token.

### 2. Get the AAP 2.6 bundle SHA256

Go to https://access.redhat.com/downloads/content/480/ and select:

- **Product**: Ansible Automation Platform 2.6
- **File**: Ansible Automation Platform 2.6 Containerized Setup Bundle
- **Architecture**: x86_64 / RHEL 9

Copy the SHA256 checksum and update `playbooks/files/distributions.yml`.

### 3. Clone the repository

```bash
git clone https://github.com/mlowcher61/aap-install.git
cd aap-install
```

### 4. Install required collections

```bash
ansible-galaxy collection install -r collections/requirements.yml -p ./collections/
```

### 5. Create and encrypt your vault file

```bash
cp playbooks/files/example_vault.yml playbooks/files/vault.yml
```

Edit `playbooks/files/vault.yml` and fill in all values, then encrypt it:

```bash
ansible-vault encrypt playbooks/files/vault.yml
```

### 6. Add your SSH public key for the ansible-svc account

```bash
cat ~/.ssh/id_rsa.pub > playbooks/files/public_keys/ansible-svc
```

### 7. Run the setup playbook

```bash
ansible-playbook playbooks/setup.yml --ask-vault-pass
```

You will be prompted for:
- **Server FQDN** — the fully qualified domain name of this RHEL 9 server
- **SSH public key path** — path to the public key for the ansible-svc account (default: `~/.ssh/id_rsa.pub`)

The playbook will:
- Validate system requirements
- Create the `ansible-svc` service account
- Download the AAP 2.6 bundle from the Red Hat Customer Portal
- Extract the bundle
- Write the `inventory-growth` file with your configuration
- Copy the vault file into the installer directory

### 8. Run the AAP installer

Switch to the `ansible-svc` account and run the installer:

```bash
su - ansible-svc
cd ~/ansible-automation-platform-containerized-setup-bundle-2.6-*
ansible-playbook -i inventory-growth ansible.containerized_installer.install -e@vault.yml --ask-vault-pass
```

---

## File reference

| File | Purpose |
|------|---------|
| `playbooks/setup.yml` | Main setup playbook |
| `playbooks/files/example_vault.yml` | Example vault — copy to `playbooks/files/vault.yml` and encrypt |
| `playbooks/files/vault.yml` | Your encrypted vault (gitignored — never commit) |
| `playbooks/files/distributions.yml` | AAP 2.6 bundle SHA256 checksum |
| `playbooks/files/public_keys/ansible-svc` | SSH public key for the ansible-svc account |
| `playbooks/templates/inventory-growth.j2` | Template rendered into the AAP bundle's inventory-growth |

---

## Notes

- The `playbooks/files/vault.yml` file is gitignored and must never be committed.
- The Red Hat offline token expires after 30 days of non-use. Refresh it at https://access.redhat.com/management/api before running the playbook if it has been idle.
- Minimum system requirements: 16 GB RAM, 4 vCPUs, 60 GB disk, RHEL 9, Podman installed.
