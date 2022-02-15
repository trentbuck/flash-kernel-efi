How did we get here?

1. EFI ESP must be FAT32 and cannot be md RAIDed meaning it is a SPOF.
   We can mitigate this by using a static, read-only ESP, identical on every computer:
   use refind instead of grub.

   As long as /refind_linux.conf sets "root=LABEL=foo"
   we can shove refind.img on any USB key or DVD, and
   refind will take care of the rest.

   If the ESP has a hardware fault, we can trivially replace it with no special tricks.
   (As opposed to trying to get grub-install to work correctly from a rescue medium).

   This works GREAT when / and /boot are ext4 or btrfs,
   as refind ships with EFI drivers for those.

   Nitpick: refind defaults to the vmlinuz-X with the
   newest mtime, so don't install kernels from BOTH
   buster and buster-backports, or it will occasionally
   prefer the non-backport version.

   Nitpick: ESP can be RAID1'd if your mainboard supports IMSM (Intel fakeraid).
   Once booted, Linux's md drivers can manage IMSM arrays using Linux code.
   The mainboard firmware's IMSM drivers will still be used at boot time.

   Nitpick: fwupd expects a read-write ESP:\ESP\debian\fw to upgrade the EFI firmware itself;
   it also needs considerable space (refind.img is a FAT12 with about 2MB unallocated).

2. If you give ZFS whole disks (not partitions),

   • ZFS can "autoreplace" a dead disk's replacement;
   • ZFS will block screwups like "dd of=/dev/sdd" when you meant sde;
   • you don't need to manually partition.

   For a server this means we put the ESP on a tiny
   read-only disk and leave all the regular hot-swap
   bays available for "real" disks.

   https://www.supermicro.com/products/nfo/SATADOM.cfm  (enterprise version)

   https://shop.westerndigital.com/products/usb-flash-drives/sandisk-ultra-fit-usb-3-1  (consumer version)

3. #1 and #2 would work neatly together, except that EFI ZFS drivers are limited.

   • refind has no drivers/zfs_x64.efi
   • efifs provides one; it is a recompile of grub's zfs.mod
   • grub zfs.mod (as at GRUB 2.02) cannot read from a pool using compression=zstd or encryption=on•ANYWHERE* in the pool.
   • efifs needs Visual Studio at compile time, so non-trivial to get into Debian

   To work around this, OpenZFS (rlaager) suggests making an entire separate ZFS pool "bpool" just for /boot.

   https://openzfs.github.io/openzfs-docs/Getting%20Started/Debian/Debian%20Buster%20Root%20on%20ZFS.html

   The advantage is if you rollback the OS, it (maybe!)
   also rolls back /boot --- but not atomically, because
   separate pools!  In any case, as at Debian 11
   bullseye, rollback support is still proof-of-concept quality:

   https://github.com/openzfs/zfs/blob/master/contrib/initramfs/README.initramfs.markdown

   The disadvantage is we need to

   • dedicate two ENTIRE DISKS to /boot, OR manually partition (losing autoreplace support); AND
   • use grub (refind is much nicer), OR package efifs for Debian.

4. So what if we just leave /boot on the main ZFS pool, but *copy* it into the ESP whenever it changes?

   This is https://github.com/trentbuck/flash-kernel-efi

   Then if the DOM has a hardware fault, the restore procedure is straightforward:

   • boot ZFS-capable live medium
   • make a new ESP on the new DOM
   • run flash-kernel-efi

   It works fine, but custom scripts being mission-critical for boot feels ugly.

   It also has a couple of minor problems:

   • it uses --exclude which is error-prone;

   • it uses ESP:\EFI\debian which is ALSO used by fwupd.
     Meaning it will break fwupd.
     Since this is only done to make the icons look nice,
     flash-kernel-efi should just be changed to use something else,
     and create a /boot/.VolumeIcon.png.

5. Instead of #4, what if we just mount the ESP at /boot, and do the copy in the other direction?

   That way the custom code is JUST a backup, and if it breaks we don't care unless the DOM also dies.

   This worked fine INITIALLY, but on a routine upgrade ran into https://bugs.debian.org/825945

   Since that is WONTFIX, we have to either do #4 or have a SECOND filesystem that's outside of the main ZFS pool:

   • ``/boot/efi`` being a FAT32 ESP (since that's the only thing guaranteed to have drivers in the mainboard's EFI firmware)

   • ``/boot``     being a filesystem for which Linux *AND* EFI drivers exist (i.e. not full-feature ZFS),
     AND supporting everything mentioned here:

     https://wiki.debian.org/Teams/Dpkg/FAQ#Q:_What_are_the_filesystem_requirements_by_dpkg.3F

     In other words, probably ext4.

   • make sure the bootloader has access to that EFI driver
     (i.e. can't just use refind_x64.efi anymore, have to also ship AT LEAST drivers/ext4_x64.efi or whatever).

This script (and the symlinks to it) implement #5:

• Backup the EFI ESP when:

  • a kernel is added/removed; or
  • a ramdisk is changed.

Known limitations:

• won't run when e.g. fwupd modifies /boot/efi/debian/fw
• won't run when refind or grub is reinstalled
