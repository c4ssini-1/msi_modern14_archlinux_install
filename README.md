# Install Arch Linux on the MSI Modern 14 C12MO

- What I will be using: LUKS, LVM, UKI, systemd-boot, Secure Boot, TPM2
- My Partitions and LVMs: 1GB EFI, 16GB SWAP, and the rest of the space would be one partition.

### Turn off Secure Boot

1. Connect your USB installer.
2. Turn on the laptop and press `delete` as soon as you see the "MSI" logo. This will load the MSI Click BIOS.
3. Go to the "Security" tab and select "Secure Boot"
4. Switch "Secure Boot" to "disabled".
5. Go to the "Boot" tab and change the boot order to have the installer USB on the top of the list.
6. Press `F10` to save and exit the bios. This should boot into the installer.

### Installation

1. `timedatectl set-ntp true`
2. Creating the following partitions: 1GB efi; rest of the space for the system:
    - `sgdisk --zap-all /dev/nvme0n1`
    - ````
      sgdisk --clear --new=1:0:+1GiB --typecode=1:ef00 --change-name=1:EFI \ 
      --new=2:0:0 --typecode=2:8300 --change-name=2:cryptsystem \ 
      /dev/nvme0n1 
      ````
3. LUKS, LVM creation and Formatting:
    - `mkfs.fat -F32 -n EFI /dev/disk/by-partlabel/EFI`
    - `cryptsetup luksFormat --align-payload=8192 -s 256 -c aes-xts-plain64 /dev/disk/by-partlabel/cryptsystem`
    - Choose a password for the partition.
    - `cryptsetup open /dev/disk/by-partlabel/cryptsystem system`
    - `pvcreate /dev/mapper/system`
    - `vgcreate live_store /dev/mapper/system`
    - `lvcreate -L 16G live_store -n swap`
    - `lvcreate -l 100%FREE live_store -n root`
    - `mkswap /dev/live_store/swap`
    - `mkfs.ext4 /dev/live_store/root`
4. Mount Everything:
    - `swapon /dev/live_store/swap`
    - `mount /dev/live_store/root /mnt`
    - `mkdir /mnt/efi`
    - `mount /dev/disk/by-partlabel/EFI /mnt/efi`
5. Prepare Install
    - `reflector --save /etc/pacman.d/mirrorlist --protocol https --latest 5 --sort age`
6. Install Arch and chroot into it.
    - `pacstrap /mnt base base-devel linux linux-firmware linux-lts linux-headers nano sudo lvm2 intel-ucode git binutils`
    - `genfstab -U /mnt >> /mnt/etc/fstab`
    - Edit `/mnt/etc/fstab`: Change /efi fmask to 0137, dmask to 0027.
    - `arch-chroot /mnt`
7. Timezones, Locales, and host information
    - `ln -sf /usr/share/zoneinfo/Asia/Kolkata /etc/localtime`
    - `hwclock --systohc`
    - Edit `/etc/locale.gen` and uncomment the locales that you want. `en_US.UTF-8` is what I use.
    - `locale-gen`
    - `echo "LANG=en_US.UTF-8" >> /etc/locale.conf`
    - `echo "KEYMAP=us" >> /etc/vconsole.conf`
    - `echo "<chosen-hostname>" >> /etc/hostname`
    - `echo "127.0.0.1 localhost" >> /etc/hosts`
    - `echo "::1       localhost" >> /etc/hosts`
    - `echo "127.0.1.1 <chosen-hostname>" >> /etc/hosts`
8. Edit `mkinitcpio.conf` with the following:
    - Change `MODULES=()` to `MODULES=(i915)`
    - Comment out the like starting with `HOOKS = (...)` 

      to `HOOKS=(systemd autodetect microcode modconf kms keyboard sd-vconsole block sd-encrypt lvm2 filesystems fsck)`
9. Change password for root password: `echo root:AspectObedienceSeltzerUnplugGiddinessMocha | chpasswd`
10. Install systemd-boot: `bootctl install`
11. Edit `/efi/loader/loader.conf` with the following:
    ````
    default arch-linux.efi
    timeout 5
    console-mode auto
    editor no
    ````
