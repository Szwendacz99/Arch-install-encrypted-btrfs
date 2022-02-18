# ArchLinux install encrypted btrfs

# Install Arch Linux on EFI system with full filesystem (including /boot) encrypted and on btrfs partition 

Official guide for basic install: [https://wiki.archlinux.org/index.php/Installation\_Guide](https://wiki.archlinux.org/index.php/Installation_Guide)  
it is always good to consult with official guide, cause arch config might change in time  
For setting up different locale, or better explanations check out Arch Wiki

## 1. Boot ISO

#### Download the ISO file from [https://www.archlinux.org](https://www.archlinux.org/)

#### Put on pendrive

```bash
dd if=archlinux.img of=/dev/sdX bs=16M && sync
```

#### Boot from the USB.

#### Optional (**experimental** approach to have desktop environment during install):

##### Extend writable space so you can install basic desktop in live environment and for example use gparted for partitioning or open this tutorial in web browser or whatever you want. 

<p class="callout warning">Remember this area is saved in your RAM, so make sure you have enough of it</p>

```
mount -o remount,size=5G /run/archiso/cowspace

pacman -Syy plasma-desktop glibc konsole xorg
pacman -Scc

startplasma-wayland
```

#### Set key map

```bash
loadkeys pl  
```

#### Update clock

```bash
timedatectl set-ntp true  
```

#### Optionally (recommended) update mirrorlist

```bash
reflector --country 'Poland' --age 24 --verbose --sort rate --save /etc/pacman.d/mirrorlist  
```

## 2. Prepare Disk

#### Update btrfs-progs

```bash
pacman -Syy btrfs-progs
```

#### Display disks and partitions

```bash
lsblk
```

#### Create partitions (if you have not already)

```bash
fdisk /dev/sdX  
```

1. 100MB EFI partition
2. 100% size partiton # ( encrypted optionally) for BTRFS partition, this partition will require formatting AFTER encryption if you do encryption

##### Swap will bin in file with CoW disabled, which will be prepared later

#### Format EFI partition

```Bash
mkfs.vfat -F32 /dev/sdX1
```

##### ----------------- encryption (optional) ------------------


#### Setup the encryption of the system,

<p class="callout info">Don't use regional letters (not in en-us keyboard) like ąęć etc. for password. This requires additional steps, which are not covered by this tutorial.</p>

#### Grub have some kind of support for luks2, but not entirely, so for more fail-safe setup use luks1

```bash
cryptsetup -c=aes-xts-plain64 --key-size=512 --hash=sha512 --iter-time=3000 --pbkdf=pbkdf2 --use-random luksFormat --type=luks1 /dev/sdX2  

cryptsetup luksOpen /dev/sdX2 MainPart 
```

### Formatting as btrfs now when it is already encrypted

```bash
mkfs.btrfs -L "Arch Linux" /dev/mapper/MainPart  
```

##### ---------------- end of encryption ------------------------

#### Format the partition if not yet formatted:

```bash
pacman -Syy btrfs-progs  

mkfs.btrfs -L "Arch Linux" /dev/sdX2  
```

#### Mount partition to be able to create btrfs subvolumes

##### If using encryption, change **/dev/sdX2** to **/dev/mapper/MainPart**:

```bash
mount /dev/sdX2 /mnt  
```

#### Create subvolumes

##### This scheme can be adjusted to your needs, I'd suggest at least one subvolume for root (@) and one for snapshots (@snapshots). varlog and tmp are created to easily disable Copy on Write on` /var/log` and `/tmp`.

```bash
btrfs su cr /mnt/@  

btrfs su cr /mnt/@home  

btrfs su cr /mnt/@varlog

btrfs su cr /mnt/@tmp  

btrfs su cr /mnt/@snapshots  

```

##### Disable copy on write on `/var/log` and `/tmp`

```bash
chattr +C /mnt/@varlog
chattr +C /mnt/@tmp  
umount /mnt  

```

#### If using encryption, change **/dev/sdX2** to **/dev/mapper/MainPart**:

```bash
mount -o defaults,noatime,discard,ssd,subvol=@ /dev/sdX2 /mnt  

mkdir /mnt/home  

mkdir -p /mnt/var/log  

mkdir /mnt/tmp  

mkdir /mnt/snapshots  

mkdir /mnt/efi # for EFI partition /dev/sdX1  
```

#### Discard and ssd options and are for ssd disks only

#### If using encryption, change **/dev/sdX2** to **/dev/mapper/MainPart**

```bash
mount -o defaults,noatime,discard,ssd,subvol=@home /dev/sdX2 /mnt/home

mount -o defaults,noatime,discard,ssd,subvol=@varlog /dev/sdX2 /mnt/var/log

mount -o defaults,noatime,discard,ssd,subvol=@tmp /dev/sdX2 /mnt/tmp

mount -o defaults,noatime,discard,ssd,subvol=@snapshots /dev/sdX2 /mnt/snapshots

mount /dev/sdX1 /mnt/efi
```

# 3. Install Arch Linux

#### Select the mirror to be used if not updated with reflector on start

```bash
vim /etc/pacman.d/mirrorlist  
```

#### Install base system:

##### This command can be customized with additional packages (**btrfs-progs is necessary to let the system boot up from btrfs partition !**)

```bash
pacstrap /mnt/ base base-devel git btrfs-progs efibootmgr linux linux-headers linux-firmware mkinitcpio dhcpcd bash-completion sudo
```

#### Generate fstab:

##### Use genfstab with -U parameter if no encryption

```bash
genfstab /mnt >> /mnt/etc/fstab
```

####  

# 4. Configure the system

#### Switch to installed system root user

```bash
arch-chroot /mnt /bin/bash
```

#### Setup system clock

```bash
ln -s /usr/share/zoneinfo/Europe/Warsaw /etc/localtime  

hwclock --systohc --utc  
```

#### Set the hostname in `/etc/hostname`

```test
myhostname
```

#### Edit vconsole in `/etc/vconsole.conf`

```text
KEYMAP=pl  
FONT=Lat2-Terminus16.psfu.gz  
FONT_MAP=8859-2  

```

#### Setup locale

##### Uncomment pl\_PL.UTF-8 in /etc/locale.gen and then run:

```bash
locale-gen
```

#### Update locale in `etc/locale.conf`

```text
LANG=en_US.UTF-8
LC_COLLATE=pl_PL.UTF-8
LC_MEASUREMENT=pl_PL.UTF-8
LC_MONETARY=pl_PL.UTF-8
LC_NUMERIC=pl_PL.UTF-8
LC_TIME=pl_PL.UTF-8

```

#### Hosts in `/etc/hosts`

```text
127.0.0.1 localhost  
::1 localhost  
127.0.1.1 myhostname.localdomain myhostname  

```

#### Now create empty (with 0 size) swap file:

#### Create separate subvolume for swapfile. This subvolume is needed to let you make snapshot of `/`, which would not be possible with any file in it with CoW disabled!

```
btrfs su create /swap

chattr +C /swap
```

#### Copy on Write should always be disabled on swap file, so it will be done in the next step

```bash
touch /swap/swapfile  
```

#### Check if C attribute is enabled (should be already if created in folder with disabled CoW attribute)

```bash
lsattr /swap/swapfile'
```

#### If not then disable CoW for swapfile manually:

```bash
chattr +C /swap/swapfile  
```

#### Expanding empty file to 4GiB swap file

```bash
dd if=/dev/zero of=/swap/swapfile bs=1024K count=4096  

chmod 600 /swap/swapfile  

```

#### Format the swap file.

```bash
mkswap /swap/swapfile  
```

#### Turn swap file on.

```bash
swapon /swap/swapfile  
```

#### You also need to update `/etc/fstab` to mount swapfile on boot:

```text
/swap/swapfile none swap sw 0 0  
```

#### Set password for root

```bash
passwd  
```

#### Add real user an set password for him

```bash
useradd -m MYUSERNAME 

passwd MYUSERNAME  
```

### Configure mkinitcpio with modules needed for the initrd image

```bash
vim /etc/mkinitcpio.conf  
```

#### Add 'keyboard', 'keymap', 'encrypt' and 'btrfs' to HOOKS before filesystems:

```
HOOKS=(base udev autodetect keyboard keymap modconf block btrfs filesystems keyboard fsck)
```

#### Add btrfsck to binaries:

```
BINARIES=(btrfsck)
```

#### **With encryption:** also add encrypt before btrfs:

```text
HOOKS=(... keyboard keymap block encrypt btrfs ... filesystems ...)  
```

######  

#### Regenerate initrd images

```bash
mkinitcpio -P  
```

# 5. Install bootloader

#### Setup grub (UEFI)

```bash
pacman -S grub efibootmgr os-prober dosfstools mtools  
```

#### -------------encryption only---------------------

#### edit `/etc/default/grub`

```text
GRUB_ENABLE_CRYPTODISK=y 
```

#### Find UUID (UUID for /dev/sdX2) of crypto partition so we can add it to grub config

```bash
blkid 
```

#### Now set this line including proper UUID in place of "&lt;device-UUID&gt;":

#### (temporarly you can use /dev/sdX2 in place of "UUID=&lt;device-UUID&gt;" and change it later easy in gui mode)

##### edit `/etc/default/grub`

```text
GRUB_CMDLINE_LINUX="cryptdevice=UUID=<device-UUID>:MainPart:allow-discards"  
```

##### allow-discards is only for ssd to let trim work with encryption enabled

#### Generate key so grub don't ask twice for password on boot

```bash
dd bs=512 count=4 if=/dev/random of=/crypto_keyfile.bin iflag=fullblock  
chmod 600 /crypto_keyfile.bin  
chmod 600 /boot/initramfs-linux*  
cryptsetup luksAddKey /dev/sdX2 /crypto_keyfile.bin  
```

#### If you change name of key file there is need to add kernel parameter like cryptkey=rootfs:path

#### Crypto\_keyfile.bin is the default name that kernel will guess anyway

#### Now add this file to `/etc/mkinitcpio.conf`

```text
FILES=(/crypto_keyfile.bin) 
```

then run:

```bash
mkinitcpio -P  
```

#### -------------encryption end---------------------

#### Install grub for 

```bash
grub-install --target=x86_64-efi --efi-directory=/efi --bootloader-id=GRUB  
grub-mkconfig -o /boot/grub/grub.cfg  
```

#### Exit new system

```bash
exit  
```

#### Unmount all partitions

```bash
swapoff -a  
umount -R /mnt
```

#### Reboot into the new system, don't forget to remove the pendrive

```bash
reboot  
```

#### or

```bash
shutdown now  
```

### 6. Addtitional tips:

#### Install AUR helper (git and base-devel packages needed to do so):

```
git clone https://aur.archlinux.org/yay.git

cd yay

makepkg -si
```

#### To get proper locale and keymap, check:

```bash
localectl status
```

#### On KDE plasma , also set settings &gt; ... &gt; keyboard layout &amp;&amp; regional settings
