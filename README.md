# Arch Linux Installation Guide

These are the steps necessary to install [Arch Linux](https://www.archlinux.org/) with [LUKS](https://en.wikipedia.org/wiki/Linux_Unified_Key_Setup) disk encryption using [Logical Volume Manager](https://en.wikipedia.org/wiki/Logical_Volume_Manager_(Linux)) booting with [UEFI](https://en.wikipedia.org/wiki/Unified_Extensible_Firmware_Interface).

---

## Prepare Installation Media

[Download](https://www.archlinux.org/download/) the ISO and create a bootable USB drive. The simplest way to create bootable media on Linux is using the dd command:

    sudo dd bs=4M if=/path_to_arch_.iso of=/dev/sdX && sync

If you prefer a graphical interface, I've heard good things about [Etcher](https://etcher.io/) and it runs on Linux, Mac, and Windows. Alternately you can use UNetbootin (on Mac or Windows) or Rufus on Windows.

---

## BIOS Configuration

Hold F12 (or whatever key is used on your system) during startup to access bios. Then...

### Turn UEFI on

Make sure UEFI is on. Most modern systems use UEFI, so it's generally on by default.

### Disable Secure Boot

If secure boot is enabled it must be turned off since Linux boot loaders don't typically have digital signatures. Note that if you intend on running a dual-boot system with Windows and Linux you won't be able to use BitLocker disk encryption on the partition containing Windows, as it requires secure boot.

### Disable Fast Startup Mode

If you are dual booting with Windows turn off Fast Startup. This feature puts Windows into hibernation when you power off. Because some systems are still active during hibernation, booting into Linux can cause various nasty problems.


---

### Boot Arch Linux from the USB drive
Hold F12 (or whatever key is used on your system) during startup to access startup menu. Select the USB drive and boot into Arch.

---

### Establish an internet connection
The most reliable way is to use a wired connection, as Arch is setup by default to connect to DHCP. However, you can usually get WiFi working by running:

	wifi-menu

To test your connection:
	
	ping -c 3 www.google.com


---

## Drive setup

__IMPORTANT:__ Make sure you change the device nodes in all of the examples to your particular nodes. For example, in this document I might use:

	/dev/sd*

But on my Dell XPS, since it uses PCIe storage, an actual node would be called:

	/dev/nvme0n1p1


### To view your disc information:

	fdisk -l


### Delete existing disk partitions
This step is only necessary if you are using a drive with existing partitions. If you are installing onto a drive with unallocated space, skip this step. To remove partitions you can use parted:

	parted -s /dev/sd* rm 1
	parted -s /dev/sd* rm 2
	parted -s /dev/sd* rm 3
	etc.


### Zero the hard drive with random data

Not really necessary, but I like knowing that I'm using a drive with no data. Here's how to do it using dd:

	dd if=/dev/urandom of=/dev/sd* status=progress

Or if you're paranoid you can use a multi-pass tool like shred.

	shred -vfz -n 5 /dev/sd*



### Partition the drive
There are a number of tools available to acomplish this. This is how to do it using parted.

__NOTE:__ _Since we're using LVM we only need two drive partitions. The first is a boot partition, the second is the root partition where our LVM will live._ To launch parted on a particular drive node use:

    parted /dev/sd*

Then run the following commands:

    (parted) mklabel gpt
	(parted) mkpart primary 1MiB 512MiB name 1 boot
	(parted) set 1 boot on
	(parted) mkpart primary 512MiB 100% name 2 root
	(parted) quit


---

## Disk Encryption
Before we setup our LVM we need to encrypt the root partition we just 


### Encrypt the root partition 

	cryptsetup luksFormat -v -s 512 -h sha512 /dev/mapper/lvm-root


### Decrypt the newly encrypted partition
	cryptsetup open /dev/mapper/lvm-root root


---



### Create a physical volume on the root partition
	pvcreate /dev/sd*


### Create a volume group

__NOTE:__ I usually name the volume group __lvm__. If you use something else you'll need to replace it in the next three steps.

	vgcreate lvm /dev/sd*

### Create logical volumes for swap and root

__NOTE:__ I typically only use two volumes (swap and root) rather than 3 (swap, root, and home) so I can encrypt the entire root volume with my home directory in it. If you prefer three volumes then you'll need to adjust all the disc operations that follow. And you'll also likely encrypt your home folder rather than root.

__ALSO:__ _The amount of swap space is a function of how much ram you have. The minimum, assuming you have at least 8GB of RAM, should be 4GB, up to 1.5 times RAM (ie, 8GB RAM = 12GB swap) if you typically run multiple RAM intensive processes simultaneously. The average user won't need much swap._

The sizes below can be specified in megabytes (100M) or gigs (10G)

	lvcreate -n swap -L 500M lvm
	lvcreate -n root -l 100%FREE lvm



### Create filesystems on the two partitions

	mkfs.vfat -F32 /dev/sda1
	mkfs.ext4 /dev/mapper/root


### Mount the volumes

	mount /dev/mapper/root /mnt

	mkdir /mnt/boot
	mount /dev/sda1 /mnt/boot

---

## Update mirrorlist
By default Arch has a selection of servers from various countries listed in the local mirrorlist. While you might get adequate results with the defaults, to ensure the best possible download speeds it's recommended that you update the mirrorlist with servers from your country. To do that you use an application called reflector.

### Install reflector:
Note that reflector has two dependencies (rsync and curl) which should be installed by default in Arch, but to be safe we specify them:

	sudo pacman -S reflector rsync curl


### Backup your local mirrorlist

	sudo cp /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist.bak

### Create the new mirrolist
Note: If you are in a different country change "United States" to your country.

	sudo reflector --verbose --country 'United States' -l 5 --sort rate --save /etc/pacman.d/mirrorlist

---

## Install Arch Linux
Yay! 

	pacstrap -i /mnt base base-devel

---


### Configure fstab
We now need to update the filesystem table on the new installation. Fstab contains the association between filesystems and mountpoints. You won't be able to boot without it. To update it run:


	genfstab -U -p /mnt >> /mnt/etc/fstab

You can verify fstab with:

	cat /mnt/etc/fstab

---

## Change to your root directory
Since we're still booted on the USB drive, in order to configure our new system we need to change root:

	arch-chroot /mnt

---

## Install and configure bootloader
While there are various bootloaders that may be used, since the Linux kernel has a built-in EFI image, all we need is a way to execute it. For that we will install systemd-boot, a minimalist boot manager:

	bootctl --path=/boot install

### Update the loader.conf file
Using nano we can edit the config file:

	nano /boot/loader/loader.conf

Make sure that only the following lines are in the file:

	default arch
	timeout 3
	editor 0

__Notes:__ The timeout setting is the number of seconds the menu is displayed. The editor setting determines whether the kernel parameters editor is accessible. For security reasons we disable this.


### Get the UUID for root
In the next step we will update the boot loader config file. But first, we need to determine the UUID of our root partition. In order to get the UUID you first need to know what device node root is on. Look it up using:

	fdisk -l

The device node will be something like

	/dev/sda2

You can now get the UUID that corrisponds to the root node you just looked up using:

	ls -l /dev/disk/by-partuuid/

Or alternately you can look it with:

	blkid -s PARTUUID -o value /dev/sda2


### Update the arch.conf file
Armed with the UUID we just gathered in the previous step we can now insert it into the arch config file. Open the file using:

	nano /boot/loader/entries/arch.conf

And add the following info. Make sure to replace __YOUR-UUID__ with the ID gathered previously, and if your root drive is formatted with something other than ext4 update that info as well.


	title   Arch Linux
	linux   /vmlinuz-linux
	initrd  /initramfs-linux.img
	options root=PARTUUID=YOUR-UUID rootfstype=ext4 add_efi_memmap

	### Update the bootloader

		bootctl update

---

### Add NVMe to mkinitcpio

This step is only necessary if your computer is running PCIe storage rather than SATA. NVMe is a specification for accessing SSDs attached through the PCI Express bus. The Linux kernel includes an NVMe driver, so we just need to tell the kernel to load it. This is done by updating the MODULES variable in mkinitcpio (which is responsible for creating the initial ramdisk).

	nano /etc/nkintcpio.conf

Add __nvme__ to the MODULES variable:

	MODULES="nvme"

Now update the initramfs image with our module change:

	mkinitcpio -p linux

---

## Set language

Open the locale.gen file and uncomment your preferred language (I'm using en_US.UTF-8):

	nano /etc/local.gen

Now save the file and generate the locale:

	locale-gen

Add your language choice to the locale.conf file:

	echo LANG=en_US.UTF-8 > /etc/locale.conf

Export the language as an environmental shell variable:

	export LANG=en_US.UTF-8

---

## Set Timezone

Invoke this command to be prompted to find your timezone:

	tzselect

Now, use the provided TZ to create a symbolic link to /etc/localtime:

	ln -s /usr/share/zoneinfo/America/Denver /etc/localtime

Update the hardware clock:

	hwclock --systohc --utc

---

## Set hostname
This is the name of your computer. Note: Change "arch" to whatever you want your host to be.
	
	echo arch > /etc/hostname

---

## Set root password:

	passwd

---

## Create the user account:

	useradd -m -G wheel,users -s /bin/bash <username>

### Set password for user

	passwd <username>


---

## Grant user sudo powers
Install sudo:

	pacman -S sudo

Then run the following command, which will open the sudoers file:

	EDITOR=nano visudo

Find this line and uncomment:

	%wheel ALL=(ALL) ALL

---

## Enable multilib repositories and Yaourt

First we need to edit the pacman.conf file:

	nano /etc/pacman.conf

Uncomment the following lines:

	[multilib]
	Include = /etc/pacman.d/mirrorlist

Then add these lines for yaourt:

	[archlinuxfr]
	SigLevel = Never
	Server = http://repo.archlinux.fr/$arch

Save the file and exit.

### Refresh the package databases:

	pacman -Syy

### Install Yaourt

	sudo pacman -S yaourt

---

## Update all packages
The installation is basically done so we now update all isntalled packages:

	pacman -Syu

---

## Cross your fingers and reboot
We should now have a working Arch Linux installation. It doesn't have a desktop environtment or any applications yet, but the base installation is done. Run the reboot command and remove the USB drive:

	reboot

