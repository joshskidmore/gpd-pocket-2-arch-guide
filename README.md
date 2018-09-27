# GPD Pocket 2 Dual Boot - Arch / Windows 10 Install Guide
This guide overviews how to do a full wipe of the GPD Pocket 2, followed by a dual boot of
Arch Linux and Windows 10.

# Step 1: Use Arch USB to reformat drive

  1. Download standard Arch linux ISO
  2. Write ISO to USB drive
  3. Boot Pocket 2 and tap `DEL` key to enter BIOS
  4. In BIOS, navigate to `Save & Exit` tab and under `Boot Override`, select the USB device containing Arch
  5. Reformat `mmcblk0` disk

    Essentially, this is what we're going for:

        NAME              FSTYPE        SIZE    NOTES                       MOUNTPOINT
        mmcblk0
        ├─mmcblk0p1       vfat          512M    efi                         /boot
        ├─mmcblk0p2       luks+lvm2     60G     linux                       /
        ├─mmcblk0p[3-5]                 ~58G    windows

    It's critical that we make the first partition (mmcblk0p1) the EFI partition. For now, we're going
    to placehold our linux partition because the Windows 10 installer will create a bunch of bullshit
    partitions. For this guide, by placing the linux partition directly after the EFI partition, we
    can let Windows create its dumpster fire of partitions, and it won't effect us.

        gdisk /dev/mmcblk0
        # o ↵ to create a new empty GUID partition table (GPT)
        # y ↵ to confirm

        # n ↵ add a new partition
        # ↵ to select default partition number of 1
        # ↵ to select default start at first sector
        # +512M ↵ make that size partition for booting
        # ef00 ↵ EFI partition type

        # n ↵ add a new partition
        # ↵ to select default partition number of 2
        # ↵ to select default start at first sector
        # +60G ↵ allocate whatever size wanted for linux

        # we're intentionally leaving space for Windows 10 partitions

        # p ↵ if you want to check the partition layout
        # w ↵ to write changes to disk
        # y ↵ to confirm

  5. Format the EFI partition to vfat
  
        `mkfs.vfat /dev/mmcblk0p1`


# Step 2: Install Windows 10

  1. Create a Windows 10 install USB
  2. Boot Pocket 2 and tap `DEL` key to enter BIOS
  3. In BIOS, navigate to `Save & Exit` tab and under `Boot Override`, select the USB device containing the Windows 10 installer
  4. Install Windows 10. The only important thing is that in the disk partitioner, let Windows use the unused space at the end of
  the drive. The Windows installer will automatically place its EFI files into the EFI partition (`mmcblk0p1`). The installer will
  warn you that it has to create additional partitions which is normal.

Windows will take over the Pocket 2 and act as if it's rolling solo and that's completely fine. It will reboot a couple times. It's
also normal that Windows 10 is rotated right until Windows drivers are installed and rotation is configured.


# Step 3: Let's Install Arch
Remove the Windows 10 install USB stick and, just like step one, insert the Arch install USB and boot into Arch.

## Configure LUKS + LVM2 partitions on `mmcblk0p2`

    # Encrypt /dev/mmcblk0p2
    # This step will ask for a password which will be used to unlock the partition on boot. Make sure you know the password.
    cryptsetup luksFormat -v -s 512 -h sha512 /dev/mmcblk0p2

    # Open/mount encrypted disk
    # Upon unlocking, this will mount the unlocked disk to /dev/mapper/luks.
    cryptsetup luksOpen /dev/mmcblk0p2 luks

    # Create LVM2 Physical Volume (PV) on /dev/mapper/luks
    pvcreate /dev/mapper/luks

    # Create LVM2 Volume Group (VG) on /dev/mapper/luks
    vgcreate rootvg /dev/mapper/luks

    # Create LVM2 Logical Volume (LV) on rootvg
    # This will create an LVM logical volume at /dev/mapper/rootvg-root.
    lvcreate -n root -l 100%FREE rootvg

    # Finally, format the logical volume to ext4
    mkfs.ext4 /dev/mapper/rootvg-root

## Wirelessly connect to the internet
Internet is needed to download packages.

    wifi-menu

