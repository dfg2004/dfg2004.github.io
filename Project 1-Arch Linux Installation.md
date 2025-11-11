### 1 Pre-installation
[Arch Linux Installation Guide](https://wiki.archlinux.org/title/Installation_guide)

**1.1 Acquire an installation image**
I downloaded arch-2025.10.01-x86_64.iso from adectra.com and the PGP signature from 
[Arch Linux Iso Download](https://archlinux.org/download/#download-mirrors)

**1.2 Verify signature**
To verify the signature I went to the directory where I put my iso and used 7zip to check the SHA256 hash of my file against the hash provided on the website
- To do this I first right-clicked on my iso file, clicked Show More Options --> 7Zip --> CRC SHA --> SHA-256

**1.4 Boot the live environment**
I used the New Virtual Machine option in VMware and gave it my Arch iso, but it didn't recognize the operating system, so I set it to a 'Other Linux 6.x kernel 64-bit' before launching the VM --> booted into a Zsh shell prompt.

**1.5 Set the console keyboard layout and font**
To set my keymap to us I used the command `loadkeys us` and set the font to Terminus p.24 with the command `setfont ter-p24b.psf.gz`.

**1.6 Verify the boot mode**
To verify the boot mode I used the command `cat /sys/firware/efi/fw_platform_size`
- This gave me "No such file or directory"
- Had to power off the machine and go to VM --> Settings --> Options --> Advanced --> change firmware to UEFI instead of BIOS
- After I did this and powered on the VM, running the command gave "64", which means that the system is booted in UEFI mode and has a 64-bit x64 UEFI

**1.7 Connect to the internet**
I was able to verify my connection with ping: `ping ping.archlinux.org`.

**1.8 Update the system clock**
To update the system clock, I ensured the system clock was synchronized with the command `timedatectl`.

**1.9 Partition the disks**
When partitioning the disks, I ran into a problem figuring out how I needed to partition it (dos or gpt), so I ran `fdisk /dev/sda` in bash and then `p` in css to find that my Disklabel type: dos, meaning my disk has the Master Boot Record (MBR) partition table and UEFI systems require a GUID partition table, so I had to convert it. To do this I followed these steps:
	1. Run `ls /sys/firmware/efi/efivars` to confirm I'm booted in UEFI mode
	2. Run `lsblk` to identify disk, which gave me "sda"
	3. Run `fdisk /dev/sda` to work with this disk
	4. To create a GPT partition table: `g`
	5. To create a new partition: `n`
	6. To assign it as a primary: `p`
	7. Press enter to select default partition for the EFI partition and root partition (1 & 2)
	8. Press enter to get default sector for each partition (1 & 2)
	9. I typed `+1G` to allocate 1 GiB for the /sda1 partition (my EFI partition) and just pressed enter to give the /sda2 partition (my root partition) the remainder of the device (7 GiB)
	10. To change the partition type from 'Linux' to 'EFI', I entered `t` and then `EF` for UEFI

**1.10 Format the partitions** 
Each partition needs to be formatted with the appropriate file system. Since I created an EFI system partition, I need to format my /sda1 partition to FAT32 with mkfs.fat `mkfs.fat -F32 /dev/sda1`. For the root partition, I created an Ext4 file system using `mkfs.ext4 /dev/sda2`.

**1.11 Mount the file systems**
The root volume needs to be mounted to /mnt with the command `mount /dev/sda2 /mnt` and the EFI system partition needs to be mounted with the command `mount --mkdir /dev/sda1 /mnt/boot`.

### 2 Installation

**2.1 Select the mirrors** 
I decided to use reflector to do this, so I looked up reflector commands to create a custom mirrorlist. Looking up these flags gave varying results, but reflector --help showed the available operations. I followed these steps to download reflector and create a custom mirrorlist:
1. Downloaded reflector with `pacman -Sy reflector` command
	1. --sort rate - sorts them by download speed
	2. --save - saves the result to the mirrorlist file
	3. -c - specify countries
	4. -l - list the number of mirrors 
2. I used to command `reflector --country "United States" --latest 3 --sort rate --save /etc/pacman.d/mirrorlist`

**2.2 Install essential packages**
After reviewing the software recommended by the installation guide, I decided on these packages:
1. intel-ucode - CPU microcode for hardware bug and security fixes
2. sof-firmware - for onboard audio, linux-firmware-marvell for Marvell wireless
3. networkmanager - network management
4. iwd - authentication software for Wi-Fi
5. modemmanager - for mobile broadband connections
6. nano - console text editor 
7. man-db, man-pages, texinfo - packages for accessing documentation in man and info pages
8. userspace utilities for file systems: I used ext4 for my root system, so I installed e2fsprogs, and I used exFAT/FAT32 for my EFI partition so I installed exfatprogs
9. RAID - for accessing and managing mdadm
10. LVM - for accessing and managing lvm2
11. The basic installation command `pacstrap -K /mnt base linux linux-firmware` installs:
	1. base - minimal Arch Linux userland
	2. linux - the latest kernel
	3. linux-firmware - firmware for common hardware
	4. -k copies your pacman keyring into the new system
Altogether the command turned into `pacstrap -K /mnt base linux linux-firmware intel-ucode sof-firmware networkmanager iwd modemmanager nano man-db man-pages texinfo e2fsprogs exfatprogs mdadm lvm2`.

### 3 Configure the system

**3.1 Fstab**
To get needed file systems mounted on startup, generate an fstab file using -U or -L to define by UUID or labels `genfstab -U /mnt >> /mnt/etc/fstab`.

**3.2 Chroot**
To directly interact with the new system's environment, tools, and configurations for the next steps as if you were booted in, you have to change root into the new system with the command `arch-chroot /mnt`.

**3.3 Time**
I set time zone with the command `ln -sf /usr/share/zoneinfo/America/Chicago /etc/localtime` and then generated the /etc/adjtime file with the command `hwclock systohc` which assumes hardware clock runs on UTC.

**3.4 Localization** 
In order to use the correct region and language specific formatting, I had to edit the /etc/locale.gen file and uncomment "en_US.UTF-8 UTF-8" before generating the locale. After this I need to create the locale.conf file and set the LANG variable as well as the vconsole.conf with the KEYMAP variable. I followed these steps to do that:
1. Generated the locales by running `nano /etc/locale.gen`
2. Uncomment en_US.UTF-8 UTF-8 --> save with CTRL+O -> enter -> CTRL+X
3. Generate the locales: locale-gen
4. Create locale.conf with `nano /etc/locale.conf` -> manually type LANG=en_US.UTF-8 -> CTRL+O -> enter -> CTRL+X
5. Create vconsole.conf with `nano /etc/vconsole/conf` -> KEYMAP=us

**3.5 Network Configuration**
I assigned my computer a hostname with the command `nano /etc/hostname -> DFG-3353`.

**3.6 Initramfs**`
For LVM, system encryption or RAID, I needed to modify mkinitcpio.conf and recreate the initramfs. To do this I ran the following commands: `nano /etc/mkinitcpio.conf` --> `mkinitcpio -P`.

**3.7 Root password**
I set my passwd with the command `passwd`.

**3.8 Boot loader**
I decided to install GRUB boot loader, so I installed grub and configured it with these commands:
	1. Install with pacman: `pacman -Sy grub efibootmgr`
	2. Install with grub-install: `grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB`
	3.Configure with grub-mkconfig: `grub-mkconfig -o /boot/grub/grub.cfg`
[GRUB Install](https://wiki.archlinux.org/title/GRUB)
### 4 Reboot

To reboot I exited the chroot environment with `exit`, unmounted all the partitions with the `umount -R /mnt` command as it recommended and restarted my machine with `reboot`.

---
### Arch Linux Modifications

**Installing a GUI-based desktop environment:** 
I first tried installing KDE Plasma which turned out to be too big for my partition, and then I tried to install MATE DE which did not include a terminal application. Ultimately, I decided to install Xfce. I installed the xfce4 and xfce4-goodies package with the command `sudo pacman -S xfce4 xfce4-goodies` before running into an error where some things were unable to be retrieved. To solve this I ran the command `sudo pacman -Syu` [Pacman Commands](https://www.geeksforgeeks.org/linux-unix/pacman-command-in-arch-linux/)
- S - sychronize the local database with main database
- y - update the local catch on the system
- u - tells pacman to upgrade the packages
This command synchronizes your databases ad updates your system/upgrades packages . After this the command worked and I rebooted my system

**Creating Users**
A new installation leaves you with only the superuser account "root" and using this for long periods of time is insecure because it can be exposed.
1. To create a new user I needed to do the command `useradd -m -G wheel -s /bin/bash username` (1) username=danie (2) username=codi
	1. -m - home directory is created
	2. -G - groups user is a member of
	3. -s - path to user's login shell
2. I then set the passwords for both of our accounts with the command `passwd username` (1) username=danie (2) username=codi
3. Need to uncomment "%wheel All=(ALL:ALL) ALL" in the /etc/sudoers file. This file has to be edited with visudo (says in file) --> had to download vi to use visudo: `sudo pacman -S vi`. To use visudo, you run `sudo visudo` and it opens the /etc/sudoers file.
	1. In order to save your changes in visudo you have to hit escape and then type `:wq`
[Users and Groups in Arch](https://wiki.archlinux.org/title/Users_and_groups) [Giving User Root Permissions](https://unix.stackexchange.com/questions/46849/how-to-give-user-root-permissions) [Sudo Arch](https://wiki.archlinux.org/title/Sudo)

**Installing LightDM**
To install LightDM you need to download the lightdm package and xorg-server. Along with these, you need to install a greeter like LightDM GTK Greeter to go with it, a greeter being a GUI that prompts the user for credentials among other things. To do this, I used to the command `sudo pacman -S xorg-server lightdm lightdm-gtk-greeter`. The last thing I needed to do was enable lightdm.service which I did with the command `sudo systemctl enable lightdm.service`.
[Arch LightDM Install](https://wiki.archlinux.org/title/LightDM)

**Installing fish**
To install fish I just needed to run the command `sudo pacman -S fish`.
[Install Fish](https://wiki.archlinux.org/title/Fish)

**Installing ssh**
To install ssh, I ran the command `sudo pacman -S openssh`. The manage daemons (computer programs that run in background), I ran `sudo systemctl enable sshd.service` and `sudo systemctl start sshd.service`.
- To check if it is running do `sudo systemctl status sshd`

**Terminal Color-coding**
To edit the bash prompt, you have to edit the bash configuration file: `nano ~/.bashrc`
- Original: `PS1='\u@\h:\w\$'`
- Colored: `PS1='[\e[35m\u@\h \w]\$ \e[0m'`
	- `\e[35m` - the magenta color
	- \u - displays username
	- \h - displays hostname
	- \w - displays current working directory
	- \$ - displays correct sign for type of user
	- `\e[0m` - makes strictly the prompt this color
- `source ~/.bashrc` - applies changes to bash
[Arch Bash Prompt Customization](https://wiki.archlinux.org/title/Bash/Prompt_customization) [Changing Color of Prompt](https://askubuntu.com/questions/689156/color-changing-in-terminal-specifically-my-input) [Possible Colors for Prompt](https://unix.stackexchange.com/questions/124407/what-color-codes-can-i-use-in-my-bash-ps1-prompt)

**Add Aliases**
To add an alias, you type "alias" and the new shortcut on the left-hand side, and on the right-hand side, you put the original command. The aliases I decided to add are:
- `alias h='history'`
- `alias j='jobs -l'
- `alias path='echo -e ${PATH//:/\\n}'`
- `alias now='date+"%T"'`
- `alias date='date +"%d-%m-%Y"`
[Example of Aliases](https://www.cyberciti.biz/tips/bash-aliases-mac-centos-linux-unix.html)

---
### Problems & Solutions

**MATE DE & GDM**
I ran into a problem with MATE DE where when I installed it, it did not have a terminal application for me to run any commands. To get back to terminal display I had to logout and do CTRL+FN+ALT+F2. I decided the best thing to do would be to uninstall the DE and try a different one. These are the steps I followed:
- Tried to do `sudo pacman -R mate` --> this gave me the error that mate was required for applications like marco and caja
- I then tried to uninstall those separately and marco specifically threw an error that marco was required for mate-control-center 
- I then found out that I could tack on more specifications for remove that allow it to uninstall mate. The command `sudo pacman -Rns mate` works.
	- -R - remove
	- -n - remove configuration files
	- -s - remove unneeded dependencies

I thought uninstalling GDM and installing LightDM for Xfce would be best since GDM is mostly tied with GNOME [Display Manager Association](https://www.reddit.com/r/debian/comments/6fjg18/what_is_a_display_manager_do_i_need_one_for_xfce4/?rdt=53583) 
- `sudo pacman -R gdm`

**VMware Station Connection Error**
Ran into the error **"VMware Workstation cannot connect to the virtual machine. Make sure you have rights to run the program, access all directories the program uses, and access all directories for temporary files."** when trying to boot the VM up. Found solution here: [VM Connection Solution](https://stackoverflow.com/questions/29328292/error-while-powering-on-vmware-player-cannot-connect-to-the-virtual-machine)
After I did this I ran into a problem with DNS --> the services that you manually stop with this solution do not automatically start again when you reopen VMware. You need to manually start them.

**Network Connectivity**
Had a network connectivity problem when I first booted up Arch Linux: had to do `systemctl start NetworkManager` which I had installed before rebooting and then `systemctl enable NetworkManager` before checking that the Wi-Fi was connected with the command `nmcli device`

**Installation Error**
Needed sudo to install a package but it wasn't a built-in command so I ran `pacman -S sudo`.