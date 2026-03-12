# Ansible Homelab

Ansible configuration for managing a Proxmox-based homelab infrastructure. This repository automates the deployment and maintenance of VMs, LXC containers, and services across multiple hosts.

## Features

### Dynamic Proxmox Inventory
Automatically discovers all VMs and LXC containers from your Proxmox cluster using the `community.proxmox` inventory plugin. No need to manually maintain host lists - just tag your VMs in Proxmox and Ansible picks them up.

### Tag-Based Role Assignment
Proxmox tags translate directly to Ansible groups. Tag a VM with `docker` in Proxmox, and it automatically joins the `role_docker` group in Ansible. This makes role assignment as simple as editing VM tags in the Proxmox UI.

### Pre-Update Snapshots
Before updating any Proxmox VM or container, the update playbook automatically creates a snapshot with configurable retention. If an update breaks something, you can roll back instantly.

### Multi-OS Support
Roles support both Debian and Alpine Linux, with automatic detection and OS-specific handling (package managers, init systems, privilege escalation).

### CI/CD Ready
Designed for use with Semaphore or other CI/CD tools. Local secrets live in a gitignored `proxmox.cfg` file; Semaphore uses environment variables as a fallback.

## Quick Start

### Prerequisites
- Ansible 2.17+
- A Proxmox VE cluster with API access
- SSH key authentication configured (default key: `~/.ssh/id_ansible`, default user: `ansible`)

#### Proxmox API Token
Create an API token in Proxmox for the `ansible@pam` user with the following permissions:

| Path | Role | Purpose |
|------|------|---------|
| `/` | `PVEAuditor` | Discover VMs/LXCs for inventory |
| `/vms` | `PVEVMAdmin` | Create pre-update snapshots |

Disable **Privilege Separation** on the token so it inherits the user's permissions.

#### Host Requirements
Each managed host must have an `ansible` user with:
- SSH public key (`~/.ssh/id_ansible.pub`) in `~/.ssh/authorized_keys`
- Passwordless sudo (`/etc/sudoers.d/10-ansible`)
- Membership in the `docker` group (for Docker hosts)

Run `bootstrap-cloudinit.yaml` to provision these on fresh VMs. For Alpine Linux hosts, ensure the `ansible` user account is not locked (`passwd -S ansible` should show `P` or `NP`, not `L`). Set a disabled-but-unlocked password with `usermod -p '*' ansible` if needed.

### Installation

```bash
# Clone the repository
git clone https://github.com/youruser/ansible-homelab.git
cd ansible-homelab

# Install required collections
ansible-galaxy install -r requirements.yml

# Configure your Proxmox connection settings
cp proxmox.cfg.example proxmox.cfg
# edit proxmox.cfg with your host and credentials

# Test connectivity
ansible all -m ping
```

### Running Playbooks

```bash
# Deploy all roles to tagged hosts
ansible-playbook homelab.yaml

# Update all systems and containers (with automatic pre-update snapshots)
ansible-playbook update.yaml

# Clean up Docker resources
ansible-playbook docker-clean.yaml

# Bootstrap a fresh cloud-init VM
ansible-playbook bootstrap-cloudinit.yaml
```

## Playbooks

| Playbook | Description |
|----------|-------------|
| `homelab.yaml` | Main deployment playbook - applies roles based on Proxmox tags |
| `update.yaml` | Updates system packages and Docker containers with pre-update snapshots |
| `bootstrap-cloudinit.yaml` | Provisions fresh cloud-init VMs with users, SSH keys, and base packages |
| `docker-clean.yaml` | Cleans up unused Docker resources |

## Roles

| Role | Description |
|------|-------------|
| `docker_host` | Installs Docker, creates users/groups, sets up directories |
| `adguard` | Deploys AdGuard Home DNS filtering via Docker Compose |
| `proxmox_backup_client` | Installs PBS client with systemd timer for scheduled backups |
| `labelprinter` | Label printer service management |

## Inventory Structure

### Dynamic Inventory (Proxmox)
The `inventory/proxmox.yaml` file configures automatic discovery from your Proxmox cluster:
- VMs/containers are grouped by status (`status_running`, `status_stopped`)
- VMs/containers are grouped by type (`type_qemu`, `type_lxc`)
- VMs/containers are grouped by node (`node_proxmox1`)
- **Tags become role groups** (`role_docker`, `role_adguard`, etc.)

### Static Inventory
The `inventory/inventory.yaml` file defines:
- Physical machines not managed by Proxmox
- Host-specific variable overrides
- Special host groups (e.g., `proxmox_hosts` for Proxmox hypervisors)

## Configuration

### Proxmox Connection Settings

All connection settings live in a local `proxmox.cfg` file (gitignored):
```bash
cp proxmox.cfg.example proxmox.cfg
# edit proxmox.cfg and fill in your values
```

For CI/CD (Semaphore), set `PVE_TOKEN_SECRET` as an environment variable instead of using the config file — it is used as a fallback when `token_secret` is absent from `proxmox.cfg`.

### Customizing for Your Environment
1. Copy `proxmox.cfg.example` to `proxmox.cfg` and set your Proxmox hostname
2. Update `ansible.cfg` with your SSH key path and default user
3. Tag your VMs in Proxmox with the roles you want applied

## Tag Mapping Examples

| Proxmox Tag | Ansible Group | Role Applied |
|-------------|---------------|--------------|
| `docker` | `role_docker` | `docker_host` |
| `adguard` | `role_adguard` | `adguard` |
| `pbs_client` | `role_pbs_client` | `proxmox_backup_client` |
| `homeassistant` | `role_homeassistant` | (excluded from updates) |

## Update Behavior

The `update.yaml` playbook has special handling for different host types:

- **Proxmox hypervisors** (`proxmox_nodes` group): Uses `apt dist-upgrade` for kernel updates
- **Regular Debian hosts**: Uses `apt full-upgrade`
- **Alpine hosts**: Uses `apk upgrade`
- **Docker hosts**: Pulls latest images and recreates containers
- **VMs/containers**: Get automatic pre-update snapshots with 3-snapshot retention

## Requirements

See `requirements.yml` for required Ansible collections:
- `community.general`
- `community.docker`
- `community.proxmox`
- `ansible.posix`

## License

MIT
