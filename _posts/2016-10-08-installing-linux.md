---
layout: post
title:  "Trivia of Installing Linux on Laptops"
date:   2016-10-08 20:47:00 -0700
categories: Linux
---
Happy second year of college at UCLA! And finally get to update my blog again. At the quarterly Installfest by the Linux User Group at UCLA, luckily we got many machines to run Linux (or Ubuntu to be more accurate) properly, though after some struggling.  

### Dell Inspiron: Dual-boot Windows 10 and Ubuntu
The first laptop we worked on is a Dell Inspiron 13'' laptop that has Windows 10 on it. No problem on booting from USB installation drive, resizing the Windows partition and adding a new one for Linux using `GParted`. Stuff happended when we tried to `grub-install` the boot loader; it always said something like cannot find BIOS location like that. A little work in command line revealed that the hard drive used *GPT (GUID Partition Table)* and the machine is booted via *UEFI*, rather than using *MBR (Master Boot Record)* and boots from *BIOS* typically done on an old laptop. (Its disk has *[Protective MBR](https://en.wikipedia.org/wiki/GUID_Partition_Table#Protective_MBR_.28LBA_0.29)* though, to maintain backward compatibilty and avoid programs from mistakenly overwriting GPT. However, we are just using GPT here.) Seems that the command line tool didn't realize this and was still thinking that we got here from BIOS. Soon we solved the problem by adding EFI options to this command.  

Then the installation wizard starts on normal installation process; unfortunately a fatal error occured and forced a restart, with no error message or code given at all. An workaround to this problem was to brutally install Ubuntu  through command line; not sure how my friend did that, but I guess it was essentially copying all system files into the destination partition. Rebooted the laptop and we found the boot option of Ubuntu in Dell's default boot manager. However, strangely enough, the machine was not booted into the system and went to the boot settings of Dell's motherboard.  

This implied the OS was at least installed and could be somewhat recognized by a boot loader. Since we believed this problem may be specifically related to Dell, Google searches led us to [this article](http://www.dell.com/support/article/us/en/19/SLN297060/en). Following it to boot in UEFI and enable *security boot* proved to be no avail. After playing around for sometime, we realized that the solution was exactly the opposite: **Disable the Security Boot**! And booting, `grub` boot loader and everything work perfectly. 

So, Security Boot is a realitively new OS security strategy used since the Windows 8 era, and strongly advocated by Microsoft. Basically the hardware vendor signed legitimate operating systems in the firmware, and only these OSs can be run when this feature is enabled. So disable the security boot is kind of similar to unlock the boot loader on Android devices. While it may prevent maliciously modified OSs from being booted, security boot is still in high controversy especially in the open source community. If the hardware manufacturer does not provide the option of disabling the feature, installing U/Linux would be impossible. [More insights on Security Boot](http://www.windowsecurity.com/articles-tutorials/misc_network_security/Secure-Boot-Controversy-What-does-mean-IT.html)

### MacBook Pro: Dual Boot OS X El Capitan and Ubuntu
Soon we realized that dual booting Linux is not as difficult as I thought before on a Mac; we were trying it on a Retina MacBook Pro 13''. The first step would be to **install *boot loader* `rEFInd` before anything else**, which does the job of `grub` on a dual-booting PC. `rEFInd` works well on Macs for even triple-booting macOS, Linux and Windows.  

Below is an illustration for a PC or a Mac's booting process after being powered on. The boot loader was loaded first after hardware are ready; then it will prompt user to choose the OS to boot, and gives control to that kernel to load.  

```bash
                                                       |--------|
                                                  ---->| macOS  |
                                                       |--------|
                                                  
|---------------------------|     |--------------|     |--------|
| Hardware check done, etc. |---->|  grub        |---->| Ubuntu |
| OS ready to be booted.    |     |  rEFInd, etc.|     |--------|
|---------------------------|     |--------------|
                                                  ---->|---------|
                                                       | Windows |
                                                       |---------|
```

Then comes the common installation routine, after partitoning using the built-in `Disk Utility`. Keep in mind that we haven't choose `rEFInd` to be the first boot option yet, so there would be no booting menu with both macOS and Linux when turning on the computer. The default *boot manager* by Apple (the menu triggered by holding `Option` after the startup *ping* sound) will not recognize the Ubuntu partition. So now we booted into the Ubuntu installation drive again, and set up the `rEFInd` to be the first boot option by `efibootmgr`. Note that there may be two options of "Mac OS X", where one was really the boot loader we just installed.

Up to now everything are good, so what's next? How about **triple-boot macOS, Ubuntu and Windows**? I've tried that last year; though didn't work out finally, the primary reason was clear: **The three OSs must be consistent on the boot scheme: BIOS or UEFI.** macOS and Ubuntu are all relying on UEFI as the main option for a long time, and Microsoft came down to support it since Windows 7/8. A little background here: years ago engineers at Apple invented a somewhat complex and annoying solution called [*Hybrid MBR*](http://www.rodsbooks.com/gdisk/hybrid.html), for users to install Windows through *Boot Camp* when it was not supporting UEFI boot yet. Basically it emulates a MBR partition table specifically for Windows, so Windows installation (and later the Windows OS itself) will be tricked to recognize the hard drive as being MBR partitioned while it is actually using GUID.  

But here is the thing: even though newer versions of Windows work perfectly fine under UEFI, Boot Camp may still install it under the Hybrid MBR mode which makes everything complicated. I strongly discourages using it though Boot Camp is still the only official way to dual-boot Windows on a Mac. Theoretically Windows 8/10 should be easily installed without any extra modification: partition the hard drive, boot from the installation USB, and install. The Windows boot loader was broken when I was trying triple-boot last year, and I bet Hybrid MBR is the reason.   

So, *Why join the Navy if you can be a pirate?* Go hacky!

### Old Acers and HP Compaq: A Piece of Cake!
These were two old Acer 9'' laptops and a HP Compaq 15'' with the first generation of AMD Dual-Core CPU; one acer was even with a Spanish keyboard layout. A woman from Linux Chix brought them here, and she got that HP with only $50. Hardware vendors hadn't come up with so many OS protection  features back in the old days, so installing another OS was fairly simple on these laptops.  

Note that a *file sharing* partition may be of use here; it can be in `NTFS` format which can be read and write by both Windows and Linux. One Acer with 40 Gigabytes of hard drive was clean installed a 32-bit Ubuntu with one partition mounted to `/`; the Acer with 160 Gig and HP with 500 Gig were dual booting with Windows XP and Windows 7.

Two things to delve deeper in the future:  
1. Swap. I didn't give an extra partition mounted to `/swap` for memory swap when installing the OS, so it will go to a swap file in the main partition if swap space is needed. That may possibly cause lose of performance on low-memory machines.  
2. *[Logical Volume Manager(https://en.wikipedia.org/wiki/Logical_Volume_Manager_(Linux))]*: we can choose to use that to manage partitions before installing Ubuntu, which is claimed to provide easier and faster partition managements in the future. Basically it is an extra layer above the physical partitions, and somehow maps to the position of each partition.

### HP: Problems with Many Partitions in MBR
There was one last problem regarding a relatively new HP laptop running Windows 7 under MBR. One of the major restrictions of MBR is that up to 4 *primary partitions* are supported on one physical drive only; any extra ones will go to *logical partitions*. That seems to be annoying when checking the hard drive on the Ubuntu installation USB: **all logical partitons, as well as some unallocated space between them, cannot be recognized.** Therefore we didn't dare to change the partition scheme in case anything goes wrong. [Extra infomation on MBR](https://en.wikipedia.org/wiki/Master_boot_record)  

### Morals
1. Considering install Linux or open source OSs on new laptop models? *Just be mentally prepared!* As mentioned above, recovery and protection features by hardware vendors are making things much more difficult. HP once had a model that includes a 30-ish Gig recovery partiton, and users would have no way of reinstalling any OS if they delete that...
2. Facing problems in command line interface? `man` pages are your best friends! Don't be afraid to read it carefully.
3. Never rush when working on system-level problems. You are not always the lucky guy; *Always backup *all* of your data before doing anything*. Organize your thoughts and understand *why* you do it before actually doing it. Stay focused, take snapshots/pictures when necessary, keep logs (what we have tried and if it works).
