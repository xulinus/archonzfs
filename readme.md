# Arch on ZFS MBP15

#### 0 På maskinen

Ställ in ett lösenord: `passwd`

Kolla ip address: `ip a`

#### 1 På remote

Ssha till maskinen att installera på `ssh root@[ip]`

Uppdatera paket:

```bash
pacman -Syy
```

##### 1.1 Fixa ZFS, partition, skapa datasets etc

Hämta ZFS moduler:

```bash
curl -s https://raw.githubusercontent.com/eoli3n/archiso-zfs/master/init | bash
```

Kontrollera att zfs installerats korrekt:

```bash
zpool list
zfs list
```

Skapa ett nytt bashskript:

```bash
nano zfssetup.sh
```

```bash
#!/bin/bash

# Prompt the user to set the DISK variable
read -p "Enter disk to use for installation (e.g., /dev/sdX): " DISK

# Display the chosen disk
echo "You chose disk: $DISK"

# Ask for confirmation
echo "WARNING! ALL DATA ON $DISK WILL BE DESTROYED"
read -p "Is this the correct disk? (Y/N): " CONFIRM

# Check the confirmation
if [[ "$CONFIRM" != "Y" && "$CONFIRM" != "y" ]]; then
  echo "Aborted."
  exit 1  # Exit the script with a non-zero status indicating an error
fi

# Ensure swap partitions are not in use.
swapoff --all

echo "Creating partions."

# Wipe earlier zfs
wipefs -a $DISK

# TRIM/UNMAP
#blkdiscard -f $DISK

# Clear the partition table.
sgdisk --zap-all $DISK

# For UEFI booting.
sgdisk     -n2:1M:+512M   -t2:EF00 $DISK

# Create boot pool.
sgdisk     -n3:0:+1G      -t3:BF01 $DISK

# Encrypted with LUKS
sgdisk     -n4:0:0        -t4:BF00 $DISK

# Make sure zfs modules are loaded
modprobe zfs

echo "Creating bpool."

# Create boot pool.
zpool create -f \
    -o ashift=12 \
    -o autotrim=on -d \
    -o cachefile=/etc/zfs/zpool.cache \
    -o feature@async_destroy=enabled \
    -o feature@bookmarks=enabled \
    -o feature@embedded_data=enabled \
    -o feature@empty_bpobj=enabled \
    -o feature@enabled_txg=enabled \
    -o feature@extensible_dataset=enabled \
    -o feature@filesystem_limits=enabled \
    -o feature@hole_birth=enabled \
    -o feature@large_blocks=enabled \
    -o feature@livelist=enabled \
    -o feature@lz4_compress=enabled \
    -o feature@spacemap_histogram=enabled \
    -o feature@zpool_checkpoint=enabled \
    -O devices=off \
    -O acltype=posixacl -O xattr=sa \
    -O compression=lz4 \
    -O normalization=formD \
    -O relatime=on \
    -O canmount=off -O mountpoint=/boot -R /mnt \
    bpool ${DISK}-part3

echo "Creating rpool."

zpool create -f \
    -o ashift=12 \
    -o autotrim=on \
    -O encryption=on -O keylocation=prompt -O keyformat=passphrase \
    -O acltype=posixacl -O xattr=sa -O dnodesize=auto \
    -O compression=lz4 \
    -O normalization=formD \
    -O relatime=on \
    -O canmount=off -O mountpoint=/ -R /mnt \
    rpool ${DISK}-part4

echo "Creating datasets."

# Create filesystem datasets to act as containers.
zfs create -o canmount=off -o mountpoint=none rpool/ROOT
zfs create -o canmount=off -o mountpoint=none bpool/BOOT

# Create filesystem datasets for the root and boot filesystems.
zfs create -o canmount=noauto -o mountpoint=/ rpool/ROOT/arch
zfs mount rpool/ROOT/arch

zfs create -o mountpoint=/boot bpool/BOOT/arch

# Create datasets.
zfs create rpool/home
zfs create -o mountpoint=/root rpool/home/root
chmod 700 /mnt/root
zfs create -o canmount=off rpool/var
zfs create -o canmount=off rpool/var/lib
zfs create rpool/var/log
zfs create rpool/var/spool
zfs create -o com.sun:auto-snapshot=false rpool/var/lib/libvirt
zfs create -o com.sun:auto-snapshot=false rpool/var/lib/docker
zfs create -o com.sun:auto-snapshot=false rpool/var/lib/lxc
zfs create -o canmount=off rpool/usr
zfs create rpool/usr/local

# Separate dataset for /tmp
zfs create -o com.sun:auto-snapshot=false  rpool/tmp
chmod 1777 /mnt/tmp

# Export and import pools
umount -R /mnt
zfs umount -a
zpool export -a
zpool import -d ${DISK}-part4 -R /mnt rpool -N
zfs load-key rpool
zpool import -d ${DISK}-part3 -R /mnt bpool -N

# Mount datasets
zfs mount rpool/ROOT/arch
zfs mount bpool/BOOT/arch
zfs mount -a

# create zpool.cache
zpool set cachefile=/etc/zfs/zpool.cache rpool
zpool set cachefile=/etc/zfs/zpool.cache bpool

# bring zpool.cache to new system
mkdir -p /mnt/etc/zfs
cp /etc/zfs/zpool.cache /mnt/etc/zfs/zpool.cache

echo "ZFS setup is done."
```

