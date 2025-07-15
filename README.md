# <img src="./misc/arch-install-guide-repo.png" width="24"/> Arch Linux Installation Guide

This guide is licensed under the [GNU Free Documentation License 1.3](./LICENSE), it is originally uploaded to [codeberg](https://codeberg.org/unixchad/arch-install-guide) and [github](https://github.com/gnuunixchad/arch-install-guide).

Configuration files with my installation can be found on [codeberg](https://codeberg.org/unixchad/dotfiles) and [github](https://github.com/gnuunixchad/dotfiles)

# 0. setup
partition:
- LVM on LUKS
- hibernation to swap partition

boot:
- UEFI
- systemd-boot
- Secure Boot with `sbctl`

```lsblk
  NAME              SIZE  TYPE  MOUNTPOINTS
  nvme1n1         931.5G  disk
  └─nvme1n1p1     931.5G  part
    └─cryptdata   931.5G  crypt /data
  nvme0n1         476.9G  disk
  ├─nvme0n1p1         1G  part  /boot
  ├─nvme0n1p2        76G  part
  │ └─cryptlvm       76G  crypt
  │   ├─vg0-swap     16G  lvm   [SWAP]
  │   └─vg0-root     60G  lvm   /
  └─nvme0n1p3     399.9G  part
    └─crypthome   399.9G  crypt /home
```

# 1. archiso
## 1.1 ventoy
Bootable ISO USB drive created with `ventoy-1.0.99`

## 1.2 increase console font
```sh
setfont /usr/share/kbd/consolefonts/iso01-12x22.psfu.gz
```

## 1.3 connect to wifi (hidden)
```sh
# get full manual of iwct
iwctl help | less

# list network interface
iwctl device list

# connect hidden wifi
iwctl --passphrase <passphrase> station <device> connect-hidden <ssid>

# check connection
ip a
ping -c 3 archlinux.org
```

## 1.4 update the system clock
```sh
timedatectl set-timezon Region/City

# check NTP
timedatectl
```

## 1.5 partitioning
```sh
# read `fdisk`'s manual
fdisk /dev/nvme0n1 <<< m | less
# `<<<` is the "here string", this command send `m` to `fdisk /dev/nvme0n1` and
# piping to `less`, useful when console screen isn't enough

# subcommands
# `p` print
# `F` Free
# `d` delete
# `n` new
# `t` type
# `w` write
# `q` quit
```

## 1.6 formatting
```sh
# format the bootloader's partition
mkfs.fat -F 32 /dev/nvme0n1p1

# format the encrypt partitions
cryptsetup luksFormat /dev/nvme0n1p2

# unlock the encrypted partitions
cryptsetup open /dev/nvme0n1p2 cryptlvm
cryptsetup open /dev/nvme0n1p3 crypthome
cryptsetup open /dev/nvme1n1p1 cryptdata

# create physical volume for LVM on the top of the LUKS container
pvcreate /dev/mapper/cryptlvm

# create the volume group for LVM, name it `vg0`
vgcreate vg0 /dev/mapper/cryptlvm

# create the logical volumes inside the volume group
lvcreate -L 16G vg0 -n swap
lvcreate -l 100%FREE vg0 -n root

# format the logical volumes for root
mkfs.ext4 /dev/vg0/root
# or
mkfs.ext4 /dev/mapper/vg0-root
# both paths links to the same device `/dev/dm-2

# format the logical volumes for swap
mkswap /dev/vg0/swap

# format the crypted home
mkfs.ext4 /dev/mapper/crypthome
```

## 1.7 mount partitions
```sh
# mount root (ext4 on lvm on luks)
mount /dev/vg0/root /mnt

# create mount points for other partitions
mkdir -p /mnt/{boot,home,data}

# mount boot (esp, not encrypted because of secure boot)
mount /dev/nvme0n1p1 /mnt/boot

# mount home (ext4 on luks)
mount /dev/mapper/crypthome /mnt/home

# mount data (ext4 on luks on another ssd)
mount /dev/mapper/cryptdata /mnt/data

# enable swap (swap on lvm on luks)
swapon /dev/vg0/swap
```

## 1.8 install the operating system and linux kernel
```sh
# enable parallel downloads for pacman
vim /etc/pacman.conf
# uncomment `#ParallelDownloads = 5`

# change mirrorlist
reflactor --save /etc/pacman.d/mirrorlist

# update keyring
pacman -Sy && pacman -S archlinux-keyring
# When you use an Arch Linux ISO that was released months ago, the included
# keyring may be outdated. The Arch Linux keyring contains the public keys used
# to verify the signatures of packages

# install packages
pacstrap -K /mnt base base-devel linux linux-headers linux-firmware intel-ucode cryptsetup lvm2 vim neovim networkmanager man-db man-pages bash-completion

# explaining packages
#    base               minimal package set to define a basic arch linux
#                       installation
#    base-devel         basic tools to build arch linux packages
#    linux              the kernel
#    intel-ucode        ucode for intel cpu, amd cpu install `amd-ucode`
#    lvm2               if this package is not installed, root filesystem on the
#                       logical volume won't be able to be used
#    man-db             database for `man`
#    bash-completion    completion for sub-commands
```

## 1.9 generate fstab
```sh
genfstab -U /mnt >> /mnt/etc/fstab
```

## 1.10 chroot into the new system
```sh
arch-chroot /mnt
```

## 1.11 configure timezone and clock
```sh
# make a symbolic link to a timezone
ln -sf /usr/share/zoneinfo/Region/City /etc/localtime

# sync system time to the hardware clock on the computer's motherboard
hwclock --systohc
```
## 1.12 configure locale

uncomment in `/etc/locale.gen`
```/etc/locale.gen
en_US.UTF-8 UTF-8
```

Generate locales
```sh
locale-gen
```

append `/etc/locale.conf`
```/etc/locale.conf
LANG=en_US.UTF-8
```

## 1.13 create hostname
```sh
echo fx507 >> /etc/hostname
```

## 1.14.1 configure localhost
edit `/etc/hosts`
```/etc/hosts
127.0.0.1       localhost
::1             localhost
127.0.1.1       fx507.localdomain fx507
```

## 1.14.2 configure DNS resolution
edit `/etc/resolv.conf`
```/etc/resolv.conf
# CloudFlare's DNS
nameserver 1.1.1.1
```

This fixes `gpg` keyserver resolve

## 1.15 add a key file to luks container
```sh
cd /root

dd if=/dev/urandom of=/root/cryptkey bs=1024 count=4

chmod 400 cryptkey
chattr +i cryptkey

cryptsetup luksAddKey /dev/nvme0n1p3 /root/cryptkey

blkid >> /etc/crypttab
```

edit `/etc/crypttab`
```/etc/crypttab
# <uuid> of the luks containers
crypthome <uuid> /root/cryptkey luks,discard
```

## 1.15.1 kill no longer used key slot
```sh
# list all key slots
cryptsetup luksDump /dev/nvme0n1p3 | less
# kill slot 1 for instance
cryptsetup luksKillSlot /dev/nvme0n1p3 1
# you will be prompted for the key's password,
# and you cannot kill a key with its own password
```

## 1.16 create initial ramdisk environment
edit `/etc/mkinitcpio.conf`
```/etc/mkinitcpio.conf
# add `encrypt`, `lvm2` and `resume` hooks and modify the line to
HOOKS=base udev autodetect microcode modconf kms keyboard keymap consolefont block encrypt lvm2 filesystems resume fsck

# the kernel modules **MUST** be called by the order
#        - block (block device)
#        - encrypt (decrypt luks container)
#        - lvm2 (load logical volumes)
#        - filesystems
#        - resume (hibernation)
```

```sh
# build initramfs image(s) according to all presets
mkinitcpio -P
```

## 1.17 configure root and new user
```sh
# create root password
passwd

# create new user, adding to the wheel group, creating home directory if
# not existing
useradd -G wheel -m nate
passwd nate

# allow users of wheel group to use sudo
visudo
```
uncomment
```visudo
%wheel ALL=(ALL:ALL) ALL
```

## 1.18 install and config systemd-boot
```sh
# install systemd-boot to `/boot`
bootctl install
```

edit `/boot/loader/loader.conf`
```/boot/loader/loader.conf
default arch.conf
timeout 3
console-mode 0
```

get encrypted device and root partition UUID
```sh
`blkid >> /boot/loader/entries/arch.conf`
```

edit `/boot/loader/entries/arch.conf`
```/boot/loader/entries/arch.conf
title Arch Linux
linux /vmlinuz-linux
initrd /intel-ucode.img
initrd /initramfs-linux.img
options cryptdevice=UUID=<UUID-OF-PHYSICAL-PARTITION>:cryptlvm root=UUID=<UUID-OF-ROOT-LOGICAL-VOLUME>

# cryptdevice is the partition of luks container (in this case /dev/nvme0n1p2)
# root is the logical volume partition (in this case /dev/vg0/root)
```

optionally `cp /boot/loader/entries/arch.conf /boot/loader/entries/arch-fallback.conf`,
edit `/boot/loader/entries/arch-fallback.conf`
```/boot/loader/entries/arch-fallback.conf
title Arch Linux
linux /vmlinuz-linux
initrd /intel-ucode.img
initrd /initramfs-linux-fallback.img
options cryptdevice=UUID=<UUID-OF-PHYSICAL-PARTITION>:cryptlvm root=UUID=<UUID-OF-ROOT-LOGICAL-VOLUME>
```

enable auto update systemd-boot
```sh
systemctl enable systemd-boot-update.service`
```

## 1.19 finish arch install
```sh
# leave chroot
exit

# unmount partitions
umoun -R /mnt
swapoff -a

# leave archiso
reboot
```

# 2. post installation
login as root

## 2.1 setfont again temporally
```sh
setfont /usr/share/kbd/consolefonts/iso01-12x22.psfu.gz
```

## 2.2 set tty font permanently (take effect after reboot)
edit `/etc/vconsole.conf`
```/etc/vconsole.conf
FONT=iso01-12x22
# for HiDPI:
FONT=latarcyrheb-sun32
```

## 2.3 config bashrc a bit
```sh
set -o vi
 ```

remove the `~/.bash_profile` if exist as `~/.bash_profile` would override
`~/.profile`

## 2.4 enable networkmanager and connect to hidden wifi
```sh
systemctl enable --now NetworkManager.service

# run the following twice, as the first attemp would fail for ssid not found
nmcli device wifi connect <ssid> password <password> hidden yes
```

## 2.5 install user packages
```markdown
### basic tools
dash zsh zsh-syntax-highlighting vim neovim lf fzf tmux git rsync openssh openbsd-netcat udisks2 zip unzip unrar-free tree bc calc pacman-contrib archlinux-contrib reuild-detector arch-install-scripts dosfstools exfat-utils jq
### system configuration
networkmanager brightnessctl tlp ufw firejail cronie bluez-utils bluetui efibootmgr sbctl

### system monitoring
btop ncdu iftop sysstat smartmontools

### file sharing
android-file-transfer samba qrtool

### web browser
w3m firefox firefox-dark-reader firefox-tridactyl firefox-ublock-origin

### window manager/wayland compositor suite
foot river wlr-randr swaybg swayidle swaylock wmenu wtype wl-clipboard cliphist dunst gammastep slurp grim wf-recorder wl-mirror wob

### audio server
pipewire pipewire-alsa pipewire-pulse pipewire-jack noise-suppression-for-voice pulsemixer

### fonts
adobe-source-code-pro-fonts noto-fonts noto-fonts-cjk noto-fonts-emoji noto-fonts-extra ttf-font-awesome ttf-nerd-fonts-symbols

### file viewer
swayimg zathura zathura-pdf-mupdf bat catimg chafa lsix gnome-epub-thumbnailer poppler ffmpegthumbnailer odt2txt

### touch typing in a terminal
ttyper

### multi-media player
mpv ncmpcpp mpd mpc

### multi-media editor
ffmpeg python-mutagen imagemagick mediainfo perl-image-exiftool perl-rename
kdenlive gimp

### virtualization
virt-manager qemu-base libvirt virt-install dnsmasq openbsd-netcat bridge-utils
qemu-hw-display-qxl qemu-hw-display-virtio-gpu qemu-hw-display-virtio-gpu-pci qemu-chardev-spice qemu-audio-spice

### input method engine
fcitx5 fcitx5-chinese-addons fcitx5-configtool fcitx5-gtk fcitx5-qt fcitx5-anthy

### downloader & torrent
yt-dlp transmission-cli httrack

### personal tools
newsboat task calcurse neomutt isync dict

### coding
jdk-openjdk openjdk-src openjdk-doc xorg-xwayland code nodejs npm

### themes
gnome-themes-extra

### video driver (nvidia only)
nvidia-open nvidia-utils nvtop

### office suite
libreoffice-still
```

## 2.5.1 aur
```markdown
### helper
yay

### xoauth2 for isync
cyrus-sasl-xoauth2-git

### river
lswt rivercarro-git river-shifttags-git

### themes
adwaita-qt5 adwaita-qt5 fcitx5-skin-fluentdark-git

### directories
dict-gcide dict-wn

### etc
wshowkeys-mao-git

### fix
lf-git code-marketplace
```

### [example] compile and install
```sh
    git clone https://aur.archlinux.org/yay.git
    makepkg
    sudo pacman -U yay-*.pkg.tar.zst
```

## 2.5.2 local builds
dwl
    - https://codeberg.org/unixchad/dwl
    - https://github.com/gnuunixchad/dwl

dam
    - https://github.com/gnuunixchad/dam
    - https://codeberg.org/unixchad/dam
    - https://codeberg.org/sewn/dam

## 2.6 config softwares
### 2.6.-1 sbctl (secure boot)
1. reboot into UEFI utilities, restore secure boot's factory keys, and enter
`setup mode`

2. boot into system, check `sbctl status`, you should see:
```sbctl status
Installed:    ✘ Sbctl is not installed
Setup Mode:   ✘ Enabled
Secure Boot:  ✘ Disabled
```

3. create your own keys
```sh
sbctl create-keys
```

4. enroll the keys, along with microsoft keys if need dual boot with Windows
```sh
sbctl enroll-keys --microsoft
```

5. sign files:
```sh
sudo sbctl sign-all
sudo sbctl sign -s /boot/EFI/systemd/systemd-bootx64.efi
sudo sbctl sign -s /boot/EFI/BOOT/BOOTX64.EFI
sudo sbctl sign -s /boot/vmlinuz-linux
```

6. reboot into UEFI utilities, secure boot should be enabled automatically, if
not, do it manually instead

7. boot into system, check `sbctl status`, you should see:
```sbctl status
Installed:	✓ sbctl is installed
Setup Mode:	✓ Disabled
Secure Boot:	✓ Enabled
Vendor Keys:	microsoft
```

8. make sure `systemd-boot-update.service` is enabled for auto signing the
future bootloaders and kernels

### 2.6.0 automatically clean pacman cache
```sh
# remove unused packages weekly by `paccache` command from `pacman-contrib`
# package. (default keeps the last 3 versions of a package)
systemctl enable --now paccache.timer
```

### 2.6.1 tlp battery charing threshold
edit `/etc/tlp.conf`
```/etc/tlp.conf
    STOP_CHARGE_THRESH_BAT1=80
```

```sh
sudo systemctl enable --now tlp.service
```

### 2.6.2 mandb database update
```sh
sudo mandb
```

### 2.6.5 calcurse import calendar
```sh
# import calendar data file
calcurse -i ~/.config/calcurse/calendar.ical
```

### 2.6.6 samba server
```sh
cp ./dotfiles/smb.conf /etc/samba/smb.conf
# or
curl 'https://git.samba.org/samba.git/?p=samba.git;a=blob_plain;f=examples/smb.conf.default' sudo tee /etc/samba/smb.conf

# adding a linux user to samba server
sudo smbpasswd -a nate

# enable smb service
sudo systemctl enable --now smb.service
```

### 2.6.7 enable time sync
```sh
systemctl enable --now systemd-timesyncd.service
timedatectl set-ntp true
```

### 2.6.8 sshd force loggin in with key file
edit `/etc/ssh/sshd_config`
```/etc/ssh/sshd_config
# uncomment this line
PasswordAuthentication no
```

```sh
# restart sshd
systemctl enable --now sshd.service
```

#### 2.6.8.1 enable ssh-agent
```sh
systemctl enable --now --user ssh-agent.service
```

#### 2.6.8.2 import ssh pub key (on nuc11)
```sh
# entering the directory where the pub key locates
cd ~/.ssh
# use smbclient to move pub key to nuc11
smbclient //192.168.xx.xx/smb -U nate
# `-U nate` can be omitted if samba server's user name is the same
# share name shall be identical in /etc/samba/smb.conf like [smb]

```smbclient
# copy the local file to the server
put ~/.ssh/id_rsa.pub
```

```sh
# on the samba server, import pub key for sshd
cat ~/smb/id_rsa.pub >> ~/.ssh/authorized_keys
sudo systemctl restart sshd.service
```

### 2.6.9 termux file syncing to android phone setup
```sh
# create termux user for syncing files to android phone
sudo useradd -m termux
# add user termux to the nate group
sudo usermod -aG nate termux

# on termux copy `~/.ssh/id_rsa.pub`, write to clipboard.txt on smb share

# if on wayland, copy to paste
cat ~/smb/clipboard.txt | wl-copy

# change user to termux with sudo (never create password for the termux user)
sudo su - termux
mkdir ~/.ssh

# import pub key to sshd
echo "<paste pub key here>" > .ssh/authorized_keys

# restart sshd
sudo systemctl restart sshd.service
```

#### 2.6.9.1 check home directory permission for sharing files
```sh
# make home directory r-x for group nate
chmod 750 ~

# make directories for sharing r-x for group nate
chmod g+rx ~/doc ~/pic ~/mus ~/vid ~/repo

# remove all permisions for other directories  for group nate
chmod g-rwx ~/.config ~/.local ~/.ssh ~/mnt ~/smb

# to sync to android phone, run the following in termux:
rsync -avh --delete --progress --ignore-errors --exclude .git termux@192.168.xx.xx:/home/nate/{doc,mus,pic,vid} storage/shared/back/
```

### 2.6.10 ufw rules
```sh
# allow incoming trafic from LAN through ssh port
sudo ufw allow from 192.168.0.0/16 to any app SSH

# allow incoming trafic from LAN through CIFS port for Samba server:
sudo ufw allow from 192.168.0.0/16 to any app CIFS

# enable ufw
sudo ufw enable
sudo systemctl enable --now ufw.service
```

### 2.6.11 firejail sandbox
```sh
# comment out `mkdir` lines to disable creating empty directories on starting
# up softwares
sudo vim /etc/firejail/newsboat.profile
sudo vim /etc/firejail/neomutt.profile

# start firejail
sudo firecfg
```

### 2.6.12 libvirt/kvm/qemu
```sh
sudo usermod nate -aG kvm,libvirt (take effect on relog)
sudo ufw allow in on virbr0 from any to any
```

edit `/etc/libvirt/network.conf`
```/etc/libvirt/network.conf
firewall_backend = "iptables"
```

```sh
sudo systemctl enable --now libvirtd
sudo virsh net-define /etc/libvirt/qemu/networks/default.xml
sudo virsh net-autostart defaul
# uncomment dnsmasq in /etc/firejail/firecfg.config
```

### 2.6.13 electron software (code/codium) themes on wayland
```sh
# make sure xdg-desktop-portal package is installed and run:
gsettings set org.gnome.desktop.interface gtk-theme "Adwaita-dark"
```

### 2.6.14 nvidia fix hibernation
```sh
sudo systemctl enable nvidia-suspend.service
sudo systemctl enable nvidia-hibernate.service
sudo systemctl enable nvidia-resume.service
sudo systemctl enable nvidia-powerd.service
```

### 2.6.15 change default power button behavior
```sh
sudo cp /etc/systemd/logind.conf /etc/systemd/logind.conf~
```

edit `/etc/systemd/logind.conf`
```/etc/systemd/logind.conf
# uncomment and modify to:
HandlePowerKey=hibernate
```

### 2.6.16 firefox
1. copy and modify `user.js`, `userChrome.css`
2. tridactyl
    - change scroll steps
        - `:bind j scrollline 2`
        - `:bind k scrollline -2`
    - add new bindings
        - `:bind n scrollline 10`
        - `:bind p scrollline -10`
    - focus movement actions on elements
        - `:bind i hint -; *`
    - make hint mode actually readable
        - `:set hintstyle.bg none`
        - `:set hintstyle.fg none`
        - `:set hintstyle.outline all`

### 2.6.17 change default shell & login shell
```sh
# a pacman [HOOK](etc/pacman.d/hooks/default-shell-symlink.hook) is needed to
# reassign symlink after every bash upgrade
sudo ln -sf /usr/bin/dash /usr/bin/sh
chsh -s /usr/bin/zsh
```

# 3.0 restore files from a backup media
```sh
# unlock and mount the backup disk
udisksctl unlock -b /dev/sdx1
udisksctl mount -b /dev/dm-x

# sync to home directory
rsync -avh --delete --progress /run/media/nate/usb-ssd0/back/{aur,doc,mus,pic,repo,vid} /home/nate/

# unmount and lock the device
udisksctl unmount -b /dev/dm-x
udisksctl lock -b /dev/sdx1
udisksctl poweroff -b /dev/sdx
```