## Mount, pacstrap and prepare for arch-chroot

    # Mount the ext4-formatted root LV
    # Note: /mnt becomes your actual Arch install.
    mount /dev/mapper/rootvg-root /mnt

    # Make the boot directory to mount the EFI (mmcblk0p1) partition
    mkdir /mnt/boot

    # Mount the EFI parition
    mount /dev/mmcblk0p1 /mnt/boot

    # Pacstrap the /mnt directory with utilities needed for arch-chroot
    pacstrap /mnt base base-devel dialog openssl-1.0 bash-completion git intel-ucode wpa_supplicant

    # Generate your fstab
    genfstab -pU /mnt >> /mnt/etc/fstab


## arch-chroot
`arch-chroot` drops you into your Arch filesystem in order to handle additional install tasks.

    arch-chroot /mnt /bin/bash

Awesome! You're now in your to-be Arch filesystem. Let's install some basic shit.

Set your hostname (yeah, this is the most difficult part .. at least for me)

    echo MYHOSTNAME > /mnt/etc/hostname

By now, you're probably sick of looking at tiny text, sideways. Let's fix the font size issue (on
reboot) by creating the file `/etc/vconsole.conf` and adding:

    FONT=latarcyrheb-sun32

Because we're using disk encryption and LVM2, we need to add and reorder mkinitcpio's hooks. Edit
`/etc/mkinitcpio.conf` and completely replace the line beginning with `HOOKS` with:

    HOOKS=(base systemd autodetect keyboard sd-vconsole modconf block sd-encrypt sd-lvm2 fsck filesystems)

With new HOOKS in-tow, regenerate your ramdisk and kernel using mkinitcpio:

    mkinitcpio -P

We'll be using `systemd-boot` as a better grub replacement. First, install systemd-booti using the
following command. The command will create a new EFI entry named `Linux Boot Manager` which will sit
alongside Windows' `Windows Boot Manager`.

    bootctl install

Windows will be included as a choice on boot, and any additional entries must be created in the
directory `/boot/loader/entries/`. We're going to create a default entry for `arch`.

First, we need the block or UUID of the mmcblk0p2 LUKS partition for our `arch` entry. Since
you're working sideways with little ass text, lets dump that ID right into the empty entry file:

    blkid | grep mmcblk0p2 | cut -d \" -f 2 > /boot/loader/entries/arch.conf

Next, add the following to the `/boot/loader/entries/arch.conf` entry. Replace `[UUID]` with the
UUID you dumped into the file. Remove the brackets.

    title Arch Linux
    linux /vmlinuz-linux
    initrd /intel-ucode.img
    initrd /initramfs-linux.img
    options rd.luks.name=[UUID]=luks root=/dev/mapper/rootvg-root rw fbcon=rotate:1

Of note is the kernel command line option `fbcon=rotate:1`. This will rotate any non-X content to the
right (after reboot).

We also need to edit (or possibly create) the file `/boot/loader/loader.conf`. This file handles
systemd-boot's general config options. Most of the following options should be self explanitory, but
one that might not be is `console-mode 2`. This option will force larger text during the handoff from
the BIOS to the kernel. The file should contain:

    default arch
    auto-firmware no
    timeout 3
    console-mode 2

Specify a timezone:

    ln -sf /usr/share/zoneinfo/US/Eastern /etc/localtime

Create `/etc/locale.conf` with the following:

    LANG=en_US.UTF-8
    LANGUAGE=en_US
    LC_ALL=C

Edit the file `/etc/locale.gen` and uncomment the proper locale line:

    en_US.UTF-8 UTF-8

Then, generate the locale using the following command:

    locale-gen

Set a root password:

    passwd

Create your user and set a password:

    useradd -m -g users -G wheel,storage,power -s /bin/bash [USERNAME]
    passwd [USERNAME]

If you wish to run sudo'd commands without entering a password, run `visudo` and
uncomment the line:

    %wheel ALL=(ALL) NOPASSWD: ALL

Ok, we're done in `arch-chroot`! Let's unmount and restart.

    # Exit arch-chroot and back into the default shell
    exit

    # Recursively unmount /mnt
    umount -R /mnt

    # Remove USB stick and reboot
    reboot


# Step 4: From Text to X
Upon reboot, immediately after the BIOS, you should see `systemd-boot` prompt you with two
options: `arch` and `Windows Boot Manager`. All text should be sized decently and rotated
correctly. If this is not the case, something has been missed.

