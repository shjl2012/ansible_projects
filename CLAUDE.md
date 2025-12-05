# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is an Ansible automation repository for deploying and configuring Axway SecureTransport (ST) and Edge servers on Rocky Linux. The repository is organized into the following main sub-projects:

1. **st/** - SecureTransport and Edge deployment
2. **os_config/** - General OS and NFS server configuration
3. **dsom/** - Infrastructure server setup (NTP, DNS, NFS for Kubernetes clusters)

## Project Structure

Each sub-project follows standard Ansible structure with its own:
- `ansible.cfg` - Project-specific Ansible configuration
- `inventory` - Host inventory (ST/Edge groups or nfs_servers group)
- `playbooks/` - Ansible playbooks
- `group_vars/` - Group-specific variables
- `templates/` - Jinja2 templates for configuration files
- `files/` - Static files (installers, licenses)
- `collections/` - Local Ansible collections (ansible.posix, community.general)

## Common Commands

### Working in the st/ Directory

```bash
cd st

# Install required collections from local tarballs
ansible-galaxy collection install -r collections/requirements.yml -p ./collections

# Run playbooks in order:
# 1. Prerequisites (install dependencies, configure OS limits, mount NFS for ST)
ansible-playbook playbooks/st_prerequisites.yml --extra-vars "@secrets.yml" -K

# 2. Install SecureTransport (copy installer, unzip, template configs, run setup)
ansible-playbook playbooks/st_install.yml --extra-vars "@secrets.yml" -K

# 3. Configure SecureTransport (deploy licenses, change admin password, configure settings)
ansible-playbook playbooks/st_config.yml --extra-vars "@secrets.yml" -K

# Using ansible-navigator (containerized execution)
ansible-navigator run playbooks/st_prerequisites.yml --extra-vars "@secrets.yml" -K
```

### Working in the os_config/ Directory

```bash
cd os_config

# Collect facts from hosts and display on screen
ansible-playbook playbooks/collect_facts.yml

# Collect and export facts to control node filesystem (/tmp/*.json and /tmp/*.yml)
ansible-playbook playbooks/collect_facts_to_files.yml

# General OS setup (create devops user, configure sudo, disable SELinux/firewall)
ansible-playbook playbooks/os_general_setup.yml --extra-vars "@secrets.yml" -K

# NFS server setup (install nfs-utils, configure exports, start services)
ansible-playbook playbooks/nfs_server_setup.yml --extra-vars "@secrets.yml" -K
```

### Working in the dsom/ Directory

```bash
cd dsom

# Run complete infrastructure server setup (NTP, DNS, NFS)
ansible-playbook playbooks/site.yml --extra-vars "@secrets.yml" -K

# Run specific service setup with tags
ansible-playbook playbooks/site.yml --extra-vars "@secrets.yml" -K --tags ntp
ansible-playbook playbooks/site.yml --extra-vars "@secrets.yml" -K --tags dns
ansible-playbook playbooks/site.yml --extra-vars "@secrets.yml" -K --tags nfs

# Validate services after setup
ansible-playbook playbooks/site.yml --extra-vars "@secrets.yml" -K --tags validate
```

### Reading Module Documentation

```bash
# View documentation for modules from local collections
ANSIBLE_COLLECTIONS_PATHS=./collections ansible-doc community.general.pam_limits
ANSIBLE_COLLECTIONS_PATHS=./collections ansible-doc ansible.posix.sysctl
```

## Architecture Details

### OS Config Deployment Workflow

The os_config project provides foundational system setup:

1. **collect_facts.yml** - Quick facts collection playbook:
   - Gathers comprehensive facts from all inventory hosts
   - Displays facts on screen (stdout)
   - No files written, useful for quick inspection

2. **collect_facts_to_files.yml** - Facts collection and export playbook:
   - Gathers comprehensive facts from all inventory hosts
   - Writes facts to separate files on the control node in /tmp/
   - Creates both JSON and YAML format files: `${hostname}_facts.json` and `${hostname}_facts.yml`
   - Generates a summary report with all hosts processed: `/tmp/facts_collection_summary.txt`
   - Uses `delegate_to: localhost` to write files on the control node, not the managed hosts
   - Safe to run repeatedly - overwrites previous fact files with current data

3. **os_general_setup.yml** - Base OS configuration:
   - Creates devops user with sudo privileges
   - Configures passwordless sudo
   - Disables SELinux and firewall (for development environments)
   - Must run as root initially (use -K flag)

4. **nfs_server_setup.yml** - NFS server configuration:
   - Installs nfs-utils package
   - Configures NFS exports based on group_vars/nfs_servers.yml
   - Starts and enables NFS services

### ST Deployment Workflow

The ST deployment follows a strict sequence:

1. **st_prerequisites.yml** - Two separate plays:
   - Play 1: ST hosts - Install dependencies, set OS limits/kernel parameters, mount NFS
   - Play 2: Edge hosts - Install dependencies, set OS limits/kernel parameters (no NFS)

2. **st_install.yml** - Single play for both ST and Edge:
   - Copy installer zip from `files/` directory
   - Unzip to temporary directory
   - Template group-specific properties files (st_installer.properties/st.properties or edge_installer.properties/edge.properties)
   - Run silent installation via `setup.sh -s`
   - Cleanup on failure

3. **st_config.yml** - Post-installation configuration:
   - Deploy license files from `files/` to `{{ axway_base }}/SecureTransport/conf/`
   - Change admin password via REST API (PATCH /api/v2.0/myself)
   - Configure SSL encryption setting via REST API

### DSOM Infrastructure Server Deployment Workflow

The dsom project sets up a complete infrastructure server for Kubernetes clusters:

1. **site.yml** - Main orchestration playbook with multiple plays:
   - **Pre-tasks** - Updates system packages, installs common utilities, enables firewalld
   - **ntp-server-setup.yml** - Configures NTP server using chrony
   - **dns-server-setup.yml** - Configures DNS server using bind (named)
   - **nfs-server-setup.yml** - Configures NFS server for Kubernetes persistent volumes
   - **Post-setup validation** - Validates all services are running correctly

2. Tag-based execution allows running specific services:
   - `--tags ntp` - Only NTP setup
   - `--tags dns` - Only DNS setup
   - `--tags nfs` - Only NFS setup
   - `--tags validate` - Only validation tasks

3. Configuration via **group_vars/all.yml**:
   - `k8s_subnet` - Kubernetes network CIDR
   - `dns_zone` and `dns_records` - DNS zone and A records for K8s nodes
   - `ntp_upstream_servers` - NTP server pool
   - `nfs_export_path` and `nfs_export_options` - NFS export configuration

### Group Variables Architecture

Variables are scoped by Ansible host groups:
- **st/group_vars/ST.yml** - ST-specific variables (includes NFS mount configuration)
- **st/group_vars/Edge.yml** - Edge-specific variables (no NFS configuration)
- **os_config/group_vars/nfs_servers.yml** - NFS export configuration
- **dsom/group_vars/all.yml** - Infrastructure server configuration (NTP, DNS, NFS settings)

Key variable patterns:
- `config_files` dict maps to different template files per group (st vs edge)
- `os_limits` and `kernel_parameters` are lists of dicts applied via loops
- `dns_records` in dsom is a list of dicts for DNS A records
- Sensitive variables (passwords, etc.) are stored in `secrets.yml` (not in repo)

### Inventory Structure

**st/inventory:**
- [ST] - ST server host(s)
- [Edge] - Edge server host(s)

**os_config/inventory:**
- [st] - ST servers (lowercase group name)
- [edge] - Edge servers (lowercase group name)
- [nfs_servers] - NFS server host(s)

**dsom/inventory:**
- [infrastructure_server] - Single infrastructure server providing NTP, DNS, and NFS for K8s clusters

Note: ST project uses capitalized group names (ST, Edge) while os_config uses lowercase (st, edge, nfs_servers).

### Authentication Model

**st and os_config projects** use the same authentication pattern:
- `remote_user: devops` - SSH connection user
- `become: true` with `become_user: root` - Privilege escalation
- `become_ask_pass: false` - Assumes passwordless sudo (configured by os_general_setup.yml)

The `os_general_setup.yml` playbook creates the devops user and configures passwordless sudo.

**dsom project** uses root or privileged user:
- `become: yes` - Uses sudo for privilege escalation
- Typically run with `-K` flag to prompt for sudo password

### Template System

Templates use Jinja2 and reference variables from group_vars:
- `st/templates/st_installer.properties.j2` - ST installer configuration
- `st/templates/st.properties.j2` - ST main configuration
- `st/templates/edge_installer.properties.j2` - Edge installer configuration
- `st/templates/edge.properties.j2` - Edge main configuration
- `os_config/templates/exports.j2` - NFS exports configuration
- `dsom/templates/` - DNS zone files, NTP config, NFS exports for K8s infrastructure

Templates are selected dynamically based on the `config_files` variable in group_vars (for st project).

### Collections Dependencies

**st and os_config projects** require:
- **ansible.posix** - For sysctl, selinux, mount modules
- **community.general** - For pam_limits module
- Collections are installed from local tarballs in `st/collections/` directory

**dsom project** uses built-in modules only:
- No external collections required
- Uses dnf, systemd, template, file, command modules

## Important Notes

**General:**
- Each sub-project has its own `secrets.yml` file (not tracked in git) for sensitive variables
- All projects support ansible-navigator with containerized execution (quay.io/ansible/ansible-runner)

**ST project:**
- License files must be placed in `st/files/` before running st_config.yml
- SecureTransport installer zip must be placed in `st/files/` before running st_install.yml
- NFS must be configured and running before executing st_prerequisites.yml on ST hosts
- Run playbooks in strict order: st_prerequisites.yml → st_install.yml → st_config.yml

**OS Config project:**
- Run os_general_setup.yml first to create devops user before running other playbooks
- collect_facts_to_files.yml writes to /tmp/ on control node

**DSOM project:**
- Designed for single infrastructure server (NTP + DNS + NFS for Kubernetes)
- Modify `group_vars/all.yml` to match your Kubernetes network topology
- DNS records should include all K8s master and worker nodes
- Post-setup validation displays next steps for K8s integration
