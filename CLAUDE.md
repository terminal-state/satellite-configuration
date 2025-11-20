# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is an Ansible-based automation framework for deploying and configuring Red Hat Satellite servers on KVM. The complete workflow includes VM creation, Satellite installation, content management configuration, and system provisioning setup.

## Architecture

### Four-Stage Deployment Pipeline

1. **KVM Host Preparation** (`01-kvmhost-prepare.yml`)
   - Prepares the KVM hypervisor environment
   - Installs required packages and dependencies

2. **VM Creation** (`02-vm-creation.yml`)
   - Creates RHEL VM using qcow2 backing files
   - Configures static networking via cloud-init
   - Uses `roles/vm_build` for VM provisioning

3. **Satellite Installation** (`03-satellite-install.yml`)
   - Registers system with Red Hat
   - Installs Satellite packages
   - Runs satellite-installer
   - Uploads and attaches manifest

4. **Satellite Configuration** (`04-satellite-configure.yml`)
   - Master orchestration playbook
   - Configures content management
   - Sets up lifecycle environments
   - Creates activation keys and provisioning templates

### Critical Configuration Order and Dependencies

The `04-satellite-configure.yml` master playbook executes configuration playbooks in a **specific order that must not be changed** due to dependencies:

```yaml
1. content-credentials.yml      # GPG keys must exist first
2. products.yml                 # Products before repositories
3. repositories.yml             # Red Hat repositories
4. external-repositories.yml    # External repositories (AlmaLinux, EPEL)
5. sync-plans.yml              # Sync schedules (optional)
6. lifecycle-environments.yml   # Environments before promotion
7. content-views.yml           # Content views reference repositories
8. publish-promote-content-views.yml  # Must happen after CV creation
9. activation-keys.yml         # Requires published content views
10. partition-tables.yml        # Templates before OS association
11. operatingsystems.yml        # Associates everything together
```

**Why Order Matters:**

- **Products → Repositories**: Repositories belong to products and cannot exist without them
- **Repositories → Content Views**: Content views reference specific repositories by name
- **Content Views → Publishing**: Content views must exist before they can be published
- **Lifecycle Environments → Promotion**: Promotion targets must exist before promoting content
- **Published CVs → Activation Keys**: Activation keys reference specific content view versions in lifecycle environments
- **Partition Tables → Operating Systems**: OS definitions reference partition table templates
- **Everything → Operating Systems**: OS configuration ties together architectures, partition tables, provisioning templates, and installation media

### Data Separation Architecture

Configuration data is separated from playbook logic:

```
Playbooks (root directory)         Configuration Data (satellite_config/)
├── repositories.yml           →   ├── repositories.yml
├── content-views.yml          →   ├── content_views.yml
├── activation-keys.yml        →   ├── activation_keys.yml
└── lifecycle-environments.yml →   └── lifecycle_environments.yml
```

Playbooks contain task logic using `redhat.satellite` modules. Configuration files contain YAML data structures that define what to create. This separation allows:
- Changing Satellite configuration without modifying tasks
- Version controlling infrastructure as data
- Reusing playbooks across different Satellite deployments

### VM Creation Architecture

The `roles/vm_build/tasks/main.yml` implements cloud-init based provisioning:

1. **Backing File Creation**: Uses `qemu-img create -b` to create copy-on-write disks from base RHEL qcow2 image
2. **Cloud-Init ISO Generation**: Creates seed ISO with:
   - Static network configuration (NetworkManager renderer)
   - SSH public key injection
   - Root password configuration
3. **Libvirt XML Definition**: Dynamically generates domain XML with:
   - Virtio disk and network drivers
   - SATA CDROM for cloud-init seed
   - CPU host-passthrough for performance
4. **VM Lifecycle**: Uses `community.libvirt.virt` module for define/start operations

### Content Management Strategy

The configuration implements a sophisticated multi-tier content view strategy:

**Base Content Views** (`CV-RHEL*_Base`):
- Contain OS repositories (BaseOS, AppStream, Kickstart, Satellite Client)
- Apply "Exclude All Errata" filter to freeze at GA release
- Serve as immutable foundation

