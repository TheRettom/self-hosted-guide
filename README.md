# Rootless Podman on Arch Linux with Services
## Preface
**THIS IS FOR ARCH LINUX AND ROOTLESS PODMAN SETUPS.** I wanted to make things easier for people who want to self-host services if they manage to find my GitHub. Why Rootless Podman and Arch Linux? Security and integration. **DO YOU NEED A HIGH-SECURITY HOSTING PLATFORM?** If you're self-hosting services for you and your family, and maybe some friends — probably not. But for those who are paranoid as a standard, or want maximum privacy and security now rather than later, this is for you.

## 📖 Documentation Tree
* [💻 Host OS Setup](./docs/Installing_Arch_Linux/arch-setup.md) - Arch Linux + `linux-hardened`
* [🛠 Install and Configure Useful Services](./docs/useful-services.md) - Snapshots, SSH, yay
* [📦 Container Runtime](./docs/rootless-podman.md) - Rootless Podman Configuration
* [🔌 Reverse Proxy for Access to Services](./Caddy/caddy-setup.md) - Caddy with Crowdsec
* [🔰 DNS Setup for Privacy and Security](./DNS/pihole-unbound.md) - Pi-hole and Unbound
* [🏠 VPN for Home Network Access While Away](./WireGuard/wireguard-setup.md) - WireGuard
* [🔒 Password Manager](./Vaultwarden/vaultwarden-setup.md) - Vaultwarden
* [👥 Powerful Photo Storage](./Immich/immich-setup.md) - Immich

## 🛡 Security Roadmap

My deployment follows a "Defense in Depth" philosophy. Below is the current status of the security stack and planned enhancements.

### 🏁 Phase 1: Hardened Foundation (Current)
- [x] **Kernel:** Running `linux-hardened` to mitigate kernel-level exploits.
- [x] **User Isolation:** All services run via **Rootless Podman**.
- [x] **Network Hardening:** Restricted `unprivileged_port_start` to 53 for DNS security.
- [x] **Memory Safety:** Configured `vm.overcommit_memory` for stability.

### 🚀 Phase 2: Mandatory Access Control (Planned)
- [ ] **AppArmor Integration:** Move from standard namespace isolation to path-based restriction.
- [ ] **Service Profiling:** Generate custom AppArmor profiles for the Caddy binary to restrict filesystem access.
- [ ] **Audit Logging:** Implement `auditd` to monitor failed syscalls within containers.

### 🔒 Phase 3: Network Privacy
- [ ] **DNS-over-TLS (DoT):** Configure Unbound to use encrypted upstreams.
- [ ] **WireGuard:** Implement a hardened VPN tunnel for remote management.

---

### FAQ

<details>
<summary><strong>Why Arch Linux?</strong></summary>

* Arch follows the KISS (Keep It Simple, Stupid) principle. When you install Arch, you get almost nothing — no pre-installed web servers, no hidden background services, and no open ports you didn't explicitly configure. You cannot exploit a service that isn't there. By building your server from the ground up, you know exactly what is running. In contrast, "easier" distros often ship with dozens of active services (like Avahi, ModemManager, or CUPS) that increase the ways a hacker can get in.

* In security, recency is safety. Vulnerabilities, often referred to as CVEs (Common Vulnerabilities and Exposures), are discovered frequently. Because Arch is a rolling release, you receive the latest security patches and kernel updates within hours or days of their release. On Stable distributions, security fixes are often backported to older versions of software. This is a complex process that can sometimes introduce new bugs or leave systems vulnerable to zero-day exploits that the latest upstream version has already fixed.

