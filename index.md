## Preliminary Setup

The first step in booting Arch was to change the boot mode from BIOS to UEFI by editing the `Arch.vmx` configuration file, then verify it by listing the UEFI configuration variables with ```ls /sys/firmware/efi/efivars```. If this returns data, the boot mode is successfully set to UEFI.

After that, it is necessary to connect to a wireless network. 

The first step is to verify that a network interface exists and is enabled, using the command ```ip link```. If you are using a wired ethernet interface, you should see a `link/ether` device listed as one of the network devices. To test whether you actually have a network connection, use the `ping` command to verify that you can connect to an external website, such as ```ping utulsa.edu```. 

After verifying that you have a network connection, the next step is to synchronize your local time and date server with an online one, by setting network time synchronization to `true` using the `timedate` control module, ```timedatectl set-ntp true```. You should verify that the time and date are correctly updated with ```timedatectl status``` afterwards.

----

## Disk Partitioning

Before partitioning out a disk, it is first necessary to list the hardware devices installed on the system, using ```lsblk```. For me, this command returned three devices: 
- `loop0` (the installation ISO file)
- `sda` (the 20G of disk space allocated by the VM)
- `sr0` (the boot manager currently open)

Given that `sda` is the only actual disk space available, the next step is to partition it out. Since I am running in UEFI mode, Arch recommends the following partitions:
- An EFI partition 
- A root directory
- A swap file

I partitioned out my disk using the `gdisk` command, as GPT is a more modern and efficient table scheme than MBR. The following is the process I used:
1. Enter the `gdisk` program on the drive, using ```gdisk /dev/sda```
2. Create a new GUID Partition Table using the `o` command
3. Create three new partitions
   - EFI Partition
      1. Create a new partition using the `n` command
      2. Assign the partition ID `1`
      3. Allow Arch to pick the default first sector for the partition
      4. Allocate 300MiB (recommended to have at least 260MiB) by setting the last sector to be `+300M` bytes
      5. Assign the GUID code `EF00`, to make it a EFI System Partition
   - Swap Partition
      1. Create a new partition using the `n` command
      2. Assign the partition ID `2`
      3. Allow Arch to pick the default first sector for the partition
      4. Allocate 4GiB of swap space by setting the last sector to be `+4G` bytes
      5. Assign the GUID code `8200`, to make it a Linux Swap partition
   - Root Partition
      1. Create a new partition using the `n` command
      2. Assign the partition ID `3`
      3. Allow Arch to pick the default first sector for the partition
      4. Allow Arch to pick the default last sector for the partition, which will allocate the remaining space as the root partition
      5. Assign the default GUID code `8300`, to make it a Linux Filesystem partition
   4. Write this created partition table to the disk using the `w` command

After creating the partitions, the next step is to format them. 
- The root partition, `sda3`, was formatted in `ext4` file system using ```mkfs.ext4 /dev/sda3`
- The swap partition, `sda2`, was initialized using ```mkswap /dev/sda2```
- Because I am using EFI system with the GRUB bootloader, the EFI partition, `sda1`, was formatted in `FAT32` using ```mkfs.fat -F32 /dev/sda1```

Once the file systems are formatted, the next step is to mount the partitions. I mounted the root volume using ```mount /dev/sda3 /mnt```. I then created an EFI mountpoint using ```mkdir /mnt/efi```, and mounted the EFI volume using ```mount /dev/sda1 /mnt/efi```. Finally, I enabled the swap partition using ```swapon /dev/sda2```.

----
## Installing packages

I verified that the list of mirror servers was acceptable using ```cat /etc/pacman.d/mirrorlist```, then installed the `base`, `kernel`, `nano`, `openssh`, `sudo`, and `firmware` packages, using ```pacstrap /mnt base linux linux-firmware nano openssh sudo```

After that, I generated a file system table using ```genfstab -U /mnt >> /mnt/etc/fstab```.

Finally, after the initial setup was finished, I entered the new system using ```arch-chroot /mnt```. 

----
## System Configuration

The first thing I did in the new system was to set the timezone. As we are in the Central Time Zone, I used the command ```ln -sf /usr/share/zoneinfo/America/Chicago /etc/localtime``` to link the time and date info for Chicago to the local time data, and ```hwclock --systohc``` to set the system clock. 

