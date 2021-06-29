# Install Arch Linux with encrypted filesystem(optional) and on btrfs partition  (UEFI)
Official guide for basic install: [https://wiki.archlinux.org/index.php/Installation_Guide](https://wiki.archlinux.org/index.php/Installation_Guide)  
it is always good to consult with official guide, cause arch config might change in time  
For setting up different locale than pl check official guide
  
# 1. Boot ISO
### Download the ISO file from [https://www.archlinux.org](https://www.archlinux.org/)  
### Put on pedrive  
>dd if=archlinux.img of=/dev/sdX bs=16M && sync   
  
### Boot from the usb.  
  
### Set keymap  
>loadkeys pl  
  
### Update clock  
>timedatectl set-ntp true  
  
### Optionally (recommended) update mirrorlist  
>reflector --country 'Poland' --age 24 --verbose --sort rate --save /etc/pacman.d/mirrorlist  
 
 # 2. Prepare Disk

### Update btrfs-progs
>pacman -Syy btrfs-progs  
  
### Display disks setup  
>fdisk -l  
  
### Create partitions (if you have not already)  
>fdisk /dev/sdX  
1. 100MB EFI partition 
2. 100% size partiton # ( encrypted optionally) for BTRFS, this partition will require formatting AFTER encryption if you do encryption  
### Swap will be as file in its own subvolume  
  
>mkfs.vfat -F32 /dev/sdX1   
  
### ----------------- encryption (optional) ------------------  
  
### Setup the encryption of the system, don't use letters outside en-us keyboard like ąęć etc. for password  
### Grub have some kind of support for luks2 now but still cannot decrypt luks2, so specify luks1 for now
>cryptsetup -c=aes-xts-plain64 --key-size=512 --hash=sha512 --iter-time=3000 --pbkdf=pbkdf2 --use-random luksFormat --type=luks1 /dev/sdX2  

>cryptsetup luksOpen /dev/sdX2 MainPart  
  
### Formatting as btrfs now when it is already encrypted  
>mkfs.btrfs -L "Arch Linux" /dev/mapper/MainPart  
  
  
### ---------------- end of encryption ------------------------  
  
### Format the partition if not yet formatted:
>pacman -Syy btrfs-progs  

>mkfs.btrfs -L "Arch Linux" /dev/sdX2  
  
### Mount partition to be able to create btrfs subvolumes  
### If using encryption, change /dev/sdX2 to /dev/mapper/MainPart:   
>mount /dev/sdX2 /mnt  
  
## Create subvolumes  
### Using more complicated sheme, (but there actually is only need for separate @swap subvolume , other files can be on default top subvolume)  
  
>btrfs su cr /mnt/@  

>btrfs su cr /mnt/@swap  

>btrfs su cr /mnt/@home  

>btrfs su cr /mnt/@var  

>btrfs su cr /mnt/@tmp  

>btrfs su cr /mnt/@snapshots  

#### disable copy on write on var, tmp and swap
>chattr +C /mnt/@var  
>chattr +C /mnt/@tmp  
>chattr +C /mnt/@swap  

>umount /mnt  
  
### If using encryption, change /dev/sdX2 to /dev/mapper/MainPart:  
>mount -o defaults,noatime,discard,ssd,subvol=@ /dev/sdX2 /mnt  
  
>mkdir /mnt/swap  

>mkdir /mnt/home  

>mkdir /mnt/var  

>mkdir /mnt/tmp  

>mkdir /mnt/snapshots  

>mkdir /mnt/efi # for EFI partition /dev/sdX1  
  
### If using encryption, change /dev/sdX2 to /dev/mapper/MainPart  
### for swap subvolume add nodatacow option to disable CoW (works only if its separate partition)
### Discard ssd and noatime are for ssd disks only  
  
>mount -o defaults,noatime,nodatacow,discard,ssd,subvol=@swap /dev/sdX2 /mnt/swap  

>mount -o defaults,noatime,discard,ssd,subvol=@home /dev/sdX2 /mnt/home  

>mount -o defaults,noatime,discard,ssd,subvol=@var /dev/sdX2 /mnt/var  

>mount -o defaults,noatime,discard,ssd,subvol=@tmp /dev/sdX2 /mnt/tmp  

>mount -o defaults,noatime,discard,ssd,subvol=@snapshots /dev/sdX2 /mnt/snapshots  

>mount /dev/sdX1 /mnt/efi  
  
  
# 3.  Install Arch Linux
  
### Select the mirror to be used if not updated with reflector on start  
>nano /etc/pacman.d/mirrorlist  
  
### This command can be customized with additional packages  
>pacstrap /mnt/ base base-devel git btrfs-progs efibootmgr linux linux-headers linux-firmware mkinitcpio dhcpcd bash-completion sudo  
  
### Use genfstab with -U parameter if no encryption  
>genfstab /mnt >> /mnt/etc/fstab  
  
### If using swapfile check if nodatacow is added for @swap  
>nano /mnt/etc/fstab  
  

  
# 4. Configure the system 
  
### Switch to installed system root user
>arch-chroot /mnt /bin/bash  
  
### Nano can be usefull when editing config files
>pacman -Syy nano  
  
### Setup system clock  
>ln -s /usr/share/zoneinfo/Europe/Warsaw /etc/localtime  

>hwclock --systohc --utc  
  
### Set the hostname  
>/etc/hostname  
>>myhostname 
  
### Edit vconsole  
>/etc/vconsole.conf  
>>KEYMAP=pl  
>>FONT=Lat2-Terminus16.psfu.gz  
>>FONT_MAP=8859-2  
  
### Setup locale  
### Uncomment pl_PL.UTF-8 in /etc/locale.gen and then:  
>locale-gen  
  
### Update locale  
>/etc/locale.conf  
>>LANG=pl_PL.UTF-8  
>>LC_ALL=pl_PL.UTF-8
  
### Hosts 
>/etc/hosts  
>>127.0.0.1 localhost  
>>::1 localhost  
>>127.0.1.1 myhostname.localdomain myhostname  
  
### Now create 4GiB swap file. nodatacow is already on @swap but if you follow exactly then @swap is on same partition as other subvolumes and nodatacow will not work for whole subvolume so you need to disavle CoW manualy :  
>touch /swap/swapfile  
### Check if C attribute is enabled with 
>lsattr /swap/swapfile'

### If not then disable COW for swapfile manually: 
>chattr +C /swap/swapfile  
  
### Expanding empty file to 4GiB swap file  
>dd if=/dev/zero of=/swap/swapfile bs=1024K count=4096  
  
>chmod 600 /swap/swapfile  
  
### Format the swap file.  
>mkswap /swap/swapfile  
  
### Turn swap file on.  
>swapon /swap/swapfile  
  
### You also need to update /etc/fstab to mount swapfile on boot:  
>/etc/fstab
>>/swap/swapfile none swap sw 0 0  
  
### Set password for root  
>passwd  
### Add real user  
>useradd -m MYUSERNAME  

>passwd MYUSERNAME  
  
### Configure mkinitcpio with modules needed for the initrd image  
>nano /etc/mkinitcpio.conf  
### Remove 'fsck' and add 'keyboard', 'keymap', 'encrypt' and 'btrfs' to HOOKS before filesystems  
### If no encryption then only remove fsck and add on that place btrfs  
>HOOKS=(... keyboard keymap block encrypt btrfs ... filesystems ...)  
  
###### optionally add BINARIES=(/usr/bin/btrfs) for rescue?  
  
### Regenerate initrd images  
>mkinitcpio -P  
  
  # 5. Install bootloader
  
### Setup grub (UEFI)  
>pacman -S grub efibootmgr os-prober dosfstools mtools  
  
  
### -------------encryption only---------------------  
>nano /etc/default/grub  
>>GRUB_ENABLE_CRYPTODISK=y  
### Find UUID (PARTUUID for /dev/sdX2) of crypto partition so we can add it to grub config  
>blkid  
### Now set this line including proper UUID in place of "\<device-UUID>":
#### (temporarly you can use /dev/sdX2 in place of "UUID=\<device-UUID>" and change it later easy in gui mode)
>/etc/default/grub 
>>GRUB_CMDLINE_LINUX="cryptdevice=UUID=\<device-UUID>:MainPart:allow-discards"  
### allow-discards is only for ssd  
  
### Generate key so grub don't ask twice for password on boot  
>dd bs=512 count=4 if=/dev/random of=/crypto_keyfile.bin iflag=fullblock  
>chmod 600 /crypto_keyfile.bin  
>chmod 600 /boot/initramfs-linux*  
>cryptsetup luksAddKey /dev/sdX2 /crypto_keyfile.bin  
### If you change name of key file there is need to add kernel parameter like cryptkey=rootfs:path  
### Crypto_keyfile.bin is the default name that kernel will guess anyway  
### Now add this file to mkinitcpio.conf  
>/etc/mkinitcpio.conf  
>>FILES=(/crypto_keyfile.bin)  
  
>mkinitcpio -P  
### -------------encryption end---------------------  
  
### Install
>grub-install --target=x86_64-efi --efi-directory=/efi --bootloader-id=GRUB  
>grub-mkconfig -o /boot/grub/grub.cfg  
  
### Exit new system  
>exit  
  
### Unmount all partitions  
>swapoff -a  
>umount -R /mnt  
  
### Reboot into the new system, don't forget to remove the CD/pendrive
>reboot  
### or  
>shutdown now  
  
## Addtitional tips
### To get proper locale and keymap, check:  
>localectl status  
### On KDE plasma , also set settings > ... > keyboard layout && regional settings
