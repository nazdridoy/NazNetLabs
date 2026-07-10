# RHEL 7.9 KVM Guest Image Lab: Manual Build and Image Builder

This lab covers two methods for producing a RHEL 7.9 KVM guest image that is functionally identical to Red Hat's official KVM Guest Image, with `cloud-user`, `cloud-init`, serial console, and `qemu-guest-agent`. The image is then imported into GNS3 as a working appliance with telnet console access.

---

## Overview

| Stage | What You Do |
|---|---|
| 1 | Install RHEL 7.9 minimal in virt-manager |
| 2 | Configure the system to match the Red Hat KVM Guest Image |
| 3 | Seal and generalize the image |
| 4 | Compact the qcow2 |
| 5 | Import the qcow2 into GNS3 as a QEMU appliance |
| 6 | Verify telnet console works |

---

## Prerequisites

- Red Hat subscription or RHEL 7.9 ISO (e.g., `rhel-server-7.9-x86_64-dvd.iso`)
- `virt-manager` installed on your Linux host
- GNS3 installed on your Linux host
- Enough disk space (10–20 GB recommended for the qcow2)

---

## Stage 1: Install RHEL 7.9 Minimal in virt-manager

### 1.1 Create the VM

1. Open **virt-manager** → **File → New Virtual Machine**
2. Select **Local install media (ISO image)** → choose your RHEL 7.9 ISO
3. Set resources:
   - RAM: `1024 MB` (1 GB)
   - CPUs: `1`
4. Create a disk image:
   - Format: **qcow2**
   - Size: **10 GB**
   - Store in a known location, e.g. `/var/lib/libvirt/images/rhel79-base.qcow2`
5. Before clicking **Finish**, check **"Customize configuration before install"**

### 1.2 Customize Hardware Before Install

In the VM hardware details:

- **Boot Options**: ensure CDROM is first
- **Video**: set to `VGA` (not QXL, for better serial console compatibility)
- **Display**: `Spice` is fine for installation
- Click **Begin Installation**

### 1.3 Anaconda Installation Settings

When the RHEL installer boots:

| Setting | Value |
|---|---|
| Installation Destination | Select your 10 GB disk → **Automatic Partitioning** |
| Partitioning Scheme | **Standard Partition** (no LVM) |
| Software Selection | **Minimal Install** |
| Network & Hostname | Enable the NIC → set hostname if desired |
| Root Password | Set a temporary root password |
| User Creation | **Do NOT create a user here**: you will create `cloud-user` manually |

> **Note**: Under **Installation Destination → Partitioning**, choose
> **"I will configure partitioning"** → select **Standard Partition** scheme
> and let Anaconda auto-create `/boot`, `/`, and `swap` without LVM.

Complete the installation and reboot into the new system.

---

## Stage 2: Configure the System (Inside the VM via virt-manager Console)

Log in as **root** via the virt-manager SPICE console.

---

### 2.1 Register and Update

```bash
subscription-manager register --username=<your-rhn-user> --password=<your-rhn-pass>
subscription-manager attach --auto
yum update -y
```

If using a local mirror or offline repo, configure `/etc/yum.repos.d/` appropriately and skip `subscription-manager`.

---

### 2.2 Install Required Packages

These are the packages present in the official Red Hat KVM Guest Image, plus important tools for diagnostics and proper SELinux management:

```bash
yum install -y \
  cloud-init \
  cloud-utils-growpart \
  qemu-guest-agent \
  NetworkManager \
  net-tools \
  vim \
  curl \
  wget \
  sudo \
  openssh-server \
  bash-completion \
  bind-utils \
  telnet \
  rsyslog \
  chrony \
  tcpdump \
  lsof \
  policycoreutils-python \
  setools-console
```

**Package notes:**

