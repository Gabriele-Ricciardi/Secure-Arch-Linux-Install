# Secure-Arch-Linux-Install
A complete guide to replicate my Arch Linux setup. It includes minimal partitioning, full disk encryption, secure boot, SELinux and more security features.

## Premise
1. Take a read, understand, investigate and take notes about the [official installation guide](https://wiki.archlinux.org/title/Installation_guide).
2. Identify your needs, whishes, and possible pain points. Arch Linux is extremely customisable, so you should approach it with a rough idea of what you want.
3. If relatively high security is a need or interest of yours, this guide will probably be of help.

## Pre-installation steps
### Prepare an installation medium
1. [Download](https://archlinux.org/download/) the latest Arch Linux ISO image from your favourite mirror.
2. [Download](https://archlinux.org/download/#checksums) and verify the GPG signature of the ISO image to ensure its integrity. Make sure to download the signature from the official Arch Linux website and not from the mirror website. 
   * In case the signatures do not match, double check that the detached signature file, .sig, refers to the same downloaded version, double check your commands, etc. In any case, do not proceed if you get an error at this step.
3. Prepare an installation medium, usually a [live USB](https://wiki.archlinux.org/title/USB_flash_installation_medium).

### Initial checks
1. Boot the live installation medium.
2. We are going to use EFI in order to use the secure boot features; thus, check the boot mode:
```
ls /sys/firmware/efi/efivars
``` 
If no errors are returned, the system is booted in UEFI mode.

3. Test the internet connection, e.g. `ping 1.1.1.1` and `ping archlinux.org`.
4. Enable clock time synchronisation: 
```
timedatectl set-ntp true
```
This will enable and start the `systemd-timesyncd.service`, which is the default Arch Linux/systemd network synchronisation service. You can check it with 
```
systemctl status systemd-timesyncd.service
```

### Drive preparation
[As suggested for drive encryption](https://wiki.archlinux.org/title/Dm-crypt/Drive_preparation), perform a secure erasure of the target hard disk drive.
1. Check and double check the name of the disk, e.g. `/dev/sda`. 
```
lsblk
```
2. Create a temporary encrypted container, "to_be_wiped", on the device: 
```
cryptsetup open --type=plain -d /dev/urandom /dev/sda to_be_wiped
```
3. Check that it does exist: 
```
lsblk
```
4. Wipe the container with zeroes (it will take a while): 
```
dd if=/dev/zero of=/dev/mapper/to_be_wiped status=progress bs=4M
```
5. Close the temporary container: 
```
cryptsetup close to_be_wiped
```

### Partition the disk
There are many possible partition configurations. I kept it as simple and minimal as possible, i.e. 1 EFI partition and 1 encrypted root partition containing the rest, including `/boot`. I did not include a swap partition. You can add it or add a swap file later on. Note that I did not use LVM.
Following is an example of partitioning using gdisk:
1. Check disks: 
```
gdisk -l /dev/sda
lsblk
```
2. Create a 500MB EFI partition and the root partition taking the rest of the disk:
```
gdisk /dev/sda
   o
   n
   (enter)
   0
   +500M
   ef00
   
   n
   (enter)
   (enter)
   (enter)
   8309
   w
```
### Format the partitions and mount the filesystem
1. Create the LUKS encrypted container in the root partition, e.g. `/dev/sda2`
```
cryptsetup luksFormat --type luks1 /dev/sda2
```
2. Check that it does exist
```
gdisk -l /dev/sda
```
3. Open the container
```
cryptsetup open /dev/sda2 cryptroot
```
The decrypted container is now available at `/dev/mapper/cryptroot`.

4. Format and mount the root partition
```
mkfs.ext4 /dev/mapper/cryptroot
mount /dev/mapper/cryptroot /mnt
```
5. Check that the mapping works as intended
```
umount /mnt
cryptsetup close cryptroot
cryptsetup open /dev/sda2 cryptroot
mount /dev/mapper/cryptroot /mnt
```
6. Format and mount the EFI partition
```
mkfs.fat -F 32 /dev/sda1
mkdir /mnt/efi
mount /dev/sda1 /mnt/efi
```
