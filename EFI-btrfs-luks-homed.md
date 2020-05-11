**Boot off the CD or USB Drive**

**Obtain Internets**

Plug into a hard network if you're able, otherwise use wireless:

```shell script
ip link set wlan0 up
iwconfig wlan0 essid "[SSID Name]"
dhcpcd wlan0
```

... or if there's encryption... 

```shell script
ip link set wlan0 up
wpa_supplicant -B -i wlan0 -c <(wpa_passphrase SSIDNAME passphrase)
dhcpcd wlan0
```

**Initial Steps and Hard Drive Setup**
```shell script
pacman -Sy gptfdisk btrfs-progs
```

```shell script
gdisk /dev/sda
```

Konami Code: o, y, n, enter, enter, +512M, ef00, n, enter, enter, enter, enter, w, y.

Adjust as necessary, but this will leave you with a 512Mb formatted EFI partition. Be sure to use the right device located in `dev`. This could be something like `/dev/nvme0n1` if you're using an NVME type SSD.

For the rest of the document, pay attention to partitions. This guide utilizes two partitions. They may be identified like `/dev/sda1` and `/dev/sda2` or they could be identified like `/dev/nvme0n1p1` and `/dev/nvme0n1p2`.

**Encryption Setup**
```shell script
cryptsetup luksFormat /dev/sda2
```

Set a password when prompted and be sure to remember it. This _can_ be changed latter if needed.

Setup a mapping of our new luks device at `/dev/mapper/secure`:

```shell script
cryptsetup open /dev/sda2 secure
```

**btrfs Setup**

For btrfs, we create three subvolumes. One is used for the entire system root, one is used to place snapshots into if we use it in the future, and the last is one dedicated to our user account. Each user account on this system requires its own subvolume.

```shell script
mkfs.btrfs -f -L Arch /dev/mapper/secure
mount /dev/mapper/secure /mnt
btrfs sub create /mnt/@
btrfs sub create /mnt/@snapshots
umount /mnt
mount -o subvol=@,ssd,compress=lzo,noatime,nodiratime /dev/mapper/secure /mnt
```

**Boot Partition Setup**
```shell script
mkfs.vfat -F 32 /dev/sda1
mkdir /mnt/boot
mount /dev/sda1 /mnt/boot
```

**System Preparation and Installation**

Packages in the `pacstrap` command can be modified, but you must have `base`, `base-devel`, `btrfs-progs`, `linux`, and `go` to complete this guide. It is strongly recommended to install all of the ones listed in order to get up to speed quickly.

Before running `pacstrap`, modify `/etc/pacman.d/mirrorlist` to contain the most relevant mirrors to your position. This might make a substantial difference in time to install.

```shell script
pacstrap /mnt \
  base \
  base-devel \
  linux \
  linux-headers \
  linux-api-headers \
  linux-firmware \
  intel-ucode \
  btrfs-progs \
  zsh \
  vim \
  git \
  networkmanager \
  go
```

**Configure Boot and Kernel Settings**
```shell script
arch-chroot /mnt /usr/bin/zsh
vim /etc/mkinitcpio.conf
```

Edit the file by pressing `i` and navigating with arrow keys. The order of the hooks in particular are important.

Modify modules to contain `crc32c`:

* `MODULES=(crc32c)`

Modify hooks to contain the following:

* `base systemd autodetect modconf keyboard block sd-encrypt filesystems fsck`

Save the file by pressing `esc` followed by `:wq`.

Finally, generate our kernel files:

```shell script
mkinitcpio -p linux
```

**Install and Configure the Bootloader**

```shell script
bootctl --path=/boot install
blkid
```

Find the entry for `/dev/sda2` (or equivalent) and note the UUID. It will be used below.

```shell script
vim /boot/loader/entries/arch.conf
```

Edit the file by pressing `i` and navigating with arrow keys. The file should look like the this, but be sure to replace `ROOT_UUID` with the UUID you got from `blkid`:

```
title Arch Linux
linux /vmlinuz-linux
initrd /intel-ucode.img
initrd /initramfs-linux.img
options rd.luks.name=ROOT_UUID=secure root=/dev/mapper/secure rootflags=subvol=@ rd.luks.options=discard rw
```

Note: if you are on an non-Intel processor, remove the entire `initrd /intel-ucode.img` line.

Save the file by pressing `esc` followed by `:wq`.

```shell script
vim /boot/loader/loader.conf
```

Edit the file by pressing `i` and navigating with arrow keys. The file should look like this:

```
default arch
timeout 5
editor 0
```

Note: we explicitly disable bootloader entry editing because leaving it open would allow anyone with physical access to the device to obtain a shell.

Save the file by pressing `esc` followed by `:wq`.

**Generate an fstab File**
```shell script
exit
genfstab -U /mnt >> /mnt/etc/fstab
arch-chroot /mnt /usr/bin/zsh
```

**Create Personal User**

Note: this section is currently experiencing massive development in the community. Check out the `systemd-homed` [page on the ArchWiki](https://wiki.archlinux.org/index.php/Systemd-homed) to make sure you're following the latest advice. My advice is "_don't do this, there's so much more to live for_" and create a regular user with `useradd -m -G wheel -s /usr/bin/zsh username` followed by setting the password: `passwd username`.

If you still want to continue, read on and replace `username` below with your own username. 

```shell script
homectl create username --shell=/usr/bin/zsh --storage=subvolume -G wheel
vim /etc/pam.d/system-auth
```

The new `systemd-homed` functionality requires that we modify PAM auth settings to support legacy applications and logging in, otherwise our user cannot do much. Edit the file by pressing `i` and navigating with arrow keys. The file should look like the this:

```
auth      sufficient  pam_unix.so     try_first_pass nullok
-auth     sufficient  pam_systemd_home.so
auth      optional    pam_permit.so
auth      required    pam_env.so

-account  sufficient  pam_systemd_home.so
account   required    pam_unix.so
account   optional    pam_permit.so
account   required    pam_time.so

-password sufficient  pam_systemd_home.so
password  required    pam_unix.so     try_first_pass nullok sha512 shadow
password  optional    pam_permit.so

-session  optional    pam_systemd_home.so
session   required    pam_limits.so
session   required    pam_unix.so
session   optional    pam_permit.so
```

Save the file by pressing `esc` followed by `:wq`.

**Finalize Root Setup**

Set the root password and configure sudo options.

```shell script
passwd
EDITOR=vim visudo
```

Edit the file by pressing `i` and navigating with arrow keys. Find this line and uncomment it:

* `%wheel ALL=(ALL) ALL`

Save the file by pressing `esc` followed by `:wq`.

**Install Yay for AUR Packages**

If you have not already, change into your newly created user with `su username`, replacing _username_ with your username.

```shell script
cd ~
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
cd ../
rm -rf yay
```

**Prepare for System Reboot**
```shell script
exit ## to leave your user
exit ## to leave the chroot
umount -R /mnt
reboot
```

After the first boot, login as root, set locale and hostname:
```shell script
hostnamectl set-hostname MyHostname
sed -i '/^#en_US.UTF/s/^#//' /etc/locale.gen
locale-gen
localectl set-locale LANG="en_US.utf8"
reboot
```

**Recovery Steps**

If you need to get back into the system from a live boot, run these commands:

```shell script
cryptsetup open /dev/sda2 secure
mount -o subvol=@ /dev/mapper/secure /mnt 
mount /dev/sda1 /mnt/boot 
arch-chroot /mnt /usr/bin/zsh
```
