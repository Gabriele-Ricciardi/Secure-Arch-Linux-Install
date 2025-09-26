# Secure-Arch-Linux-Install
A collection of personal notes about my Arch Linux setup. It includes minimal partitioning, full disk encryption, secure boot, and more security features.

2025/09/25 update: Since the time of writing this notes, there have been some interesting changes in Arch Linux. [This new official guide](https://wiki.archlinux.org/title/Dm-crypt/Encrypting_an_entire_system#LUKS_on_a_partition_with_TPM2_and_Secure_Boot) is available and is quite similar to my original setup. Even though I use Arch (btw), I am not a fan of customisation. I like (good) stock systems. Indeed, I have changed much of my previous configuration to better adhere to some new standards shown in that guide (like [systemd GPT partition automounting](https://wiki.archlinux.org/title/Systemd#GPT_partition_automounting)).

What changed:

* To avoid rewriting the whole Arch Linux installation guide, I have severely trimmed down this notes to just important or interesting parts, referencing the relevant steps.
   * This means the guide is no longer a full guide but a collection of notes.
* Partitioning follows the discoverable partitions specifications, so we can use systemd automounting.
   * This means setting the correct partitions type GUID so `fstab` is not needed anymore.
* No more `GRUB`. I use `systemd-boot` instead.
   * This means no more encryption password double prompt problem, so no more keyfile.
 * Secure Boot configuration now uses `systemd-ukify`, and not [`cryptboot`](https://github.com/xmikos/cryptboot) (which I had long since replaced with `sbctl` anyway).
   * This means the AUR is not required anymore for this setup.
 * I now use Plymouth
   * This makes the booting process look very nice!
 * I have uploaded some of my configuration files

## Premise
1. Take a read, understand, investigate and take notes about the [official installation guide](https://wiki.archlinux.org/title/Installation_guide).
2. Identify your needs, whishes, and possible pain points. Arch Linux is extremely customisable, so you should approach it with a rough idea of what you want.
3. If you have an interest in security and want to try out some features, you may find this notes of interest.

## Pre-installation
* Make an effort and [verify](https://wiki.archlinux.org/title/Installation_guide#Verify_signature) the ISO GPG signature.
* [Verify](https://wiki.archlinux.org/title/Installation_guide#Verify_the_boot_mode) the boot mode. This is important as UEFI is needed for Secure Boot.
* To partition the disks, follow [the new guide mentioned above](https://wiki.archlinux.org/title/Dm-crypt/Encrypting_an_entire_system#LUKS_on_a_partition_with_TPM2_and_Secure_Boot).
    * My advice here is to stick to the guide to benefit from Systemd automounting.
    * Yes, you should perform a [secure erasure](https://wiki.archlinux.org/title/Dm-crypt/Drive_preparation) of the drive when preparing your disk. "Cyber Security" doesn't sound so sexy anymore, huh?

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
* Say your prayers to the Omnissiah and reboot. The rest can be done without the installation live environment.

## Plymouth
* This is purely cosmetic, but it is easy to setup and makes the whole early booting process much nicer to look at and interact with.
* Basically [read and follow the relevant page](https://wiki.archlinux.org/title/Plymouth).
* Remember to apped
  * `splash` (and `quiet`) to the kernel parameters (e.g. `/etc/cmdline.d/standard.conf`)
  * `plymouth` in the initramfs generator configuration (i.e. `/etc/mkinitcpio.conf` hooks, right after `systemd` hook).
* Select a theme!
  * There are many and very cool looking! But I like stock stuff, so I went for `bgrt`.
  * `sudo plymouth-set-default-theme -R bgrt`
  * To be fair, I like the smooth transition: OEM logo -> OEM logo + arch logo at the bottom -> password prompt + arch logo at the bottom -> GDM with arch logo at the bottom

## Secure Boot

* For configuring secure boot I have used the quite recent (`systemd v257+`) [`systemd-ukify` configuration](https://wiki.archlinux.org/title/Unified_Extensible_Firmware_Interface/Secure_Boot#Assisted_process_with_systemd), and not `sbctl`.
* [**READ THIS VERY IMPORTANT WARNING**](https://wiki.archlinux.org/title/Unified_Extensible_Firmware_Interface/Secure_Boot#Using_your_own_keys)
  * I had no problem with my (old) Thinkpad E560, using the BIOS/UEFI menu to remove the platform key, but be warned.
* Follow the instructions and just take care to:
  * Set a (strong) firmware Supervisor Password
  * Put the firmware to Setup Mode properly
  * Manually sign the boot loader
  * Add the [recommended pacman hook](https://wiki.archlinux.org/title/Systemd-boot#Signing_for_Secure_Boot).
    * `/path/to/keyfile.key` = `/etc/kernel/secure-boot-private-key.pem`
    * `/path/to/certificate.crt` = `/etc/kernel/secure-boot-certificate.pem`
  * Configure the ESP for auto-enrollment
  * set `secure-boot-enroll force` in `/boot/loader/loader.conf` and reboot to enroll the keys in the firmware.
    * I have later commented out this option, just in case.
  * Actually enable Secure Boot and test it.
    * A successful boot is already a win, but also check with `sudo bootctl status`

## TPM

Currently [there is a bug](https://github.com/systemd/systemd/pull/39089) with older firmware (TCG < 1.1) that do not support `GetPcrBanks()`. Thus, on my Lenovo Thinkpad E560, `systemd` does not recognize the TPM 2.0. Once this is fixed I will update this part.

## User configuration
The system is now able to boot and is fully functional. However, you may want to set a non root user account. Also, you should go through the [General Recommendations.](https://wiki.archlinux.org/title/General_recommendations) 

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
Now it is a good time to [install](https://wiki.archlinux.org/title/General_recommendations#Graphical_user_interface) the graphical part of the system, such as the needed drivers and desktop environment.

As a general recommendation, I suggest avoiding the temptation (and hassle) to set all the graphical user interface manually, as I did before, and simply pick a desktop environment. You may have noticed that I like (good) stock stuff, so my choice is obviously [GNOME](https://wiki.archlinux.org/title/GNOME).

## Ideas for future projects
I would like to implement `apparmor` and other in-system security features. SELinux is cool, but it is not officially supported and requires a big reliance on the AUR, which I prefer to use as little as possible.

## In Case of Emergency - Rescuing a non bootable system
Yes, this may happen. In fact, it happened very recently to me with [this upstream bug](https://github.com/systemd/systemd/issues/38932).

My honest advice is to keep a dedicated USB with an Arch Linux bootable live environemnt (e.g. your live installation medium itself) in your bag or headphones case. Maybe update it once a year. The procedure to get into your unbootable system is as follows:
* Get into the BIOS/UEFI and disable secure boot
* Choose to boot from a temporary device, and boot the USB
* Once inside the live environment:
  * `lsblk` and identify the root (e.g. `sda2`) and ESP partition (e.g. `sda1`)
  * `cryptsetup open /dev/sda2 root` to open the encrypted partition
  * `mount /dev/mapper/root /mnt` to mount the unlocked root partition
  * `mount /dev/sda1 /mnt/boot` to mount the ESP partition
  * `arch-chroot /mnt`
* You are now inside your normal installation.
* _When we embark on a journey, we often discover more of ourselves than of the world_
