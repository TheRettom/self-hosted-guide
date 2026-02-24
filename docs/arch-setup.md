# Arch Linux Configuration

<details>
<summary><strong>Why Arch Linux versus Windows for a server?</strong></summary>

<br>

I will disclose my bias first: I hate Windows. Microsoft has ruined the Windows OS over time in my opinion. More forced telemetry, more forced products, more advertising, and less control and privacy with each iteration. On to the facts:

* Windows views itself as the priority. It will force updates, reboot your server without permission for a "quality patch", and run background telemetry that eats IOPS and minor CPU usage. On Arch, a service runs until you tell it to stop.

* Windows does not have a native space for containerization like Linux. To run Podman or Docker, Windows has to utilize WSL2, a virtual machine. You end up running a Linux kernel inside a Windows OS to run a container. Arch is the native metal; your packets go straight from the NIC to the container with no tax.

* Windows includes a GUI, print spoolers, font parsers, and thousands of legacy APIs like Win32 that create a massive attack surface. Arch gives you full control for setup: If you don't install it, the vulnerability doesn't exist.

* A "light" Windows install idles at ~2GB of RAM. Arch, even with linux-hardened and Btrfs, will idle at ~150-300MB. That’s 1.7GB of RAM we can give back to our services.

Using Arch for this project means your server is exactly what you made it, and nothing more. There are no hidden services, and no telemetry pings.

</details>

<details>
<summary><strong>Why Arch vs. Other Linux Distros (Debian, Ubuntu, RHEL)</strong></summary>

<br>

This is where the debate gets nuanced.

* While Debian is stable, Arch is early. Most distros (Ubuntu/Debian) patch their software. They take a piece of code, change it to fit their OS, then ship it. Arch ships Vanilla software. In my experience of using Arch as a home server for two years, I've never had anything break because I updated it, even with rolling releases. When Podman dropped with a major security fix or a new Btrfs feature, Arch got it in 24 hours. On Debian "Stable," you might wait months or even years.

* The Arch Linux Wiki is the best technical documentation in the world. Because Arch doesn't use "magïk" configuration scripts, like Ubuntu’s `netplan`, you learn exactly where your config files are and how things work. If your DNS fails, you know it's in /etc/resolv.conf or your container config.

* Arch was one of the first to embrace the modern Btrfs/ZFS stack. Because we are using Reflinks for Podman storage, we need a rolling-release kernel to ensure you have the latest bug fixes for the Btrfs CoW engine. Older "stable" distros often run kernels that have known performance regressions with Btrfs. Reflinks in Btrfs allow for efficient file copying by sharing the same data blocks, which can improve performance when using the overlay filesystem with Podman. This feature is particularly beneficial during operations like "copy ups" when modifying files in a container's root filesystem.

Using Arch for this project means your server is exactly what you made it, and nothing more. No waiting years for security features like native-overlay.

</details>

<details>
<summary><strong>Why linux-hardened kernel instead of linux-lts or something else?</strong></summary>

* _Hardened BPF (Berkeley Packet Filter) JIT (Just-In-Time) compiler._ Our system will be constantly processing data from the outside world.
If a vulnerability were found in the way the kernel handles networking, a hardened BPF compiler ensures that an attacker cannot use the kernel's JIT compiler to attack us.

* _Restricted ptrace to prevent processes from inspecting others._ In a standard kernel, one process can often "peek" into the memory of another. linux-hardened restricts this, making it much harder for a compromise in, for example, a Caddy container to scrape sensitive data from other running processes.

* _Disabled Unprivileged User Namespaces (by default)._ While this is the setting we manually toggle for Podman, the fact that the kernel forces you to consciously enable it is a security feature. It prevents random, malicious scripts from creating namespaces to bypass restrictions.

* _ASLR (Address Space Layout Randomization)._ linux-hardened uses much stronger randomization than LTS or linux kernels. This means an attacker cannot predict where "Function A" is in memory, making "Return-to-libc" attacks nearly impossible.

* _Kernel Page Table Isolation (KPTI)._ While standard kernels have this to mitigate Meltdown/Spectre, the hardened kernel often implements more aggressive variants that further isolate the kernel's memory from the user's memory.

* _linux-lts has delayed features._ LTS focuses on fixing bugs in old code. If a new, more secure way of handling networking or encryption is released, it may take a long time to reach LTS.

We choose linux-hardened because we are building a fortress, not a workstation. It turns our Arch Linux install from a hobbyist's playground into a production-grade security appliance.
</details>

<details>
<summary><strong>What hardware do I need?</strong></summary>
As long as it is functional, it'll probably work. Literally, that's all there is to it. Any old PC is fine.
</details>

## Getting Arch Linux and Creating a Bootable Drive / Live Installation

⌛ Estimated Time: 20 minutes (depending on internet speeds)

This is going to require a few things. There are many ways to install it, but the simplest is to get a thumb drive at least 8GB in size and put the Arch Linux ISO on it. This is the only way I'm documenting it. This guide is assuming that you have a dedicated machine specific for hosting, and will **not** be dual-booting.

<details>
<summary><strong>Starting from Linux</strong></summary>

