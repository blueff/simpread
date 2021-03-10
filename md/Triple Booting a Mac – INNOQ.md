> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.innoq.com](https://www.innoq.com/en/blog/triple-booting-a-mac/)

### [Introduction](#introduction)

Installing Windows 10 alongside with Mac OS is pretty straightforward with Apple’s BootCamp. It will ask you how much space you want to allocate for each environment, ask you for a Windows 8 or 10 ISO or CD-ROM, and then start the process. Once finished, you can select which OS to boot by pressing “alt” when the boot chime sounds. But BootCamp doesn’t give you enough flexibility. What if you want to share data between the two operating systems, for instance? Windows can’t access Mac’s HFS+ file system and Apple’s implementation of NTFS drivers to access Windows file systems is so buggy, they only dare to give you read access. And what if you’d like to not just install a dual boot but triple or quadruple boot? Of course, you might overcome most challenges by simply using a virtual machine in VirtualBox, Parallels or VMWare Fusion. But there are some use cases where you need to be able to boot natively into an OS - and it’s not just gaming in Windows, e.g. scrolling of console output in Eclipse is only usable under Linux. Windows and OSX do too much soft scrolling and cause a growing lag between real output and display. Another advantage might be to use Docker natively in Linux instead of a VM.

I spent literally 2.5 days (that’s 36 hours) on the internet, and trying and testing to remedy all the quirks and problems. I’ve tried many things that in the end didn’t work out. To save you that time, I decided to write this guide. Please let me know if it worked or where you got stuck to make it more complete.

### [Overview](#overview)

*   Use BootCamp to install Windows
*   Shrink the Windows partition
*   Create additional partitions
*   Install Linux via USB stick
*   Fix the Windows boot process (error 0xc000000e)
*   Install reFind
*   Install ntfs-3g in OSX and Linux to access the shared data partition
*   optional: Install & Configure VirtualBox to boot Windows/Linux from Raw disk in OSX

**NOTE**: The following procedure is not without risk. Do not follow it unless you know what you are doing and most of all: **create a backup first** ! I will not take any responsibility for loss of data or wrecked OS installations (incl. the original OSX).

### [Preparation](#preparation)

To make necessary changes to system files, you must disable the integrity checks of El Capitan

1.  Reboot your Mac and at the boot chime, press _cmd-r_ to boot into the recovery image.
2.  Once loaded, in the top menu select _Utilities->Terminal_
3.  To disable the integrity protection, enter `csrutil disable`
4.  Type `reboot` to reboot into standard OSX. You can check with `csrutil status` if it’s enabled or not.

### [Installation of Windows](#installationofwindows)

#### [Preconditions](#preconditions)

*   There should be only one partition on the disk for OSX - otherwise BootCamp will refuse to do its work.
*   On a CD-less Mac, you’ll need the ISO images of Windows 8 or 10, not the CD. To download them without direct access to Microsofts Download center, use [Windows ISO-Downloader](http://www.chip.de/downloads/Microsoft-Windows-ISO-Downloader_89861132.html) and get e.g. Win10_1511_1_English_x64.iso)

#### [Steps](#steps)

1.  Start BootCamp (_Applications/Utilities/Bootcamp Assistant_)
2.  Specify the downloaded ISO and select the partition sizes. Here, only the OSX partition size will be final - let Windows take the rest (for now). I used 100GB for OSX.
3.  Let Bootcamp install Windows and restart as desired.
4.  Once it’s (almost) finished and logs you in to the desktop, don’t do _anything_ else yet! Keep the automatically started BootCamp installer open but don’t proceed yet. If you do, you might not be able to shrink the Windows partition sufficiently, and you’ll have to start over. (ask me how I know…)
5.  Start the Windows partition manager (right click on your _computer -> manage -> disk management ->_ right click on _C:_ -> _Shrink Volume_. Now, enter the amount of MB to shrink the Windows Partition. I chose a value so the final size of the windows partition is 100GB.
6.  As it’s so convenient, you can use the Windows tool to create the remaining partitions. Don’t worry, although we already have 4 partitions, it’s a GPT partitioned disk (hopefully) which doesn’t have the limitation of 4 primary disks like a MBR disk. I added a 25GB partition for Ubunutu Linux and used the remaining 250GB for a shared NTFS Data partition (all 3 OS will use it to store Documents, Source Code, Java Applications, Music, Movies). If you want you can add a partition for Linux Swap space (I use a swapfile on the 25GB partition instead). **DO NOT change anything on the first three partitions !!!** The first one is the EFI partition required to boot the system. The second is the HFS+ partition which contains the OSX and the third is the OSX recovery partition (the one you boot into with cmd-r).
7.  Once the partitions are configured to your satisfaction, continue the BootCamp installer.

#### [Postcondition](#postcondition)

You have a dual boot system with partitions prepared for Linux and data sharing. You can switch between the OS by pressing the Alt key during boot.

### [Installation of Linux](#installationoflinux)

#### [Precondition](#precondition)

*   You have prepped all partitions as described above.
*   You have downloaded a 64bit variant of a Linux distribution (I chose Ubuntu Desktop). Make sure to use the 64-bit variant and _not_ the one built for Macs. The Mac image doesn’t support EFI partitions yet. It will boot up in BIOS mode and write a MBR to the disk instead of the necessary EFI files on the first partition
*   You also have a 2GB+ USB stick at hand which can/will be erased.

#### [Steps](#steps)

2.  Reboot the Mac and press the Alt key during boot.
    
3.  Select the USB stick in the boot selection menu (choose the EFI option).
    
4.  When prompted, choose “Try ubuntu” and NOT “Install Ubuntu” - once the desktop has loaded, start “Install Linux” from there.
    
5.  _IMPORTANT:_ Once you arrive at the partitioning step, **do NOT let Ubuntu repartition your disk**. Choose “Do something else” - otherwise you really do something else (like pay a visit to the Mac store to re-install OSX + recovery image).
    
6.  Check that the first partition is recognized as EFI boot partition (/dev/sda1).
    
7.  Select the partition intended to install Linux (in my setup it was the /dev/sda5, 25GB) and click on “Change”. Format the disk as ext4 partition and assign it to “/”.
    
8.  _IMPORTANT:_ before you continue, select **/dev/sda1** as target for the bootloader installation. This will add Linux' EFI bootloader to the existing collection.
    
9.  Check again in the partition overview that only the partitions you want to install Linux on, or want to use as swap disk, have the “Format?” checkbox checked. If not, start over by clicking “revert”.
    
10.  Let the installation finish (do not install updates during installation as the wifi probably won’t work yet). If you did not choose a swap partition, you might get a warning. Ignore it.
    
11.  After the installation of Linux, Windows 10 will not boot anymore and stop with a blue screen error 0xd000000e. This is because the Linux installation creates a hybrid MBR which is completely unnecessary. Windows will not boot in EFI mode but in BIOS mode instead and crash.
    
12.  Open a terminal and start `sudo gdisk /dev/sda` (if it’s not installed, reboot into Linux, try again or install with `sudo apt-get install gdisk`)
    
13.  Type `p` to view the partition table to verify you’re working on the correct disk. If not, type `q` to quit without saving your changes and try again with another device.
    
14.  Type `x` to enter the experts' menu.
    
15.  Type `n` to create an empty protective MBR. Note that gdisk won’t confirm a change; it’ll just show you a new experts' prompt.
    
16.  Type `w` to save your changes. You’ll be asked to confirm this action. Do so.
    
17.  Reboot into Linux (if not done already)
    
18.  Click on _System Settings -> Display_ and set _Scale for menu and title bars_ to 1.50
    
19.  Optional: to fix small fonts of grub during boot, execute `sudo grub-mkfont -s 36 -o /boot/grub/DejaVuSansMono.pf2 /usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf`
    
    and add the following line at the end `sudo vi /etc/default/grub`:
    
    ```
    GRUB_FONT=/boot/grub/DejaVuSansMono.pf2
    
    
    ```
    
20.  To fix freezes with SSD disks edit `/etc/default/grub` so the line with GRUB_CMD_LINUX reads:
    
    ```
    GRUB_CMDLINE_LINUX="libata.force=noncq"
    
    
    ```
    
    then execute `sudo update-grub`
    
21.  Install and configure powertop: `sudo apt-get install powertop` and make it to start automatically: edit `/etc/rc.local` and insert the line before the one that says `exit 0`
    
    ```
    powertop --auto-tune &
    
    
    ```
    
22.  Fix keyboard layout (to fix some special characters):
    
    ```
    sudo su -
    echo "options hid-apple iso_layout=0" > /etc/modprobe.d/hid-apple.conf
    
    
    ```
    

### [Postcondition](#postcondition)

You have a Mac which can boot to OSX, Windows 10 and Linux by pressing the Alt key during boot.

### [Installation of reFind](#installationofrefind)

#### [Precondition](#precondition)

*   You have installed one or more additional OS with the above procedure and are able to boot into them by pressing the Alt key during boot
*   Your currently running OS is OSX

#### [Steps](#steps)

1.  Download the [reFind installer](http://www.rodsbooks.com/refind/getting.html) (the binary zip file)
2.  In Finder, double-click on zip file
3.  Open a terminal, go to the unzipped refind folder
4.  execute `./refind-install` (this is the action which requires the integrity tool to be switched off)

#### [Postcondition](#postcondition)

During boot you will get a nice boot menu which automatically boots into the previously selected OS after a certain timeout.

Note: After Windows or Linux updates you might loose the reFind bootloader. If so, boot into OSX by pressing the Alt key during boot and re-install reFind. (you will have to temporarely disable integrity checks first, refer to “Preparation” and “Cleanup”).

### [Installation of ntfs-3g](#installationofntfs-3g)

To be able to access the shared data partition from all OS, you must install ntfs-3g on OSX and configure it in Linux.

#### [Linux](#linux)

1.  Linux already comes with ntfs-3g pre-installed.
2.  To make it mount the shared data during boot, add the following line to /etc/fstab (replace the partition number 6 with the correct one and the mount point with the one you like):
    
    ```
    /dev/sda6 /data   ntfs    defaults,umask=007,gid=46 0       0
    
    
    ```
    

#### [OSX](#osx)

1.  Download and install [OSXFUSE](http://osxfuse.github.io/) (it’s a pre-condition for ntfs-3g)
2.  Download and install Open Source [ntfs-3g](http://www.tuxera.com/community/open-source-ntfs-3g/) . Although this free edition doesn’t offer caching, it works fine and sufficiently fast. You might also choose the commercial edition maintained by [Tuxera](http://www.tuxera.com/products/tuxera-ntfs-for-mac/)
3.  I did not follow the guidelines and did not create an entry in /etc/fstab. Instead, I did the following in a terminal which replaces the OSX original ntfs implementation with the ntfs-3g and the integration in OSX is seamless (new NTFS volumes are automatically mounted in r/w mode under /Volumes)
    
    ```
    cd /sbin
    sudo mv mount.ntfs mount.ntfs.orig
    sudo ln -s /usr/local/sbin/mount_ntfs mount.ntfs
    
    
    ```
    

### [Installation of VMWare to boot the raw Windows / Linux partition in OSX (optional)](#installationofvmwaretoboottherawwindowslinuxpartitioninosxoptional)

You can not only boot into Windows or Linux natively now, but also by using virtualization tools like Oracle’s VirtualBox to boot from the raw partition. But be careful _NOT_ to suspend the OS in native mode and then boot via VirtualBox or vice versa. This will cause **disk corruption** as Windows/Linux will write data to the disk assuming a wrong status once you re-start the suspended image. –> As you can also read in the “Caveats” section, the best idea is to **always shut-down Windows and Linux and never put it to sleep or hibernation** (natively and virtualized). Only put OSX to sleep, nothing else.

#### [Precondition](#precondition)

You have installed Windows and/or Linux accordingly.

#### [Steps](#steps)

1.  Download and install [VirtualBox](https://www.virtualbox.org/) and its additions in OSX.
2.  Create a new Virtual Machine e.g. for Windows 10. Do not add a virtual disk yet.
3.  Open a Terminal and enter the following (replace the “4” in line 2 and 4 with the correct partition number):

```
sudo chmod og+rw /dev/disk0s1
sudo chmod og+rw /dev/disk0s4
cd <path to your virtual machine>
sudo VBoxManage internalcommands createrawvmdk -filename win10.vmdk -rawdisk /dev/disk0 -partitions 1,4

```

1.  Open the settings of your virtual box, go to “Storage”, add an IDE controller with PIIX4 chip-set (as SATA won’t work yet) and add the just created win10.vmdk as disk to the configuration.
2.  After a reboot the permission to the devices are reset, so I created a small script to quickly boot into the Win 10 VM. Save it as “Windows 10.command” (replace “Windows 10” with the name you gave to the VM) :

```
#!/bin/bash
sudo chmod og+rw /dev/disk0s1
sudo chmod og+rw /dev/disk0s4
VirtualBox -startvm "Windows 10" &
disown

```

1.  To make the terminated Terminal close automatically, go to it’s Preferences -> Profiles -> Shell -> When the shell exits: “Close if shell exited cleanly.”
2.  Make the script executable with chmod.
3.  Start the virtual machine. At first startup it will install some additional drivers.
4.  If you put the VM into full screen mode, you can easily switch between OSX and Windows by swiping two fingers over your mouse. Very nice ! :)

You may do the same for the Linux raw partition.

#### [Postcondition](#postcondition)

*   You can boot not only natively into your operating systems but also virtualized while having OSX running as host OS.
*   To access the shared data partition, add it as a shared folder in the VM configuration and mount it accordingly in the VM.
*   You don’t need to worry about multiple mounting of the shared data partition (e.g. natively by OSX and by the VM). As you only gave access to the EFI and the OS partition, Windows or Linux will not be able to access the data partition directly from within the VM. But I would recommend to **not access the windows OS partition** mounted on /Volumes while the Windows VM is running.

### [Cleanup](#cleanup)

Once everything works well and you’re happy, re-enable the integrity system check (CSR):

1.  Reboot, press cmd-r to boot into the recovery image
2.  Open a terminal and enter `csrutil enable` and `reboot`

### [Caveats](#caveats)

*   Suspending the computer in Windows or Linux will cause the Mac to freeze –> it’s best to disable this in the respective power configuration tools.
*   Bluetooth and the facetime camera don’t work properly yet. There are some beta drivers available but they might not work properly yet.

### [References](#references)

*   [https://www.hongkiat.com/blog/install-windows-on-mac](https://www.hongkiat.com/blog/install-windows-on-mac)
*   [https://www.makeuseof.com/tag/install-linux-macbook-pro](https://www.makeuseof.com/tag/install-linux-macbook-pro)
*   [https://www.makeuseof.com/tag/how-to-boot-a-linux-live-usb-stick-on-your-mac/](https://www.makeuseof.com/tag/how-to-boot-a-linux-live-usb-stick-on-your-mac/)
*   [https://www.rodsbooks.com/refind/](https://www.rodsbooks.com/refind/)
*   [https://superuser.com/questions/508026/windows-detects-gpt-disk-as-mbr-in-efi-boot](https://superuser.com/questions/508026/windows-detects-gpt-disk-as-mbr-in-efi-boot)
*   [https://help.ubuntu.com/community/MacBookPro12–1/Wily](https://help.ubuntu.com/community/MacBookPro12-1/Wily)

 [![](https://uploads.innoq.com/hve1glibm5jff60nrab5qn2xj6cz?response-content-disposition=inline%3B%20filename%3D%22michaeln_portrait.jpg%22%3B%20filename%2A%3DUTF-8%27%27michaeln_portrait.jpg&response-content-type=image%2Fjpeg&X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIAJHA5PEZW3OX32LYA%2F20210310%2Feu-central-1%2Fs3%2Faws4_request&X-Amz-Date=20210310T111026Z&X-Amz-Expires=300&X-Amz-SignedHeaders=host&X-Amz-Signature=ab039e450aad96e4a1570b43a159665c9c46c83278db636fb08fbee544886dce)](https://www.innoq.com/en/staff/michael-neuweiler/ "Michael Neuweiler") 

Michael worked as a Principal Consultant for INNOQ until October 2020. He has over 10 years of practical consulting experience in architecture, software development and middleware solutions. His main areas of interest are digitalization of industrial solutions and finding the right balance between latest and proven technologies to create sustainable solutions which fit the customer’s needs, culture and capabilities.

### Recommended

[Blog Post

iOS: Writing to Core Data in your Today extension
-------------------------------------------------

This blog post gives guidance on how to propagate changes in Core Data made by a Today extension to the host app and handle them properly.

![Portrait von Carmen Burmeister](https://uploads.innoq.com/ugn2d9x0og9toh3kmn45j562dhb0?response-content-disposition=inline%3B%20filename%3D%22carmen-burmeister-ava.jpg%22%3B%20filename%2A%3DUTF-8%27%27carmen-burmeister-ava.jpg&response-content-type=image%2Fjpeg&X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIAJHA5PEZW3OX32LYA%2F20210310%2Feu-central-1%2Fs3%2Faws4_request&X-Amz-Date=20210310T111028Z&X-Amz-Expires=300&X-Amz-SignedHeaders=host&X-Amz-Signature=8be26ecba5dc7ff286e74ee0efa7b989deacb0620d10045fc9fd7500a726c1e6) Carmen Burmeister ](/en/blog/ios-writing-core-data-in-today-extension/) [ Blog Post

When you have reached Acme
--------------------------

Notes on working with a minimalistic text editor/programming environment.

![Portrait von Martin Kühl](https://uploads.innoq.com/fjkzztclwiu178azlqxlk99c6udn?response-content-disposition=inline%3B%20filename%3D%22martin-kuehl-expert.png%22%3B%20filename%2A%3DUTF-8%27%27martin-kuehl-expert.png&response-content-type=image%2Fpng&X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIAJHA5PEZW3OX32LYA%2F20210310%2Feu-central-1%2Fs3%2Faws4_request&X-Amz-Date=20210310T111028Z&X-Amz-Expires=300&X-Amz-SignedHeaders=host&X-Amz-Signature=999dd989b05158dc32145daec6dceebec49ebd51674176dd29ea49d5e9446aae) Martin Kühl ](/en/blog/mkhl-acme-setup/) [ Blog Post

Mundane Bash Tips
-----------------

![Portrait von Frederik Dohr](https://uploads.innoq.com/w58rctlfg6syjwzfrvitdhmw93s1?response-content-disposition=inline%3B%20filename%3D%22fnd.png%22%3B%20filename%2A%3DUTF-8%27%27fnd.png&response-content-type=image%2Fpng&X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIAJHA5PEZW3OX32LYA%2F20210310%2Feu-central-1%2Fs3%2Faws4_request&X-Amz-Date=20210310T111028Z&X-Amz-Expires=300&X-Amz-SignedHeaders=host&X-Amz-Signature=9964f0f01f39ef5c4e49938ff2654b7abb95aef018012daa3939ecd936a3975f) Frederik Dohr ](/en/blog/bash-cheat-sheet/) 

### Comments

Please accept our cookie agreement to see full comments functionality. [Read more](https://www.innoq.com/en/datenschutz/)