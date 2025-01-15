# Base Install Packages
- linux-hardened: My preferred Kernel.
- intel-ucode: - microcode support.
- btrfs-progs: Necessary for working with Btrfs.
- dosfstools: Important if you are using a FAT filesystem for partitions like EFI.
- util-linux: A collection of essential utilities for Linux systems.
- unzip: To handle .zip files.
- sbctl: Secure Boot utility if you’re using UEFI with Secure Boot enabled.
- networkmanager: Useful for managing your network connections.
- sudo: Essential for user privileges management.
- git & kitty to setup my DE later.

## The Basic Technologies
[Authenticated Boot and Disk Encryption on Linux](https://0pointer.net/blog/authenticated-boot-and-disk-encryption-on-linux.html)

1. **LUKS/`dm-crypt`/`cryptsetup`**:  
   These provide disk encryption and optionally, data authentication. Disk encryption ensures that data can only be read in plaintext if you possess a secret, usually a password or passphrase. Data authentication prevents unauthorized modifications to disk data. Most distributions only enable encryption by default; authentication is a newer feature not widely adopted yet.  
   Closely related tools include:  
   - `dm-verity` (authenticates immutable volumes)  
   - `dm-integrity` (authenticates writable volumes).  

2. **UEFI Secure Boot**:  
   Secure Boot authenticates boot loaders and pre-OS binaries before execution. When the boot process uses similar authentication steps throughout, a chain of trust ensures only trusted code runs on the system. Authentication relies on cryptographic signatures, validated against certificates preloaded into most PCs (commonly signed by Microsoft).  

3. **TPMs (Trusted Platform Modules)**:  
   TPMs secure secrets, releasing them only if the boot code can be authenticated. This process involves:  
   - Hashing components of the boot process (code, configuration, etc.) and writing the hash values into TPM's write-once Platform Configuration Registers (PCRs).  
   - Releasing keys for decryption only if boot matches expected values.  
   TPMs enforce anti-hammering (limited unlock attempts) and utilize unique seed keys, making physical duplication of the chip nearly impossible.

---

## Disk Preparation

We'll use a 512MB FAT32 partition for EFI and a LUKS-encrypted `btrfs` partition for root. We’ll also use correct partition types for systemd detection.  

[Discoverable Partitions Specification](https://uapi-group.org/specifications/specs/discoverable_partitions_specification/)

---

1. **Prepare Disk**  
   Zap the drive:  
   ```
   $ sgdisk -Z /dev/nvme0n1
   ```

2. **Create Partitions**  
   Create a 512MB EFI partition and the root partition:  
   ```
   $ sgdisk -n1:0:+512M -t1:ef00 -c1:EFI -N2 -t2:8304 -c2:LINUXROOT /dev/nvme0n1
   ```

3. **Confirm Partitions**  
   ```
   $ partprobe -s /dev/nvme0n1
   ```

```
lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS

vda    254:0    0   30G  0 disk 
├─nvme0n1p1 254:1    0  512M  0 part
└─nvme0n1p2 254:2    0 469.5G  0 part
```

Looking good, next, let's encrypt our root partition with LUKS, using `cryptsetup`:
---

### Encrypt the Root Partition

Encrypt `/dev/nvme0n1p2` with LUKS:  
```
$ cryptsetup luksFormat --type luks2 /dev/nvme0n1p2
$ cryptsetup luksOpen /dev/nvme0n1p2 linuxroot
```

---

### Create Filesystems  

1. Format EFI and root partitions:  
   ```
   $ mkfs.vfat -F32 -n EFI /dev/nvme0n1p1
   $ mkfs.btrfs -f -L linuxroot /dev/mapper/linuxroot
   ```
Let's mount our partitions, and create our btrfs subvolumes.

2. Mount Partitions:  
   ```
   $ mount /dev/mapper/linuxroot /mnt
   $ mkdir /mnt/efi
   $ mount /dev/vda1 /mnt/efi
   $ btrfs subvolume create /mnt/home
   $ btrfs subvolume create /mnt/srv
   $ btrfs subvolume create /mnt/var
   $ btrfs subvolume create /mnt/var/log
   $ btrfs subvolume create /mnt/var/cache
   $ btrfs subvolume create /mnt/var/tmp
   ```

---

## Base Install  

1. Update mirrors:  
   ```
   $ reflector --country AU --age 24 --protocol http,https --sort rate --save /etc/pacman.d/mirrorlist
   ```
This command updates your mirror list to prioritize **Australian (AU)** mirrors, selecting only those updated within the last 24 hours and sorted by download rate.

2. Install base system:  
   ```
   $ pacstrap -K /mnt base base-devel linux-hardened linux-firmware intel-ucode vim nano cryptsetup btrfs-progs dosfstools util-linux git unzip sbctl kitty networkmanager sudo
   ```

---

### System Configuration

1. Update locales:  
   ```
   $ sed -i -e "/^#en_US.UTF-8/s/^#//" /mnt/etc/locale.gen
   $ arch-chroot /mnt locale-gen
   ```

2. Configure first boot:  
   ```
   $ systemd-firstboot --root /mnt --prompt
   ```

3. Create user and configure sudo:  
   ```
   $ arch-chroot /mnt useradd -G wheel -m yokoszn
   $ arch-chroot /mnt passwd yokoszn
   $ sed -i -e '/^# %wheel ALL=(ALL:ALL) NOPASSWD: ALL/s/^# //' /mnt/etc/sudoers
   ```

---

## UKIs + Secure Boot

We want to create [Unified Kernel Images](https://uapi-group.org/specifications/specs/unified_kernel_image/), so first, let's create our kernel cmdline file. This doesn't need to contain anything because we're using Discoverable Partitions, we just do it so `mkinitcpio` doesn't complain:

```
$ echo "quiet rw" >/mnt/etc/kernel/cmdline
```

Let's create the EFI folder structure:
```
$ mkdir -p /mnt/efi/EFI/Linux
```

Open `/mnt/etc/mkinitcpio.conf` in nano / vim .

We need to change the HOOKS in `mkinitcpio.conf` to use systemd, so make yours look like:

```
# vim:set ft=sh
 MODULES=()  
 
 BINARIES=()  
 
 FILES=()  
 
 HOOKS=(base systemd autodetect modconf kms keyboard sd-vconsole sd-encrypt block filesystems fsck)
```

And now let's update the .preset file, to generate a UKI:

Open `/mnt/etc/mkinitcpio.d/linux.preset` in nano / vim.
(or `/mnt/etc/mkinitcpio.d/linux-hardened.preset` if you have been following along )

comment out "default_image", "default_config" "fallback_image" & "fallback_config"

```
# mkinitcpio preset file for the 'linux-hardened' package

#ALL_config="/etc/mkinitcpio.conf"
ALL_kver="/boot/vmlinuz-linux-hardened"

PRESETS=('default' 'fallback')

#default_config="/etc/mkinitcpio.conf"
#default_image="/boot/initramfs-linux-hardened.img"
default_uki="/efi/EFI/Linux/arch-linux-hardened.efi"
default_options="--splash /usr/share/systemd/bootctl/splash-arch.bmp"

#fallback_config="/etc/mkinitcpio.conf"
#fallback_image="/boot/initramfs-linux-hardened-fallback.img"
fallback_uki="/efi/EFI/Linux/arch-linux-hardened-fallback.efi"
fallback_options="-S autodetect"
```

And now let's generate our UKIs:
```
$ arch-chroot /mnt mkinitcpio -P
```

Once that is complete, confirm they have been created

```
$ ls -lR /mnt/efi
```
## Example Result:

```
/mnt/efi:
total 4
drwxr-xr-x 3 root root 4096 Aug 25 20:51 EFI

/mnt/efi/EFI:
total 4
drwxr-xr-x 2 root root 4096 Aug 25 21:02 Linux

/mnt/efi/EFI/Linux:
total 118668
-rwxr-xr-x 1 root root 29928960 Aug  9 14:07 arch-linux.efi
-rwxr-xr-x 1 root root 91586048 Aug  9 14:07 arch-linux-fallback.efi
```

## Services and Boot Loader

OK, we're just about done in the archiso, we just need to enable some services, and install our bootloader:

```
$ systemctl --root /mnt enable systemd-resolved systemd-timesyncd NetworkManager 
$ systemctl --root /mnt mask systemd-networkd 
$ arch-chroot /mnt bootctl install --esp-path=/efi
```

OK, let's reboot, and then finish off the installation. Whilst you're rebooting, head into your UEFI/BIOS and put Secure Boot into "Setup Mode", you'll need to check with your PC/Motherboard manufacturer for exact details on how to do that:

```
$ sync
$ systemctl reboot --firmware-setup
```

Confirm Secure Boot is in setup mode / disabled: 
```
$ sbctl status
Installed:  ✓ sbctl is installed
Setup Mode: ✗ Enabled
Secure Boot:    ✗ Disabled
Vendor Keys:    none
```
Looks good. Let's first create and enroll our personal Secure Boot keys:

```
$ sudo sbctl create-keys 
$ sudo sbctl enroll-keys -m
```
We use the `-m` option to enroll the Microsoft vendor key along with our self-created platform key. If you're absolutely certain that none of your hardware has OPROMs signed by Microsoft, you can leave this option out. **WARNING**: Mistakes can potentially brick your system. Although it’s not ideal, it’s generally safer to include the Microsoft vendor key. _You have been warned!_

### Signing EFI Files with `sbctl`

Next, we’ll use `sbctl` to sign the required `.efi` files. These include the systemd-boot `.efi` file and the UKI files generated by `mkinitcpio`. Using the `-s` option ensures `sbctl` automatically resigns them whenever the kernel or bootloader is updated through `pacman`.

```
$ sudo sbctl sign -s -o /usr/lib/systemd/boot/efi/systemd-bootx64.efi.signed /usr/lib/systemd/boot/efi/systemd-bootx64.efi
$ sudo sbctl sign -s /efi/EFI/BOOT/BOOTX64.EFI
$ sudo sbctl sign -s /efi/EFI/Linux/arch-linux-hardened.efi
$ sudo sbctl sign -s /efi/EFI/Linux/arch-linux-hardened-fallback.efi
```
### Reinstall Kernel to Confirm Automatic Signing

To ensure everything is working as expected, reinstall the kernel. During this process, the UKI should be resigned automatically:

```
$ sudo pacman -S linux-hardened ...
```

Output confirmation example:
```
(4/4) Signing EFI binaries...
Generating EFI bundles....
✓ Signed /efi/EFI/Linux/arch-linux-fallback.efi
✓ Signed /efi/EFI/Linux/arch-linux.efi
```
### Configure Automatic Unlocking of Root Filesystem

After rebooting, configure the system to automatically unlock the root filesystem by binding a LUKS key to the TPM. This involves generating a new key, adding it to the volume for unlocking, and binding it to PCRs 0 and 7 (corresponding to the system firmware and Secure Boot state).

1. **Generate a Recovery Key**

First, create a recovery key for emergency use in case of future issues. Keep this key secure and hidden.

```
$ sudo systemd-cryptenroll /dev/gpt-auto-root-luks --recovery-key
```

---

2. **Enroll Firmware and Secure Boot State**

This step ensures the TPM can unlock the encrypted drive if the expected system firmware and Secure Boot state remain unchanged.  

```
$ sudo systemd-cryptenroll --tpm2-device=auto --tpm2-pcrs=0+7 /dev/gpt-auto-root-luks
```

For additional security, you can require a PIN during boot by including the `--tpm2-with-pin=yes` option:

```
$ sudo systemd-cryptenroll --tpm2-device=auto --tpm2-pcrs=0+7 /dev/gpt-auto-root-luks --tpm2-with-pin=yes
```

After rebooting, your system will be setup with Secure Boot, disk encryption, and TPM integration for automatic unlocking.

Consult the [Arch Wiki Hardware Pages](https://wiki.archlinux.org/title/Category:Laptops) for additional configuration tips tailored to your device, such as touchpad gestures, power management, or device-specific quirks.

---

## Resources:

- [Unified Extensible Firmware Interface/Secure Boot](https://archlinux.org/packages/?name=intel-media-driver)
- [Secure your boot process: UEFI + Secureboot + EFISTUB + Luks2 + lvm + ArchLinux](https://nwildner.com/posts/2020-07-04-secure-your-boot-process/)  
- [How is hibernation supported on machines with UEFI Secure Boot?](https://security.stackexchange.com/questions/29122/how-is-hibernation-supported-on-machines-with-uefi-secure-boot)  
- [Authenticated Boot and Disk Encryption on Linux](https://0pointer.net/blog/authenticated-boot-and-disk-encryption-on-linux.html)

---
