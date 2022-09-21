**Table of contents:**
[toc]

<div style="page-break-after: always;"></div>

# Preamble

The following procedure describes how to create a small bootable [cloud-init](https://cloud-init.io/) template with [Alpine Linux](https://alpinelinux.org/) for creating virtual machines on a [Proxmox VE](https://www.proxmox.com/en/proxmox-ve) server/cluster.

The official [Cloud Images](https://alpinelinux.org/cloud/) are currently (Sept. 2022) only available for AWS, this shows how to make a `NoCloud` cloud-init image suitable for Proxmox VE.

## References

### Alpine Linux

1. [Cloud Init package README.Alpine](https://git.alpinelinux.org/aports/tree/community/cloud-init/README.Alpine)

2. [git project for creating official images](https://gitlab.alpinelinux.org/alpine/cloud/alpine-cloud-images/)

### Proxmox VE

1. [Cloud-Init Support](https://pve.proxmox.com/wiki/Cloud-Init_Support)

2. [Cloud-Init FAQ - Creating a custom cloud image](https://pve.proxmox.com/wiki/Cloud-Init_FAQ#Creating_a_custom_cloud_image)

<div style="page-break-after: always;"></div>

# Create cloud-init image

## Create base Image

1. [Download "**VIRTUAL**" ISO](https://alpinelinux.org/downloads/) to shared ISO storage on PVE cluster (Target: **cephfs**).
2. Create the virtual machine (VM):
	- Use high number identify it later when it is a template (e.g. 9000)
	- No installation CD/DVD (will be applied later)
	- 1 GB system drive on shared VM drive which can be grown later (Target: **osd**).
	- 1024 MiB RAM with 512 MiB minimum
	- All cores (**4**)
3. Configure VM:
	- Remove default "IDE 2" DVD/CDROM
	- Add uploaded ISO as DVD/CDROM - SCSI 2
	- Disable NET Boot
4. Install Alpine Linux in the VM:
	- Follow the normal installation instructions
	- Add default user "alpine", another default user can be created using cloud-init if required.
	- Reboot
5. Create a **snaphot**, really.

<div style="page-break-after: always;"></div>

## Copy keys to VM for root ssh access with a password
Using the Proxmox `Console` to make configurtion as root easier, the default configuration disables password access for root.

In the VM:
```sh
# Enable password root login for sshd
vi /etc/ssh/sshd_config
# Add
PermitRootLogin yes
# Restart SSH daemon
service sshd restart
```

On the PC with the keys to be copied:
```sh
# Copy ssh ids from current PC to VM with ssh-copy-ids
ssh-copy-ids root@vm-name
```

In the VM:
```sh
# disable root login for sshd
vi /etc/ssh/sshd_config
# Remove
PermitRootLogin yes
# Restart SSH daemon
service sshd restart
```

<div style="page-break-after: always;"></div>

## Configure VM
Connect to VM `ssh root@vm-name`

### Prepare operating system

```sh
# Enable all repositories (test is required for kubernetes executables)
vi /etc/apk/repositories

# Update & upgrade system
apk upgrade --no-cache --available

# Install required packages
apk --no-cache --cache-max-age 30 add \
        cloud-init \
        util-linux \
        chrony \
        openssh-server-pam \
        doas \
        sudo \
        e2fsprogs \
        e2fsprogs-extra \
        dosfstools \
        gettext \
        lsblk \
        parted \
        tzdata

# Update the kernel options
sed -Ei \
  -e "s|^[# ]*(default_kernel_opts)=.*|\1=\"console=ttyS0,115200n8 cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory\"|" \
  -e "s|^[# ]*(serial_port)=.*|\1=ttyS0|" \
  -e "s|^[# ]*(modules)=.*|\1=sd-mod,usb-storage,ext4|" \
  -e "s|^[# ]*(default)=.*|\1=virt|" \
  -e "s|^[# ]*(timeout)=.*|\1=1|" \
  "/etc/update-extlinux.conf"
/sbin/extlinux --install /boot
/sbin/update-extlinux --warn-only

# Disable getty for physical ttys, enable getty for serial ttyS0.
sed -Ei -e '/^tty[0-9]/s/^/#/' -e '/^#ttyS0:/s/^#//' "/etc/inittab"

```
. . . 

<div style="page-break-after: always;"></div>

```sh
# configure sudo and doas

# --- no password required ---
echo '%wheel ALL=(ALL) NOPASSWD: ALL' > "/etc/sudoers.d/wheel"
echo 'permit nopass :wheel' > "/etc/doas.d/wheel.conf"

# --- with required user password ---
echo '%wheel ALL=(ALL) ALL' > "/etc/sudoers.d/wheel"
echo 'permit persist :wheel' > "/etc/doas.d/wheel.conf"

# explicitly lock the root account
/bin/sh -c "/bin/echo 'root:*' | /usr/sbin/chpasswd -e"
/usr/bin/passwd -l root

# Update services
rc-update add chronyd default
```

<div style="page-break-after: always;"></div>

# Configure cloud-init

Now cloud-init itself needs to be configured.

## Configure VM

The `cloud-init` package was installed previously with the other prerequisites.

A mixture of information from Proxmox VE and alpine linux sources

```sh
# Setup cloud-init defaults
setup-cloud-init

# Set Proxmox options as Datasources
echo 'datasource_list: [ NoCloud, ConfigDrive ]' > "/etc/cloud/cloud.cfg.d/99_pve.cfg"

# Shut down the VM and do **NOT** restart it
poweroff
```

## Configure Proxmox
In a shell on the **proxmox** server (not the VM), here 9000 is the VM ID number.

Add virtual CDROM for cloud-init configuration (cloud-init reads configuration from mounted drive)
```sh
qm set 9000 --ide2 local-lvm:cloudinit
```

Set boot device:
```sh
qm set 9000 --boot c --bootdisk scsi0
```

Enable serial device:
```sh
qm set 9000 --serial0 socket --vga serial0
```

<div style="color:red;">
<div style="font-weight:bold;display:inline-block;">n.b.</div> In Proxmox the default configuration for the NIC is an <div style="display:inline-block;text-decoration:underline;">empty static IP address</div>! Select the VM (9000) and configure the template to use DHCP (`Cloud-Init -> IP Conifg (net0)`). A static IP address can always be defined in clones made from the template if required.
</div>

So, everything is set to test it. It would be possible to convert this to a template right now but it is better to be safe than sorry.

<div style="page-break-after: always;"></div>

### Testing the container before converting it to a template

Either in the Proxmox UI or terminal:

```sh
qm clone 9000 1024 --name AlpineTest
```


In the `Cloud-Init` options for VM *1024*, try settting "User", "Password" and "SSH public key" **BEFORE** starting the VM for the first time. Remeber to press the  `Regenerate Image` button to update the *CDROM* containing the `Cloud-Init` configuration.

Start `AlpineTest` and you should now have a VM with the configured information.

## Convert VM image to a Proxmox template
<div style="color:red;font-weight:bold;display:inline-block; margin-left: 20%;">WARNING: This can not be undone!</div>

Either in the Proxmox UI or terminal (you will have to remove any snapshots beforehand):

```sh
qm template 9000
```

<div style="page-break-after: always;"></div>

# Create VM from template

Don't forget to switch `Mode:` to `Full Clone`!

Here are a few *optional changes* where *1024* is the VM ID number of the clone:

## Resize drive (**BEFORE** starting!)

Increase the size of the drive **BEFORE** starting the clone, cloud-init will **automatically** expand the root parttition to match the new size.

In a shell on the **proxmox** server (not the VM) add 5G to the image's drive capacity:
```sh
qm resize 1024 scsi0 +5G
```

## Increase RAM

Enable guest agent support in Proxmox `VM (1024) -> Hardware -> Memory`

## Install QEMU Guest Agent

### VM
```sh
sudo apk add qemu-guest-agent
```

### Proxmox
Enable guest agent support in Proxmox `VM (1024) -> Options -> QEMU Guest Agent`

## Install a different shell

The default shell in Alpine Linux is `/bin/ash`, you can install another shell, e.g. *bash* (bourne again shell).

Install `bash` and `bash-completion`:
```sh
sudo apk add bash bash-completion
```

Change the user's shell in `/etc/passwd` using `chsh`:
```sh
chsh -s /bin/bash
```

Reconnect . . . .

# Credits

Inspired to try out [cloud-init](https://cloud-init.io/) after watching [Perfect Proxmox Template with Cloud Image and Cloud Init](https://www.youtube.com/watch?v=shiIi38cJe4) by [TechnoTim](https://github.com/techno-tim/).