| Package | Why it's needed |
|---|---|
| `cloud-init` | First-boot provisioning (SSH keys, hostname, etc.) |
| `cloud-utils-growpart` | Expands the root partition to fill available disk |
| `qemu-guest-agent` | Hypervisor integration (snapshots, IP reporting) |
| `NetworkManager` | Interface management; required by cloud-init |
| `rsyslog` | System logging daemon; cloud-init logs to syslog |
| `chrony` | NTP time sync; replaces `ntpd` on RHEL 7 |
| `bind-utils` | `dig`, `nslookup` for DNS troubleshooting in labs |
| `telnet` | Useful for testing SMTP, POP3, and other TCP services |
| `tcpdump` | Packet capture for network debugging |
| `lsof` | List open files and listening ports |
| `policycoreutils-python` | `semanage` command for managing SELinux contexts |
| `setools-console` | `sesearch`, `seinfo` for SELinux policy inspection |

---

### 2.3 Create the `cloud-user` Account

The official KVM Guest Image uses `cloud-user` as the default unprivileged user:

```bash
# Create cloud-user
useradd -m -s /bin/bash cloud-user

# Add to wheel group for sudo access
usermod -aG wheel cloud-user

# Set a password (optional, cloud-init can inject SSH keys instead)
passwd cloud-user
```

#### Allow passwordless sudo for wheel group (exactly like the official image):

```bash
visudo
```

Find and uncomment this line:

```
## %wheel ALL=(ALL) NOPASSWD: ALL
```

Change to:

```
%wheel  ALL=(ALL)  NOPASSWD: ALL
```

---

### 2.4 Configure cloud-init

The official KVM Guest Image uses `cloud-init` for first-boot provisioning (SSH key injection, hostname, etc).

> **Note**: On a minimal RHEL base install, `/etc/cloud/cloud.cfg` does **not** exist until
> `cloud-init` is installed (Step 2.2 above). After `yum install cloud-init`, a generic
> default `cloud.cfg` is created, but you must **overwrite it entirely** with the official
> Red Hat KVM Guest Image version below.

#### Write `/etc/cloud/cloud.cfg`: exact verbatim copy from the official Red Hat KVM Guest Image:

```bash
cat > /etc/cloud/cloud.cfg << 'EOF'
users:
 - default

disable_root: 1
ssh_pwauth:   0

mount_default_fields: [~, ~, 'auto', 'defaults,nofail,x-systemd.requires=cloud-init.service', '0', '2']
resize_rootfs_tmp: /dev
ssh_deletekeys:   1
ssh_genkeytypes:  ~
syslog_fix_perms: ~
disable_vmware_customization: false

cloud_init_modules:
 - disk_setup
 - migrator
 - bootcmd
 - write-files
 - growpart
 - resizefs
 - set_hostname
 - update_hostname
 - update_etc_hosts
 - rsyslog
 - users-groups
 - ssh

cloud_config_modules:
 - mounts
 - locale
 - set-passwords
 - rh_subscription
 - yum-add-repo
 - package-update-upgrade-install
 - timezone
 - puppet
 - chef
 - salt-minion
 - mcollective
 - disable-ec2-metadata
 - runcmd

cloud_final_modules:
 - rightscale_userdata
 - scripts-per-once
 - scripts-per-boot
 - scripts-per-instance
 - scripts-user
 - ssh-authkey-fingerprints
 - keys-to-console
 - phone-home
 - final-message
 - power-state-change

system_info:
  default_user:
    name: cloud-user
    lock_passwd: true
    gecos: Cloud User
    groups: [adm, systemd-journal]
    sudo: ["ALL=(ALL) NOPASSWD:ALL"]
    shell: /bin/bash
  distro: rhel
  paths:
    cloud_dir: /var/lib/cloud
    templates_dir: /etc/cloud/templates
  ssh_svcname: sshd

# vim:syntax=yaml
EOF
```

**Key fields to understand:**

| Field | Value | Meaning |
|---|---|---|
| `disable_root` | `1` | Root SSH login disabled |
| `ssh_pwauth` | `0` | Password SSH auth disabled (key-only) |
| `ssh_deletekeys` | `1` | Old SSH host keys deleted and regenerated on each boot |
| `lock_passwd` | `true` | cloud-user password locked; login via SSH key only by default |
| `groups` | `[adm, systemd-journal]` | Not wheel; sudo granted via the `sudo:` directive directly |
| `distro` | `rhel` | Activates RHEL-specific cloud-init behaviors |
| `ssh_svcname` | `sshd` | SSH service name used by cloud-init to restart SSH |

