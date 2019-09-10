**Boot off the CD or USB Drive**

**Obtain Internets**

Use the regular installer through the network setup... or...

```
# ip link set wlan0 up
# iwconfig wlan0 essid "[SSID Name]"
# dhcpcd wlan0
```

... or if there's encryption... 

```
# ip link set wlan0 up
# wpa_supplicant -B -i wlan0 -c <(wpa_passphrase ARPANET sw0rdfish)
# dhcpcd wlan0
```


**Initial Steps and Hard Drive Setup**
```
# pacman -Sy gptfdisk btrfs-progs
```

```
# gdisk /dev/sda
```

Konami Code: o, y, n, enter, enter, +512M, ef00, n, enter, enter, enter, enter, w, y.

Adjust as necessary, but this will leave you with a 512Mb formatted EFI partition. Be sure to use the right device located in `dev`. This could be something like `/dev/nvme0n1` if you're using an NVME type SSD.

For the rest of the document, pay attention to partitions. This guide utilizes two partitions. They may be identified like `/dev/sda1` and `/dev/sda2` or they could be identified like `/dev/nvme0n1p1` and `/dev/nvme0n1p2`.

**BTRFS Setup**
```
#!/bin/sh

mkfs.vfat -F 32 /dev/sda1
mkfs.btrfs -L Arch /dev/sda2
mkdir /btrfs
mount -o defaults,noatime /dev/sda2 /btrfs
btrfs sub create /btrfs/__active
## btrfs subvolume create /btrfs/__active/home
## btrfs subvolume create /btrfs/__active/usr
## btrfs subvolume create /btrfs/__active/var
mount -o subvol=__active /dev/sda2 /mnt
mkdir /btrfs/boot /mnt/boot
mount /dev/sda1 /btrfs/boot
mount --bind /btrfs/boot /mnt/boot
mkdir -p /mnt/var/lib/Arch /btrfs/__snapshot
chmod -R 0755 /btrfs/__active
```

The above commented sections are because BTRFS is yet to support recursive snapshots, meaning if you took a snapshot of __active and tried to boot off of it, arch would not be able to see home, usr, or var because they are snapshots on their own. You could snapshot just those directories and replace those at will manually if you choose to, but without the necessity of a separate home partition, there is no real point. I'll just snapshot the whole thing.

**System Preparation and Installation**

```
#!/bin/sh

mkdir /mnt/{proc,dev,sys,var/lib/pacman}
pacstrap /mnt base base-devel btrfs-progs zsh vim git
mount -o bind /dev /mnt/dev
mount -t sysfs none /mnt/sys
mount -t proc none /mnt/proc
```

**EFI System Partition**

Check "/sys/firmware/efi" to see if it exists. If this directory does not exist, you're not on and EFI boot. You must be on EFI for this to work. Additionally, run efibootmgr to ensure that you do not have any garbage entries from previous installs. If you do, you can remove them with `efibootmgr -B -b 001A`. Replace the hex as appropriate. You can view old entries with `efibootmgr -v`.

```
# echo "initrd=\initramfs-linux-fallback.img root=/dev/sda2 rootflags=subvol=__active ro" | iconv -f ascii -t ucs2 > bootfallback
# efibootmgr --create --gpt --disk /dev/sda --part 1  --label "Arch Linux Fallback" --loader '\vmlinuz-linux' --append-binary-args bootfallback
# echo "initrd=\initramfs-linux.img root=/dev/sda2 rootflags=subvol=__active ro" | iconv -f ascii -t ucs2 > boot
# efibootmgr --create --gpt --disk /dev/sda --part 1  --label "Arch Linux" --loader '\vmlinuz-linux' --append-binary-args boot
```

Or if you're on an NVME storage device:

```
# echo "initrd=\initramfs-linux-fallback.img root=/dev/nvme0n1p2 rootflags=subvol=__active ro" | iconv -f ascii -t ucs2 > bootfallback
# efibootmgr --create --gpt --disk /dev/nvme0n1 --part 1  --label "Arch Linux Fallback" --loader '\vmlinuz-linux' --append-binary-args bootfallback
# echo "initrd=\initramfs-linux.img root=/dev/nvme0n1p2 rootflags=subvol=__active ro" | iconv -f ascii -t ucs2 > boot
# efibootmgr --create --gpt --disk /dev/nvme0n1 --part 1  --label "Arch Linux" --loader '\vmlinuz-linux' --append-binary-args boot
```

**Install Yay and mkinitcpio Hook**
```
# pacman-key --init --gpgdir /mnt/etc/pacman.d/gnupg
# pacman-key --populate archlinux --gpgdir /mnt/etc/pacman.d/gnupg
# cp /etc/resolv.conf /mnt/etc/resolv.conf
# chroot /mnt
```

Create your standard user so we can use Yay without root. Replace `username` with your user.

```
# useradd -m -g users -G wheel username
# passwd
# visudo 
```

Edit the sudoers file to allow wheel group users to use sudo.

Switch to our new user and set password:

```
# su username
# passwd
```

Install Yay and then the btrfs hooks for early init:

```
# git clone https://aur.archlinux.org/yay.git
# cd yay
# makepkg -si
# yay -S mkinitcpio-btrfs
# exit
# vim /etc/mkinitcpio.conf
```

Add `crc32c` to the MODULES section. Add `btrfs_advanced` to the end of the HOOKS section and remove the `fsck` hook. Save the file and regenerate the initial ramdisk.

```
# mkinitcpio -p linux
```

**Edit The fstab File**

```
# vim /etc/fstab
```

Should look like this... mostly...

```
/dev/sda2 / btrfs defaults,noatime 0 0
/dev/sda2 /var/lib/Arch btrfs defaults,noatime,subvol=/__active 0 0
/dev/sda1 /boot vfat auto defaults 0 0
```

Set the proper locale information:

```
# sed -i '/^#en_US.UTF/s/^#//' /etc/locale.gen
# locale-gen
```


**Prepare for System Reboot**

```
# exit
# cd /
# umount /mnt/{dev,proc,sys,boot}
# umount /mnt
# reboot
```

After the first boot, login as root, set locale and hostname:

```
# localectl set-locale LANG="en_US.utf8"
# hostnamectl set-hostname MilosThinkPad
# reboot
```