**Update Content Views** (`CV-RHEL*_Security_Only`, `CV-RHEL*_Updates`):
- Security-only views: Filter to include only security errata
- Updates views: Include enhancement and bugfix errata
- Allow selective patching strategies

**Composite Content Views** (`CCV-*`):
- Combine base + update views
- Use `latest: true` for automatic component version selection
- Examples:
  - `CCV-RHEL9.6_Security_Only`: Base + Security patches
  - `CCV-RHEL9.6_All_Updates`: Base + Security + Enhancements + Bugfixes

**Lifecycle Promotion Path**:
```
Library → Development → QA → Pre-Production → Disaster Recovery → Production
```

**Automated vs Manual Promotion**:
- Automated: Library → Development (via `publish-promote-content-views.yml`)
- Manual: All subsequent promotions (Development onwards) must be done manually via Satellite UI or hammer CLI

## Common Commands

### Prerequisites

Install required Ansible collections:
```bash
ansible-galaxy collection install -r requirements.yml
```

Required collections:
- `redhat.satellite` - Satellite configuration modules
- `community.libvirt` - KVM/libvirt management

### Complete Deployment Workflow

**Step 1: Configure Variables**

Edit `group_vars/all.yml`:
```yaml
# VM settings
vm_name: satellite-rhel9
vm_ip_address: 192.168.100.50
domain: lab.local

# Red Hat subscription
rhsm_org: "Your_Organization"
rhsm_activationkey: "Your_Activation_Key"

# Satellite admin credentials
foreman_admin_password: "YourSecurePassword"
satellite_manifest_org: "{{ rhsm_org }}"
```

Create `inventory.ini`:
```ini
[kvmhost]
localhost ansible_connection=local

[satellite]
satellite-rhel9 ansible_host=192.168.100.50 ansible_user=root
```

**Step 2: Deploy Infrastructure**

```bash
# Prepare KVM host
ansible-playbook -i inventory.ini 01-kvmhost-prepare.yml

# Create VM (takes ~5 minutes)
ansible-playbook -i inventory.ini 02-vm-creation.yml

# Wait for VM to boot and cloud-init to complete (~2 minutes)
# Verify SSH access: ssh root@192.168.100.50

# Install Satellite (takes ~30-45 minutes)
ansible-playbook -i inventory.ini 03-satellite-install.yml

# Configure Satellite content management (takes ~15-30 minutes depending on sync)
ansible-playbook -i inventory.ini 04-satellite-configure.yml
```

**Step 3: Verify Deployment**

Access Satellite Web UI:
```
https://192.168.100.50
Username: admin
Password: <value of foreman_admin_password>
```

Check content views:
```bash
ssh root@192.168.100.50
hammer content-view list --organization "Your_Organization"
hammer activation-key list --organization "Your_Organization"
```

### Selective Configuration Using Tags

Run specific configuration stages:

```bash
# Only configure repositories (useful after adding new repos)
ansible-playbook -i inventory.ini 04-satellite-configure.yml --tags repositories

# Reconfigure lifecycle environments and content views
ansible-playbook -i inventory.ini 04-satellite-configure.yml --tags lifecycle_environments,content_views

# Publish and promote content views
ansible-playbook -i inventory.ini 04-satellite-configure.yml --tags publish,promote

# Skip sync plans (useful for testing)
ansible-playbook -i inventory.ini 04-satellite-configure.yml --skip-tags syncplans
```

**Available Tags**:
- `credentials` - Content credentials (GPG keys)
- `repositories` - Repository configuration
- `syncplans` - Sync plan setup
- `lifecycle_environments` - Lifecycle environment creation
- `content_views` - Content view creation and filters
- `publish` - Publish content views
- `promote` - Promote content views to Development
- `activation_keys` - Activation key creation
- `partition_tables` - Partition table templates
- `operatingsystems` - Operating system configuration

### Individual Playbook Execution

**WARNING**: When running playbooks individually, you must respect the dependency order. Running out of order will cause failures.