#### Set the datasource for local/GNS3 use (no AWS/OpenStack metadata server):

```bash
cat > /etc/cloud/cloud.cfg.d/99_datasource.cfg << 'EOF'
datasource_list: [NoCloud, None]
EOF
```

> **Note**: `NoCloud` datasource allows cloud-init to read configuration from a local
> ISO or seed directory. This makes it work in GNS3/KVM without a cloud metadata server.

#### Enable cloud-init services:

```bash
systemctl enable cloud-init-local
systemctl enable cloud-init
systemctl enable cloud-config
systemctl enable cloud-final
```

---

### 2.5 Configure GRUB (Serial Console + eth0 Naming)

This section covers two things at once: **serial console** (required for GNS3 telnet) and
**`eth0`-style interface naming**, because both are kernel arguments and belong in the same file.

Write the **exact official Red Hat KVM Guest Image `/etc/default/grub`** using a heredoc:

```bash
cat > /etc/default/grub << 'EOF'
GRUB_TIMEOUT=1
GRUB_DISTRIBUTOR="$(sed 's, release .*$,,g' /etc/system-release)"
GRUB_DEFAULT=saved
GRUB_DISABLE_SUBMENU=true
GRUB_TERMINAL_OUTPUT="console"
GRUB_CMDLINE_LINUX="console=tty0 crashkernel=auto console=ttyS0,115200n8 no_timer_check net.ifnames=0"
GRUB_DISABLE_RECOVERY="true"
EOF
```

**Key values to note vs. a default minimal install:**

| Setting                  | Official Value | Why                                  |
| ------------------------ | -------------- | ------------------------------------ |
| `GRUB_TIMEOUT`           | `1`            | Fast boot, no wait                   |
| `GRUB_TERMINAL_OUTPUT`   | `console`      | Output to serial/text console        |
| `console=ttyS0,115200n8` | `115200` baud  | Match official, faster than 9600    |
| `no_timer_check`         | present        | Fixes KVM timer issues               |
| `net.ifnames=0`          | present        | Forces `eth0` style naming           |
| `crashkernel=auto`       | present        | Enable kdump                         |
| `rhgb quiet`             | **absent**     | No graphical boot; serial only      |
| `biosdevname=0`          | **absent**     | Official image does not include this |

#### Regenerate GRUB config:

```bash
grub2-mkconfig -o /boot/grub2/grub.cfg
```

#### Enable serial getty service:

```bash
systemctl enable serial-getty@ttyS0.service
systemctl start serial-getty@ttyS0.service
```

---

### 2.6 Enable and Configure qemu-guest-agent

```bash
systemctl enable qemu-guest-agent
systemctl start qemu-guest-agent
```

---

### 2.7 Configure NetworkManager

The `net.ifnames=0` kernel argument (already set in GRUB in Step 2.5) handles `eth0`-style
interface naming. All that remains here is creating the interface config file and enabling the service.

#### Create the interface config for eth0:

```bash
cat > /etc/sysconfig/network-scripts/ifcfg-eth0 << 'EOF'
TYPE=Ethernet
BOOTPROTO=dhcp
NAME=eth0
DEVICE=eth0
ONBOOT=yes
EOF
```

#### Enable NetworkManager:

```bash
systemctl enable NetworkManager
```

---

### 2.8 SSH Configuration

```bash
# Ensure SSH is enabled
systemctl enable sshd

vi /etc/ssh/sshd_config
```

Set these options to match the guest image:

```
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
UseDNS no
```

> **Note**: `PasswordAuthentication no` matches the official image where only SSH key login
> works for `cloud-user`. For GNS3 lab use, set `PasswordAuthentication yes` if you need
> direct password logins.

---

### 2.9 Set SELinux to Enforcing

```bash
vi /etc/selinux/config
```

Ensure:

```
SELINUX=enforcing
SELINUXTYPE=targeted
```

---

### 2.10 Clean Yum Cache

```bash
yum clean all
rm -rf /var/cache/yum
```