You should also be prompted to enter the LUKS partition decryption password that you created
earlier. If entered correctly, the system will continue to load and provide a terminal login.

Login with the user you created, *not* `root`.

## Wirelessly connect to the internet

    wifi-menu

## Add some repos to pacman

Edit `/etc/pacman.conf` and uncomment these lines:

    [multilib]
    Include = /etc/pacman.d/mirrorlists

In `/etc/pacman.conf`, append these lines to the bottom

    [archlinuxfr]
    SigLevel = Never
    Server = http://repo.archlinux.fr/$arch

## Sync pacman to retrieve latest package information

    sudo pacman -Sy

## Build and install `yay` package manager tool in order to install AUR packages
If you prefer another AUR/pacman tool, feel free to replace this step and `yay` with that tool.

    git clone https://aur.archlinux.org/yay.git
    cd yay
    makepkg -si

    # after install, remove the yay build folder
    cd ..; rm -rf yay

    ## Sync yay
    yay -Sy

## thermald
`thermald` protects your Pocket 2 from overheating.

    # Install
    yay -S thermald

    # Enable + start
    sudo systemctl enable thermald.service
    sudo systemctl start thermald.service

## More timezone setup

    sudo tzselect

## NetworkManager

    # Install
    yay -S networkmanager network-manager-applet nm-connection-editor

    # Enable + start
    sudo systemctl enable NetworkManager
    sudo systemctl start NetworkManager

    # Connect to a wireless network using nmtui
    nmtui
      # Activate a Connection -> (choose wi-fi connection) -> Activate -> Back -> Quit

## Sound

    # Install pulseaudio packages
    yay -S pulseaudio pulseaudio-alsa pulseaudio-bluetooth pulseaudio-ctl

    # (optional) Install nice traybar utils to work with audio
    yay -S pasystray-gtk3-standalone pavucontrol

## Bluetooth

    # Install
    yay -S bluez bluez-utils bluez-tools

    # Enable + start
    sudo systemctl enable bluetooth
    sudo systemctl start bluetooth

    # (optional) Install nice traybar utils
    yay -S blueman blueberry

## TLP
TLP helps reduce power by setting some sane power defaults.

    # Install
    yay -S tlp

    # Enable + start
    sudo systemctl enable tlp
    sudo systemctl start tlp

## Xorg

### Install Xorg packages

    yay -S xorg-server xorg-xev xorg-xinit xorg-xkill xorg-xmodmap xorg-xprop xorg-xrandr xorg-xrdb xorg-xset xinit-xsession

### Create Pocket 2 Xorg configs

Create Intel xorg config at `/etc/X11/xorg.conf.d/20-intel.conf` with the following:

    Section "Device"
      Identifier    "Intel Graphics"
      Driver        "intel"
      Option        "AccelMethod"            "sna"
      Option        "TearFree"               "true"
      Option        "DRI"                    "3"
    EndSection

Create display config at `/etc/X11/xorg.conf.d/30-display.conf` with the following:

    Section "Monitor"
      Identifier    "eDP1"
      Option        "Rotate"                 "right"
    EndSection

Create touchscreen config at `/etc/X11/xorg.conf.d/99-touchscreen.conf` with the following:

    Section "InputClass"
      Identifier    "calibration"
      MatchProduct  "Goodix Capacitive TouchScreen"
      Option        "TransformationMatrix"   "0 1 0 -1 0 1 0 0 1"
    EndSection

### Install Intel video drivers

    yay -S xf86-video-intel


## XFCE4

    yay -S xfce4 xfce4-goodies
    # hit enter to install all the goodies

## Create `~/.xinitrc`

    # Simple hack to allow mouse scrolling when right click button is held down
    xinput --set-prop pointer:"HAILUCK CO.,LTD USB KEYBOARD Mouse" "libinput Middle Emulation Enabled" 1
    xinput --set-prop pointer:"HAILUCK CO.,LTD USB KEYBOARD Mouse" "libinput Button Scrolling Button" 3
    xinput --set-prop pointer:"HAILUCK CO.,LTD USB KEYBOARD Mouse" "libinput Scroll Method Enabled" 0 0 1

    # Start XFCE4
    (xset s 120) & exec startxfce4

## Start X

    startx
