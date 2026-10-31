# Arch Linux Installation Guide

## 1. Download Arch Linux

1. Download the Arch Linux ISO from the [official Arch Linux download page](https://archlinux.org/download/).
2. Create a virtual machine (VM) and attach the ISO.
3. Configure the VM:
   - Enable EFI.
   - Set memory allocation.
   - Set CPU allocation.
   - Configure storage.
4. Launch the VM.
5. Select **Arch Linux install medium** from the boot menu.

## 2. Initial Setup

### Keyboard Layout

Set the keyboard layout and font if necessary. The default keyboard layout is US.

### Verify Boot Mode

Check that the system booted in UEFI mode:

```bash
cat /sys/firmware/efi/fw_platform_size
```

A result of `64` indicates a 64-bit UEFI environment.

### Configure Network

Verify that the network interface is listed and enabled:

```bash
ip link
```

Test the network connection:

```bash
ping ping.archlinux.org
```

### Update System Clock

Verify that the system clock is accurate:

```bash
timedatectl
```

## 3. Disk Partitioning

### Identify Disks

List available disks and partitions:

```bash
fdisk -l
lsblk
```

### Configure Partitions

Launch `cfdisk`:

```bash
cfdisk
```

Create the following partitions:

**Partition 1: EFI System Partition**

1. Select the available free space.
2. Choose **New**.
3. Enter `500M` for the partition size.
4. Select **Type**.
5. Choose **EFI System**.

**Partition 2: Linux Filesystem**

1. Select the remaining free space.
2. Choose **New**.
3. Use all remaining available space.
4. The partition type should automatically be Linux filesystem.

Write the partition changes and exit.

### Format Partitions

Format the EFI partition as FAT32:

```bash
mkfs.fat -F 32 /dev/sda1
```

Format the Linux partition as ext4:

```bash
mkfs.ext4 /dev/sda2
```

### Mount Partitions

Mount the Linux root partition:

```bash
mount /dev/sda2 /mnt
```

Create the boot directory and mount the EFI partition:

```bash
mount --mkdir /dev/sda1 /mnt/boot
```

## 4. Install Essential Packages

Install the base system, Linux kernel, and firmware:

```bash
pacstrap -K /mnt base linux linux-firmware
```

## 5. Configure the Base System

### Generate fstab

Generate the filesystem table:

```bash
genfstab -U /mnt >> /mnt/etc/fstab
```

Verify the generated configuration:

```bash
cat /mnt/etc/fstab
```

### Enter the Installed System (Chroot)

Change root into the newly installed system:

```bash
arch-chroot /mnt
```

This allows you to configure the installed system as if you had booted directly into it.

## 6. System Configuration

### Set Time Zone

Set the time zone to America/Chicago:

```bash
ln -sf /usr/share/zoneinfo/America/Chicago /etc/localtime
```

Synchronize the hardware clock:

```bash
hwclock --systohc
```

### Install Nano

Install the Nano text editor:

```bash
pacman -S nano
```

### Configure Localization

Open the locale configuration file:

```bash
nano /etc/locale.gen
```

Uncomment the following line:

```text
en_US.UTF-8 UTF-8
```

Generate the locale:

```bash
locale-gen
```

Create the locale configuration file:

```bash
nano /etc/locale.conf
```

Insert:

```text
LANG=en_US.UTF-8
```

### Configure Hostname

Create or edit the hostname file:

```bash
nano /etc/hostname
```

Insert your desired hostname.

### Configure Networking

Install NetworkManager:

```bash
pacman -S networkmanager
```

Enable NetworkManager to start automatically at boot:

```bash
systemctl enable NetworkManager
```

Configure the hosts file:

```bash
nano /etc/hosts
```

Insert the following entries, replacing `hostname` with your chosen hostname:

```text
127.0.0.1   localhost
::1         localhost
127.0.1.1   hostname.localdomain hostname
```

### Configure Initramfs

Regenerate the initial RAM filesystem:

```bash
mkinitcpio -P
```

### Set Root Password

Set a password for the root account:

```bash
passwd
```

Follow the prompts to enter and confirm the password.

## 7. Install and Configure the Bootloader

This guide uses **systemd-boot**.

### Install systemd-boot

```bash
bootctl install
```

### Identify the Root Partition UUID

Retrieve the UUID of the Linux root partition:

```bash
blkid /dev/sda2
```

Copy the UUID for use in the bootloader configuration.

### Create Bootloader Entry

Open the Arch Linux boot entry:

```bash
nano /boot/loader/entries/arch.conf
```

Insert the following configuration, replacing the UUID with the one retrieved earlier:

```text
title Arch Linux
linux /vmlinuz-linux
initrd /initramfs-linux.img
options root=UUID=YOUR-UUID-HERE rw
```

### Configure systemd-boot

Open the bootloader configuration:

```bash
nano /boot/loader/loader.conf
```

Insert:

```text
default arch.conf
timeout 5
console-mode max
editor no
```

## 8. Reboot into Arch Linux

Exit the chroot environment:

```bash
exit
```

Reboot the VM:

```bash
reboot
```

After the VM powers off or restarts:

1. Remove the ISO from the VM's virtual optical drive.
2. Start the VM again.
3. Log in using the root account and password.

## 9. Update Arch Linux

Update the package database and installed packages:

```bash
pacman -Syu
```

## 10. Install a Desktop Environment

Install Xorg, LXQt, SDDM, and Breeze icons:

```bash
pacman -S xorg-server lxqt sddm breeze-icons
```

## 11. Create a Regular User

Create a user account with a home directory and add it to the `wheel` group:

```bash
useradd -m -G wheel -s /bin/bash username
```

Replace `username` with your desired username.

Set the user's password:

```bash
passwd username
```

## 12. Install Zsh

Install Zsh and its completion scripts:

```bash
pacman -S zsh zsh-completions
```

Launch Zsh:

```bash
zsh
```

## 13. Configure SSH

Install OpenSSH:

```bash
pacman -Syu openssh
```

Start the SSH service:

```bash
systemctl start sshd
```

Enable SSH to start automatically at boot:

```bash
systemctl enable sshd
```

## 14. Customize the Terminal

### Enable Pacman Colors

Open the Pacman configuration:

```bash
nano /etc/pacman.conf
```

Find the following line and uncomment it:

```text
Color
```

### Customize Bash Prompt

Open the Bash configuration file:

```bash
nano ~/.bashrc
```

Add or replace the `PS1` variable with:

```bash
PS1='\[\e[01;32m\]\u@\h\[\e[00m\]:\[\e[01;34m\]\w\[\e[00m\]\$ '
```

This provides a colored terminal prompt:
- Green username and hostname.
- Blue current working directory.
- Default-colored command prompt symbol.

## 15. Boot into the Graphical Desktop

Enable SDDM:

```bash
systemctl enable sddm
```

Set the default system target to graphical:

```bash
systemctl set-default graphical.target
```

Reboot to start the graphical login manager:

```bash
reboot
```

## 16. Create Aliases

Add convenient command shortcuts to your shell configuration.

```bash
alias ..='cd ..'
alias c='clear'
```

Add these to `~/.bashrc` for Bash or `~/.zshrc` for Zsh to make them persistent.