---

## Stage 3: Seal and Generalize the Image

This makes the image reusable as a base template, exactly like Red Hat's distributed image.

```bash
# Remove SSH host keys (new ones are generated on first boot)
rm -f /etc/ssh/ssh_host_*

# Truncate machine-id (new one generated on boot)
truncate -s 0 /etc/machine-id

# Clear cloud-init state so it re-runs on next boot
cloud-init clean --logs

# Remove any leftover network state
rm -f /etc/udev/rules.d/70-persistent-net.rules
rm -f /var/lib/NetworkManager/*.lease

# Remove root's bash history
unset HISTFILE
history -c
rm -f /root/.bash_history

# Shutdown
poweroff
```

> [!CAUTION]
> After running `cloud-init clean`, the **next boot** will re-run all first-boot cloud-init tasks.
> Do not boot the VM again via virt-manager before importing into GNS3, or cloud-init's first run
> will be consumed without effect.

---

## Stage 4: Compact the qcow2 (Recommended)

After poweroff, on your **host machine**, compact the image to reduce file size:

```bash
# Find the image path
ls /var/lib/libvirt/images/

# Compact it
qemu-img convert -O qcow2 \
  /var/lib/libvirt/images/rhel79-base.qcow2 \
  /var/lib/libvirt/images/rhel79-gns3.qcow2

# Check the info
qemu-img info /var/lib/libvirt/images/rhel79-gns3.qcow2
```

Use `rhel79-gns3.qcow2` as your GNS3 appliance image going forward.

---

## Stage 5: Import into GNS3 as a QEMU Appliance

### 5.1 Add the QEMU VM Template

1. Open **GNS3** → **Edit → Preferences → QEMU VMs**
2. Click **New**
3. Fill in:
   - **Name**: `RHEL-7.9`
   - **RAM**: `1024 MB`
   - **Console type**: `telnet`
   - **Disk image**: Browse to `/var/lib/libvirt/images/rhel79-gns3.qcow2`

### 5.2 Configure QEMU Options

In the **Advanced** tab, set **Additional QEMU options**:

```
-cpu host -nographic
```

> [!IMPORTANT]
> Do **not** add `-serial telnet::CONSOLEPORT,...` here. GNS3 automatically appends
> the serial console binding (`-serial telnet:127.0.0.1:<port>,server,nowait`) based
> on the **Console type = telnet** setting above. Adding it manually causes a literal
> `CONSOLEPORT` string error and the VM will fail to start.

### 5.3 Network Adapters

In the **Network** tab:
- **Adapters**: `1` (or more if you need multi-interface labs)
- **Type**: `virtio-net-pci`

### 5.4 Save the Template

Click **OK** → **Apply** → **OK**.

---

## Stage 6: Test in GNS3

1. Drag the **RHEL-7.9** node onto the GNS3 canvas
2. Right-click → **Start**
3. Wait 30–45 seconds for boot
4. Right-click → **Console**

You should see the serial login prompt:

```
Red Hat Enterprise Linux Server 7.9 (Maipo)
Kernel 3.10.0-1160.xx.x.el7.x86_64 on an x86_64

localhost login:
```

Log in as `cloud-user` with the password you set.

---

## Comparison: Your Image vs. Official Red Hat KVM Guest Image

| Feature | Official Red Hat KVM Guest Image | Your Built Image |
|---|---|---|
| `cloud-user` account | Yes | Yes |
| Passwordless sudo via wheel | Yes | Yes |
| `cloud-init` (NoCloud datasource) | Yes | Yes |
| `qemu-guest-agent` | Yes | Yes |
| Serial console (`ttyS0`) | Yes | Yes |
| `eth0`-style interface naming | Yes | Yes |
| No LVM (standard partitions) | Yes | Yes |
| SSH host key regeneration on boot | Yes | Yes |
| Unique `machine-id` per instance | Yes | Yes |
| GNS3 telnet console | Yes | Yes |

---

## Troubleshooting

### Telnet console shows nothing / blank

