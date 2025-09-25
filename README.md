# Secure-Arch-Linux-Install
A collection of personal notes about my Arch Linux setup. It includes minimal partitioning, full disk encryption, secure boot, and more security features.

2025/09/25 update: Since the time of writing this notes, there have been some interesting changes in Arch Linux. [This new official guide](https://wiki.archlinux.org/title/Dm-crypt/Encrypting_an_entire_system#LUKS_on_a_partition_with_TPM2_and_Secure_Boot) is available and is quite similar to this setup. Even though I use Arch (btw), I am not a fan of customisation. I like (good) stock systems. Indeed, I have changed some of my configurations in this guide to better adhere to some new standards shown in that guide (like [systemd GPT partition automounting](https://wiki.archlinux.org/title/Systemd#GPT_partition_automounting)).

To avoid rewriting the whole Arch Linux installation guide, I have severely trimmed down this notes to just important or interesting parts, referencing the relevant steps.

## Premise
1. Take a read, understand, investigate and take notes about the [official installation guide](https://wiki.archlinux.org/title/Installation_guide).
2. Identify your needs, whishes, and possible pain points. Arch Linux is extremely customisable, so you should approach it with a rough idea of what you want.
3. If you have an interest in security and want to try out some features, you may find this notes of interest.

## Pre-installation
1. Make an effort and [verify](https://wiki.archlinux.org/title/Installation_guide#Verify_signature) the ISO GPG signature.
2. [Verify](https://wiki.archlinux.org/title/Installation_guide#Verify_the_boot_mode) the boot mode. This is important as UEFI is needed for Secure Boot.
3. To partition the disks, follow [the new guide mentioned above](https://wiki.archlinux.org/title/Dm-crypt/Encrypting_an_entire_system#LUKS_on_a_partition_with_TPM2_and_Secure_Boot).
    * Yes, you should perform a [secure erasure](https://wiki.archlinux.org/title/Dm-crypt/Drive_preparation) of the drive when preparing your disk. "Cyber Security" doesn't sound so sexy anymore, huh?
    * My advice here is to stick to the guide to benefit from Systemd automounting.

## Installation
At this point the partitions have been created and formatted and are ready for the actual system install.

### Install essential packages
The following list is just for personal reference, but is a good starting point. Note the [microcode package](https://wiki.archlinux.org/title/Microcode).
```
pacstrap /mnt base base-devel linux linux-firmware networkmanager man-db man-pages texinfo vim intel-ucode
```

### Generate the File System Table
Generate the fstab file to preserve your file system layout.
```
genfstab -U /mnt >> /mnt/etc/fstab
```
Review the file for any misconfiguration.
```
cat /mnt/etc/fstab
```
You should see that `/dev/mapper/cryptroot` is mounted as `/`, and has `ext4` formatting, while `/dev/sda1` (in standard configurations) is mounted as `/efi`, and has `vfat`/`fat32` formatting.

## Configure the system
Now the base system is installed, but is not fully working and needs some configuration.
### Change root
Chroot into the newly installed system
```
arch-chroot /mnt
```
Now your base root will be `/mnt`, so that it will be like "normally" using your new system. Take in mind this `live USB boot` - `partition mount` - `change root` procedure, as this is the standard recovery procedure in case the system becomes unbootable.

### Set time zone
Set the time zone. In my case it will be Rome time zone.
```
ln -sf /usr/share/zoneinfo/Europe/Rome /etc/localtime
```

Set the Real Time Clock (RTC) from the system time (usually UTC).
```
hwclock --systohc
```
In case you are dual booting with Windows, read [here](https://wiki.archlinux.org/title/System_time#UTC_in_Microsoft_Windows).

### Localisation
Edite the locale file and choose the needed locale. In my case `en_GB.UTF-8 UTF-8`.
```
vim /etc/locale.gen
```
Generate the locale.
```
locale-gen
```
Create the `locale.conf` file and set the `LANG` variable. In my case `LANG=en_GB.UTF-8`
```
vim /etc/locale.conf
```

### Network configuration
Create the configuration file and choose a name for the hostname by simply writing it into the file, e.g. `myArch`.
```
vim /etc/hostname
```
Set the hosts in `/etc/hosts`. The file should look something like this:
```
127.0.0.1 localhost
127.0.1.1 myArch
::1       localhost
```
You may want to enable the `NetworkManager` service, so that you do not need to start it manually at each boot.
```
systemctl enable NetworkManager.service
```

### Set the root password
Set the root password.
```
passwd
```
You know the drill about passwords. Anyway, just be reminded that in case the system is expected to offer any kind of networking service, such as `SSH` connections, extra care should be applied when considering a root password complexity and lenght.

### Install the bootloader
This is where things get more interesting. I recommend a critical read of the [Archwiki's GRUB - UEFI page](https://wiki.archlinux.org/title/GRUB#UEFI_systems). Yes, that is kind of a rabbit hole of manuals, but that is how you learn things. Also, you should have already done this.

Install the required packages
```
pacman -S grub efibootmgr
```
Setup the initial ramdisk environment by modifying the `HOOKS` variable in the `/etc/mkinitcpio.conf` file, so that it will support encryption. An example is as follows:
```
HOOKS=(base udev autodetect keyboard consolefont modconf block encrypt filesystems fsck)
```

Configure GRUB to allow booting from `/boot` on a LUKS1 encrypted partition. Do it by setting in `/etc/default/grub`
```
...
GRUB_ENABLE_CRYPTODISK=y
...
GRUB_CMDLINE_LINUX=" ... cryptdevice=UUID=device-UUID:cryptroot root=/dev/mapper/cryptroot ... "
...
```
Where `device-UUID` is your encrypted device UUID. It can be retrieved by
```
lsblk -dno UUID /dev/sda2
```

Now recreate the initramfs image
```
mkinitcpio -P
```
Install GRUB
```
grub-install --target=x86_64-efi --efi-directory=/efi --bootloader-id=GRUB --recheck
```
> Note: I had some problems reinstalling GRUB with this configuration while chroot-ing into the system. GRUB would complain that it could not find the `/boot` partition mount point, although `/boot` was a simple directory inside `/`. I will investigate further about it, but in the meantime, as a workaround, simply exit from chroot-ing and install GRUB from outside the system. For example, with the previous configuration:
```
grub-install --target=x86_64-efi --efi-directory=/mnt/efi --boot-directory=/mnt/boot --bootloader-id=GRUB --recheck
```
> Remember to modify as above the `/etc/default/grub` file also in the external live USB system.

Generate GRUB configuration file.
```
grub-mkconfig -o /boot/grub/grub.cfg
```

### Install microcode updates

Install the [microcode](https://wiki.archlinux.org/title/microcode) updates relative to your CPU manufacturer to ensure system stability and security. So, either Intel
```
pacman -S intel-ucode
```
or AMD.
```
pacman -S amd-ucode
```

### Avoid encryption password double prompt
With the current setup, at boot you will be prompted twice for the encryption password. This happens because the first time it is asked by GRUB to unlock the LUKS1 encrypted partition, `cryptroot`, the second time for the initramfs. This section explains how to configure the system to ask the password only once at boot, in GRUB. The way this works is by embedding a keyfile in the initramfs. This procedure is taken from the relative [Archwiki article](https://wiki.archlinux.org/title/dm-crypt/Encrypting_an_entire_system#Avoiding_having_to_enter_the_passphrase_twice).

Create a keyfile and add it as LUKS key
```
dd bs=512 count=4 if=/dev/random of=/root/cryptroot.keyfile iflag=fullblock
chmod 000 /root/cryptroot.keyfile
cryptsetup -v luksAddKey /dev/sda2 /root/cryptroot.keyfile
```
> Note for future investigation: perhaps substitute `/dev/random` with `/dev/urandom`. The second one is the one generally suggested.

Add the keyfile to the initramfs image by modifying `/etc/mkinitcpio.conf`, adding
```
FILES=(/root/cryptroot.keyfile)
```
Recreate the initramfs image
```
mkinitcpio -P
```

Secure the embedded keyfile
```
chmod 600 /boot/initramfs-linux*
```

Add (i.e. append) the keyfile option in `/etc/default/grub`
```
GRUB_CMDLINE_LINUX="... cryptkey=rootfs:/root/cryptroot.keyfile"
```
Re-generate the `grub.cfg` file
```
grub-mkconfig -o /boot/grub/grub.cfg
```

## User configuration
The system is now able to boot and is fully functional. However, you might want to set a non root user account.

### Install your favourite shell
Install a different shell, if you wish so. I prefer `zsh`
```
pacman -S zsh
```

### Setup non root user
Add a new user with home directory, the needed groups (depending on your setup) and the chosen default shell. Then setup a password
```
useradd -m -G <the_needed_groups> -s /usr/bin/zsh <username>
passwd <username>
```

### Setup sudo
In a personal computer, usually, this user will be your main user. You may want to use [sudo](https://wiki.archlinux.org/title/sudo) to grant the user the capabilities to manage the computer as root when needed.

Sudo supports fine-grained configurations and checks to decide which user can use higher privileges and when. This is especially important in shared machines. In this case the non root user will have all the privileges as the root user, but consider adjusting this configuration depending on your use case.

Install `sudo`
```
pacman -S sudo
```

Grant root privileges to the main user by modifying the file `/etc/sudoers` using
```
visudo
```
and adding a line
```
<username>  ALL=(ALL:ALL) ALL
```

### Installing the desktop environment
Now it is a good time to [install](https://wiki.archlinux.org/title/General_recommendations#Graphical_user_interface) the graphical part of the system, such as the needed drivers and desktop environment. I will expand more upon this in the near future, once the more security-related things are covered.

As a general recommendation, I suggest avoiding the hassle to set all the graphical user interface manually, as I did before, and simply pick a desktop environment. I switched to [GNOME](https://wiki.archlinux.org/title/GNOME).

### Secure boot
With the current encryption setup, the kernel image and the initramfs cannot be modified by an attacker when the data is at rest. However, there is one single file that remains unencrypted and unprotected: `/efi/EFI/GRUB/grubx64.efi`. This file is indeed the bootloader. Because of this, the system is vulnerable to [Evil Maid attacks](https://www.schneier.com/blog/archives/2009/10/evil_maid_attac.html).

Evil maid attacks are relatively sophisticated and complex, and require (temporary) physical access to the target machine. However, it is still interesting to mitigate such vulnerability. A possible solution is using UEFI Secure Boot, and [`cryptboot`](https://github.com/xmikos/cryptboot) makes its setup simple enough.

Start by booting into your machine UEFI firmware setup utility. There, enable Secure Boot. In order to enroll your own keys into the firmware, Secure Boot needs to be in Setup Mode. In order to enter in Setup Mode, clear all preloaded Secure Boot keys. Also, setup the UEFI supervisor password, so that no unauthorised party can boot into the UEFI setup utlity and disable Secure Boot.

Install [`yay`](https://github.com/Jguer/yay#installation) or a similar package manager that handles the AUR packages and then install `cryptboot`
```
yay -S cryptboot
```
Generate your new UEFI Secure Boot keys
```
cryptboot-efikeys create
```
Enroll the keys into the UEFI firmware
```
cryptboot-efikeys enroll
```
`cryptboot` [does not support](https://github.com/xmikos/cryptboot/issues/2) single encrypted partition setups, so it is not possible to simply run the `cryptboot update-grub` command to update and sign the bootloader. Instead, do it manually:
```
grub-mkconfig -o /boot/grub/grub.cfg
grub-install --target=x86_64-efi --boot-directory=/boot --efi-directory=/efi --bootloader-id=GRUB

sudo sed -i ‘s/SecureBoot/SecureB00t/’ /efi/EFI/GRUB/grubx64.efi
cryptboot-efikeys sign /efi/EFI/GRUB/grubx64.efi
```
> Note: if you have just recently installed your system, you can skip the first two commands.

> [More info](https://wejn.org/2021/09/fixing-grub-verification-requested-nobody-cares/) about _that sed_

> Use this commands to update GRUB (i.e. after a GRUB package upgrade, when you are told to run `grub-install` to use the new features included with the update).

Reboot into the UEFI firmware setup utility and verify that Secure Boot is still active. Then reboot and check that you are able to normally boot into your system.
