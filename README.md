# Local Storage and Sharing Setup in Proxmox

This repository provides a comprehensive guide on how to create local storage and enable sharing in Proxmox.

## Prerequisites

- A Proxmox server
- Basic knowledge of Proxmox and Linux commands

## Steps

### 1. Create a ZFS Pool

The first step is to create a ZFS pool. This will be used for local storage.

### 2. Create a Container in Proxmox

Configure the container with the following settings:
- **Hostname:** fileserver
- **Privilege:** Yes
- **Features:** NFS, Nesting, SMB
- **Mount Point:** zpool:/mnt/rust (with specific storage number)

### 3. Login to the Container

Login to the container using the password set during the creation process.

### 4. Execute the Bash Script

Copy the provided bash script and paste it into the terminal of the container to set up the necessary users and install Cockpit.

#### Bash Script

```bash
#!/bin/bash
## 
apt update -y && apt upgrade -y

# Create a user "admin" and ask for password
read -p "Enter password for user admin: " -s password
echo
adduser --gecos "" admin
echo "admin:$password" | chpasswd

# Add "admin" to the sudo group
usermod -aG sudo admin

# Install Cockpit
apt update
apt install --no-install-recommends cockpit -y

# Enable Cockpit on boot
systemctl enable cockpit.socket

# Install curl
apt install -y curl

# Download Cockpit File Sharing
wget https://github.com/45Drives/cockpit-file-sharing/releases/download/v3.3.4/cockpit-file-sharing_3.3.4-1focal_all.deb
# Download Cockpit Navigator
wget https://github.com/45Drives/cockpit-navigator/releases/download/v0.5.10/cockpit-navigator_0.5.10-1focal_all.deb
# Download Cockpit Identities
wget https://github.com/45Drives/cockpit-identities/releases/download/v0.1.12/cockpit-identities_0.1.12-1focal_all.deb
# Install them
apt install ./*.deb -y
# It will complain about being unable to delete the deb files, so we will do that now
rm ./*.deb

# Create group "slow"
groupadd slow
# Create user "autoadmin" and ask for password
read -p "Enter password for user autoadmin: " -s password
echo
adduser --gecos "" autoadmin
echo "autoadmin:$password" | chpasswd
# Add "autoadmin" to group "slow"
usermod -aG slow autoadmin
```
### 5. Access the Web Interface

Login to the Cockpit web interface using:
https://fileserver.lan:9090
- **Username:** admin

### 6. Configure File Sharing

1. Navigate to "File Sharing".
2. Create a new share named `slow`.
3. Set permissions for `autoadmin` and group `slow`.
4. Choose Windows ACL with Linux support.

### 7. Test the Connection

Ensure that the connection to the newly created share is working correctly.

### 8. Create and Configure Sanoid

Create and configure the `/etc/sanoid/sanoid.conf` file with the provided settings:

#### Sanoid Configuration Script

```bash
#!/bin/bash

# Ensure the sanoid directory exists
mkdir -p /etc/sanoid

# Create the sanoid.conf file with the provided content
cat <<EOL >/etc/sanoid/sanoid.conf
# you can also handle datasets recursively in an atomic way without the possibility to override settings for child datasets.
[data/video]
        use_template = production
        recursive = zfs
[data/media]
        use_template = production
        recursive = zfs

#############################
# templates below this line #
#############################

# name your templates template_templatename. you can create your own, and use them in your module definitions above.

# Using a lot of frequently at 30min for Shadow Copy, since it isn't a fan of the differently named snapshots.
[template_production]
        frequently = 144
        frequent_period = 30
        hourly = 0
        daily = 30
        monthly = 3
        yearly = 0
        autosnap = yes
        autoprune = yes
EOL

# Confirm the creation and content of the sanoid.conf file
echo "sanoid.conf has been created with the provided settings:"
cat /etc/sanoid/sanoid.conf
```
### 9. Configure Advanced File Sharing Options
Add the following advanced options to the "file sharing" configuration:
``` options
map acl inherit = yes
vfs objects = shadow_copy2 acl_xattr
shadow:snapdir = .zfs/snapshot
shadow:sort = desc
shadow:format = autosnap_%Y-%m-%d_%H:%M:%S_frequently
```
