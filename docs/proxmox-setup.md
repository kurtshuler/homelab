---
layout: default
title: Proxmox Setup
nav_order: 2
---

# Proxmox Setup
{: .no_toc }

Just the Docs has some specific configuration parameters that can be defined in your Jekyll site's \_config.yml file.
{: .fs-6 .fw-300 }

<details open markdown="block">
  <summary>
    Table of contents
  </summary>
  {: .text-delta }
1. TOC
{:toc}
</details>




---
## Make a bootable USB drive with OS images and tools using Ventoy
The latest Ventoy installers are at [https://sourceforge.net/projects/ventoy/files/](https://sourceforge.net/projects/ventoy/files/)

{: .note }
The Ventoy USB can be created in Linux or Windows only. **For Mac**, use Parallels Windows VM or Linux VM. After you have created the Ventoy USB, you can copy files to it using your Mac or PC.
   
Important ISO images:
   - System Rescue ISO:                      (https://www.system-rescue.org/Download/)                         
   - Proxmox PVE and PBS ISOs:               https://www.proxmox.com/en/downloads                                 
   - Ubuntu Server Live Install ISO:         https://releases.ubuntu.com/24.04/ubuntu-24.04-live-server-amd64.iso 
   - Ubuntu Server Cloud-Init Install ISO:   https://cloud-images.ubuntu.com/noble/current/                       
   - Ubuntu DESKTOP ISO:                     https://ubuntu.com/download/desktop/                                 

## Proxmox post-install setup
### Check that SSH is running
```
systemctl status ssh.service
```
### Run tteck's Proxmox VE Post Install Script
tteck's Helper-Scripts are at https://tteck.github.io/Proxmox/

- WARNING: Run tteck scripts from the **Proxmox GUI shell**, not SSH!

```
bash -c "$(wget -qLO - https://github.com/tteck/Proxmox/raw/main/misc/post-pve-install.sh)"
```
### Set up IKoolcore-specific Proxmox summary
Follow the steps in the iKoolcore R2 wiki at https://github.com/KoolCore/Proxmox_VE_Status

Add iKoolcore R2 hardware stats to the Proxmox summary page by running this shell script that I modified https://github.com/kurtshuler/proxmox-ubuntu-server/blob/main/Proxmox%20files/Proxmox_VE_Status_zh.sh
```
cd Proxmox_VE_Status
```
```
bash ./Proxmox_VE_Status_zh.sh
```

### Run tteck's Proxmox VE Processor Microcode Script
tteck's Helper-Scripts are at https://tteck.github.io/Proxmox/
```diff
- WARNING: Run tteck scripts from the **Proxmox GUI shell**, not SSH!
```
```
bash -c "$(wget -qLO - https://github.com/tteck/Proxmox/raw/main/misc/microcode.sh)"
```
Reboot
```
reboot
```

## Set up the Proxmox terminal
### Install neofetch
```
sudo apt install neofetch
```
### Install Oh My Bash
```
bash -c "$(curl -fsSL https://raw.githubusercontent.com/ohmybash/oh-my-bash/master/tools/install.sh)"
```
Reload `.bashrc`
```
source ~/.bashrc
```
If error loading OMB, set proper OMB file location in `.bashrc`
```
export OSH='/root/.oh-my-bash'
```
### Add plugins and completions to `.bashrc`
Edit `.bashrc` by copying and comparing to GitHub Proxmox [`.bashrc`](/Proxmox%20files/.bashrc)
```
nano .bashrc
```
Reload `.bashrc`
```
source ~/.bashrc
```

### Install iTerm shell integration:
In the iTerm 2 GUI, click on `iTerm2 → Iterm Shell Integration`

## Configure Proxmox alerts
This guide is adapted from Techno Tim's [Set up alerts in Proxmox before it's too late!](https://technotim.live/posts/proxmox-alerts/) article.

### Install dependencies
```
apt update
apt install -y libsasl2-modules mailutils
```
### Configure app passwords on your Google account
https://myaccount.google.com/apppasswords

### Configure postfix
```
echo "smtp.gmail.com your-email@gmail.com:YourAppPassword" > /etc/postfix/sasl_passwd
```
#### Update permissions
```
chmod 600 /etc/postfix/sasl_passwd
```
#### Hash the file

```
postmap hash:/etc/postfix/sasl_passwd
```
Check to to be sure the db file was created

```
cat /etc/postfix/sasl_passwd.db
```
#### Edit postfix config

```
nano /etc/postfix/main.cf
```
Comment out line 26
```
### relayhost =
```
Add this text at end of file
```
# google mail configuration

relayhost = smtp.gmail.com:587
smtp_use_tls = yes
smtp_sasl_auth_enable = yes
smtp_sasl_security_options =
smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd
smtp_tls_CAfile = /etc/ssl/certs/Entrust_Root_Certification_Authority.pem
smtp_tls_session_cache_database = btree:/var/lib/postfix/smtp_tls_session_cache
smtp_tls_session_cache_timeout = 3600s
```
Save file 

#### Reload postfix
```
postfix reload
```
#### Send a test email
```
echo "This is a test message sent from postfix on my Proxmox Server" | mail -s "Test Email from Proxmox" shulerpve1@gmail.com
```
### Fix the "from" name in email

#### Install dependency
```
apt update
apt install postfix-pcre
```
#### Edit config
```
nano /etc/postfix/smtp_header_checks
```
Add the following text
```
/^From:.*/ REPLACE From: pve1-alert <pve1-alert@something.com>
```
#### Hash the file
```
postmap hash:/etc/postfix/smtp_header_checks
```
Check the contents of the file
```
cat /etc/postfix/smtp_header_checks.db
```
#### Add the module to our postfix config
```
nano /etc/postfix/main.cf
```
Add to the end of the file
```
smtp_header_checks = pcre:/etc/postfix/smtp_header_checks
```
#### Reload postfix service
```
postfix reload
```
#### Send another  test email
```
echo "This is a second test message sent from postfix on my Proxmox Server" | mail -s "Second Test Email from Proxmox" shulerpve1@gmail.com
```
## Set up iGPU passthrough in Proxmox Host

**NOTE:** Additional steps are required in **each VM** to 
### 5.1. Make IOMMU changes at boot
**NOTE:** There are two possible boot systems, Systemd (EFI) or Grub.

According to the [Proxmox manual](https://pve.proxmox.com/pve-docs/pve-admin-guide.html#sysboot): "For EFI Systems installed with ZFS as the root filesystem `systemd-boot` is used, unless Secure Boot is enabled. All other deployments use the standard GRUB bootloader (this usually also applies to systems which are installed on top of Debian)."

➡️ BOTTOM LINE: If you did not install Proxmox on ZFS, it's normal that GRUB is used for booting in UEFI mode and you will use the first method below.

#### For Grub boot, edit `/etc/default/grub`
Open `/etc/default/grub`
``` sh
nano /etc/default/grub
```
Change this line to:
```EditorConfig
GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt"
```
Save file and close

Run:
```
update-grub
```
#### For Systemd (EFI) boot, edit `/etc/kernel/cmdline`
**NOTE:** These steps are for EFI boot systems.

Open `/etc/kernel/cmdline`
```
nano /etc/kernel/cmdline
```
Add this to first line:

>**NOTE** All commands in `/etc/kernel/cmdline` must be in a **single line** on the **first line!**
```EditorConfig
intel_iommu=on iommu=pt
```
Save file and close

Run:
```
proxmox-boot-tool refresh
```
### Load VFIO modules at boot
Open `/etc/modules`
```
nano /etc/modules
```

Add these lines:
```
vfio
vfio_iommu_type1
vfio_pci
```

Save file and close

### Configure VFIO for PCIe Passthrough

#### Find your GPU PCI identifier
   
It will be something like `00:02`
```
lspci
```

#### Find your GPU's PCI HEX values

Enter the PCI identifier (`00:02`) from above into the `lspci` command: 
```
lspci -n -s 00:02 -v
```
You will see an associated HEX value like `8086:46d0`

#### Edit `/etc/modprobe.d/vfio.conf`

Copy the HEX values from your GPU into this command and hit enter:
```
echo "options vfio-pci ids=8086:46d0 disable_vga=1"> /etc/modprobe.d/vfio.conf
```

#### Apply all changes
```
update-initramfs -u -k all
```

### Blacklist Proxmox host device drivers

This ensures nothing else on Proxmox can use the GPU that you want to pass through to a VM.

#### Edit `/etc/modprobe.d/iommu_unsafe_interrupts.conf`
```
echo "options vfio_iommu_type1 allow_unsafe_interrupts=1" > /etc/modprobe.d/iommu_unsafe_interrupts.conf
```

#### Edit `/etc/modprobe.d/blacklist.conf`
```
echo "blacklist i915" >> /etc/modprobe.d/blacklist.conf
echo "blacklist nouveau" >> /etc/modprobe.d/blacklist.conf
echo "blacklist nvidia" >> /etc/modprobe.d/blacklist.conf
```

#### Apply all changes
```
update-initramfs -u -k all
```

#### Reboot to apply all changes
```
reboot
```

### Verify all changes

#### Verify `vfio-pci` kernel driver being used:
```
lspci -n -s 00:02 -v
```

In the output, you should see: 
   ```
   Kernel driver in use: vfio-pci
   ```
#### Verify IOMMU is enabled:
```
dmesg | grep -e DMAR -e IOMMU
```
In the output, you should see: 
   ```
   DMAR: IOMMU enabled
   ```
#### Verify IOMMU interrupt remapping is enabled:
```
dmesg | grep 'remapping'
```
In the output, you should see something like: 
   ```
   DMAR-IR: Enabled IRQ remapping in x2apic mode
   ```
#### Verify that Proxmox recognizes the GPU:
```
lspci -v | grep -e VGA
```
In the output, you should see something like: 
   ```
   00:02.0 VGA compatible controller: Intel Corporation Alder Lake-N [UHD Graphics] (prog-if 00 [VGA controller])
```
## Set up Proxmox SSL certificates using Lets Encrypt and Cloudflare

Instructions are at: https://www.derekseaman.com/2023/04/proxmox-lets-encrypt-ssl-the-easy-button.html

## Set up NUT UPS Monitoring

Instructions for NUT server and client installation on Proxmox bare metal server are at: https://technotim.live/posts/NUT-server-guide/

## Set up Proxmox Backup Server (PBS) backup to Synology NAS

Instructions are at: https://www.derekseaman.com/2023/04/how-to-setup-synology-nfs-for-proxmox-backup-server-datastore.html


