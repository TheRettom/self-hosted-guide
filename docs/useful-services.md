# Useful Services

These are configurations and services that make your life much easier, and improve your server overall.

## yay
[`yay`](https://github.com/Jguer/yay) is an AUR (Arch User Repository) helper, but more specifically, it is a Pacman wrapper. `yay` (Yet Another Yogurt) has become the industry standard for Arch users.

<details>
<summary>Why `yay` versus another AUR helper?</summary>

1: By default, `yay` asks if you want to see the `PKGBUILD` and `.install` scripts before executing them. On a hardened server, this is mandatory. You should never install an AUR package without checking who is maintaining it and what scripts it’s running as root.

2: `yay` is excellent at handling "ghost" dependencies and orphaned packages. It uses a separate cache for AUR builds, keeping your main system directories clean.

3: You don't have to remember if a package is in the official repos or the AUR. `yay -S package-name` checks both.

4: `yay` asks all its questions about dependencies, providers, and clean-builds at the beginning of the process. You don't have to sit and watch a 10-minute compile just to answer "Yes" to a final prompt. You may, however, need to do other things, like entering the `sudo` password later on during the install.

5: `yay` can show you the `diff` (the exact changes) between your current version and the new version of a package. This is a massive security win — you can see exactly what code was added in an update.

Without an AUR helper, you have to:

1: git clone the repository.

2: `cd` into it.

3: Run `makepkg -si`.

4: Later on, manually check for updates.

`yay` simply automates that tedious loop while keeping the logic transparent.

</details>

* The initial installation of `yay` can be done by cloning the PKGBUILD and building with `makepkg`. We also need to make sure we have the base-devel package group installed. Install with the following commands:

```
sudo pacman -S --needed git base-devel
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
```

## SSH

If you're like me, you don't want to have dedicated peripherals for the server. That means you'll need to use [SSH](https://wiki.archlinux.org/title/OpenSSH) to access the server remotely from your devices. I'm running you through configuring it the way I like it. If you want to configure it differently, refer to the linked Arch Linux Wiki.

> 💡 **I highly recommend a hardware security key for security.** It makes your life easier. People might recognize the name YubiCo or YubiKey. Personally, I have only bought the classic [OnlyKey](https://onlykey.io/products/onlykey-color-secure-password-manager-and-2-factor-token-u2f-yubikey-otp-google-auth-make-password-hacking-obsolete) and don't care to use anything else. The fact it requires a pin to use is important to me, especially when I take it outside the home, but that's usually one step that will make people not use it. I also like it because it automatically inputs text and can change input fields if you program that. Do your research and find one you WILL use.

### On the Server

* Get SSHD with:

```
sudo pacman -Sy openssh
```

* Enable the service:

```
sudo systemctl enable sshd.service
```

* Check that the config directory is enabled and that the `Port 22` is commented out:

```
micro /etc/ssh/sshd_config
```

At the top should be:

```
# Include drop-in configurations                                                                                                                            
Include /etc/ssh/sshd_config.d/*.conf  
```

and the first real parameter should have the port commented out, like `#Port 22`. If it is not, do it so our upcoming configuration is not conflicting.


* Make configuration changes, particularly with the listening port. Ideally put it on a random port higher than 1024, like 39901. Edit the configuration file:

```
sudo micro /etc/ssh/sshd_config.d/20-hardened.conf
```

Make changes as necessary and input the following:

```
Port portnumber
AllowUsers yourusername
PermitRootLogin no
PubkeyAuthentication yes
```

Later, we will disable password logins entirely and use public key authentication.

<details>
<summary>Why disable the password and use a public key?</summary>

If a client cannot authenticate through a public key, by default, the SSH server falls back to password authentication, thus allowing a malicious user to attempt to gain access by brute-forcing the password. One of the most effective ways to protect against this attack is to disable password logins entirely, and force the use of SSH keys.

</details>

<details>
<summary>Why change the port from 22?</summary>

If the server is to be exposed to the WAN, this is basically a requirement. Changing the SSH port is a security-by-obscurity move that, while not a replacement for keys, will reduce the number of log entries caused by automated authentication attempts, but will not eliminate them. See [Port knocking](https://wiki.archlinux.org/title/Port_knocking) for related information.

</details>

> 💡 Note: `nftables` is stateless by default in some configs. Ensure you have `ct state established,related accept` in your rules, otherwise, you will be able to send an SSH request but won't receive the reply! I'll add them in the next step.

* Open the firewall for that port. Run:

```
sudo micro /etc/nftables.conf
```

Under `chain input`, edit the port number if you want and add the line `tcp dport YOUR-PORT-NUMBER accept comment "allow sshd on custom port"`

```
table inet filter {                                                                                                                                         
  chain input {
      ct state invalid drop comment "early drop of invalid connections"                                                                                       
      ct state {established, related} accept comment "allow tracked connections"  
      ... existing rules ...
      tcp dport 39901 accept comment "allow sshd on custom port"
      ... existing rules ...
  }
}
```

**Before we reload the firewall configuration:** If there is a typo in `/etc/nftables.conf`, the reload might fail, or worse, leave the firewall in an inconsistent state. Run the following:

```
sudo nft -c -f /etc/nftables.conf
```

`-c` means check and will report errors without applying the changes. If all is good, reload `nftables`:

```
sudo systemctl reload nftables
```

* Start the SSH service:

```
sudo systemctl start sshd.service
```

### On the Client (your regular PC/laptop)

* Get SSHD with:

```
sudo pacman -Sy openssh
```

* Make configuration changes to simplify connecting. Run:

```
micro ~/.ssh/config
```

Edit the following as necessary and put it in the file:

```
# global options
User user1

# host-specific options
Host myserver
    Hostname server-ip-address
    Port     port
    User     username
```

With such a configuration, the following commands are equivalent:

```
$ ssh -p port username@server-ip-address
$ ssh myserver
```

* Create an [SSH key](https://wiki.archlinux.org/title/SSH_keys). Ed25519 keys generated via ssh-keygen (stored on disk) are different from Resident Keys or FIDO2 keys stored on the hardware. Edit the comment (`-C "comment"`) and run:

```
ssh-keygen -t ed25519 -a 100 -C "user@yourserver"
```
> 💡 Note: If you want to use OnlyKey or YubiKey for the SSH auth, the command is slightly different: `ssh-keygen -t ed25519-sk`. You would also need the `libfido2` package to be installed on the client.

When it asks for a passphrase, **use one**. Even a simple one is better than none. It encrypts the key on your local disk.

* Now you need to put the public part of the key onto the server. Edit `your-port-number` and `username@server-ip` as necessary and run:

```
ssh-copy-id -p your-port-number username@server-ip
```

> Note: `id_ed25519.pub` is your public key. This stays on the server in ~/.ssh/authorized_keys. It is safe to share. `id_ed25519` stays only on your local device. Never upload this anywhere.

* Try using SSH now to remotely access your server. If it's successful, congrats! You can press `Ctrl`+`D` to exit if you'd like. **BUT**...

* **BACK ON YOUR SERVER**, add the following line to `/etc/ssh/sshd_config.d/20-hardened.conf`:

```
PasswordAuthentication no
```

Restart `sshd` on the server with:

```
sudo systemctl restart sshd.service
```

> ### ⚠️ **Any time you make changes to SSH configurations after initially connecting, DO NOT close your current terminal session until you have verified the new connection works in a separate window. If there is a typo in your config, your current session is your only way to fix it!**

### WAN Access

If you want to SSH into your server from outside the LAN (local area network, as in your router), then you need to set up port forwarding on your router. I do not cover that on this guide, so refer to your router's manual on how to do that.

## Systemd Timers

A healthy server requires automated upkeep. We use Systemd Timers to handle this.

### 1: `paccache.timer`: Cleans the Pacman cache to save space (Weekly).

<details>
<summary>About paccache.timer</summary>

* Pacman stores its downloaded packages in `/var/cache/pacman/pkg/` and does not remove the old or uninstalled versions automatically. This has some advantages:
    
    * It allows to downgrade a package without the need to retrieve the previous version through other means, such as the Arch Linux Archive.
    * A package that has been uninstalled can easily be reinstalled directly from the cache directory, not requiring a new download from the repository.

However, it is necessary to deliberately clean up the cache periodically to prevent the directory to grow indefinitely in size.

* The `paccache` script, provided within the pacman-contrib package, deletes all cached versions of installed and uninstalled packages, except for the most recent three, by default.

```
paccache -r
```

* The `paccache.timer` discards unused packages weekly. You can configure the arguments for the service in `/etc/conf.d/pacman-contrib`.

</details>

```
sudo systemctl enable paccache.timer
```

### 2: `btrfs-scrub@.timer`: Checks for data corruption/bit-rot (Monthly).

```
sudo systemctl enable --now btrfs-scrub@-.timer
sudo systemctl enable --now btrfs-scrub@home.timer
```

<details>
<summary>About btrfs-scrub.timer</summary>

The `btrfs-progs` package (that we installed when installing Arch Linux) has the `btrfs-scrub@.timer` unit for monthly scrubbing the specified mountpoint. Enable the timer with an escaped path, e.g. `btrfs-scrub@-.timer` for `/` (root) and `btrfs-scrub@home.timer` for `/home`.

You can also run the scrub by starting `btrfs-scrub@.service` (with the same encoded path). The advantage of this over btrfs scrub (as the root user) is that the results of the scrub will be logged in the systemd journal.

> ⚠️ **Warning: On large NVMe drives with insufficient cooling (e.g. in a laptop), scrubbing can read the drive fast enough and long enough to get it very hot. If you are running scrubs with systemd, you can easily limit the rate of scrubbing with the `IOReadBandwidthMax` option described in systemd.resource-control by using a drop-in file. [Refer to this article](https://wiki.archlinux.org/title/Btrfs#Start_with_a_service_or_timer).**

</details>

### 3: `timeshift.timer`: Creates snapshots to restore to on a timed interval.

```
yay timeshift-systemd-timer
```

* Check the `timeshift` timer status:

```
systemctl list-timers | grep timeshift
```

* Make sure Timeshift is configured: Run `sudo timeshift --list` to make sure you've actually selected your Btrfs drive and snapshot levels (Daily/Weekly), otherwise the timer will trigger but have nothing to do. We should see `@` and `@home`.

> Note: Try to keep total partition usage below 80%. If you go higher, Btrfs snapshots may fail to create, or the system may slow down. Use `timeshift --delete-all` to clear out old snapshots if you're running low on space."

## Pacman Hooks

* The `timeshift` hook.

In case the pacman hook directory is not created, run:

```
sudo mkdir /etc/pacman.d/hooks
```

Create the timeshift hook:

```
sudo micro /etc/pacman.d/hooks/00-timeshift-autosnap.hook
```

Input:

```
[Trigger]                                                                                                                                                   
Operation = Upgrade                                                                                                                                         
Type = Package                                                                                                                                              
Target = *                                                                                                                                                  
                                                                                                                                                            
[Action]                                                                                                                                                    
Description = Creating Timeshift snapshot before upgrade...                                                                                                 
Depends = timeshift                                                                                                                                         
When = PreTransaction                                                                                                                                       
Exec = /usr/bin/timeshift-autosnap                                                                                                                          
AbortOnFail
```


> 💡 Tip: To keep the attack surface minimal, periodically review your explicitly installed packages with pacman -Qe. Remove unnecessary software using pacman -Rs to ensure dependencies are cleared out along with the main package.


## Post-Update Procedures

After running `sudo pacman -Syu`, your work isn't quite done. Upgrades (`sudo pacman -Syu`) are typically not applied to existing processes. You must restart processes to fully apply the upgrade.

The `archlinux-contrib` package provides scripts called `checkservices` to check for processes running with outdated libraries and prompts the user if they want them to be restarted, and `pacdiff` to merge `.pacnew` files.

The kernel is particularly difficult to patch without a reboot. A reboot is always the most secure option, but if this is very inconvenient [kernel live patching](https://wiki.archlinux.org/title/Kernel_live_patching) can be used to apply upgrades without a reboot.

* Get the package:

```
sudo pacman -Sy archlinux-contrib
```

* Because `archlinux-contrib` relies on the VIM editor and we have micro installed, we need to change the configuration. Run:

```
micro ~/.bashrc
```

Add these lines at the bottom:

```
export EDITOR='micro'
export VISUAL='micro'
export DIFFPROG='micro -d'
```

Then source the file to apply it now:

```
source ~/.bashrc
```

> ⚠️ Warning: Avoid merge conflicts. When using `pacdiff`, ensure you fully resolve any conflicts. If you see `<<<<<<<` or `>>>>>>>` in a config file after a merge, the file is corrupted and will cause service failures or Pacman errors.
If a command like `podman` or `pacman` suddenly throws a syntax error or `expected = but got <` error after an update, you likely have a merge conflict. Search the config file for `<<<<<<<`. These markers are injected if a `pacdiff` session wasn't finished correctly. Delete the markers and the duplicate lines to restore the service.

## Updating Mirrors

You should periodically update pacman's mirrorlist, as the quality of mirrors can vary over time, and some might go offline or their download rate might degrade. `reflector` can automatically update the mirror list weekly. Reflector is a Python script which can retrieve the latest mirror list from the Arch Linux Mirror Status page, filter the most up-to-date mirrors, sort them by speed and overwrite the file `/etc/pacman.d/mirrorlist`.

* Get it:

```
sudo pacman -Sy reflector
```

Reflector ships with `reflector.service`, which runs Reflector using the options specified in `/etc/xdg/reflector/reflector.conf`. Default options are fine. It also has a `systemd timer` which runs weekly, so...

* We're going to enable it:

```
sudo systemctl enable reflector.timer
```

> 💡 Note: If changing the configuration, it is typically not a good idea to filter by country; there are only a finite number of mirrors in a single country. Network throughput is only partly determined by geographical distance.

* If you ever need an emergency fix, run the following:

```
sudo reflector --latest 20 --protocol https --sort rate --save /etc/pacman.d/mirrorlist
```

It'll take some time.

## Next Step

* [📦 Configure Rootless Podman](./rootless-podman.md) - Necessary changes for our setup