12. Create UKI:-
     - Comment and uncomment lines in `/etc/mkinitcpio.d/linux.preset` to make it look like below:
       ````
        # mkinitcpio preset file for the 'linux' package
        
        ALL_config="/etc/mkinitcpio.conf"
        ALL_kver="/boot/vmlinuz-linux"
        
        PRESETS=('default')
        
        #default_config="/etc/mkinitcpio.conf"
        #default_image="/boot/initramfs-linux.img"
        default_uki="/efi/EFI/Linux/arch-linux.efi"
        default_options="--splash /usr/share/systemd/bootctl/splash-arch.bmp"
        
        #fallback_config="/etc/mkinitcpio.conf"
        #fallback_image="/boot/initramfs-linux-fallback.img"
        fallback_uki="/efi/EFI/Linux/arch-linux-fallback.efi"
        fallback_options="-S autodetect"
       ````
     - Make the same edits to `/etc/mkinitcpio.d/linux-lts.preset`.
     - `mkinitcpio -P`
     - `systemctl enable systemd-boot-update.service`
     - `bootctl update`
13. Install applications and create a user.
     - Uncomment the following lines in `/etc/pacman.conf`:
       ````
       Color
       ParallelDownloads = 5
       [multilib]
       Include = /etc/pacman.d/mirrorlist
       ````
     - ````
       pacman -Syu networkmanager gptfdisk ttf-dejavu sbctl polkit network-manager-applet dialog wpa_supplicant \
       mtools dosfstools linux-headers avahi xdg-user-dirs xdg-utils gvfs gvfs-smb nfs-utils inetutils dnsutils \
       bluez bluez-utils cups hplip alsa-utils pipewire pipewire-alsa pipewire-pulse pipewire-jack bash-completion \
       openssh rsync reflector acpi acpi_call virt-manager qemu qemu-emulators-full edk2-ovmf bridge-utils dnsmasq \
       vde2 openbsd-netcat iptables-nft ipset firewalld flatpak sof-firmware nss-mdns acpid os-prober ntfs-3g \
       terminus-font etckeeper htop hwinfo cpupower neofetch expat powertop power-profiles-daemon p7zip zip unrar \
       unarchiver lzop lrzip xorg-server xf86-video-intel mesa lib32-mesa vulkan-intel lib32-vulkan-intel intel-media-driver
       ````
     - Enable services:
       ````
       systemctl enable NetworkManager
       systemctl enable bluetooth
       systemctl enable cups.service
       systemctl enable sshd
       systemctl enable avahi-daemon
       systemctl enable reflector.timer
       systemctl enable fstrim.timer
       systemctl enable libvirtd
       systemctl enable firewalld
       systemctl enable acpid
       ````
     - Add user: `useradd -m -g users -G sys,network,power,rfkill,video,storage,lp,input,audio,wheel,libvirt <username>`
     - Change their password: `echo <username>:<choose-a-password> | chpasswd`
     - Add them to sudoers: `echo "user ALL=(ALL) ALL" >> /etc/sudoers.d/user`
14. `exit`, `umount -R /mnt`, and `reboot`.

### Activate Setup Mode in BIOS

1. On reboot, press `delete` to enter BIOS.
2. Press the following keys in order to show hidden options:
   - `Shift`+`fn`+`Ctrl`+`Alt`+`F2`
3. Go to the "Security" tab and select "Secure Boot".
4. Select "Reset to Setup Mode".
5. Remove the USB installer.
6. Log into root.
7. `pacman -S sbctl`
8. Run `sbctl status`. If is says "Setup Mode X Enabled", then continue on. If it says "Setup Mode ✓ Disabled", then redo all steps in this section.
9. `systemd-cryptenroll --tpm2-device=list`
10. `systemd-cryptenroll /dev/nvme0n1p2 --recovery-key `. Enter the drive encryption password and save the shown recovery key.
11. Enroll a TPM2 key: `systemd-cryptenroll --tpm2-device=/dev/tpmrm0 --tpm2-pcrs=0+7 --tpm2-with-pin=yes /dev/disk/by-partlabel/cryptsystem`
    - Enter drive encryption password.
    - Choose a TPM2 key.
12. Reboot into the BIOS.
13. Go to "Security" and select "Secure Boot".
14. Switch secure boot to enabled.
15. Press `F10` to save and reboot.

You can now use the laptop.
