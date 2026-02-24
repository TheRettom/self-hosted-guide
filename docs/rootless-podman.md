# Configuring Podman to Be Rootless

This document explains how to configure Podman to run without root privileges on Arch Linux Hardened and enable settings for future services. Running rootless ensures that even if a container is compromised, the attacker does not gain root access to the host.

## Notes

* `~` in a directory string refers to the user's home location, which by default should be `/home/$USER`.
* `%h` is another way for `systemd` to refer to the home directory.

## Ensure Packages Are Present

* Run the following command:

```
sudo pacman -Sy podman passt
```

## sysctl Changes

Because linux-hardened disables many unprivileged features by default, and leaves some settings at low values, the following parameters are required in /etc/sysctl.d/99-sysctl.conf to allow Podman and Networking to function.

* Run the following command:

```
sudo micro /etc/sysctl.d/99-sysctl.conf
```

Input this in the file:

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

* `vm.overcommit_memory = 1` This tells the kernel to not be as strict when apps ask for memory. Instead of checking if it currently has enough RAM to cover every potential request, it grants the request and assumes the app won't use it all at once. Many containerized applications (and languages like Go, which Caddy is built in) pre-allocate large chunks of virtual memory. Without this setting, the kernel might refuse to start a container because it thinks it's out-of-memory even when the physical RAM is mostly empty.

* `net.netfilter.nf_conntrack_udp_timeout = 180` UDP is a "stateless" protocol — it doesn't have a close-connection signal like TCP. The kernel has to guess when a UDP conversation is over so it can clear its tracking table. The default for this is 30 seconds. In the DNS setup we will be configuring, extending this to 180 seconds reduces the overhead of constantly rebuilding "connection" entries for repeated DNS queries from the same devices. It helps the network feel slightly more responsive and reduces firewall CPU usage.

</details>

## Add Ranges of UIDs and GIDs (User IDs and Group IDs) for Services to Run Rootless While Remaining Secure

When podman runs in rootless mode, a user namespace is automatically created for the user, defined in `/etc/subuid` and `/etc/subgid`. Containers created by a non-root user are not visible to other users and are not seen or managed by Podman running as root.

* Create the files if they're missing:

```
sudo touch /etc/subuid /etc/subgid
```

* Add the ID ranges:

```
sudo usermod --add-subuids 10000-75535 $USER
sudo usermod --add-subgids 10000-75535 $USER
```

* Podman doesn't always "see" the new ranges immediately if you already touched Podman once. Run:

```
podman system migrate
```

This refreshes Podman's internal map of the user namespaces to match the files we just edited.

## Help Services with Persisting

We need to have `linger` enabled. Linger allows user services to continue running even after the user has logged out. This is useful for running long-running processes or services, and it is necessary to start services immediately after rebooting without needing the user to be actively logged in.
* Run:

```
loginctl enable-linger $USER
```

## Create Directories for Podman

Create the directories necessary for Podman. The user directory that Podman reads from to generate the systemd `.service` files are in `~/.config/containers/systemd`. The container storage directory is `.local/share/containers/storage`. These are all compliant with OCI (Open Container Initiative) standards.

* Run the following command:

```
mkdir -p ~/.config/containers/systemd
mkdir -p ~/.local/share/containers/storage
```

<details>
<summary>What is `mkdir -p`?</summary>

Make the specified directory, and if any parent directory is missing, create those as well.

</details>

## Image Registries

By default, no container image registries are configured in Arch Linux. This means unqualified searches like `podman search httpd` or `podman pull pihole:latest` will not work. To make pulling images easier, we need to create a [`registries.conf`](https://man.archlinux.org/man/containers-registries.conf.5). The user directory that Podman reads from is in `~/.config/containers`.

* Run the following command:

```
micro ~/.config/containers/registries.conf
```

Add the line:

```
unqualified-search-registries = ["docker.io", "ghcr.io"]                                                                                                  
                                                                                                                                                          
[[registry]]                                                                                                                                              
location = "docker.io"                                                                                                                                    
                                                                                                                                                          
[[registry]]                                                                                                                                              
location = "ghcr.io" 
```

## Container Settings

Name resolution is handled by subsystems of Podman (`aardvark-dns`), which provide both external DNS (usually through the host's DNS resolver) and container name resolution (e.g. webserver.dns.podman talking to database.dns.podman). Since we will be running Pihole, we need to change the port used by Podman on the host to another port.

* Run the following command:

```
micro ~/.config/containers/containers.conf
```

Add the following:

```
dns_bind_port = 20053
```

## Storage Settings

Since we are on Arch Linux with a modern kernel, Podman will automatically use Native OverlayFS. While there is a one-time delay when building/pulling an image for the first time, the runtime performance is superior to fuse-overlayfs and more stable for rootless users than the specialized `btrfs` driver. Previously I would have suggested `fuse-overlayfs` or `btrfs`, but that is no longer the case. Nothing needs to be reconfigured.

* To verify that `Native Overlay` is used, run:

```
podman info | grep -i overlay
```
It should show `graphDriverName: overlay` and `Native Overlay Diff: "true"`.

---

## Next Step

Before running any service, we need to configure Caddy with Crowdsec.

* [🔌 Reverse Proxy for Access to Services](https://github.com/TheRettom/self-hosted-guide/Caddy/README.md) - Caddy with Crowdsec

**or**

* [🛠 Install and Configure Useful Services](./docs/useful-services.md) - `yay`, SSH, `systemd` timers, `pacman` hooks, tips

If you missed setting up useful services, consider it now. It's optional, but **highly recommended**.
