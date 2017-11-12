---
title: Migration to SSD and reconfig of filesystem setup
#author: Jake
date: '2017-10-15'
categories:
  - Laptop
tags:
  - SSD
  - Arch
slug: ssd-migration
---


# Migration to SSD and reconfig of filesystem setup
I was finally about to migrate to a SSD on my laptop when I started to think about switching to Btrfs and getting rid of LVM while I’m at it. 
To achieve that I tried different things but came down to the following. At this point I also had the SSD installed in my laptop already and 
the old HDD pluggable with SATA-to-USB adapter.

Basic idea: Via live usb, create /boot and LUKS container (with /) on SSD. Format / and /boot to Btrfs. Migrate data with rsync. Do the usual 
steps at the end of an arch installation (fstab, bootloader, mkinitcpio etc).

Detailed steps: (based on [ArchWiki:Dm-crypt-Encrypting an entire 
system](https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system#Btrfs_subvolumes_with_swap))

- `fdisk /dev/sdSSD` and create new MBR (DOS) table (because BIOS booting on my x220 doesn’t work with GPT). Create `/boot` partition for grub 
(in my case 350MiB, here `/dev/sdSSD1`) and `/` (rest of space, here `/dev/sdSSD2`), both of type linux file system. Mark `/boot` bootable.
- Setup LUKS container: (DISCLAIMER: Thats important, please read the arch wiki before deciding on your parameters: 
[ArchWiki:Dm-crypt-Encryption options for LUKS 
mode](https://wiki.archlinux.org/index.php/Dm-crypt/Device_encryption#Encryption_options_for_LUKS_mode) )
    - `cryptsetup --key-size 512 --hash sha512 --iter-time 5000 luksFormat /dev/sdSSD2`
    - Verify with `cryptsetup luksOpen /dev/sdSSD2 cryptroot`. Now that partition is also mapped and available for formatting continue with 
the following.
- Format `mkfs.btrfs -L root /dev/mapper/cryptroot` and mount `mount -o compress=lzo /dev/mapper/cryptroot /mnt`. More on compression: 
[ArchWiki:Btrfs](https://wiki.archlinux.org/index.php/Btrfs#Compression)
- Next is creating subvolumes which will replace classic partitions in this setup. I’m new to Btrfs and just know the basics. There is more 
information here: [ArchWiki:Dm-crypt-Encrypting an entire 
system](https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system#Creating_btrfs_subvolumes)
    - `btrfs subvolume create /mnt/@` and same with `/mnt/@snapshots` and `/mnt/@home`
 - Now with everything setup for `/` next step is copying the data.
    - Unmount `unmount /mnt` and mount new “real” root `mount -o compress=lzo,subvol=@ /dev/mapper/cryptroot /mnt`
    - Plug in old HDD. `cryptsetup luksOpen /dev/sdHDD2 oldroot` and `mkdir /mntOld` to `mount /dev/mapper/vg-root /mntOld` (my old setup had 
root, home and swap at this LVM layer).
    - Copy data from old disk to new. Attention: one little trailing `/` gave me some headache 
([1](https://serverfault.com/questions/529287/rsync-creates-a-directory-with-the-same-name-inside-of-destination-directory)) until I executed 
rsync the following way.
    - `rsync -aAXHs --numeric-ids --info=progress2 
--exclude={"/dev/*","/proc/*","/sys/*","/tmp/*","/run/*","/mnt/*","/media/*","/lost+found",”/var/cache/pacman/pkg/*”} /mntOld/ /mnt`
    - Almost same for /home: `umount /mnt` and same for `/mntOld`
    - `mount -o compress=lzo,subvol=@home /dev/mapper/cryptroot /mnt` and `mount /dev/mapper/vg-home /mntOld`
    - `rsync -aAXHs --numeric-ids --info=progress2 
--exclude={"/lost+found",”*/.thumbnails/*”,”*/.cache/mozilla/*”,”*/.cache/chromium/*”,”*/.local/share/Trash/*”,”*/.gvfs”} /mntOld/ /mnt`
- Preparing and installing bootloader to `/boot`.
    - Because I was using syslinux and learned while reading the wiki and documentation that it doesn’t support compressed Btrfs. I just 
installed GRUB as new bootloader. This seems to be a good idea anyway because Syslinux wasn’t updated in quite a while. In many situations 
keeping the bootloader might work just fine. But if you choose the apadt another one remember to migrate hooks etc too.
    - `umount /mnt` and `umount /mntOld`
    - `mkfs.btrfs -L boot /dev/sdSSD1`
    - Finally mount the almost ready system 
        - `mount -o compress=lzo,subvol=@ /dev/mapper/cryptroot /mnt`
        - `mount -o compress=lzo,subvol=@home /dev/mapper/cryptroot /mnt/home`
        - `mount -o compress=lzo,subvol=@snapshots /dev/mapper/cryptroot /mnt/.snapshots`
        - `mount -o compress=lzo /dev/sdSSD1 /mnt/boot`
        - Update filesystem table `genfstab -U /mnt >> /mnt/etc/fstab`. Modify with editor so old lines are commented or removed.
    - Config internet access on live system and chroot into new system `arch-chroot /mnt /bin/zsh`
        - Install grub package and install grub itself `grub-install --target=i386-pc /dev/sdSSD` (important to address whole disk here so 
GRUB can be written into MBR)
        - Edit `/etc/default/grub`: modify line to `GRUB_CMDLINE_LINUX="cryptdevice=/dev/sda2:cryptroot:allow-discards"` (discarding is 
necessary for SSD trimming later) and uncomment `GRUB_ENABLE_CRYPTODISK=y` line.
        - Edit `mkinitcpio.conf`: check or modify lines `MODULES=”btrfs”` and `BINARIES="/usr/bin/btrfs"`; check if `encrypt` hook and 
everything else needed is there. Remove `lvm2` hook.
        - Reinstall linux `pacman -S linux` to get necessary files and start mkinitcpio. Image generation must be successful!
        - Write changes `grub-mkconfig -o /boot/grub/grub.cfg` to GRUB. You should see messages of found images. 
    - Last but not least: config SSD behavior
        - TLP config: check or modify line to `SATA_LINKPWR_ON_BAT=max_performance`
        - LUKS parameter was given above already (`:allow-discards`).
        - Enable weekly trimming `systemctl enable fstrim.timer`.
    - `exit` chroot. Cleanly unmount everything `umount -R /mnt` and reboot.

Everything should work now. Additionally I removed `pacman -Rs hdapsd` shock protection for mechanical disk because that’s obviously not 
necessary anymore.

Last friendly disclaimer: The way I described above is just the way I fiddled for my specific situation. Sadly I’m just a linux enthusiast and 
not an expert so there might be design flaws or other errors included.
