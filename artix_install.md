# Artix Linux Installation Text Guide
Fresh Artix Linux minimal install for ThinkPad (or wherever).
Artix is Arch-based, systemd-free distribution with runit as its init system.

More info: [Artix Website](https://artixlinux.org/)


## Preparations
[Download iso here](https://artixlinux.org/download.php)

Burn it to usb stick (I recommend [Ventoy](https://www.ventoy.net/en/index.html) - a tool that allows you to boot from local iso files directly)

### Wirless network setup 
More info: [Arch wiki ConnMan](https://wiki.archlinux.org/title/ConnMan)

```bash
# Check that wifi unblocked and unblock if neccessary

rfkill list
rfkill unblock wifi

# To connect to protected network we have to use interactive mode

connmanctl

# Scan for any networks around:

connmanctl> scan wifi

# Get the scan results:

connmanctl> services

# Turn on agent to handle user requests:

connmanctl> agent on

# Connect to your network (you have to use service name, tab autocompletion works):

connmanctl> connect wifi_dc85de828967_38303944616e69656c73_managed_psk

# Provide the password and wait for network response:

Agent RequestInput wifi_dc85de828967_38303944616e69656c73_managed_psk
  Passphrase = [ Type=psk, Requirement=mandatory ]
  Passphrase?  *******
Connected wifi_dc85de828967_38303944616e69656c73_managed_psk

# Quit, network should work now

connmanctl> quit
```

### Drive partitioning
In this example will be used two partitions: ```/boot``` and encrypted ```/``` with legacy boot (BIOS).

#### More info: 
Partitioning: [Arch Wiki Partitioning](https://wiki.archlinux.org/title/Partitioning) 

Encryption: [dm-crypt/Device encryption](https://wiki.archlinux.org/title/dm-crypt/Device_encryption)
```bash
# Check your drives
lsblk

# In most cases fdisk will work. You can use *parted*, *cfdisk*, etc
# Replace *sda* with your drive (sdb, nvme01, etc). Run fdisk as root or with *sudo*

fdisk /dev/sda

Welcome to fdisk (util-linux 2.38.1).
Changes will remain in memory only until you decide to write them.
Be careful before using the write command.

# Delete partitions if neccessary. In the example below 2 partitions deleted and new are created.

Command (m for help): d
Partition number (1,2, default 2): 2

Partition 2 has been deleted.

Command (m for help): d
Selected partition 1
Partition 1 has been deleted.

# Create new partition for /boot. Press enter for default values
# Type +1G for last sector to make 1GB partition
# Type Y for confirmation if sector contains another filesystem signature

Command (m for help): n
Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)

Select (default p): 
Press enter for default (1)
Partition number (1-4, default 1): 
First sector (2048-2097151, default 2048):
Last sector, +sectors or +size{K,M,G,T,P} (2048-2097151, default 2097151): +1G

Partition #1 contains an ext4 signature.
Do you want to remove the signature? [Y]es/[N]o: Y

# Encrypt your root partition
# Type YES in capital letters for confirmation
# Enter your password (it cannot be changed after)

cryptsetup luksFormat /dev/sda2

WARNING!

========

This will overwrite data on /dev/sda2 irrevocably.

Are you sure? (Type 'yes' in capital letters): YES

Enter passphrase for /dev/sda2: 
Verify passphrase: 

# Open an encrypted partition we just created, *system* is a friendly name

cryptsetup open /dev/sda2 system

# Create filesystems for boot and root partitions and mount them 
# Note that mkfs for root run on mapper device (a friendly name from previous step)

mkfs.fat -F32 /dev/sda1
mkfs.btrfs /dev/mapper/system

mount /dev/mapper/system /mnt
mkdir /mnt/boot
mount /dev/sda1 /mnt/boot
```

## Installation
The essential packages. If you get warnings about missing firmware modules, more info [mkinitcpio](https://wiki.archlinux.org/title/mkinitcpio)
```bash
basestrap -i /mnt base base-devel runit elogind-runit linux linux-firmware linux-firmware-qlogic grub networkmanager networkmanager-runit cryptsetup lvm2 lvm2-runit vim neovim
```

### Configure target system

```bash
# Press alt+right to open a new terminal session, login with credetinals provided by livecd 
# Chroot into new system
artix-chroot /mnt bash

# Set the timezone and sync your clock
ln -s /usr/share/zoneinfo/Europe/Berlin /etc/localtime
hwclock --systohc

# Set the locale
# Uncomment your language in /etc/locale.gen
# In this example, en_US
sed -i '/en_US/s/^#//g' /etc/locale.gen

echo 'export LANG="en_US.UTF-8"' >> /etc/locale.conf
echo 'export LC_COLLATE="C"' >> /etc/locale.conf

# Generate locales:
locale-gen

# Set the hostname, "crypt"
echo "crypt" > /etc/hostname

# Edit /etc/hosts for networking
cat <<EOT >> /etc/hosts
127.0.0.1    localhost
::1          localhost
127.0.1.1    crypt.localdomain  crypt
EOT

cat /etc/hosts

# Enable networkmanager to run on boot:
ln -s /etc/runit/sv/NetworkManager /etc/runit/runsvdir/current

# Set root password
passwd

# Create new user and set its password
useradd -G wheel -m username
passwd username

# Optional - make your user automatically login

vim /etc/runit/sv/agetty-tty1/conf
---
GETTY_ARGS="--noclear --autologin username"
---
```

### Generate fstab 
Get the UUIDs for grub and fstab, in a new livecd non-chroot session (alt+left/right)
```bash
lsblk -f >> /mnt/etc/default/grub
fstabgen -U /mnt >> /mnt/etc/fstab

# Now back to chrooted session
# Setup the decrypt on boot in 
# Add in the HOOKS string before the *filesystem*
vim /etc/mkinitcpio.conf
---
encrypt lvm2
---

# Generate new boot image:
mkinitcpio -p linux

# Edit the grub file:
# Change ```GRUB_CMDLINE_LINUX_DEFAULT``` line and put UUIDs there accordingly
# Lastly, remove the lsblk junk at the end of the file, or your system will not boot.
vim /etc/default/grub
---
GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quite cryptdevice=UUID=<encrypted-uuid-value-here>:system root=UUID=<decrypted-uuid-value-here>"
---


# Install grub
grub-install --root-directory=/mnt /dev/sda
grub-mkconfig -o /boot/grub/grub.cfg

# That's it, reboot without the USB stick and you should be ready to go
exit
reboot
```
