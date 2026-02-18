# Rootless Podman on Arch Linux with Services
## Preface
**THIS IS FOR ARCH LINUX AND ROOTLESS PODMAN SETUPS.** I wanted to make things easier for people who want to self-host services if they manage to find my GitHub. Why Rootless Podman and Arch Linux? Security and integration. **DO YOU NEED A HIGH-SECURITY HOSTING PLATFORM?** If you're self-hosting services for you and your family, and maybe some friends — probably not. But for those who are paranoid as a standard, or want maximum privacy and security now rather than later, this is for you.

## 📖 Documentation Tree
* [🌐 Overview](#architecture-overview)
* [💻 Host OS Setup](./docs/Installing Arch Linux/arch-setup.md) - Arch Linux + `linux-hardened`
* [📦 Container Runtime](./docs/Configuring Rootless Podman/rootless-podman.md) - Rootless Podman & Sysctl
* [🛠 Service Configuration](./docs/Pihole and Unbound/pihole-unbound.md) - Pi-hole, Unbound, & Caddy
* [🛡 Security Roadmap](#security-roadmap)

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
