# arch-box
Fresh arch-minimal install for ThinkPad (or wherever)


## Preparations
Obtain iso here https://artixlinux.org/download.php

Burn it to usb stick (I recommend [Ventoy](https://www.ventoy.net/en/index.html) - a tool for multi boot drives)

### Wirless network setup 
More info: [Arch wiki](https://wiki.archlinux.org/title/ConnMan)

Check that wifi unblocked and unblock if neccessary

```
rfkill list
sudo rfkill unblock wifi
```
To connect to protected network we have to use interactive mode
```
connmanctl
```
Scan for any networks around:
```
connmanctl> scan wifi
```
Get the scan results:
```
connmanctl> services
```
Turn on agent to handle user requests:
```
connmanctl> agent on
```
Connect to your network (you have to use service name, tab autocompletion works):
```
connmanctl> connect wifi_dc85de828967_38303944616e69656c73_managed_psk
```
Provide the password and wait for network response:
```
Agent RequestInput wifi_dc85de828967_38303944616e69656c73_managed_psk
  Passphrase = [ Type=psk, Requirement=mandatory ]
  Passphrase?  *******
Connected wifi_dc85de828967_38303944616e69656c73_managed_psk
```
Quit, network should work now
```
connmanctl> quit
```

### Drive partitioning
Check your drives
```
lsblk
```
In most cases fdisk will work. You can use *parted*, *cfdisk*, etc
Replace *sda* with your drive (sdb, nvme01, etc). Run fdisk as root or with *sudo*
```
# fdisk /dev/sda

Welcome to fdisk (util-linux 2.38.1).
Changes will remain in memory only until you decide to write them.
Be careful before using the write command.
```
Delete partitions if neccessary. In the example below 2 partitions deleted and new are created.
```
Command (m for help): d
Partition number (1,2, default 2): 2

Partition 2 has been deleted.

Command (m for help): d
Selected partition 1
Partition 1 has been deleted.
```
Create new partition for /boot:
```
Command (m for help): n
Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)
```
Press enter for default (primary)
```
Select (default p): 
```
Press enter for default (1)
```
Partition number (1-4, default 1): 
```
Press enter for default first sector:
```
First sector (2048-2097151, default 2048):
```
Type +1G for last sector to make 1GB partition
```
Last sector, +sectors or +size{K,M,G,T,P} (2048-2097151, default 2097151): +1G
```
Type Y for confirmation if sector contains another filesystem signature
```
Partition #1 contains an ext4 signature.
Do you want to remove the signature? [Y]es/[N]o: Y
```
Encrypt your root partition, more info: [dm-crypt/Device encryption](https://wiki.archlinux.org/title/dm-crypt/Device_encryption)
```
cryptsetup luksFormat /dev/sda2
```
Type YES in capital letters for confirmation
```
WARNING!

========

This will overwrite data on /dev/sda2 irrevocably.

Are you sure? (Type 'yes' in capital letters): YES
```
Enter your password (it cannot be changed after)
```
Enter passphrase for /dev/sda2: 
Verify passphrase: 
```
Open an encrypted partition we just created, *system* is a friendly name and you can change that:
```
cryptsetup open /dev/sda2 system
```
Create filesystems for boot and root partitions. Note that mkfs for root run on mapper device (a friendly name from previous step):
```
mkfs.fat -F32 /dev/sda1
mkfs.btrfs /dev/mapper/system
```
Mount the drives
```
mount /dev/mapper/system /mnt
mkdir /mnt/boot
mount /dev/sda1 /mnt/boot
```

## Installation
The essential packages. If you get warnings about missing firmware modules, more info here: https://wiki.archlinux.org/title/mkinitcpio
```
basestrap -i /mnt base base-devel runit elogind-runit linux linux-firmware linux-firmware-qlogic grub networkmanager networkmanager-runit cryptsetup lvm2 lvm2-runit vim neovim
```

### Chroot into your system
Press alt+right to open a new terminal session, login with credetinals on your screen and chroot into new system
```
artix-chroot  /mnt bash
```
Set the timezone and sync your clock
```
ln -s /usr/share/zoneinfo/Europe/Berlin /etc/localtime
hwclock --systohc
```
Set the locale. Create a file:
```
vim /etc/locale.conf
```
Add the following:
```
export LANG="en_US.UTF-8"
export LC_COLLATE="C"
```
Uncomment your language in:
```
vim /etc/locale.gen
```
Generate locales:
```
locale-gen
```
Set the hostname:
```
echo "crypt" > /etc/hostname
```
Add following to /etc/hosts
```
vim /etc/hosts
```
Enable networkmanager to run on boot:
```
ln -s /etc/runit/sv/NetworkManager /etc/runit/runsvdir/current
```

Set root password
```
passwd
```
Create new user and set its password
```
useradd -G wheel -m username
passwd username
```
_Optional_ make your user automatically login
```
vim /etc/runit/sv/agetty-tty1/conf
GETTY_ARGS="--noclear --autologin username"
```

Setup the decrypt on boot
```
vim /etc/mkinitcpio.conf
```
Add in the HOOKS string before the *filesystem*
```
encrypt lvm2
```
Generate new boot image:
```
mkinitcpio -p linux
```
Get the UUIDs. Return to the livecd session (alt+left) and run:
```
lsblk -f >> /mnt/etc/default/grub
```
In the same session, generate fstab:
```
fstabgen -U /mnt >> /mnt/etc/fstab
```
Go back to chroot (alt+right) and edit the grub file:
```
vim /etc/default/grub
```
Edit ```GRUB_CMDLINE_LINUX_DEFAULT``` line and put UUIDs there accordingly:
```
GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quite cryptdevice=UUID=<encrypted-uuid-value-here>:system root=UUID=<decrypted-uuid-value-here>"
```
Remove the lsblk junk at the end of the file, or your system will not boot.

Once more return to livecd session (alt+left). Install grub:
```
grub-install --root-directory=/mnt /dev/sda
grub-mkconfig -o /boot/grub/grub.cfg
```
That's it, reboot without the USB stick and you should be ready to go

## Post-install
### Connect to Wi-Fi using [NetworkManager](https://wiki.archlinux.org/title/NetworkManager)
```
nmcli device wifi list
nmcli device wifi connect <SSID> password <password>
```
Run the pacman to syncrhonise the databases (yy is forcing re-download the package database):
```
pacman -Syyu
```
Try to install something useful:
```
pacman -S git
```