Safe examples:
```bash
# Re-enable repositories (safe to re-run)
ansible-playbook -i inventory.ini repositories.yml

# Recreate content views (safe if repositories exist)
ansible-playbook -i inventory.ini content-views.yml

# Publish latest changes
ansible-playbook -i inventory.ini publish-promote-content-views.yml
```

Unsafe example (will fail):
```bash
# This will FAIL because content views don't exist yet
ansible-playbook -i inventory.ini activation-keys.yml
```

### Troubleshooting Commands

Check VM status:
```bash
virsh list --all
virsh console satellite-rhel9  # Ctrl+] to exit
```

Monitor Satellite installation:
```bash
ssh root@192.168.100.50
tail -f /var/log/foreman-installer/satellite.log
```

Check repository sync status:
```bash
hammer task list --search 'label = Actions::Katello::Repository::Sync'
hammer repository list --organization "Your_Organization"
```

Manually promote content view:
```bash
# Get content view version
hammer content-view version list --content-view "CCV-RHEL9.6_All_Updates" --organization "Your_Organization"

# Promote to QA
hammer content-view version promote \
  --content-view "CCV-RHEL9.6_All_Updates" \
  --version 1.0 \
  --to-lifecycle-environment QA \
  --organization "Your_Organization"
```

## Configuration Guidelines

### Required Variables in group_vars/all.yml

**VM Configuration:**
```yaml
vm_name: satellite-rhel9               # VM name in libvirt
vm_memory_mb: 20480                    # 20GB RAM minimum
vm_vcpus: 4                            # 4 vCPUs minimum
vm_disk_gb: 500                        # 500GB disk minimum
vm_ip_address: 192.168.100.50          # Static IP address
vm_gateway: 192.168.100.1              # Network gateway
vm_dns_servers: [192.168.100.1, 1.1.1.1]
domain: lab.local                      # DNS domain
qcow_image_path: /var/lib/libvirt/images/rhel-9.7-x86_64-kvm.qcow2
ssh_public_key_path: /home/ansible/.ssh/id_ed25519.pub
```

**Red Hat Subscription:**
```yaml
rhsm_org: "Default_ORG"                # Red Hat organization ID
rhsm_activationkey: "YOUR_KEY"         # Activation key with Satellite entitlements
satellite_version: "6.18"              # Satellite version to install
```

**Satellite Configuration:**
```yaml
foreman_admin: "admin"                 # Admin username
foreman_admin_password: "Admin123"     # Admin password
satellite_server_url: "https://192.168.100.50"
satellite_username: "admin"
satellite_password: "Admin123"
satellite_validate_certs: false
satellite_manifest_org: "Default_ORG"  # Organization name in Satellite
satellite_manifest_path: "/root/manifest.zip"  # Path to manifest file
```

**Derived Variables:**
```yaml
daily_sync: "Daily SYNC"               # Referenced in repository configs
weekly_sync: "Weekly SYNC"             # Referenced in repository configs
```

### Adding Repositories

**Red Hat Repositories** (`satellite_config/repositories.yml`):

```yaml
redhat_repositories:
  - name: "RHEL 9 BaseOS x86_64"
    product: "Red Hat Enterprise Linux for x86_64"
    label: "rhel-9-for-x86_64-baseos-rpms"
    releasever: "9.6"                  # Specific version
    sync_plan: "{{ daily_sync }}"
    content_type: "yum"
    enabled: true

  - name: "RHEL 9 BaseOS x86_64 Kickstart"
    product: "Red Hat Enterprise Linux for x86_64"
    label: "rhel-9-for-x86_64-baseos-kickstart"
    releasever: "9.6"                  # Required for kickstart repos
    sync_plan: "{{ daily_sync }}"
    content_type: "yum"
    enabled: true
```

**Note**: Kickstart repositories are required for PXE/network provisioning. They must be added to base content views.

**External Repositories** (`satellite_config/external_repositories.yml`):