* The [Arch Wiki](https://wiki.archlinux.org) is widely considered the best technical documentation in the Linux world. You aren't following "black box" tutorials, you are learning how the system actually works. A secure server is only as good as the administrator's understanding of it. When you configure `Podman` or `sysctl` on Arch, the Wiki forces you to understand why you are changing those bits, reducing the risk of accidental misconfiguration.

* Most distros make it difficult to run anything other than the "stock" kernel. Arch provides linux-hardened as a primary, officially supported package.

* Arch was among the first to move to systemd-resolved, Wayland, and Pipewire, all of which include modern security improvements, like better sandboxing, over their legacy predecessors.
   

</details>

<details>
<summary><strong>Why use Podman versus Docker?</strong></summary>
Docker uses a daemon, whilst Podman is daemonless. Why is daemonless better? For a few reasons.

* **Stability**

The Docker `daemon` is responsible for managing images, containers, networking, and storage. **This daemon runs as a privileged root process**, making it a single point of failure. This means, if the `daemon` crashes, all running containers are affected. Podman's fork-exec model is more stable and secure, as containers can be managed independently. The fork system creates a  process that is a copy of the `parent` process, but it gets its own independent memory space, as in a `child` process. This means that if a `child` process crashes or becomes unstable, it won't directly affect the `parent` process or other unrelated processes. *Stability*.

* **Security**

By creating a new process for each program execution, the fork-exec model limits the potential damage that a compromised program can inflict. Even if a `child` process is exploited, an attacker's access is typically limited to the resources associated with that specific process. That also means the `parent` process can run with higher privileges and perform necessary setup tasks (e.g., opening files, establishing network connections), and then the `child` process can be executed with lower privileges, reducing the potential impact of vulnerabilities in the `child` program. *Security*.

* **Lighter Resources and Integration with Systemd if running Linux**

Because of the fork-exec model, it integrates very nicely into `systemd`. Systemd is the standard system and service manager in most Linux distributions. Podman's integration allows you to manage containers using the same tools, that is `systemctl`.<br>`systemd` provides robust process monitoring, automatic restarts on failure, resource limits, and dependency management. It also enables you to start containers automatically at boot time, ensuring critical services have minimal downtime upon system restart. If you're not using Linux, you're wrong. [spoiler]/s[/spoiler]


Transitioning would be putting you in familiar territory. Podman's CLI (command line interface) is essentially the same as Docker's, so the only meaningful difference is using `podman` as the initial command instead of `docker`. Podman even offers a conversion process you can enable so that `docker` works in place of `podman`. The only downside is [Podman doesn't have great](https://wiki.archlinux.org/title/Podman#Docker_Compose) `compose.yaml` [integration](https://wiki.archlinux.org/title/Podman#Docker_Compose), as there are lots of [translation issues](https://github.com/containers/podman-compose/issues/491). You'd have to use their Quadlets format. It doesn't take too long to figure out, just like learning Docker Compose. Don't let it overwhelm you. You can also use Kubernetes with Podman, but I don't use it and can't tell you much about it.
</details>

<details>
<summary><strong>Why rootless Podman?</strong></summary>
Running containers as root significantly increases the potential damage if a container is compromised. With rootless Podman, even if an attacker gains control of a container, their access is limited to the user's privileges, preventing them from easily escalating to root and taking over the entire system.
</details>



<details>
<summary><strong>Am I safe using Docker or host networking?</strong></summary>
Factually, there are more avenues for hoodlums to take over your system with Docker and/or host networking. It depends on the types of attackers you're trying to prevent. We want it safe enough that random script-kiddies can't do anything, and we don't want malicious actors looking for an easy way in with random probing, but you don't necessarily *need* NSA-tier security using everything here plus SELinux or App Armor.

If you're configuring an average home server, whether it be with Windows, Linux, Raspberry Pi, or other kinds of hardware and software, Docker and host networking is likely to be safe enough. Unless you're doing some dirty things, nobody should be targeting you specifically.
</details>

<details>
<summary><strong>Should I get a domain?</strong></summary>
Who's to say? There are free options like [DuckDNS](https://www.duckdns.org/) that are absolutely fine for most home server applications. If you want to use a shorter `domain` name to access services, getting your own `domain` can be worth it. With your own `domain`, you need to understand basic networking outside of a local network. It's not too demanding, and there are enough guides out there to help you.

Simply for more privacy, custom naming schemes, and me being in control (mostly) of my domain and its `DNS`, I got my own domain. You can get them as cheap as $5 a year, sometimes even less. I literally paid no more than $3 for my first year, and after that it goes up a little. It all depends on the demand for the name, and the `tld` (top-level domain, like `.com`/`.net`/`.org`) is probably the biggest factor in the pricing. There are a few domain providers out there, but I decided to go with [Namecheap](https://www.namecheap.com/).

If you do want a domain, do your own research and don't pick Namecheap just because it's in this guide. There are lots of other options like [Cloudflare](https://domains.cloudflare.com/?index), [GoDaddy](https://www.godaddy.com/), [IONOS](https://www.ionos.com/), and [DreamHost](https://www.dreamhost.com/).
</details>

<details>
<summary><strong>Why not use host networking?</strong></summary>
Because we want security, not ease of setup. A container using `host` networking shares the host's network stack. This means the container has access to all the host's network interfaces and can potentially interact with any service running on the host, regardless of `port mappings` or `firewalls` within the container.  If the container is compromised, the attacker has a much larger attack area and can potentially gain control of the entire `host` system, making the additional security with basic containerization completely useless.<br>

You also lose the ability to easily manage `ports` using Docker or Podman's port mapping (`-p` or `PublishPort=`), so `ports` are managed directly on the host. That can become more complex, especially with multiple containers.
</details>