- Confirm `console=ttyS0,115200n8` is in GRUB cmdline: `cat /proc/cmdline`
- Confirm `serial-getty@ttyS0` is active: `systemctl status serial-getty@ttyS0`
- Confirm GNS3 **Console type** is set to `telnet` in the QEMU VM template (not VNC or Spice)
- Do **not** have `-serial telnet::CONSOLEPORT,...` in Additional QEMU options; GNS3 adds this automatically

### VM gets no IP address

- Confirm `eth0` exists: `ip link show`
- Check `ifcfg-eth0` exists in `/etc/sysconfig/network-scripts/`
- Ensure GNS3 network adapter type is `virtio-net-pci`
- Restart NetworkManager: `systemctl restart NetworkManager`

### cloud-init runs every boot

- Means `cloud-init clean` was not run, or `machine-id` was not truncated
- Fix: `truncate -s 0 /etc/machine-id && cloud-init clean`

### SSH key login not working

- Check `~cloud-user/.ssh/authorized_keys` exists with correct permissions:
  ```bash
  chmod 700 /home/cloud-user/.ssh
  chmod 600 /home/cloud-user/.ssh/authorized_keys
  chown -R cloud-user:cloud-user /home/cloud-user/.ssh
  ```

---

## Quick Reference: All Commands in Order

```bash
# ---- Inside the VM as root ----

# 1. Install packages
yum install -y cloud-init cloud-utils-growpart qemu-guest-agent \
  NetworkManager net-tools vim curl wget sudo openssh-server bash-completion \
  bind-utils telnet rsyslog chrony tcpdump lsof \
  policycoreutils-python setools-console

# 2. Create cloud-user
useradd -m -s /bin/bash cloud-user
usermod -aG wheel cloud-user
passwd cloud-user

# 3. Passwordless sudo (edit /etc/sudoers via visudo and uncomment NOPASSWD line)

# 4. cloud-init datasource
cat > /etc/cloud/cloud.cfg.d/99_datasource.cfg << 'EOF'
datasource_list: [NoCloud, None]
EOF

# 5. Serial console + eth0 naming in GRUB
cat > /etc/default/grub << 'EOF'
GRUB_TIMEOUT=1
GRUB_DISTRIBUTOR="$(sed 's, release .*$,,g' /etc/system-release)"
GRUB_DEFAULT=saved
GRUB_DISABLE_SUBMENU=true
GRUB_TERMINAL_OUTPUT="console"
GRUB_CMDLINE_LINUX="console=tty0 crashkernel=auto console=ttyS0,115200n8 no_timer_check net.ifnames=0"
GRUB_DISABLE_RECOVERY="true"
EOF
grub2-mkconfig -o /boot/grub2/grub.cfg

# 6. Enable services
systemctl enable serial-getty@ttyS0.service cloud-init-local cloud-init \
  cloud-config cloud-final qemu-guest-agent NetworkManager sshd

# 7. Network config
cat > /etc/sysconfig/network-scripts/ifcfg-eth0 << 'EOF'
TYPE=Ethernet
BOOTPROTO=dhcp
NAME=eth0
DEVICE=eth0
ONBOOT=yes
EOF

# 8. Seal the image
rm -f /etc/ssh/ssh_host_*
truncate -s 0 /etc/machine-id
cloud-init clean --logs
rm -f /etc/udev/rules.d/70-persistent-net.rules
yum clean all
history -c
poweroff

# ---- On the host ----

# 9. Compact the qcow2
qemu-img convert -O qcow2 rhel79-base.qcow2 rhel79-gns3.qcow2
```

---

# Alternative: Official Method Using Red Hat Image Builder (`osbuild`)

> **Image Builder** is Red Hat's official tool for producing customized OS images.
> This is the same toolchain Red Hat uses internally to build the KVM Guest Images
> you download from the customer portal. It produces a sealed, production-ready qcow2
> with no manual steps required inside the VM.

---

## Image Builder Overview

| Component | Role |
|---|---|
| `osbuild-composer` | Backend service that builds images |
| `composer-cli` | Command-line frontend |
| `cockpit-composer` | Web UI frontend (optional) |
| **Blueprint** | TOML file describing what goes into the image |
| **Compose** | A single build job that produces a qcow2 |