* [Download Arch Linux](https://archlinux.org/download/). If you don't want to use BitTorrent to download, then use an HTTP direct download. You should also download the [`sha256sums.txt`](https://archlinux.org/iso/2026.02.01/sha256sums.txt) or [`b2sums.txt`](https://archlinux.org/iso/2026.02.01/b2sums.txt). **The links for the checksums I have will likely be outdated at some point**, so reference [the Arch Linux downloads page](https://archlinux.org/download/). **Verifying the checksums is a critical step. It ensures the ISO wasn't corrupted or tampered with before it even touches your hardware.**

* Open the terminal. Change the directory to Downloads:

```
cd ~/Downloads
```

* Run either commands for the checksums:

```
sha256sum -c sha256sums.txt
b2sum -c b2sums.txt
```

If the output is `archlinux-xxxx.iso: OK`: Everything is perfect.

If the output is `archlinux-xxxx.iso: FAILED`: The file is corrupt or tampered with.

If the output is `FAILED open or read`: This just means you have other files listed in the .txt that aren't in your folder. You can ignore those as long as your ISO says OK.

* There are many tools in various distributions to create a bootable flash drive, but I'm only going to explain the basic one that every Linux distro will have, and it's through the terminal. __If you want to use something else, like a GUI-based application, go ahead to the next section in the guide and consult it when you finish creating the ISO for the drive.__

* Run the following:

```
lsblk
```

* Look for the device that matches your USB's size (e.g., 8G, 16G, 32G). It will likely be `sdb` or `sdc`.

> **⚠️ WARNING**
> You must replace sdX with your actual thumb drive (like sdb or sdc). If you point this at `/dev/sda` (your internal drive), dd will faithfully overwrite your entire operating system __without__ asking "Are you sure?**"

* Edit the `/dev/sdX` in the command below to whatever matches it and run it:

```
sudo dd if=archlinux-*.iso of=/dev/sdX bs=4M status=progress oflag=direct
```

<details>
<summary>Breakdown of that command</summary>

1: `sudo`: Superuser Do. Writing directly to a device like `/dev/sdX` is a restricted action. Without sudo, Linux will block the command.

2: `dd`: Data Definition. It copies data from one place to another bit-for-bit.

3: `if=archlinux-*.iso`: Input File. This tells `dd` where the data is coming from. The asterisk (*) is a wildcard that tells the shell to find the file that matches that pattern, like `archlinux-2026.02.01-x86_64.iso`.

4: `of=/dev/sdX` Output File. This is the most dangerous part of the command. In Linux, a USB drive is treated as a file located in `/dev/`.

5: `bs=4M`: Block Size. This tells dd to read and write in 4-megabyte chunks. The default block size is only 512 bytes. Writing a 1GB ISO in 512-byte increments is like moving a pile of sand with a teaspoon — it works, but it takes forever. Using 4MB is like using a large shovel; it’s much faster and matches the internal "page size" of most USB flash memory.

6: `status=progress`: A Visualizer. By default, `dd` is a silent command. You would see a blinking cursor for 10 minutes and wonder if it's frozen. This flag provides a live update of how many bytes have been copied, the elapsed time, and the transfer speed.

7: `oflag=direct`: Direct I/O. Normally, Linux uses a "buffer cache" — it says "I've finished writing!" when the data is actually just sitting in your RAM waiting to be sent to the USB. `oflag=direct` tells the kernel to bypass that cache and write directly to the physical USB hardware. This makes the status=progress bar much more accurate and prevents the command from "hanging" at 99% while the system clears the cache.

</details>

</details>

<details>
<summary><strong>Starting from Windows</strong></summary>

* [Download Arch Linux](https://archlinux.org/download/). If you don't want to use BitTorrent to download, then use an HTTP direct download. You should also download the [`sha256sums.txt`](https://archlinux.org/iso/2026.02.01/sha256sums.txt) or [`b2sums.txt`](https://archlinux.org/iso/2026.02.01/b2sums.txt). **The links for the checksums I have will likely be outdated at some point**, so reference [the Arch Linux downloads page](https://archlinux.org/download/). **Verifying the checksums is a critical step. It ensures the ISO wasn't corrupted or tampered with before it even touches your hardware.** I recommend just downloading the `sha256sums.txt` instead of `b2sums.txt`.

* Open the command prompt with `Windows Key`+`R`, then type `cmd` and press `Enter`. Type `powershell` and press `Enter`.

* Check the SHA256 checksum: Run this command:

```
Get-FileHash -Algorithm SHA256 archlinux-*.iso
```

Copy the resulting hash. Open the `sha256sums.txt` file and press `Ctrl`+`F`, then paste with `Ctrl`+`V` to see if it matches. Once verified, you can close the command prompt and the document. If it's not right, re-download the ISO and go through these steps again.

* [Download Rufus](rufus.ie). Rufus is a lightweight, "no-install" executable that is the gold standard for creating Linux media on Windows.

* Plug in your thumb drive and open Rufus.

* `Device`: Under `Device`, ensure your USB drive is selected.

* `ISO`: Click SELECT and find your `Arch Linux ISO`.

* `Partition Scheme`: Select GPT. This is for modern UEFI systems. MBR is old-school.

* Ensure Target system says `UEFI`.

* Click `START`.

* Rufus will ask if you want to write in `ISO Image mode` or `DD Image mode`. Choose DD Image mode. Why? Arch Linux ISOs are hybrid images. Using DD mode creates a bit-for-bit clone of the ISO, which is the most reliable way to ensure the Arch installer boots correctly without modified file structures.

* At the bottom-right of your screen in the taskbar, look for the `Safely Remove Hardware and Eject Media` icon. Safely eject the thumb drive and wait for the "Safe to Remove Hardware" message before unplugging it. If the icon is not visible, you can access it through the Settings app or use the Run command to bring up the dialog.

</details>

<details>
<summary><strong>Starting from macOS</strong></summary>

* [Download Arch Linux](https://archlinux.org/download/). If you don't want to use BitTorrent to download, then use an HTTP direct download. You should also download the [`sha256sums.txt`](https://archlinux.org/iso/2026.02.01/sha256sums.txt) or [`b2sums.txt`](https://archlinux.org/iso/2026.02.01/b2sums.txt). **The links for the checksums I have will likely be outdated at some point**, so reference [the Arch Linux downloads page](https://archlinux.org/download/). **Verifying the checksums is a critical step. It ensures the ISO wasn't corrupted or tampered with before it even touches your hardware.** I recommend the SHA256, as BLAKE2b needs Homebrew.

* Open Terminal and navigate to your Downloads folder with the following:

```
cd ~/Downloads
```

* For the `SHA256` checksum, run the following:

```
shasum -a 256 archlinux-*.iso
```

Copy the output, then open the `sha256sums.txt` file and press `Command`+`F`, then paste with `Command`+`V` and verify that the long string of letters and numbers matches.

* For `BLAKE2b`, run the following:

```
b2sum -c b2sums.txt
```

* Verify the output. Once verified, you can close the terminal and the document if you checked the SHA256. If it's not right, re-download the ISO and go through these steps again.

* Plug in your thumb drive.

* Open the Terminal and run:

```
diskutil list
```

Look for your thumb drive in the list. You can identify it by the SIZE and NAME. Note the identifier: It will look like `/dev/disk4` (the number may vary).  For the rest of these instructions, I will use `/dev/diskN` — replace N with your actual number.

* macOS automatically mounts USB drives when you plug them in. You cannot write the ISO to a mounted disk. Edit the following as needed an run:

```
diskutil unmountDisk /dev/diskN
```

You should see a message saying "Unmount of all volumes on diskN was successful".

* We use the dd command to copy the raw data. Pro-Tip: Use `/dev/rdiskN` (the "r" stands for raw) instead of `/dev/diskN`. On macOS, raw disk access is significantly faster. Edit the following as needed an run:

```
sudo dd if=/path/to/archlinux-*.iso of=/dev/rdiskN bs=1m status=progress
```

<details>
<summary>Breakdown of the command</summary>

1: `sudo`: Requires your Mac admin password.

2: `if=`: The input file. Drag and drop the ISO file into the terminal window to auto-fill the path.

3: `of=`: The output file, as in your thumb drive.

4: `bs=1m`: Sets the block size to 1 megabyte to speed up the transfer.

5: `status=progress`: Shows you how much has been copied.

</details>

* Once the command finishes, macOS might pop up a warning saying, `The disk you inserted was not readable by this computer.` **This is normal because your Mac doesn't recognize the Linux filesystem.**

* Eject the drive properly before pulling it out. Edit the following as needed an run:

```
diskutil eject /dev/diskN
```

</details>

# Installing Arch Linux

⌛ Estimated Time: 1 hour

The Arch Linux Wiki already has a fantastic writeup of [installing Arch Linux](https://wiki.archlinux.org/title/Installation_guide). For beginners, however, they may still get a little lost, as a lot of pre-requisite knowledge is needed. I will do my best to give you instructions and explaining along the way. There are different ways to install Arch, as we are in charge of everything for booting it up and installing additional packages.

For MY guide, I will not be including software RAID setup, as I'm poor and can't afford new hardware, and have no experience setting it up. In the future, this may change and my guide will be updated to include it.

## Pre-Requisites and Notes

* Arch Linux installation images do not support Secure Boot. You will need to disable Secure Boot to boot the installation medium. If desired, Secure Boot can be set up after completing the installation.

* Know whether your motherboard is capable of UEFI bootloading or if it can only do MBR bootloading. I am only covering UEFI in this guide.

* If you want to enable Wake-on-LAN in the BIOS, do it now.

* Know what kind of partition layout you want.

* Know if you want secure boot now.

* Know if you will be utilizing software RAID.

* Some motherboards, like MSI, don't play nice with GRUB (Grand Unified Bootloader), and require the boot image to be placed somewhere else. I found this out the hard way many times. We'll work around that by default.

* When using shortcuts in a CLI (command line interface), you'll probably need to use `Ctrl`+`Shift` instead of just `Ctrl` for copy and paste commands. This is because `Ctrl+C` sends an interrupt signal (SIGINT) to a running process in the CLI, which usually terminates it. So to copy, use `Ctrl`+`Shift`+`C`, and to paste in a CLI, use `Ctrl`+`Shift`+`V`.

* Unlike Podman or Docker for Windows, it does not need virtualization to work. Podman is using the host's kernel.

## Boot the Installation Media

The [necessary key to press to get into BIOS settings](https://www.lifewire.com/bios-setup-utility-access-keys-for-popular-motherboards-2624462) is based on motherboard manufacturer.

For Windows systems, if you don't know your motherboard manufacturer, press the Windows Key and search for System Information, then open it. It is BaseBoard Manufacturer.

* Look throughout your settings to make sure UEFI mode is not disabled and that Secure Boot is off. Refer to your motherboard manual.

* Some systems, like Thinkpad firmware, have a setting called "Boot Order Lock" which needs to be disabled for efibootmgr to be able to add/remove entries so your system can boot.

* Restart the computer and press the key to open the boot selector. Refer to your motherboard manual. Start the live installation we created.

* Select `Arch Linux install medium` and press `Enter`.

* You will be logged in on the first virtual console as the root user, presented with a Zsh shell prompt.

* The default console keymap is US. Available layouts can be listed with:

## Set the Console Keyboard Layout

```
localectl list-keymaps
```

To set the keyboard layout, pass its name to [`loadkeys`](https://man.archlinux.org/man/loadkeys.1). For example, to set a German keyboard layout:

```
loadkeys de-latin1
```

## Verify the Boot Mode

* Verify the boot mode with the following:

```
cat /sys/firmware/efi/fw_platform_size
```

* Reference the following for the output results:

    * If the command returns 64, the system is booted in UEFI mode and has a 64-bit x64 UEFI.
    * If the command returns 32, the system is booted in UEFI mode and has a 32-bit IA32 UEFI. While this is supported, it will limit the boot loader choice to those that support mixed mode booting.
    * If it returns No such file or directory, the system may be booted in BIOS (or CSM) mode.

If the system did not boot in the mode you desired (UEFI vs BIOS), refer to your motherboard's manual.

## Internet Connection

* If you are using Ethernet, you likely only need to verify your internet connection with `ping`. To set up a network connection in the live environment, go through the following steps:

* Ensure your network interface is listed and enabled, for example with ip-link:

```
ip link
```

* For wireless and WWAN, make sure the card is not blocked with [`rfkill`](https://wiki.archlinux.org/title/Network_configuration/Wireless#Rfkill_caveat). Run the command and see the example output. We want `unblocked`.

```
rfkill
---------------------------------------
ID TYPE      DEVICE      SOFT      HARD
 0 bluetooth hci0   unblocked unblocked
 1 wlan      phy0   unblocked unblocked
```

* Connect to the network. You got three options:
    * Ethernet — plug in the cable if you haven't already.
    * Wi-Fi — authenticate to the wireless network using [`iwctl`](https://wiki.archlinux.org/title/Iwd#iwctl).
    * Mobile broadband modem — connect to the mobile network with the ModemManager [`mmcli`](https://wiki.archlinux.org/title/Mobile_broadband_modem#ModemManager) utility.
* Configure the network connection:
    * DHCP: dynamic IP address and DNS server assignment (provided by systemd-networkd and systemd-resolved) should work out of the box for Ethernet, WLAN, and WWAN network interfaces.
    * Static IP address: follow [Network configuration#Static IP address](https://wiki.archlinux.org/title/Network_configuration#Static_IP_address).
* Verify internet connectivity with `ping`, or you're going nowhere:

```
ping ping.archlinux.org
```

## Update the System Clock

The live system needs accurate time to prevent package signature verification failures and TLS certificate errors. The systemd-timesyncd service is enabled by default in the live environment and time will be synchronized automatically once a connection to the internet is established.

* Use `timedatectl` to ensure the system clock is synchronized:

```
timedatectl
```

## Before Partitioning Disks

**This one of the most important parts of the setup.** If you're here for this guide, you probably won't understand how to do conversion or re-partitioning procedures when you've been running your system for a while. If you are working beyond what the guide is going over with RAID, take time to plan a long-term partitioning scheme. If you want to create any stacked block devices for LVM, system encryption or RAID, do it now. In this guide, I'm only covering the `BTRFS` file system on `GPT` (GUID Partition Table) for a single drive. I'll include steps with SWAP and without, but any partition other than EFI, SWAP, and root is up to you to figure out.

GPT (GUID Partition Table) is a modern partitioning scheme that supports larger drives and more partitions than MBR (Master Boot Record), which is an older standard limited to 2 TB and four primary partitions. GPT is preferred for new systems due to its advantages in flexibility and data security.

* When recognized by the live system, disks are assigned to a block device such as `/dev/sda`, `/dev/nvme0n1` or `/dev/mmcblk0`. To identify these devices, use `fdisk`.

```
fdisk -l
```

Results ending in `rom`, `loop` or `airootfs` may be ignored. `mmcblk*` devices ending in `rpmb`, `boot0` and `boot1` can be ignored. For you RAID users, if the disk does not show up, [make sure the disk controller is not in RAID mode](https://wiki.archlinux.org/title/Partitioning#Drives_are_not_visible_when_firmware_RAID_is_enabled).

* You don't have to, but **I highly recommend you check that your NVMe drives and Advanced Format hard disk drives are using the optimal logical sector size before partitioning.** It's a bit of a read, so if you want to do it, [read here](https://wiki.archlinux.org/title/Advanced_Format). What happens if you don't and the logical sector size is different than it should be?
    * This turns one write into two, effectively cutting your write speeds in half and increasing latency.
    * For NVMe and SSDs, ther is potential for significantly reduced hardware lifespan because misalignment forces the controller to perform unnecessary extra writes to the NAND cells.
    * When partitions are misaligned, the controller has to work much harder to move data around during background cleanup cycles, which can cause the system to stutter during heavy I/O.

To save you some panic, **you're probably fine.** To verify, run the following and see the example output:

```
lsblk -td
```

See the example output:

```
NAME    ALIGNMENT MIN-IO OPT-IO PHY-SEC LOG-SEC ROTA SCHED       RQ-SIZE   RA WSAME
sda             0    512      0     512     512    0 mq-deadline      64  128    0B
sdb             0    512      0     512     512    1 mq-deadline      64 8192    0B
nvme0n1         0    512      0     512     512    0 none           1023  128    0B
```

The important thing is to have the `PHY-SEC` and `LOG-SEC` values match. If they are, you're good to go.

<details>
<summary><strong>If you are installing on an NVMe drive, read here.</strong></summary>

You could increase performance by a small amount if you can verify that your SSD/NVMe can use a 4K block value. If you need any further details than I provide, refer to [this guide](https://wiki.archlinux.org/title/Advanced_Format#NVMe_solid_state_drives). However, I will give you a tidbit of information for it.

* Use pacman to get the `nvme-cli` package so we can check:

```
pacman -Sy nvme-cli
```

* When it's done, run the following assuming it is `/dev/nvme0n1`:

```
nvme id-ns -H /dev/nvme0n1 | grep "Relative Performance"
```

See the example output:

```
LBA Format  0 : Metadata Size: 0   bytes - Data Size: 512 bytes - Relative Performance: 0x2 Good (in use)
LBA Format  1 : Metadata Size: 0   bytes - Data Size: 4096 bytes - Relative Performance: 0x1 Better
```

> If the `Better` option is available, you will probably want to use that. __**A caveat: Some NVMe SSDs report they support 4K sectors but encounter sporadic instability upon enabling them, especially under heavy random read loads. Make sure to test the new configuration thoroughly before storing data you cannot afford to lose on it. If encountering this issue, simply revert back to a 512 sector configuration.**__

* `smartctl` can also display the supported logical block address sizes, but it does not provide user friendly descriptions. Run:

```
smartctl -c /dev/nvme0n1
```

Example output below:

```
Supported LBA Sizes (NSID 0x1)
Id Fmt  Data  Metadt  Rel_Perf
 0 +     512       0         2
 1 -    4096       0         1
```

We can see we want `Id 1`. So let's change it to that. Modify as needed based on your previous output and run:

```
nvme format --lbaf=1 /dev/nvme0n1
```

You'll get a warning:

```
You are about to format nvme0n1, namespace 0x1.
WARNING: Format may irrevocably delete this device's data.
You have 10 seconds to press Ctrl-C to cancel this operation.

Use the force [--force] option to suppress this warning.
```

Since we want to do it, let it do it. On success, you'll see:

```
Sending format operation ... 
Success formatting namespace:1
```

</details>

## Partition the Disk

* Now we need to partition. If you saw any information saying it's important to have proper alignment for your disks, don't worry, `fdisk` handles it automatically, and `fdisk` has defaults so you can USUALLY press `Enter` to go through it quickly.

> ### ⚠️ For this guide I'm going to assume we're formatting an NVMe, so __**if it's something else, adapt `/dev/nvme0n1` accordingly.**__

Run the following:

```
fdisk /dev/nvme0n1
```

* Press `g` to create a new GPT partition table. You'll see the output `Created a new GPT disklabel (GUID: a long string).`

* Press `n` to create a new partition. It will ask `Partition number (1-128, default 1):`. Press `Enter`.

* It will ask `First sector (2048-1953525134, default 2048):`. All it's asking is where on the disk to start. Press `Enter`.

* It will ask `Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-1953525134, default 1953523711):`. **The big number, and correspondingly the default option, depends on your disk size.** We're going to have our first partition be the EFI System partition, and we only need it to be 2 gigabytes, so type `+2G` and press `Enter`. It will report `Created a new partition 1 of type 'Linux filesystem' and of size 2 GiB.` Now we need to list the type of partition for the EFI System. Press `t`. It will report `Selected partition 1` and ask `Partition type or alias (type L to list all):`. Don't worry, I gotchu. Type `1` again and press `Enter`. It will say `Changed type of partition 'Linux filesystem' to 'EFI System'`.

* Press `n` to create another partition. It will ask `Partition number (2-128, default 2):`. Press `Enter`.

* It will ask `First sector (4196352-1953525134, default 4196352):`. Again, **the big number, and correspondingly the default option, depends on your disk size.** The first sector for this second partition is just after the ending of the first partition. Press `Enter`.

* It will ask `Last sector, +/-sectors or +/-size{K,M,G,T,P} (4196352-1953525134, default 1953523711):`. This depends on whether or not you want SWAP.

<details>
<summary><strong>Do I want SWAP?</strong></summary>

[SWAP](https://wiki.archlinux.org/title/Swap) space can be used for two purposes: to extend the virtual memory beyond the installed physical memory (RAM), and for suspend-to-disk support. Suspend-to-disk, commonly referred to as Hibernation, is not desirable for a server because resuming from a 16GB+ SWAP image takes significant time. Put simply, for our use-case as a server with high-availability, we don't hibernate or put the computer to sleep. You only want SWAP if you don't have enough physical RAM. __Most__ of these services are computationally cheap, they just need to be available fast. SWAP is much slower than RAM. If you have less than 8GB for a simple setup, maybe consider it. If you have at least 8GB for a simple setup, you're fine without it. If you're running more complex services like [Immich](https://github.com/immich-app/immich), you may want at least 16GB of RAM or more.

**Tl;dr, only if you have less than 8GB of RAM.**

</details>

* So your options are as follows:

    * If you don't want [`SWAP`](https://wiki.archlinux.org/title/Swap) space, then press have this be the root partition. Type `+64G` and press `Enter`. It will report `Created a new partition 2 of type 'Linux filesystem' and of size 64 GiB.` Press `t` and it will ask `Partition number (1-2, default 2):`. Press `Enter`. It will ask `Partition type or alias (type L to list all):`. Type `23` and press `Enter`. It will report `Changed type of partition 'Linux filesystem' to 'Linux root (x86-64)'.`
    * If you want [`SWAP`](https://wiki.archlinux.org/title/Swap) space, you need to determine how much is necessary. I'll leave that to you, but 8-16GB is probably fine. In that case, for the last sector of partition 2, type `+xG` replacing `x` with the number of gigabytes you need, then press `Enter`. It will report `Created a new partition 2 of type 'Linux filesystem' and of size x GiB.` Obviously, `x` is whatever space you allocated. Press `t`. It will ask `Partition number (1,2, default 2):`. Press `Enter`. It will ask `Partition type or alias (type L to list all):`. Type `19` and press `Enter`. It will report `Changed type of partition 'Linux filesystem' to 'Linux swap'.`

> ### 💡 __**The rest of this guide is listing the example outputs as if you created a SWAP partition. Adapt accordingly!**__

* **If you didn't create a SWAP partition, ignore this step and skip to the next bullet point**. Let's create the root partition. Press `n`. It will ask `Partition number (3-128, default 3):`. Press `Enter`. It will ask `First sector (x-x, default x):`. Press `Enter`. It will then ask `Last sector, +/-sectors or +/-size{K,M,G,T,P} (x-x, default x):`. Type `+64G` and press `Enter`. It will report `Created a new partition 3 of type 'Linux filesystem' and of size 64 GiB.` Press `t` and it will ask `Partition number (1-3, default 3):`. Press `Enter`. It will ask `Partition type or alias (type L to list all):`. Type `23` and press `Enter`. It will report `Changed type of partition 'Linux filesystem' to 'Linux root (x86-64)'.`

* Last partition. Press `n`. It will ask `Partition number (4-128, default 4):`. Press `Enter`. We will use the remaining space on the drive for the home partition, so press `Enter` twice. It will report `Created a new partition 4 of type 'Linux filesystem' and of size x GiB.` Press `t` and it will ask `Partition number (1-4, default 4):`. Press `Enter`. Type `42` and press `Enter`. It will report `Changed type of partition 'Linux home' to 'Linux home'.`

* Let's verify everything is correct. Press `p`, and you'll see an output like below:

```
Disk /dev/nvme0n1: 931.51 GiB, 1000204886016 bytes, 1953525168 sectors
Disk model: Samsung SSD 970 EVO Plus 1TB            
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: 2ED807B6-4DE9-40BF-B8BA-D35896964D0E

Device             Start        End    Sectors   Size Type
/dev/nvme0n1p1      2048    4196351    4194304     2G EFI System
/dev/nvme0n1p2   4196352   37750783   33554432    16G Linux swap
/dev/nvme0n1p3  37750784  171968511  134217728    64G Linux root (x86-64)
/dev/nvme0n1p4 171968512 1953523711 1781555200 849.5G Linux home
```

If it looks correct, type `w` and press `Enter` to save our partition scheme.

## Format the Partitions

Let's format. We want to use [`Btrfs`](https://wiki.archlinux.org/title/Btrfs) as a file system instead of [`ext4`](https://wiki.archlinux.org/title/Ext4) or something else.
* Run `mkfs.fat -F 32 /dev/nvme0n1p1`
* If you created SWAP, run `mkswap /dev/nvme0n1p2`
* Run `mkfs.btrfs -L root /dev/nvme0n1p3`
* Run `mkfs.btrfs -L home /dev/nvme0n1p4`

## Creating Btrfs Partitions and Mounting

* Mount the root partition so we're directly modifying files on it. Run the following:

```
mount /dev/nvme0n1p3 /mnt
```

* Create the following Btrfs partitions:

```
btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@cache
btrfs subvolume create /mnt/@log
```

Unmount root so we can remount it properly with Btrfs:

```
umount /mnt
```

Mount volumes with specific security flags and Btrfs:

```
mount -o subvol=@,compress=zstd,ssd,noatime /dev/nvme0n1p3 /mnt
mkdir -p /mnt/{boot,efi,home,var/cache,var/log}
mount -o subvol=@cache,compress=zstd,ssd,noatime /dev/nvme0n1p3 /mnt/var/cache
mount -o subvol=@log,compress=zstd,ssd,noatime /dev/nvme0n1p3 /mnt/var/log
mount -o compress=zstd,ssd,noatime,nodev,nosuid /dev/nvme0n1p4 /mnt/home
```

<details>
<summary>What are these flags?</summary>

1: `nodev`: Disallows the creation or use of "device nodes" (like a fake /dev/sda) in the home folder. This prevents a specific type of privilege escalation.

2: `nosuid`: Prevents "Set User ID" bits from working. Even if a malicious file in the home folder is marked "run as root," the kernel will ignore it.

3: `compress=zstd`: Saves space and NAND wear for Btrfs.

4: `ssd`: Enables Btrfs-specific flash optimizations.

5: `noatime`: Huge for servers. It stops the drive from writing a "last accessed" timestamp every time a file is read.
    
</details>

* Mount the EFI system:

```
mount /dev/nvme0n1p1 /mnt/efi
```

* If you created SWAP, enable it:

```
swapon /dev/nvme0n1p2
```

## Install Arch Linux onto the System

* No configuration (except for /etc/pacman.d/mirrorlist) gets carried over from the live environment to the installed system. The only mandatory package to install is base, which does not include all tools from the live installation, so installing more packages is frequently necessary. I'm going to do my best to get all you need right here, but [you need to research exactly what you need](https://wiki.archlinux.org/title/Installation_guide#Install_essential_packages).
    * CPU microcode updates, `amd-ucode` or `intel-ucode`, for hardware bug and security fixes
    * We did not cover either parameter, but if you did it on your own, utilities for accessing and managing RAID or LVM
    * Specific firmware for other devices not included in `linux-firmware`, though that should cover just about everything for most people
    * People can be picky about console text editors. After using a few, I've settled on and prefer `micro`, so I will be installing that and utilizing that editor for the rest of all guides.
    * I added `NetworkManager` so you don't have to manually mess with networks. If you only have a static Ethernet connection, it's just unnecessary. I'll add both configurations.
    * A firewall frontend, like `ufw` or `firewalld`. For the sake of minimal packages and granual control, we're sticking with the modern version of `iptables`, `nftables`.

**Do not blindly install packages unless you know you need them.** That includes what I'm listing below. For those who decide not to listen, I'm only going to do my best to include packages that nobody should have a problem having and won't be unnecessary for anyone, but you will have a broken installation without figuring out what else you need. Append (add at the end of my command before pressing `Enter`) with the packages you need for your configuration, especially the vendor-specific CPU microcode (`amd-ucode` or `intel-ucode`).

```
pacstrap -K /mnt base linux-hardened linux-firmware btrfs-progs man-db man-pages texinfo sudo grub grub-btrfs timeshift efibootmgr podman networkmanager micro iptables-nft
```

* We need to make a file that looks at everything we just manually mounted (including the Btrfs subvolumes with the specific nodev,nosuid security flags. The file is a permanent record so the system knows how to boot itself. Run:

```
genfstab -U /mnt >> /mnt/etc/fstab
```

Verification is mandatory here. Because we are building a hardened system, let's verify the security flags actually made it into the file. Run:

```
micro /mnt/etc/fstab
```

Look for the line ending in /home. It should contain `nodev,nosuid` in the options column.

* To directly interact with the new system's environment, tools, and configurations for the next steps as if you were booted into it, change root into the new system:

```
arch-chroot /mnt
```

## Details about `micro`

* You're starting to get how to use a terminal and some of Linux's workings, aren't ya? Since I'm going to use micro, and assume that you are, let me explain some of the hotkeys. Basically, it works like a normal word processor.
    * `Ctrl`+`Shift`+`C` is copy from an external source, as in outside `micro`.
    * `Ctrl`+`Shift`+`V` is paste from an external clipboard, like the one from the system.
    * `Ctrl`+`C` is copy from inside `micro`.
    * `Ctrl`+`V` is paste from inside `micro`.
    * `Ctrl`+`S` is how you save the current file.
    * `Ctrl`+`Q` is how you close the current file. It will ask to save before actually quitting if you have made changes in the file. You would press `y` to save and quit, `n` to discard changes and quit, or press `Esc` to cancel the closing-file action.
    * `Ctrl`+`F` is how you can find things in a file.
[Here is the full list of default hotkeys](https://github.com/micro-editor/micro/blob/master/runtime/help/defaultkeys.md).

## Set System Time

* Let's set the system time zone. [Find which timezone applies to you here](https://gist.github.com/adamgen/3f2c30361296bbb45ada43d83c1ac4e5). Edit below and run:

```
ln -sf /usr/share/zoneinfo/Area/Location /etc/localtime
```

For me being in Utah, it would be `ln -sf /usr/share/zoneinfo/America/Boise /etc/localtime`

* Run `hwclock` to generate /etc/adjtime:

```
hwclock --systohc
```

This command assumes the hardware clock is set to UTC.

* To prevent clock drift and ensure accurate time, set up time synchronization. Run:

```
micro /etc/systemd/timesyncd.conf
```

Uncomment (remove the hash `#`) the line for `FallbackNTP`. It should look like below:

```
[Time]
#NTP=
FallbackNTP=0.arch.pool.ntp.org 1.arch.pool.ntp.org 2.arch.pool.ntp.org 3.arch.pool.ntp.org
#...
```

I usually use `Ctrl`+`Q` to save changes and go back to the command line interface. So press that and then press `Y` to save and exit. Otherwise, press `Ctrl`+`S` to save and then `Ctrl`+`Q` to quit.

* I believe `systemd-timesyncd` is enabled by default, but just in case, run:

```
systemctl enable systemd-timesyncd.service
```

## Regional Settings

* We need to use the correct region and language-specific formatting. Uncomment the UTF-8 locale you need. Run:

```
micro /etc/locale.gen
```

If you're American, all you need to really do is uncomment `en_US.UTF-8`.

* Generate the locale. Run:

```
locale-gen
```

* Create a `locale.conf` file and set the language with the same locale as before. Run:


```
micro /etc/locale.conf
```

* If you changed the console keyboard layout from the default US keymap (the first command you would have done after booting the live environment), make the changes persistent in `vconsole.conf`. Run:

```
micro /etc/vconsole.conf
```

Change as necessary. For example:

```
KEYMAP=de-latin1
```

## Create a `hostname`

* Assign a consistent, identifiable name to this system. It makes things like SSH easier. Run:

```
micro /etc/hostname
```

I use something simple like `homeserver`, but put the hostname you'll remember in it. And yes, the single word is all that will be in the file.

## Enable Services

* Enable internet access.

<details>
<summary>NetworkManager</summary>

    If you used `NetworkManager` service, run:

```
systemctl enable networkmanager
```

</details>

<details>
<summary>systemd-networkd (standard)</summary>

You have two choices for your server's IP. You can use DHCP and set a Static Reservation in your router (easiest), or you can manually define a Static IP in our network config file. For a server, a static assignment is highly recommended to prevent network-wide outages. **Only create a manual configuration instead of DHCP if you know what you are doing.**

* Find the information for your network controller first. Run:

```
networkctl
```

For Ethernet, it will start with `en`.

* Create a configuration for a single connection off of Ethernet. Run the following command:

```
micro /etc/systemd/network/20-wired.network
```

Input the following and edit as necessary:

```
[Match]
Name=en*

[Network]
Address=192.168.1.200/24
Gateway=192.168.1.1
DNS=1.1.1.1
# Optional: Uncomment DHCP=yes here ONLY if you want it to 
# fall back to DHCP if the static assignment fails.
# DHCP=yes
```

</details>

* Enable the firewall. Run:

```
systemctl enable nftables
```

* Enable Timeshift to create snapshots. Run:

```
systemctl enable timeshift
```

We do, however, need to create a retention policy. I believe 5 daily and 3 weekly is sufficient. Edit the JSON config file:

```
micro /etc/timeshift/timeshift.json
```

Ensure the following keys are set to true:

```
  "btrfs_mode" : "true",                                                                                                                                    
  "include_btrfs_home" : "true",                                                                                                                                                                                                                                                                                                                                                                                 
  "schedule_weekly" : "true",                                                                                                                               
  "schedule_daily" : "true",        
```

Look for these keys and set them to your preference:

```
    "count_daily" : "5"

    "count_weekly" : "3"

    "count_monthly" : "0"
```

* Enable GRUB entries for the Snapshots. Run:

```
systemctl enable grub-btrfsd.service
```

* To make grub-btrfsd work with Timeshift, edit the service by running:

```
systemctl edit --full grub-btrfsd
```

and change the line:

```
ExecStart=/usr/bin/grub-btrfsd /.snapshots --syslog
```

to:

```
ExecStart=/usr/bin/grub-btrfsd --syslog --timeshift-auto
```

## Quick Notes

* Creating a new `initramfs` at this point is usually not required, because [`mkinitcpio`](https://wiki.archlinux.org/title/Mkinitcpio) was run on installation of the kernel package with `pacstrap`.

* For [LVM](https://wiki.archlinux.org/title/Install_Arch_Linux_on_LVM#Adding_mkinitcpio_hooks), [system encryption](https://wiki.archlinux.org/title/Dm-crypt), or [RAID](https://wiki.archlinux.org/title/RAID#Configure_mkinitcpio), modify [`mkinitcpio.conf`](https://man.archlinux.org/man/mkinitcpio.conf.5) and recreate the initramfs image. Remember, I didn't cover any of those things.

* If you have changed the default console keymap (first thing you would have done after booting live installation), only recreating the initramfs is required. Run:

```
mkinitcpio -P
```

## Set Root Password

* Set a secure password for the root user for the big-boy commands. This helps protect your system in case of unauthorized access. Run:

```
passwd
```

## Create New User

Follow the instructions.

* Create your primary user. Replace `username` with the one you want and run:

```
useradd -m -G wheel -s /bin/bash username
```

<details>
<summary>What does this all do?</summary>

1: `-m`: Creates the /home/username directory.

2: `-G wheel`: Adds the user to the "wheel" administrative group.

3: `-s /bin/bash`: Sets the default shell to Bash.

</details>

* Set a password. Replace username with your username and run:

```
passwd username
```

## Allowing User to Use Root Commands

Follow the instructions.

* Add `wheel` to the `sudo` list. `sudo` is short for "Superuser Do". By default, the `wheel` group exists, but it doesn't have "permission" to do anything. You have to tell the system that anyone in the wheel group is allowed to use sudo. Instead of editing the main `/etc/sudoers` file which can be overwritten during updates, the we're going to use a drop-in file. Run:

```
micro /etc/sudoers.d/10-admin
```

Add this single line:

```
%wheel ALL=(ALL:ALL) ALL
```

## Install GRUB (Grand Unified Bootloader)

* We need to install GRUB so we can allow the system to actually boot Linux. Run:

```
grub-install --target=x86_64-efi --efi-directory=/efi --bootloader-id=GRUB --removable
```

After the above installation completed, the main GRUB directory is located at `/boot/grub/`. We used the `--removable` option so GRUB will be installed to `/efi/EFI/BOOT/BOOTX64.EFI` and you will have the additional ability of being able to boot from the drive in case EFI variables are reset, or you move the drive to another computer. **Some desktop motherboards will only look for an EFI executable in this location, making this option mandatory, in particular with MSI boards.**

* We need to add things to the `GRUB` configuration. Run:

```
micro /etc/default/grub
```

Find the line starting with `GRUB_CMDLINE_LINUX_DEFAULT`. It should be the sixth line down. It probably looks like this right now:

```
GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet"
```

Append the line with `lsm=landlock,lockdown,yama,integrity,apparmor,bpf`. So it should look like this, wrapped in quotes:

```
GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet lsm=landlock,lockdown,yama,integrity,apparmor,bpf"
```

<details>
<summary>What does this all do?</summary>

1: `landlock`: Allows unprivileged processes (like your Podman containers) to sandbox themselves.

2: `lockdown`: Prevents even the root user from tampering with the running kernel memory.

3: `yama`: Restricts ptrace (stops one process from "snooping" on the memory of another).

4: `integrity`: Prepares the system for EVM/IMA (verifying that files haven't been modified).

5: `apparmor`: The big-boy. Restricts what files/network ports applications can touch.

6: `bpf`: Enables the modern "eBPF" security features (used for high-performance monitoring).

</details>


* The main configuration file /boot/grub/grub.cfg needs to be generated. Run:

```
grub-mkconfig -o /boot/grub/grub.cfg
```

## Disable Root Account

* Now that we have a user with `sudo` powers, we should lock the root account. This prevents anyone from logging in directly as root, even if they guess the password, forcing everyone to use your named user account instead. Using a root account can be dangerous. Run:

```
passwd -l root
```

## Get Into Your System

We're ready.

* Exit the `chroot` environment by pressing `Ctrl`+`D`, or typing `exit` and pressing `Enter`.

* Before unmounting, tell the kernel to write any data remaining in RAM to the physical NVMe. This ensures your newly created Btrfs metadata and logs are finalized. Run:

```
sync
```

* Unmount everything. Run:

```
umount -R /mnt
```

If you want, you can check if anything is still attached with:

```
lsblk
```

Your partitions should no longer show anything in the `MOUNTPOINT` column.

* Reboot. Simply type `reboot` and press `Enter`. Assuming this is the only drive on the computer, it should automatically start GRUB, but if not, consult the motherboard's manual and go into the boot menu again, selecting GRUB.

---

# Next Steps:

* [🛠 Install and Configure Useful Services](./useful-services.md) - Snapshots, SSH, yay, cron jobs

**or**

* [📦 Configure Rootless Podman](./rootless-podman.md) - Necessary changes for our setup
