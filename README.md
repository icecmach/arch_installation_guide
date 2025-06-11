### YouTube Tutorial

Based on this [video tutorial](https://www.youtube.com/watch?v=Qgg5oNDylG8).
![image](https://github.com/user-attachments/assets/926911a5-a673-46e4-9a0c-c3b4a0b26d61)

###### 02 March 2025

# Arch Install [btrfs + encryption + timeshift + hyprland]

These installation instructions form the foundation of the Arch system that I use on my own machine. Always consult the official Arch wiki [install guide](https://wiki.archlinux.org/title/Installation_guide), sometimes you may find your preferences deviating from the the official guide, and so my intention here is to provide a walkthrough on setting up your own system with the following:

- [btrfs](https://btrfs.readthedocs.io/en/latest/): A feature rich, copy-on-write filesystem for Linux.
- [encryption](https://gitlab.com/cryptsetup/cryptsetup/): LUKS disk encryption based on the dm-crypt kernel module.
- [timeshift](https://github.com/linuxmint/timeshift): A system restore tool for Linux.
- [Hyprland](https://hyprland.org/): Dynamic tiling window manager.

## Step 1: Creating a bootable Arch media device

Here we will follow the Arch wiki:

1. Acquire an installation image [here](https://archlinux.org/download/)
2. Verify the signature on the downloaded Arch ISO image (1.2 of the installation guide)
3. Write your ISO to a USB (check out [this](https://www.scaler.com/topics/burn-linux-iso-to-usb/) guide)
4. Insert the USB into the device you intend to install Arch linux on and boot into the USB

## Step 2: Setting Up Our System with the Arch ISO

1. **[optional]** To ssh into the target machine you will need to:

- Create a password for the ISO root user with the **passwd** command
- Ensure that **ssh** is running with **systemctl status sshd** (if it isn't start it with **systemctl start ssdhd**)

2. Set the console keyboard layout (US by default):

```bash
# list available keymaps
localectl list-keymaps
# load the keymap
loadkeys <your keymap here>
```

3. **[optional]** set the font size with **setfont ter-132b**
4. Verify the UEFI boot mode **cat /sys/firmware/efi/fw_platform_size**. This installation is written for a system with a 62-bit x64 UEFI. This isn't required, but if you are on a different boot mode, consult section 1.6 of the official guide
5. Connect to the internet:

- I use the **iwctl** utility for this purpose
- Confirm that your connection is active with **ping -c 2 archlinux.org**

6. **[optional]** Obtain your IP Address with **ip addr show**, now you're ready to ssh into your target machine
7. Set the timezone (replace with your preferred timezone):

```bash
timedatectl list-timezones
timedatectl set-timezone America/Toronto
timedatectl set-ntp true
```

8. Partition your disk:

- list your partitions with **lsblk**
- delete the existing partitions on the target disk **[WARNING: your data will be lost]**

```bash
gdisk /dev/sda
?
o
Y
```

- create two partitions:
  > **NOTE:** The official Arch Linux installation guide suggests implementing a swap partition and you are welcome to take this route. You could also create a swap subvolume within BTRFS, however, snapshots will be disabled where a volume has an active swapfile. In my case, I have opted instead of `zram` which works by compressing data in RAM, thereby stretching your RAM further. zram is only active when your RAM is at capacity.
- **efi** = 500mb

```bash
n
ENTER
ENTER
+500M
ef00
```

- **main** = allocate all remaining space (or as otherwise fit for your specific case) noting that BTRFS doesn't require pre-defined partition sizes, but rather allocates dynamically through subvolumes which act in a similar fashion to partitions but don't require the physical division of the target disk.

```bash
n
ENTER
ENTER
ENTER
ENTER

w
Y
```

9. Format your main partition:

```bash
# setup encryption
cryptsetup luksFormat /dev/nvme0n1p2
# open your encrypted partition
cryptsetup luksOpen /dev/nvme0n1p2 main
# format your partition
mkfs.btrfs /dev/mapper/main
# mount your main partition for installation
mount /dev/mapper/main /mnt
# cd into the /mnt directory
cd /mnt
```

- create our subvolumes:

```bash
# root
btrfs subvolume create @
# home
btrfs subvolume create @home
```

> **NOTE:** you can create your own subvolume names, but make sure you know what you are doing, because these subvolumes will also be referenced later when taking snapshots with timeshift.

```bash
# go back to the original (root) directory
cd
# unmount our mnt partition
umount /mnt
# mount root subvolume
mount -o noatime,ssd,compress=zstd,space_cache=v2,discard=async,subvol=@ /dev/mapper/main /mnt
# create home mounting point and mount home subvolume
mkdir /mnt/home
mount -o noatime,ssd,compress=zstd,space_cache=v2,discard=async,subvol=@home /dev/mapper/main /mnt/home
```

10. format your efi partition:

```bash
mkfs.fat -F32 /dev/nvme0n1p1
```

- create boot mounting point and mount efi partition:
  ```bash
  mkdir /mnt/boot
  mount /dev/nvme0np1 /mnt/boot
  ```

11. Configure reflector

```bash
reflector -c Canada -a 12 --sort rate --save /etc/pacman.d/mirrorlist
```

12. Install base packages:

```bash
pacstrap /mnt base linux linux-headers linux-firmware
```

13. generate the file system table:

```bash
genfstab -U -p /mnt >> /mnt/etc/fstab
cat /mnt/etc/fstab
```

14. change root into the new system:

```bash
arch-chroot /mnt
```

You are now working from within in your new arch system - i.e. not from the ISO - and you will now see that your prompt start with **"#". Great work so far!**

## Step 3: Working Within Our New System

We are now working within our Arch system on our device, but it's important to note that we can't yet reboot our machine. Let's continue with a few steps that we need to repeat again (such as setting our root password, timezones, keymaps and language) given the previous settings were in the context of our ISO.

1. set your local time and locale on your system:

```bash
ln -sf /usr/share/zoneinfo/America/Toronto /etc/localtime
hwclock --systohc
pacman -S vim
```

- Execute **vim /etc/locale.gen** and uncomment your locale, example: **en_CA.UTF-8 UTF-8** write and exit and then run:
  ```bash
  locale-gen
  # for locale
  echo "LANG=en_CA.UTF-8" >> /etc/locale.conf
  # for keyboard
  echo "KEYMAP=en..." >> /etc/vconsole.conf
  ```

2. change the hostname

```bash
echo "arch-vm" >> /etc/hostname
```

3. set your root password: **passwd**
4. set up a new user (replace **ice** with your preferred username):

```bash
useradd -m -g users -G wheel ice
# give your user a password with
passwd ice
# add your user to the sudoers group
pacman -S sudo
echo "ice ALL=(ALL) ALL" >> /etc/sudoers.d/ice
```

Next, we will install all of the packages we need for our system. Refer to the bottom of this guide for a short summary on each package being installed. It's imperative to always know what you are doing, and what you are installing!

6. install the main packages that our system will use:

```bash
pacman -Syu base-devel btrfs-progs grub efibootmgr mtools networkmanager network-manager-applet openssh git iptables-nft ipset firewalld reflector acpid grub-btrfs
```

7. install the following based on the manufacturer of your CPU:

- **intel:**
  ```bash
  pacman -S intel-ucode
  ```
- **amd:**
  ```bash
  pacman -S amd-ucode
  ```

8. install your window manager of choice:

```bash
pacman -S hyprland
```

> **NOTE:** Any tiling window manager or graphical user environment can be installed at this stage.

9. install other useful packages:

```bash
pacman -S man-db man-pages texinfo bluez bluez-utils alsa-utils pipewire pipewire-pulse pipewire-jack sof-firmware ttf-firacode-nerd alacritty thunar tumbler
```

10. edit the mkinitcpio file for encrypt:

- **vim /etc/mkinitcpio.conf**
  - add `btrfs` to the MODULES: **MODULES=(btrfs)**
  - add encrypt to HOOKS (before filesystems): **encrypt filesystems fsck)**
- recreate the mkinitcpio.conf with **mkinitcpio -p linux**

11. setup grub for the bootloader so that the system can boot linux:

```bash
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB
grub-mkconfig -o /boot/grub/grub.cfg
```

- run **blkid** and obtain the UUID for the main partition
- edit the grub config **vim /etc/default/grub**
- **GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet cryptdevice=UUID=d33844ad-af1b-45c7-9a5c-cf21138744b4:main root=/dev/mapper/main**
- make the grub config with **grub-mkconfig -o /boot/grub/grub.cfg**

12. enable services:

```bash
systemctl enable NetworkManager
systemctl enable bluetooth
systemctl enable sshd
systemctl enable firewalld
systemctl enable reflector.timer
systemctl enable fstrim.timer
systemctl enable acpid
systemctl enable systemd-timesyncd.service
exit
```

Now for the moment of truth. Make sure you have followed these steps above carefully, then reboot your system with the **reboot** command.

## Step 4: Tweaking our new Arch system

When you boot up you will be presented with the grub bootloader menu, and then, once you have selected to boot into arch linux (or the timer has timed out and selected your default option) you will be prompted to enter your encryption password. Upon successful decryption, you will be presented with the login screen. Enter the password for the user you created earlier.

1. install [yay](https://github.com/Jguer/yay):

```bash
sudo pacman -S --needed base-devel
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
yay -Y --gendb
yay -Syu --devel
yay -Y --devel --save
```

2. install [timeshift](https://github.com/linuxmint/timeshift):

```bash
sudo pacman -S timeshift
sudo timeshift --list-devices
sudo timeshift --create --comments "20250303 Base Install" --tags D
sudo systemctl edit --full grub-btrfsd
# NOTE:
# rm : ExecStart=/usr/bin/grub-btrfsd --syslog /.snapshots
# add: ExecStart=/usr/bin/grub-btrfsd --syslog -t
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

3. Pacman hooks

```bash
[Trigger]
Operation = Install
Operation = Remove
Type = Package
Target = *
[Action]
When = PostTransaction
Exec = /bin/sh -c '/usr/bin/pacman -Qqet > /home/username/pkglist.txt'
```

4. Sensors for hardware monitoring, temperatures and fan speed

Install [lm_sensors](https://wiki.archlinux.org/title/Lm_sensors)

```bash
sudo pacman -S lm_sensors
# Asrock B650M Pro RS / B850M Pro RS / X870 Pro RS
sudo echo "options nct6775 force_id=0xd801" > /etc/modprobe.d/nct6775.conf
sudo modprobe nct6775
sudo sensors-detect --auto
# Check output of sensors
sensors
# Load nct6775.ko at boot
sudo echo "nct6775" > /etc/modules-load.d/nct6775.conf
```

## Hyprland configuration

...