---

## Part A: Install Image Builder on the Build Host

Image Builder runs on your RHEL host machine (not inside a VM). It needs a registered RHEL system with an active subscription.

```bash
# Install Image Builder components
yum install -y osbuild-composer composer-cli

# Optional: Web UI via Cockpit
yum install -y cockpit cockpit-composer
systemctl enable --now cockpit.socket

# Enable and start the Image Builder socket
systemctl enable --now osbuild-composer.socket

# Add your user to the weldr group to use composer-cli without sudo
usermod -aG weldr $(whoami)

# Re-login or use newgrp to apply group change immediately
newgrp weldr
```

### Verify it works:

```bash
composer-cli status show
```

Expected output:

```
API server status:
    Database version:   0
    Database supported: true
    Schema version:     0
    API version:        1
    Backend:            osbuild-composer
    Build:              NEVRA:osbuild-composer-xx.x-x.el7.x86_64
```

---

## Part B: Create a Blueprint

A Blueprint is a TOML file that defines:
- Image name and description
- Packages to include
- User accounts
- Kernel arguments
- SSH keys

### Create the blueprint file:

```bash
vi rhel79-kvm-gns3.toml
```

Paste the following: this replicates the official Red Hat KVM Guest Image:

```toml
name = "rhel79-kvm-gns3"
description = "RHEL 7.9 KVM image identical to Red Hat Guest Image, ready for GNS3"
version = "1.0.0"
distro = ""

# ---- Packages (matches official KVM Guest Image + lab tools) ----
[[packages]]
name = "cloud-init"
version = "*"

[[packages]]
name = "cloud-utils-growpart"
version = "*"

[[packages]]
name = "qemu-guest-agent"
version = "*"

[[packages]]
name = "NetworkManager"
version = "*"

[[packages]]
name = "net-tools"
version = "*"

[[packages]]
name = "vim"
version = "*"

[[packages]]
name = "curl"
version = "*"

[[packages]]
name = "wget"
version = "*"

[[packages]]
name = "sudo"
version = "*"

[[packages]]
name = "openssh-server"
version = "*"

[[packages]]
name = "bash-completion"
version = "*"

[[packages]]
name = "rsyslog"
version = "*"

[[packages]]
name = "chrony"
version = "*"

[[packages]]
name = "bind-utils"
version = "*"

[[packages]]
name = "telnet"
version = "*"

[[packages]]
name = "tcpdump"
version = "*"

[[packages]]
name = "lsof"
version = "*"

[[packages]]
name = "policycoreutils-python"
version = "*"

[[packages]]
name = "setools-console"
version = "*"

# ---- cloud-user account ----
[[customizations.user]]
name = "cloud-user"
description = "Cloud User"
groups = ["wheel"]
shell = "/bin/bash"
# Set a password hash (generate with: openssl passwd -6 'yourpassword')
# Leave blank to rely on SSH key injection via cloud-init
password = "$6$rounds=656000$..."

# ---- Kernel arguments ----
# Adds serial console support (required for GNS3 telnet) and eth0 naming
[customizations.kernel]
append = "console=tty0 console=ttyS0,115200n8 no_timer_check net.ifnames=0"

# ---- SSH service ----
[customizations.services]
enabled = ["sshd", "cloud-init", "cloud-init-local", "cloud-config", "cloud-final",
           "qemu-guest-agent", "NetworkManager", "serial-getty@ttyS0"]
```

### Generate a password hash for cloud-user:

```bash
# Generate a SHA-512 password hash
openssl passwd -6 'your-password-here'
```

Copy the output and paste it into the `password = "..."` field in the blueprint.

---

## Part C: Push the Blueprint and Start a Compose

### Push the blueprint to Image Builder:

```bash
composer-cli blueprints push rhel79-kvm-gns3.toml
```

### Verify it was accepted:

```bash
composer-cli blueprints list
composer-cli blueprints show rhel79-kvm-gns3
```

### Check what packages will be included (dependency resolution):

```bash
composer-cli blueprints depsolve rhel79-kvm-gns3
```

