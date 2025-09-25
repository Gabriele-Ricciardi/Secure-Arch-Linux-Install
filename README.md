# Secure-Arch-Linux-Install
A collection of personal notes about my Arch Linux setup. It includes minimal partitioning, full disk encryption, secure boot, and more security features.

2025/09/25 update: Since the time of writing this notes, there have been some interesting changes in Arch Linux. [This new official guide](https://wiki.archlinux.org/title/Dm-crypt/Encrypting_an_entire_system#LUKS_on_a_partition_with_TPM2_and_Secure_Boot) is available and is quite similar to this setup. Even though I use Arch (btw), I am not a fan of customisation. I like (good) stock systems. Indeed, I have changed some of my configurations in this guide to better adhere to some new standards shown in that guide (like [systemd GPT partition automounting](https://wiki.archlinux.org/title/Systemd#GPT_partition_automounting)).

To avoid rewriting the whole Arch Linux installation guide, I have severely trimmed down this notes to just important or interesting parts, referencing the relevant steps.

## Premise
1. Take a read, understand, investigate and take notes about the [official installation guide](https://wiki.archlinux.org/title/Installation_guide).
2. Identify your needs, whishes, and possible pain points. Arch Linux is extremely customisable, so you should approach it with a rough idea of what you want.
3. If you have an interest in security and want to try out some features, you may find this notes of interest.

## Pre-installation
* Make an effort and [verify](https://wiki.archlinux.org/title/Installation_guide#Verify_signature) the ISO GPG signature.
* [Verify](https://wiki.archlinux.org/title/Installation_guide#Verify_the_boot_mode) the boot mode. This is important as UEFI is needed for Secure Boot.
* To partition the disks, follow [the new guide mentioned above](https://wiki.archlinux.org/title/Dm-crypt/Encrypting_an_entire_system#LUKS_on_a_partition_with_TPM2_and_Secure_Boot).
    * Yes, you should perform a [secure erasure](https://wiki.archlinux.org/title/Dm-crypt/Drive_preparation) of the drive when preparing your disk. "Cyber Security" doesn't sound so sexy anymore, huh?
    * My advice here is to stick to the guide to benefit from Systemd automounting.

## Installation
At this point the partitions have been created and formatted and are ready for the actual system install.

* The following list of essential packages is just for personal reference, but is a good starting point. Note the [microcode package](https://wiki.archlinux.org/title/Microcode).
```
pacstrap /mnt base base-devel linux linux-firmware networkmanager man-db man-pages texinfo vim intel-ucode
```

## Configure the system

* If you did follow the [discoverable partitions specification](https://uapi-group.org/specifications/specs/discoverable_partitions_specification/), as in the guide, you can skip generating the file system table (fstab).
  * Have you considered already what to do with those extra 262 bytes of yours?
* Remember to set the root password (`passwd`)
   * You know the drill about passwords. Nonetheless, just be reminded that in case the system is expected to offer any kind of networking service, such as `SSH` connections, extra care should be applied when considering a root password complexity and length.
* I recommend installing `systemd-boot`, as [in the guide](https://wiki.archlinux.org/title/Dm-crypt/Encrypting_an_entire_system#Installing_the_boot_loader).
* Double check that the initramfs generation was successful (`mkinitcpio -P`).



# The following part is yet to be updated

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