```bash
chmod +x zfssetup.sh
./zfssetup.sh
```

Kontrollera att alla datasets monterats:

```bash
zfs list
df -k
```

##### 1.2 Installera Arch

Install the base system

```bash
pacstrap -K /mnt base linux linux-firmware nano vim
```

Change root into the new system

```
arch-chroot /mnt
```

Add Arch ZFS repository to /etc/pacman.conf

```bash
echo -e '
[archzfs]
Server = https://archzfs.com/$repo/x86_64' >> /etc/pacman.conf
```

Hämta key: [https://wiki.archlinux.org/index.php/Unofficial\_user\_repositories#archzfs](https://wiki.archlinux.org/index.php/Unofficial_user_repositories#archzfs)

```shell
pacman-key -r DDF7DB817396A49B2A2723F7403BD972F75D9D76
pacman-key --lsign-key DDF7DB817396A49B2A2723F7403BD972F75D9D76
pacman -Syyuu
```

Installera zfs-linux

```
pacman -S zfs-utils
pacman -S zfs-linux
```

Boot setup

```bash
nano /etc/mkinitcpio.conf
```

Lägg till ZFS i MODULES `MODULES=(zfs)`

Ändra HOOKS =(...) till: `HOOKS=(base udev autodetect modconf block keyboard keymap zfs filesystems)`

Generate image

```
mkinitcpio -p linux
```

Installera övriga paket

```bash
pacman -S --noconfirm grub efibootmgr zsh networkmanager network-manager-applet wpa_supplicant \
xorg plasma plasma-wayland-session kde-applications git sudo firefox screenfetch \
dnsutils inetutils tcpdump signal-desktop wireguard-tools openresolv ksshaskpass tcpdump \
gimp rawtherapee libreoffice noto-fonts-cjk 
```

##### 1.3 Installera GRUB

Kontrollera att ZFS boot system känns igen:

```
grub-probe /boot
```

Finally, let's set up GRUB. First, make sure to create the `/boot/grub` directory. Edit the file `/etc/default/grub` and add `zfs=zroot/rootfs` to the kernel parameters at `GRUB_CMDLINE_LINUX_DEFAULT`. Generate the GRUB configuration files and install GRUB with the according tools:

```bash
pacman -S --noconfirm dosfstools

DISK=

mkdosfs -F 32 -s 1 -n EFI ${DISK}-part2

mkdir /boot/efi
echo /dev/disk/by-uuid/$(blkid -s UUID -o value ${DISK}-part2) \
   /boot/efi vfat defaults 0 0 >> /etc/fstab
mount /boot/efi
#apt install --yes grub-efi-amd64 shim-signed

nano /etc/default/grub 
# GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3"
# GRUB_CMDLINE_LINUX="root=ZFS=rpool/ROOT/arch"
grub-mkconfig -o /boot/grub/grub.cfg

grub-install --target=x86_64-efi --efi-directory=/boot/efi \
    --bootloader-id=arch --recheck --no-floppy
```

##### 1.4 Avsluta installationen

We need to enable some services for systemd to be able to handle and mount our ZFS datasets. Remember to enable your display manager service, too.

```
systemctl enable zfs.target zfs-import-cache \
  zfs-mount zfs-import.target NetworkManager sddm
```

When running ZFS on root, the machine's hostid will not be available at the time of mounting the root filesystem. There are two solutions to this. You can either place your spl hostid in the kernel parameters in your boot loader. For example, adding spl.spl\_hostid=0x00bab10c, to get your number use the hostid command.

The other, and suggested, solution is to make sure that there is a hostid in /etc/hostid, and then regenerate the initramfs image which will copy the hostid into the initramfs image. To write the hostid file safely you need to use the zgenhostid command.

```
zgenhostid $(hostid)
```

Byt hostname

```
echo baal > /etc/hostname
echo -e '127.0.0.1 localhost\n::1 localhost\n127.0.1.1 baal' >> /etc/hosts
```

Sätt tidszon

```bash
ln -sf /usr/share/zoneinfo/Europe/Stockholm /etc/localtime # Change according to location…
hwclock --systohc # Sync with HW clock
```

Configure locals

```bash
echo -e '
sv_SE.UTF-8 UTF-8
en_US.UTF-8 UTF-8
zh_CN.UTF-8 UTF-8' >> /etc/locale.gen
echo 'KEYMAP=sv-latin1' > /etc/vconsole.conf
echo 'LANG=en_US.UTF-8' > /etc/locale.conf
locale-gen
```

Add user

```bash
groupadd sudo
useradd -m -G sudo linus
usermod -a -G wheel linus
EDITOR=nano visudo # uncomment sudo group
passwd linus
```

Add admin privileges in KDE to wheel

```bash
echo 'polkit.addAdminRule(function(action, subject) {
    return ["unix-group:wheel"];
});' >> /etc/polkit-1/rules.d/40-default.rules
```

#### 2 Unmount and restart

```bash
exit
# Back in the installer shell…
umount -R /mnt
zfs umount -a
zpool export -a

shutdown
```

#### 3 Fix at first login

Config ssh-add to use Kwallet

```bash
echo "[Desktop Entry]
Exec=ssh-add -q
Name=ssh-add
Type=Application" > ~/.config/autostart/ssh-add.desktop

mkdir -p ~/.config/environment.d/
echo "SSH_ASKPASS='/usr/bin/ksshaskpass'
SSH_ASKPASS_REQUIRE=prefer" > ~/.config/environment.d/ssh_askpass.conf

# edit ~/.zshrc
# add eval "$(ssh-agent)" > /dev/null
# after plugins=() 
```

Install yay

Install apple-emoji

```bash
yay -S ttf-apple-emoji
```

Fixa root cert för Entropy Center

```bash
cat "-----BEGIN CERTIFICATE-----
MIIBuDCCAV6gAwIBAgIRANOqDHCHaQ0k1kRe+pgTf50wCgYIKoZIzj0EAwIwOjEX
MBUGA1UEChMORW50cm9weSBDZW50ZXIxHzAdBgNVBAMTFkVudHJvcHkgQ2VudGVy
IFJvb3QgQ0EwHhcNMjMwNDIyMjAzOTIwWhcNMzMwNDE5MjAzOTIwWjA6MRcwFQYD
VQQKEw5FbnRyb3B5IENlbnRlcjEfMB0GA1UEAxMWRW50cm9weSBDZW50ZXIgUm9v
dCBDQTBZMBMGByqGSM49AgEGCCqGSM49AwEHA0IABDVoDnVMe/QRn/tSLHlSK2S/
MjTwv9LZnIuXnmuVDuQBagR4cbw8J0Kjlx50cMxnAdGrBXdtZNnNH2/v4v2J8yOj
RTBDMA4GA1UdDwEB/wQEAwIBBjASBgNVHRMBAf8ECDAGAQH/AgEBMB0GA1UdDgQW
BBSJYHMSUxgM01cyvK9ljBzur1wcCTAKBggqhkjOPQQDAgNIADBFAiB0MIpMOE1B
1i3nfANEObpT0qFurxujBRKOYoZ9NoceagIhAJciPWWzJAgT7k5C6sd3Yy95r9Ya
r+McI9QrVhJR0/IS
-----END CERTIFICATE-----" > ~/entropy_cetner_root_ca.crt

sudo trust anchor --store ~/entropy_cetner_root_ca.crt
```

#### A Källor

[https://wiki.archlinux.org/title/Install\_Arch\_Linux\_on\_ZFS](https://wiki.archlinux.org/title/Install_Arch_Linux_on_ZFS)

[https://openzfs.github.io/openzfs-docs/Getting%20Started/Debian/Debian%20Bullseye%20Root%20on%20ZFS.html](https://openzfs.github.io/openzfs-docs/Getting%20Started/Debian/Debian%20Bullseye%20Root%20on%20ZFS.html)

[https://gist.github.com/GrumpyChunks/11c912f40faaa6a5a3e8ee1afa40096f](https://gist.github.com/GrumpyChunks/11c912f40faaa6a5a3e8ee1afa40096f)

[https://wiki.archlinux.org/title/Installation\_guide](https://wiki.archlinux.org/title/Installation_guide)

[https://github.com/eoli3n/archiso-zfs](https://github.com/eoli3n/archiso-zfs)

[https://blog.timo.page/installing-arch-linux-on-zfs](https://blog.timo.page/installing-arch-linux-on-zfs)

[https://dev.to/henrybarreto/pacman-s-simple-guide-for-apt-s-users-5hc4](https://dev.to/henrybarreto/pacman-s-simple-guide-for-apt-s-users-5hc4)

[https://www.youtube.com/watch?v=CcSjnqreUcQ](https://www.youtube.com/watch?v=CcSjnqreUcQ)

[https://wiki.archlinux.org/title/Polkit#Administrator\_identities](https://wiki.archlinux.org/title/Polkit#Administrator_identities)