### Start the build (qcow2 format):

```bash
composer-cli compose start rhel79-kvm-gns3 qcow2
```

This returns a **UUID** for the build job, e.g.:

```
Compose c1f2d9a4-3b5e-4f2a-8c1d-7e3a6b9f0c2d added to the queue
```

---

## Part D: Monitor the Build

```bash
# Check all compose jobs
composer-cli compose status

# Watch until it shows FINISHED
watch -n 10 'composer-cli compose status'
```

Status values:

| Status | Meaning |
|---|---|
| `WAITING` | Queued, not started yet |
| `RUNNING` | Build in progress |
| `FINISHED` | Build complete, ready to download |
| `FAILED` | Build failed; check logs |

### If the build fails, check the logs:

```bash
# List the UUID from status output, then:
composer-cli compose log <UUID>

# Or fetch full logs
composer-cli compose logs <UUID>
```

---

## Part E: Download the Built Image

```bash
# Download the qcow2 (creates a .tar file containing the qcow2)
composer-cli compose image <UUID>

# This creates a file like:
# <UUID>-disk.qcow2

# Rename for clarity
mv <UUID>-disk.qcow2 rhel79-gns3.qcow2

# Verify the image
qemu-img info rhel79-gns3.qcow2
```

Expected output:

```
image: rhel79-gns3.qcow2
file format: qcow2
virtual size: 10G (10737418240 bytes)
disk size: 1.2G
cluster_size: 65536
```

The image is already **sealed**; Image Builder automatically:
- Removes SSH host keys (regenerated on first boot)
- Sets `machine-id` to empty (new one generated on boot)
- Runs `cloud-init clean` as part of the finalization step

---

## Part F: Import into GNS3

Follow the exact same steps as **Stage 5** in the manual guide above:

1. **GNS3** → **Edit → Preferences → QEMU VMs** → **New**
2. Set **Console type** = `telnet`
3. Set **Disk image** = path to `rhel79-gns3.qcow2`
4. In **Advanced** tab, **Additional QEMU options**:
   ```
   -cpu host -nographic
   ```
5. **Network** tab → Adapter type: `virtio-net-pci`
6. Save → test with right-click → **Console**

---

## Part G: Useful composer-cli Reference

```bash
# List all blueprints
composer-cli blueprints list

# Show a specific blueprint
composer-cli blueprints show <name>

# Delete a blueprint
composer-cli blueprints delete <name>

# List all compose jobs (history)
composer-cli compose list

# Check status of all composes
composer-cli compose status

# Cancel a running compose
composer-cli compose cancel <UUID>

# Delete a finished compose (frees disk space)
composer-cli compose delete <UUID>

# List available image output formats
composer-cli compose types

# Show what distros are available
composer-cli distros list
```

---

## Part H: Optional: Web UI via Cockpit

If you installed `cockpit-composer`:

1. Open a browser on your RHEL host: `https://localhost:9090`
2. Log in with your system credentials
3. Navigate to **Image Builder** in the left sidebar
4. Create blueprints and start composes via the GUI; it does everything `composer-cli` does but visually

---

## Image Builder vs Manual Install: Comparison

| Aspect | Manual Install (Stage 1-6) | Image Builder |
|---|---|---|
| Time to build | 60-90 minutes | 5-15 minutes (automated) |
| Reproducibility | Poor (manual steps) | Excellent (blueprint = code) |
| Sealing/generalization | Manual | Automatic |
| Used by Red Hat | No | Yes, exactly this tool |
| Requires subscription | No (just ISO) | Yes (registered host) |
| Good for learning | Yes | Less hands-on |
| Good for production | No | Yes |
| Version controlled | No | Yes (blueprint TOML in git) |

---

## Summary: Which Method to Use

```
Lab learning / understanding the internals  →  Manual install (Stage 1-6 above)
Repeatable builds for multiple VMs          →  Image Builder (this section)
CI/CD pipeline image builds                 →  Packer + QEMU builder
```

For GNS3 lab environments where you need to quickly spin up multiple RHEL nodes,
Image Builder is the fastest and most reliable path after initial setup.
