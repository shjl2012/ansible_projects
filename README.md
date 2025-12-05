# ansible自動化設定包

## Ansible Installation

There are three common ways to use or install Ansible on your control node:

### 1. Install Ansible Locally (on your system)
This method installs Ansible directly on your system. Installation steps vary by operating system.

**For complete installation instructions for all distributions, see:**
https://docs.ansible.com/projects/ansible/latest/installation_guide/installation_distros.html

#### Ubuntu/Debian (WSL or Linux)
```bash
# Add Ansible PPA repository (for latest version)
sudo apt update
sudo apt install -y software-properties-common
sudo add-apt-repository --yes --update ppa:ansible/ansible

# Install Ansible
sudo apt install -y ansible python3-pip

# (Optional) Upgrade pip and install additional dependencies
python3 -m pip install --upgrade pip
```

#### RHEL/CentOS/Rocky Linux
```bash
# Enable EPEL repository
sudo dnf install -y epel-release

# Install Ansible
sudo dnf install -y ansible python3-pip

# (Optional) Upgrade pip
python3 -m pip install --upgrade pip
```

#### Fedora
```bash
# Install Ansible (available in default repositories)
sudo dnf install -y ansible python3-pip

# (Optional) Upgrade pip
python3 -m pip install --upgrade pip
```

#### macOS (using Homebrew)
```bash
# Install Ansible via Homebrew
brew install ansible

# Install pip if needed
brew install python3
```

#### Install via pip (Any OS with Python 3.9+)
```bash
# Install Ansible using pip (recommended for latest version)
python3 -m pip install --user ansible

# Verify installation
ansible --version
```

### 2. Use ansible-navigator with an Execution Environment (No local Ansible install required)
This method only requires ansible-navigator and a container runtime (like Docker or Podman). Ansible and all dependencies are provided by the execution environment image.

```bash
# Install ansible-navigator using pip
python3 -m pip install ansible-navigator

# (Optional) Create or edit ansible-navigator.yml for project-specific settings
# Example: ansible-navigator.yml

# Run playbooks using ansible-navigator (Ansible runs inside the container)
ansible-navigator run playbooks/site.yml -i inventory
```

### 3. Use Ansible via Official Docker Image (No local Ansible or ansible-navigator required)
This method uses Docker to run Ansible directly from the official container image.

```bash
# Pull the official Ansible image
docker pull quay.io/ansible/ansible-runner

# Run Ansible playbook using the container (adjust volume paths as needed)
docker run --rm -it -v $(pwd):/workspace -w /workspace quay.io/ansible/ansible-runner ansible-playbook -i inventory playbooks/site.yml
```

## Ansible Initialization

### 1. Create General Project Folder Structure
A typical, general-purpose Ansible project structure:

```
ansible_project/
├── README.md
├── ansible.cfg
├── inventory
├── playbooks/
│   ├── site.yml
│   └── other_playbook.yml
├── group_vars/
│   ├── all.yml
│   └── group1.yml
├── host_vars/
│   ├── host1.yml
│   └── host2.yml
├── roles/
│   └── myrole/
│       ├── tasks/
│       ├── handlers/
│       ├── templates/
│       ├── files/
│       └── vars/
├── templates/
│   └── mytemplate.j2
├── files/
│   └── myfile.txt
├── collections/
│   └── ...
└── requirements.yml
```

- Place playbooks in the `playbooks/` directory.
- Use `templates/` and `files/` for shared resources.
- Use `collections/` for local Ansible collections.
- Use `roles/` for reusable role definitions.
- Use `group_vars/` and `host_vars/` for variable scoping.

### 2. Generate a Sample ansible.cfg

#### Example ansible.cfg (based on your st/ansible.cfg):
```ini
[defaults]
inventory = ./inventory
collections_paths = ./collections
roles_path = ./roles
host_key_checking = False
retry_files_enabled = False
stdout_callback = yaml
```

#### Official Method (from Ansible documentation):
Generate a full sample config with all options (disabled by default):
```powershell
ansible-config init --disabled > ansible.cfg
```
You can then edit this file to enable and customize the settings you need.

### 3. Generate a Sample ansible-navigator.yml (Optional)
If you use ansible-navigator, create `ansible-navigator.yml` in your project root:

```yaml
---
ansible-navigator:
  execution-environment:
    enabled: true
    image: quay.io/ansible/ansible-runner
    pull-policy: missing
  logging:
    level: info
  app:
    color: true
```

## Useful Ansible Project Commands

### 1. Checking Documentation

```bash
# Read documentation for a module from a local collection
ANSIBLE_COLLECTIONS_PATHS=./collections ansible-doc community.general.pam_limits

# Read documentation for a built-in module
ansible-doc copy

# List all available modules
ansible-doc -l

# Show documentation for a role (if using roles)
# (Navigate to the role directory and check README.md or docs/)
cat roles/myrole/README.md
```

### 2. Installing and Managing Collections & Roles

```bash
# Install a collection locally
ansible-galaxy collection install community.general -p ./collections

# List installed collections in the local path
ansible-galaxy collection list -p ./collections

# Upgrade a collection in the local path
ansible-galaxy collection install community.general -p ./collections --force

# Install roles from a requirements file
ansible-galaxy role install -r requirements.yml -p ./roles

# Download the official community.general collection as a tarball
ansible-galaxy collection download community.general -p ./collections

# Install the collection tarball into your local collections path
ansible-galaxy collection install community-general-*.tar.gz -p ./collections
```

### 3. Running Playbooks

```bash
# Run a playbook using the default inventory
ansible-playbook -i inventory playbooks/site.yml

# Run a playbook with a specific inventory and extra variables
ansible-playbook -i inventory playbooks/site.yml -e "foo=bar"

# Run a playbook and pass a secrets file (YAML vars)
ansible-playbook -i inventory playbooks/site.yml --extra-vars "@secrets.yml"

# Run a playbook using local collections
ansible-playbook -i inventory playbooks/site.yml

# Run a playbook with ansible-navigator (if configured)
ansible-navigator run playbooks/site.yml -i inventory

# Run a playbook and prompt for sudo/root password
ansible-playbook -i inventory playbooks/site.yml -K

# Run a playbook with a secrets file and prompt for sudo/root password
ansible-playbook -i inventory playbooks/site.yml --extra-vars "@secrets.yml" -K

# Run a playbook with an encrypted secrets file (Ansible Vault) and prompt for both vault and sudo/root password
ansible-playbook -i inventory playbooks/site.yml --extra-vars "@secrets.yml" --ask-vault-pass -K
```

---
Add more commands as needed for your workflow!