```yaml
external_repositories:
  - name: "EPEL 9"
    product: "Extra Packages for Enterprise Linux"
    label: "epel-9-x86_64"
    content_type: "yum"
    arch: "x86_64"
    download_policy: "on_demand"
    mirroring_policy: "mirror_complete"
    url: "https://dl.fedoraproject.org/pub/epel/9/Everything/x86_64/"
    gpg_key: "https://dl.fedoraproject.org/pub/epel/RPM-GPG-KEY-EPEL-9"
```

### Content View Configuration

**Simple Content View with Filter** (`satellite_config/content_views.yml`):

```yaml
content_views:
  - name: "CV-RHEL9.6_Base"
    description: "Red Hat Enterprise Linux 9.6 Base Content View"
    repositories:
      - name: "Red Hat Enterprise Linux 9 for x86_64 - BaseOS RPMs 9.6"
        product: "Red Hat Enterprise Linux for x86_64"
      - name: "Red Hat Enterprise Linux 9 for x86_64 - AppStream RPMs 9.6"
        product: "Red Hat Enterprise Linux for x86_64"
      - name: "Red Hat Enterprise Linux 9 for x86_64 - BaseOS Kickstart 9.6"
        product: "Red Hat Enterprise Linux for x86_64"
      - name: "Red Hat Enterprise Linux 9 for x86_64 - AppStream Kickstart 9.6"
        product: "Red Hat Enterprise Linux for x86_64"
    auto_publish: false
    filters:
      - name: "Exclude All Errata"
        filter_type: "erratum"
        inclusion: false              # Exclude all errata = GA freeze
```

**Content View with Filter Rules**:

```yaml
  - name: "CV-RHEL9.6_Security_Only"
    description: "RHEL 9.6 with only security updates"
    repositories:
      - name: "Red Hat Enterprise Linux 9 for x86_64 - BaseOS RPMs 9.6"
        product: "Red Hat Enterprise Linux for x86_64"
    filter_rules:
      - name: "Security Updates Only"
        type: "rpm"
        inclusion: true
        original_packages: true
        description: "Include only security updates"
        rules:
          - name: "Security Errata"
            type: "errata_type"
            types: ["security"]         # Only security errata
```

**Composite Content View**:

```yaml
  - name: "CCV-RHEL9.6_All_Updates"
    description: "Composite view with base + all updates"
    composite: true
    components:
      - content_view: "CV-RHEL9.6_Base"
        latest: true                   # Always use latest published version
      - content_view: "CV-RHEL9.6_Updates"
        latest: true
      - content_view: "CV-RHEL9.6_Security_Only"
        latest: true
    auto_publish: false
```

### Activation Key Configuration

Activation keys reference published content views in specific lifecycle environments:

```yaml
activation_keys:
  - name: "rhel9.6-dev"
    lifecycle_environment: "Development"
    content_view: "CCV-RHEL9.6_All_Updates"
    release_version: "9.6"
    subscriptions:
      - name: "Red Hat Enterprise Linux Server, Standard (Physical or Virtual Nodes)"
    content_overrides:
      - label: "satellite-client-6-for-rhel-9-x86_64-rpms"
        override: enabled
```

### Operating System Configuration

Operating systems tie together multiple components:

```yaml
operatingsystems:
  - name: "RedHat"
    major: "9"
    minor: "6"
    description: "Red Hat Enterprise Linux 9.6"
    family: "Redhat"                    # Used for initial creation
    release_name: "Plow"
    architectures:
      - x86_64
    ptables:
      - RHEL9_Standard                  # Partition table template name
    media:
      - RHEL 9.6 Kickstart Media        # Installation medium name
    provisioning_templates:
      - name: "Kickstart default"
        template_kind: provision
      - name: "Kickstart default finish"
        template_kind: finish
```

**CRITICAL**: The `operatingsystems.yml` playbook uses `family` for initial OS creation but requires `os_family` when updating associations (architectures, partition tables). This is handled automatically by the playbook.

### Lifecycle Environments

Define promotion path:

