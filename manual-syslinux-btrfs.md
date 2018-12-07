**Boot off the CD**

**Obtain Internets**

Use the regular installer through the network setup... or...

<code>
# ip link set wlan0 up
# iwconfig wlan0 essid "[SSID Name]"
# dhcpcd wlan0
</code>

**Initial Steps and Hard Drive Setup**
<code>
pacman -Sy gptfdisk btrfs-progs vim-minimal
</code>

<code>
gdisk /dev/sda
</code>

Konami Code: o, y, n, enter, enter, +1M, ef02, n, enter, enter, enter, enter, x, a, 2, 2, enter, w, y.

(Adjust as necessary, but this will leave you with a 1Mb formatted Bios partition, valid if grub is ever used over syslinux. Just to prepare for whatever that could be. You can optionally create only one partition just by clicking enter a few times. Read the instructions.)

**BTRFS Setup**
<code>
mkfs.btrfs -L ArchSSD /dev/sda2
mkdir /btrfs
mount -o defaults,noatime /dev/sda2 /btrfs
btrfs sub create /btrfs/__active
## btrfs subvolume create /btrfs/__active/home
## btrfs subvolume create /btrfs/__active/usr
## btrfs subvolume create /btrfs/__active/var
mount -o subvol=__active /dev/sda2 /mnt
mkdir /btrfs/boot /mnt/boot /btrfs/__snapshot
mount --bind /btrfs/boot /mnt/boot
mkdir -p /mnt/var/lib/ArchSSD
chmod -R 0755 /btrfs/__active
</code>

The above commented sections are because BTRFS is yet to support recursive snapshots, meaning if you took a snapshot of __active and tried to boot off of it, arch would not be able to see home, usr, or var because they are snapshots on their own. You could snapshot just those directories and replace those at will manually if you choose to, but without the necessity of a separate home partition, there is no real point. I'll just snapshot the whole thing.

**System Preparation and Installation**

<code>
mkdir /mnt/{proc,dev,sys,var/lib/pacman}
pacstrap /mnt base base-devel syslinux btrfs-progs haveged zsh vim
mount -o bind /dev /mnt/dev
mount -t sysfs none /mnt/sys
mount -t proc none /mnt/proc
chroot /mnt extlinux -i /boot/syslinux
cat /mnt/usr/lib/syslinux/bios/gptmbr.bin > /dev/sda
cp /mnt/usr/lib/syslinux/bios/*.c32 /mnt/boot/syslinux/
nano /mnt/boot/syslinux/syslinux.cfg 
</code>

Edit your syslinux config file to match the boot partition (sda2 in this guide.)
Also ADD quirks and rootflags for btrfs boot.
<code>
LABEL arch
    MENU LABEL Arch Linux
    LINUX ../vmlinuz-linux
    APPEND root=/dev/sd#2 rw rootflags=subvol=__active usbhid.quirks=0x1B1C:0x1B12:0x4
    INITRD ../initramfs-linux.img

</code>

**Install Yaourt and mkinitcpio Hook**
<code>
cp /etc/resolv.conf /mnt/etc/resolv.conf
chroot /mnt
haveged -w 1024
pacman-key --init
pacman-key --populate archlinux
pkill haveged
nano /etc/pacman.conf
</code>

Uncomment the multilib repo and comment the SigLevel so that signature checking is disabled.

Add Arch Linux France's Repo to the list: 

<code>
[archlinuxfr]
Server = http://repo.archlinux.fr/$arch
</code>

<code>
vim /etc/pacman.d/mirrorlist
</code>

Uncomment a mirror

<code>
pacman -Syy yaourt
yaourt -S mkinitcpio-btrfs
vim /etc/mkinitcpio.conf
</code>

Add "crc32c" to the MODULES section. Add "btrfs_advanced" to the end of the HOOKS section and remove the fsck hook.

<code>
mkinitcpio -p linux
</code>

Note the btrfs ID for your __active subvolume:

<code>
btrfs subvolume list -t /
</code>

**Edit The fstab File**

<code>
vim /etc/fstab
</code>

Should look like this... replace the subvolid with the value taken above in the subvolume list command.

<code>
/dev/sda2 / btrfs defaults,noatime 0 0
/dev/sda2 /var/lib/ArchSSD btrfs defaults,noatime,subvolid=0 0 0
/var/lib/ArchSSD/boot /boot none rw,bind 0 0
</code>

Set the proper locale information:

<code>
sed -i '/^#en_US.UTF-8/s/^#//' /etc/locale.gen
locale-gen
</code>

And after the first boot, run this:

<code>
localectl set-locale LANG="en_US.utf8"
</code>

Set root password: passwd

**Prepare for System Reboot**

<code>
exit
cd /
umount /mnt/{dev,proc,sys,boot}
umount /mnt
reboot
</code>

Hold your breath...

If the system stalls at the emergency shell with an error about finding /sbin/init, your btrfs mounting failed. Remove the subvolid entry from the fstab and set the btrfs partition to use __active as its default subpartition using the values from the subvol list:

<code>
mkdir /btrfsmnt
mount /dev/sda2 /btrfsmnt
btrfs subvolume list -t /btrfsmnt
btrfs subvolume set-active 1234 /btrfsmnt
reboot
</code>
