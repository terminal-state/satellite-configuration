# Red Hat Satellite Automated Deployment and Configuration

This repository contains Ansible playbooks and configuration files for automating the complete deployment of Red Hat Satellite, from VM creation to fully configured content management.

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Repository Structure](#repository-structure)
- [Quick Start](#quick-start)
- [Detailed Usage Guide](#detailed-usage-guide)
- [Configuration Files](#configuration-files)
- [Customization](#customization)
- [Troubleshooting](#troubleshooting)

## Overview

This automation framework provides an end-to-end solution for:
- Creating a RHEL VM on KVM for Satellite installation
- Installing Red Hat Satellite
- Configuring repositories, lifecycle environments, content views
- Setting up activation keys and partition tables
- Managing external repositories (AlmaLinux, EPEL, etc.)

## Architecture

The automation follows this workflow:
1. **VM Creation**: Deploy RHEL VM on KVM with cloud-init
2. **Satellite Installation**: Install and configure Red Hat Satellite
3. **Content Configuration**: Set up repositories, sync plans, and content views
4. **Lifecycle Management**: Create environments and promotion paths
5. **System Registration**: Configure activation keys for client systems

## Prerequisites

### Host Requirements
- KVM hypervisor with libvirt
- Ansible 2.9+ with required collections
- RHEL 9.x KVM guest image (qcow2)
- Valid Red Hat subscription with Satellite entitlements
- SSH key pair for authentication

### Required Ansible Collections
```bash
ansible-galaxy collection install -r requirements.yml
```

### Network Requirements
- Static IP allocation for Satellite VM
- DNS resolution for Satellite hostname
- Internet access for package downloads

## Repository Structure
```
satellite-configuration/
├── 01-kvmhost-prepare.yml       # Prepare KVM host
├── 02-vm-creation.yml           # Create Satellite VM
├── 03-satellite-install.yml     # Install Satellite
├── 04-satellite-configure.yml   # Master configuration playbook
├── activation-keys.yml          # Create activation keys
├── content-credentials.yml      # GPG key management
├── content-views.yml           # Content view configuration
├── external-repositories.yml   # External repo setup
├── lifecycle-environments.yml  # Lifecycle environments
├── partition-tables.yml        # Partition table templates
├── products.yml               # Product definitions
├── publish-promote-content-views.yml  # CV publishing
├── repositories.yml           # Repository enablement
├── sync-plans.yml            # Sync plan configuration
├── group_vars/
│   └── all.yml               # Global variables
├── satellite_config/         # Configuration definitions
│   ├── activation_keys.yml
│   ├── content_views.yml
│   ├── external_repositories.yml
│   ├── lifecycle_environments.yml
│   ├── partition_tables.yml
│   ├── products.yml
│   ├── repositories.yml
│   └── sync_plans.yml
├── roles/
│   └── vm_build/            # VM creation role
└── files/                   # GPG keys and other files
```

## Quick Start

### 1. Clone and Configure
```bash
git clone https://github.com/terminal-state/satellite-configuration.git
cd satellite-configuration
```

### 2. Update Variables

Edit `group_vars/all.yml`:
```yaml
# VM Configuration
vm_name: satellite-rhel9
vm_ip_address: 192.168.100.50
domain: lab.local

# Red Hat Registration
rhsm_org: "Your_Organization"
rhsm_activationkey: "Your_Activation_Key"

# Satellite Admin
foreman_admin: "admin"
foreman_admin_password: "YourSecurePassword"

# Image Path
qcow_image_path: "/var/lib/libvirt/images/rhel-9.7-x86_64-kvm.qcow2"
```

### 3. Create Inventory

Create `inventory.ini`:
```ini
[kvmhost]
localhost ansible_connection=local

[satellite]
satellite-rhel9 ansible_host=192.168.100.50 ansible_user=root
```

### 4. Run Complete Setup
```bash
# Step 1: Prepare KVM host
ansible-playbook -i inventory.ini 01-kvmhost-prepare.yml

# Step 2: Create VM
ansible-playbook -i inventory.ini 02-vm-creation.yml

# Wait for VM to be accessible, then:

# Step 3: Install Satellite
ansible-playbook -i inventory.ini 03-satellite-install.yml

# Step 4: Configure Satellite (runs all configuration)
ansible-playbook -i inventory.ini 04-satellite-configure.yml
```

## Detailed Usage Guide

### Individual Playbook Execution

The master playbook (`04-satellite-configure.yml`) runs all configuration tasks. You can also run individual components:
```bash
# Enable Red Hat repositories only
ansible-playbook -i inventory.ini repositories.yml

# Configure lifecycle environments
ansible-playbook -i inventory.ini lifecycle-environments.yml

# Create content views
ansible-playbook -i inventory.ini content-views.yml

# Publish and promote content views
ansible-playbook -i inventory.ini publish-promote-content-views.yml

# Create activation keys
ansible-playbook -i inventory.ini activation-keys.yml
```

### Using Tags

The master configuration playbook supports tags for selective execution:
```bash
# Only configure repositories
ansible-playbook -i inventory.ini 04-satellite-configure.yml --tags repositories

# Only lifecycle environments and content views
ansible-playbook -i inventory.ini 04-satellite-configure.yml --tags lifecycle_environments,content_views

# Skip sync plans
ansible-playbook -i inventory.ini 04-satellite-configure.yml --skip-tags syncplans
```

Available tags:
- `credentials` - Content credentials (GPG keys)
- `repositories` - Repository configuration
- `syncplans` - Sync plan setup
- `lifecycle_environments` - Lifecycle environments
- `content_views` - Content view creation
- `promote` / `publish` - Content view publishing
- `activation_keys` - Activation key creation
- `partition_tables` - Partition table templates

## Configuration Files

### Repository Configuration (`satellite_config/repositories.yml`)

Defines Red Hat repositories to enable:
```yaml
redhat_repositories:
  - name: "RHEL 8 BaseOS x86_64"
    product: "Red Hat Enterprise Linux for x86_64"
    label: "rhel-8-for-x86_64-baseos-rpms"
    releasever: "8.10"
    sync_plan: "{{ daily_sync }}"
```

### Content Views (`satellite_config/content_views.yml`)

Defines content views and composite content views:
```yaml
content_views:
  - name: "CV-RHEL8.10_Base"
    description: "RHEL 8.10 Base Content View"
    repositories:
      - name: "Red Hat Enterprise Linux 8 for x86_64 - BaseOS RPMs 8.10"
    filters:
      - name: "Exclude All Errata"
        filter_type: "erratum"
        inclusion: false
```

### Lifecycle Environments (`satellite_config/lifecycle_environments.yml`)

Defines promotion path:
```yaml
lifecycle_environments:
  - name: "Development"
    description: "Development environment"
    prior: "Library"
  - name: "QA"
    description: "QA environment"
    prior: "Development"
```

### Activation Keys (`satellite_config/activation_keys.yml`)

Defines activation keys for system registration:
```yaml
activation_keys:
  - name: "rhel8.10"
    lifecycle_environment: "Production"
    content_view: "CCV-RHEL8.10_All_Updates"
    release_version: "8.10"
```

## Customisation

### Adding External Repositories

Edit `satellite_config/external_repositories.yml`:
```yaml
external_repositories:
  - name: "EPEL 9"
    product: "Extra Packages for Enterprise Linux (EPEL)"
    url: "https://dl.fedoraproject.org/pub/epel/9/Everything/x86_64/"
    gpg_key: "EPEL-9"
```

### Modifying VM Specifications

Edit `group_vars/all.yml`:
```yaml
vm_memory_mb: 32768  # 32GB for larger deployments
vm_vcpus: 8          # More CPUs for better performance
vm_disk_gb: 1000     # 1TB for more content
```

### Custom Partition Tables

Add to `satellite_config/partition_tables.yml`:
```yaml
- name: "Custom_Layout"
  os_family: "Redhat"
  layout: |
    zerombr
    clearpart --all --initlabel
    autopart --type=lvm --fstype=xfs
```

## Troubleshooting

### VM Creation Issues

Check KVM logs:
```bash
virsh list --all
virsh dominfo satellite-rhel9
journalctl -u libvirtd -f
```

### Satellite Installation

Monitor installation progress:
```bash
tail -f /var/log/foreman-installer/satellite.log
```

### Repository Sync Issues

Check Satellite logs:
```bash
tail -f /var/log/foreman/production.log
hammer task list --search 'sync'
```

### Content View Errors

Verify repository names:
```bash
hammer repository list --organization "Your_Organization"
```

### Common Issues and Solutions

1. **"Repository not found" errors**
   - Verify exact repository names in Satellite
   - Check that repositories are enabled in your manifest
   - Ensure sync plans have completed

2. **Activation key creation failures**
   - Content views must be published and promoted first
   - Verify lifecycle environment names match

3. **Network connectivity issues**
   - Check firewall rules
   - Verify DNS resolution
   - Test proxy settings if configured

## System Registration

After setup, register clients using:
```bash
# Using activation key
subscription-manager register \
  --org="Your_Organization" \
  --activationkey="rhel8.10"

# Or with bootstrap script
curl -k https://satellite-rhel9.lab.local/pub/bootstrap.py | \
  sudo python - \
  --organization="Your_Organization" \
  --activation-key="rhel8.10"
```

## Contributing

1. Fork the repository
2. Create a feature branch
3. Test thoroughly in a lab environment
4. Submit a pull request

## License

[Specify your license]

## Support

For issues:
- Check Red Hat Satellite documentation
- Review Ansible module documentation
- Open an issue in this repository