```yaml
lifecycle_environments:
  - name: "Development"
    description: "Development environment for testing"
    prior: "Library"                    # Promotes from Library
  - name: "QA"
    description: "Quality assurance environment"
    prior: "Development"                # Promotes from Development
  - name: "Production"
    description: "Live production environment"
    prior: "QA"                         # Promotes from QA
```

**Promotion Flow**: Library → Development (automated) → QA (manual) → Production (manual)

## Architecture Patterns

### VM Provisioning with Cloud-Init

The VM creation uses a cloud-init NoCloud datasource with static networking:

1. **user-data**: Configures users, SSH keys, passwords, and network settings
2. **meta-data**: Sets instance-id and hostname
3. **Seed ISO**: Created with `genisoimage` and attached as CDROM

This approach ensures:
- Predictable static IP configuration before first boot
- Automatic SSH key injection
- No DHCP dependency
- NetworkManager compatibility (RHEL 9 default)

### Repository Synchronization

Repositories can be configured with sync plans:
- **Daily Sync**: Runs nightly, suitable for security repositories
- **Weekly Sync**: Runs weekly, suitable for large repositories

Sync policies are defined in `satellite_config/sync_plans.yml` and referenced by repositories via `sync_plan: "{{ daily_sync }}"`.

### Content View Publishing Workflow

1. **Create** content views (defines which repositories to include)
2. **Publish** to Library environment (creates immutable version)
3. **Promote** to Development (automated in this configuration)
4. **Promote** to QA/Production (manual process)

Component content views must be published before composite content views that reference them.

## Important Implementation Notes

- **Idempotency**: Most playbooks are idempotent and safe to re-run. The `04-satellite-configure.yml` uses `ignore_errors: true` for most plays to allow partial configuration recovery.
- **Backing Files**: VM disks use qcow2 backing files (`qemu-img create -b`) to save disk space and enable rapid VM cloning.
- **Certificate Validation**: `satellite_validate_certs: false` is used by default for lab environments. Set to `true` for production.
- **Module Parameter Differences**: The `redhat.satellite.operatingsystem` module uses `family` for creation but `os_family` for updates - playbooks handle this automatically.
- **Repository Names**: Repository names in content views must exactly match the names in Satellite. Use `hammer repository list` to verify exact names.
- **Manifest Upload**: The `03-satellite-install.yml` playbook expects a manifest at `/root/manifest.zip`. Upload your manifest to the Satellite VM before running installation.

## System Registration

After Satellite configuration, register clients:

**Using Activation Key**:
```bash
subscription-manager register \
  --org="Your_Organization" \
  --activationkey="rhel9.6-dev"
```

**Using Bootstrap Script**:
```bash
curl -k https://satellite-rhel9.lab.local/pub/bootstrap.py | \
  python - \
  --organization="Your_Organization" \
  --activation-key="rhel9.6-dev"
```

## Common Modification Scenarios

### Adding a New Operating System Version

1. Add repositories to `satellite_config/repositories.yml`
2. Create content view in `satellite_config/content_views.yml`
3. Add to promotion list in `publish-promote-content-views.yml`
4. Create activation key in `satellite_config/activation_keys.yml`
5. Add OS definition in `satellite_config/operatingsystems.yml`
6. Run: `ansible-playbook -i inventory.ini 04-satellite-configure.yml`

### Changing VM Specifications

Edit `group_vars/all.yml`:
```yaml
vm_memory_mb: 32768  # 32GB
vm_vcpus: 8          # 8 CPUs
vm_disk_gb: 1000     # 1TB
```

Then recreate VM:
```bash
virsh destroy satellite-rhel9
virsh undefine satellite-rhel9
rm /var/lib/libvirt/images/satellite-rhel9.qcow2
ansible-playbook -i inventory.ini 02-vm-creation.yml
```

### Re-publishing Content Views

To publish latest content:
```bash
# Publish all configured CVs to Library and promote to Development
ansible-playbook -i inventory.ini publish-promote-content-views.yml

# Or use hammer directly for specific CV
hammer content-view publish --name "CV-RHEL9.6_Base" --organization "Your_Organization"
```