I then edited `/etc/locale.gen` to uncomment the `en_US.UTF-8` entry, then ran ```locale-gen``` to generate the locale files, and I added ```LANG=en_US.UTF-8``` to `/etc/locale.conf`.

To configure the network, I created `/etc/hostname`, and added my computer's hostname to it. I chose `halnine` as my hostname, after a somewhat famous computer you are likely familiar with, though I hope that my installation does not follow in its footsteps should I ever take it to space. I then added localhost and Hal to `/etc/hosts`, resulting in the following contents:

```
127.0.0.1   localhost
::1         localhost
127.0.1.1   halnine
```

As I like to use Cloudflare 1.1 as a DNS server, I modified `/etc/resolv.conf` to use
```
nameserver 1.0.0.1 
nameserver 1.1.1.1 
``` 
instead of the default DNS server.

I then generated the ramdisk using ```mkinitcpio -P```, and set a root password using ```passwd``` *(not telling you what it is)*

The final thing I did in the configuration step was to install the GRUB bootloader, by first installing it with ```pacman -S grub efibootmgr```. I then ran ```grub-install --target=x86_64-efi --efi-directory=/efi --bootloader-id=GRUB``` to install GRUB to the `/mnt/efi` partition created in `/dev/sda1` earlier in the guide. After installing GRUB, I generated a config file for it using its command ```grub-mkconfig -o /boot/grub/grub.cfg```. 

Realizing that I forgot network connectivity, I installed the `netctl` and `dhcpcd` packages, then checked my internet interfaces with `ip link` and found that my network adapter's name was `ens33`. I then ran ```cp /etc/netctl/examples/ethernet-static /etc/netctl/ens33``` to generate a network card profile, then edited it to add the line `Interface=ens33`. Finally, I enabled the network card with ```systemctl enable dhcpcd``` and ```systemctl start dhcpcd```.

Finally, after completing the system configuration and crossing my fingers, I `exit`ed the `/mnt` partition and unmounted it using ```umount -R /mnt```. I then `reboot`ed the system, chose `Arch Linux` in the GRUB bootloader menu, and signed in as `root`. 

----
## After installation

By some miracle, I was able to boot into Arch linux and sign in on the first try. 

For some inexplicable reason, `/etc/resolv.conf` did not save my changes from the setup process, but I was able to edit it to re-add `nameserver 1.1.1.1` and then ```sudo systemctl restart systemd-resolved.service``` to fix this error and regain connectivity. 

>*Author's note: it is around here where I tried to save some time by copying a random DE install command off Reddit and instead irreparably broke my install and had to start over, causing me to miss the deadline for extra credit. Valuable lesson learned I guess*

The first thing I did was realize I needed a few common packages that were not by default installed, so I installed them via ```pacman -S man-db```.

Next, I created my user account, using ```useradd -m camp```, and set my password using ```passwd camp```. I then gave myself sudo permissions by editing the `visudo` file with `nano`, using ```EDITOR=nano visudo```,and gave myself all permissions by adding the line ```camp ALL=(ALL) ALL``` to the sudoers file, granting me full permission when I use the `sudo` prefix before a command.

Finally, needing a desktop environment, I decided to go with **KDE**, largely because Codi mentioned it in class. I first had to install `xorg` and `xorg-server`, and, because this is running in a VM, `xf86-video-vmware` video drivers, instead of Intel/AMD/NVidia specific drivers. I then installed the required packages for KDE, `plasma`, `plasma-wayland-session`, and `kde-applications`, and finally enabled the display manager and network manager with the commands ```systemctl enable sddm.service``` and ```systemctl enable NetworkManager.service```. I edited `/usr/lib/sddm/sddm.conf.d/default.conf` to set my default theme to `breeze`, and then rebooted with `sudo systemctl reboot`. When the system restarted, I was within the KDE desktop, and could navigate with the GUI successfully.

I modified my `.bashrc` file to add colors to the terminal and display my current directory as well as the hostname of my computer, and added a few useful aliases to the system-wide `/etc/bash.bashrc`:
- `archbtw` runs `neofetch`
- `la` runs `ls -la`
- `meminfo` runs `free -m -l -t`

I then created user accounts for `codi` and `sal`, making Sal's default shell `zsh` since he seems like he would like a more technical shell, then set both their passwords to *GraceHopper1906*, and made them both expire on login using ```sudo passwd -e sal``` and ```sudo passwd -e codi```. With that all done, I did a few visual changes in the GUI, and had a working Arch installation.
