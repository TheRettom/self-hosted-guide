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

# Installing Arch Linux
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

* **Warning: You must replace sdX with your actual thumb drive (like sdb or sdc). If you point this at `/dev/sda` (your internal drive), dd will faithfully overwrite your entire operating system __without__ asking "Are you sure?**" Edit the `/dev/sdX` in the command below to whatever matches it and run it:

```
sudo dd if=archlinux-*.iso of=/dev/sdX bs=4M status=progress oflag=direct
```

<details>
<summary>Breakdown of that command</summary>

* `sudo`: Superuser Do. Writing directly to a device like `/dev/sdX` is a restricted action. Without sudo, Linux will block the command.

* `dd`: Data Definition. It copies data from one place to another bit-for-bit.

* `if=archlinux-*.iso`: Input File. This tells `dd` where the data is coming from. The asterisk (*) is a wildcard that tells the shell to find the file that matches that pattern, like `archlinux-2026.02.01-x86_64.iso`.

* `of=/dev/sdX` Output File. This is the most dangerous part of the command. In Linux, a USB drive is treated as a file located in `/dev/`.

* `bs=4M`: Block Size. This tells dd to read and write in 4-megabyte chunks. The default block size is only 512 bytes. Writing a 1GB ISO in 512-byte increments is like moving a pile of sand with a teaspoon — it works, but it takes forever. Using 4MB is like using a large shovel; it’s much faster and matches the internal "page size" of most USB flash memory.

* `status=progress`: A Visualizer. By default, `dd` is a silent command. You would see a blinking cursor for 10 minutes and wonder if it's frozen. This flag provides a live update of how many bytes have been copied, the elapsed time, and the transfer speed.

* `oflag=direct`: Direct I/O. Normally, Linux uses a "buffer cache" — it says "I've finished writing!" when the data is actually just sitting in your RAM waiting to be sent to the USB. `oflag=direct` tells the kernel to bypass that cache and write directly to the physical USB hardware. This makes the status=progress bar much more accurate and prevents the command from "hanging" at 99% while the system clears the cache.

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

* `sudo`: Requires your Mac admin password.

* `if=`: The input file. Drag and drop the ISO file into the terminal window to auto-fill the path.

* `of=`: The output file, as in your thumb drive.

* `bs=1m`: Sets the block size to 1 megabyte to speed up the transfer.

* `status=progress`: Shows you how much has been copied.

</details>

* Once the command finishes, macOS might pop up a warning saying, `The disk you inserted was not readable by this computer.` **This is normal because your Mac doesn't recognize the Linux filesystem.**

* Eject the drive properly before pulling it out. Edit the following as needed an run:

```
diskutil eject /dev/diskN
```

</details>


## sysctl Changes
Because linux-hardened disables many unprivileged features by default, the following parameters are required in /etc/sysctl.d/99-hardened.conf to allow Podman and Networking to function:
    /etc/sysctl.d/99-sysctl.conf
```
kernel.unprivileged_userns_clone = 1
net.core.rmem_max = 8388608  
net.core.wmem_max = 8388608
net.ipv4.tcp_rmem = 4096 87380 4194304
net.ipv4.tcp_wmem = 4096 16384 4194304
net.ipv4.ip_unprivileged_port_start=53
vm.overcommit_memory = 1
net.netfilter.nf_conntrack_udp_timeout = 180
```

<details>
<summary>What are these doing?</summary>

* `kernel.unprivileged_userns_clone = 1` On the linux-hardened kernel, the ability for a regular user to create a "User Namespace" is disabled by default for security. However, Rootless Podman cannot function without this. It uses these namespaces to map the local user to a virtual root inside the container. While enabling this slightly increases the attack surface for local privilege escalation, it is the prerequisite for running your services without actual root privileges, which is a massive net gain for security.

* `net.core.rmem_max` & `net.core.wmem_max = 8388608` These set the maximum OS receive and send buffer sizes for all types of connections. For a DNS server like Unbound, which handles thousands of small UDP packets, having a larger buffer prevents the kernel from dropping packets during a sudden spike in traffic.

* `net.ipv4.tcp_rmem` & `net.ipv4.tcp_wmem` These define the (Min, Default, Max) memory bytes for TCP sockets. We set these to scale up to 4MB. This ensures that Caddy, serving our container web traffic as a reverse proxy, can handle high-bandwidth requests smoothly without running out of memory for individual socket connections.

* `net.ipv4.ip_unprivileged_port_start = 53` Traditionally, only root can use ports below 1024. By setting this to 53, we are allowing ourselves as a non-root user to bind to port 53 specifically for Pihole. We want Pihole to listen on its standard DNS port (53) without needing to start the container as root or use complex port-forwarding hacks which might not even work.

* `vm.overcommit_memory = 1` This tells the kernel to not be as strict when apps ask for memory. Instead of checking if it currently has enough RAM to cover every potential request, it grants the request and assumes the app won't use it all at once. Many containerized applications (and languages like Go, which Caddy is built in) pre-allocate large chunks of virtual memory. Without this setting, the kernel might refuse to start a container because it thinks it's "out of memory" even when the physical RAM is mostly empty.

* `net.netfilter.nf_conntrack_udp_timeout = 180` UDP is a "stateless" protocol — it doesn't have a close-connection signal like TCP. The kernel has to guess when a UDP conversation is over so it can clear its tracking table. The default for this is 30 seconds. In the DNS setup we will be configuring, extending this to 180 seconds reduces the overhead of constantly rebuilding "connection" entries for repeated DNS queries from the same devices. It helps the network feel slightly more responsive and reduces firewall CPU usage.

</details>


To ensure AppArmor (Phase 2 of the roadmap) is ready for activation, the following line is added to the GRUB/systemd-boot configuration:
Plaintext

lsm=landlock,lockdown,yama,integrity,apparmor,bpf
