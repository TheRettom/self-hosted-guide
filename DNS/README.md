# Private and Secure DNS Configuration

> Note: I will be linking the configuration files throughout the guide. As some of them are huge, the contents of some files will not be in the body of the guide itself. Refer to the linked files.

## Create Directories

* Create the necessary directories:

```
mkdir -p ~/.config/containers/storage/pihole/etc-pihole && mkdir ~/.config/containers/storage/pihole/logs && mkdir -p ~/.config/containers/storage/unbound && mkdir -p ~/.local/share/containers/storage/unbound/conf.d && mkdir ~/.local/share/containers/storage/unbound/log.d && mkdir ~/.local/share/containers/storage/unbound/certs.d && mkdir ~/.local/share/containers/storage/unbound/zones.d
```
## Disable Copy-on-Write for Specific Directory

* Pi-hole’s FTL (Faster Than Light) engine writes to a database for every single DNS request. On Btrfs, this leads to extreme fragmentation of pihole-FTL.db. Ensure your etc-pihole volume is set to NoCoW (+C). This keeps the dashboard responsive even after months of logging millions of queries.

Run:

```
sudo chattr +C ~/.local/share/containers/storage/pihole/etc-pihole
```

## Pull Images

* Pull the images with:
```
podman pull pihole/pihole:latest madnuttah/unbound:latest busybox redis:alpine
```

## Create Podman Network

* Create the `dns.network` file with:
```
micro ~/.config/containers/systemd/dns.network
```
and input:
```
[Network]
NetworkName=dns
Subnet=172.17.0.0/24
```

## Deconflict Port 53

* We need to stop `systemd-resolved` from listening on port 53 so Pihole can utilize it.

Create the directory first:

```
mkdir -p /etc/systemd/resolved.conf.d
```

* Create the config:

```
sudo micro /etc/systemd/resolved.conf.d/80-stublistener.conf
```

Add:

```
[Resolve]
DNSStubListener=no
```

* Restart the service:

```
sudo systemctl restart systemd-resolved
```

## Open the Port

* Add the following rules to your `input` chain:

```
sudo micro /etc/nftables.conf
```

Adjust the subnet as necessary. We only want the local network to use this DNS:

```
table inet filter {                                                                                                                                         
  chain input { 
      ... existing rules ...
      ip saddr 192.168.1.0/24 udp dport 53 accept
      ip saddr 192.168.1.0/24 tcp dport 53 accept
      ... existing rules ...
  }
}
```

> Note: If for whatever reason you don't want only the local network to use the DNS, remove the subnet range portion `ip saddr 192.168.1.0/24`

**Before we reload the firewall configuration:** If there is a typo in `/etc/nftables.conf`, the reload might fail, or worse, leave the firewall in an inconsistent state. Run the following:

```
sudo nft -c -f /etc/nftables.conf
```

`-c` means check and will report errors without applying the changes. If all is good, reload `nftables`:

```
sudo systemctl reload nftables
```

## Port Forwarding

I am not covering how to do this, as it is very specific to each router. Pihole Documentation already has some routers listed under [`Router setup`](https://docs.pi-hole.net/routers), so start there. All you need to do is forward all local traffic to the homeserver IP under port 53 TCP and UDP.

## Configuring All Devices to Use Pihole as DNS

Same thing, this is determined based on your router. Pihole Documentation already has some routers listed under [`Router setup`](https://docs.pi-hole.net/routers), so start there.

However, there are three methods to do so, [as stated on Pihole's Discourse](https://discourse.pi-hole.net/t/how-do-i-configure-my-devices-to-use-pi-hole-as-their-dns-server/245). Follow that guide based on your available options.

---

# Pihole

Let's start with making the directory.

* Run:

```
mkdir ~/.config/containers/storage/pihole/etc-pihole && mkdir ~/.config/containers/storage/pihole/etc-dnsmasq.d && mkdir ~/.config/containers/storage/pihole/logs
```

## Create Pihole's Secret

We want to make a password that will be hidden with [Podman Secrets](https://docs.podman.io/en/latest/markdown/podman-secret.1.html) for Pihole's login.

* Use this command to create the `environmental variable` that will be utilized by Podman Secrets:

```
FTLCONF_webserver_api_password="password" podman secret create --env=true FTLCONF_webserver_api_password FTLCONF_webserver_api_password
```

## Pihole's Container

* Now let's create [Pihole's `.container` file](./pihole.container):

```
micro ~/.config/containers/systemd/pihole/pihole.container
```

Input the following:

```
[Container]
ContainerName=pihole
Image=pihole/pihole:latest
Environment=FTLCONF_dns_listeningMode=all
Environment=FTLCONF_dns_upstreams="172.17.0.20#5335"
# Sometimes DNSMASQ_USER needs to be root. Try running it without the variable if you can.
Environment=DNSMASQ_USER=root
# Sometimes the log file location needs to be explicitly mentioned in rootless Podman. Run without if you can.
Environment=FTLCONF_files_log_ftl=/var/log/pihole/FTL.log
Secret=FTLCONF_webserver_api_password,type=env,target=FTLCONF_webserver_api_password

PublishPort=53:53/tcp
PublishPort=53:53/udp
Network=dns.network
IP=172.17.0.5
AddHost=unbound:172.17.0.10

Volume=%h/.local/share/containers/storage/pihole/etc-pihole:/etc/pihole:Z
Volume=%h/.local/share/containers/storage/pihole/logs:/var/log:Z
# The next two volume mounts are for timezone accuracy
Volume=/etc/localtime:/etc/localtime:ro
Volume=/etc/timezone:/etc/timezone:ro

Memory=256m
HealthCmd=dig +short +norecurse +retry=0 @127.0.0.1 pi.hole || exit 1
HealthInterval=60s
HealthStartPeriod=30s

[Install]
WantedBy=default.target

[Service]
Restart=always
```

`IP=` is setting a static IP for the container.

---

# Unbound

We've got a few things to do, especially since we're running Rootless Podman instead of Docker. [I had to work with `madnuttah`](https://github.com/madnuttah/unbound-docker/issues/92) on why I couldn't get DNSSEC to work, and after a fair bit of troubleshooting, I was able to get it to work with specific changes. Unfortunately that means more steps, but it is more secure.

Since there are many containers for the overall service, we will create a Pod.

<details>
<summary>What is a Pod?</summary>

In Podman, a Pod is the smallest unit of management, borrowed from the Kubernetes world. Think of it as a "secure wrapper" around a group of containers that need to act like a single machine.

In a standard container setup, each container has its own network namespace. To talk to each other, they have to have a bridge network. This is fine in a few scenarios, and we're going to be utilizing both a Pod network and a bridge network for Unbound. But in a Pod, containers are in the same network namespace.

* They share:
    * The same IP address: To the rest of your network, the Pod is one entity.
    * The Loopback interface: They can talk to each other over localhost.
    * IPC (Inter-Process Communication): They can share Unix sockets in memory.

We need to use Redis as a cache for Unbound (via `unbound-db.container`). Using a Pod here isn't just "neat" — it's a security and performance optimization. Unbound needs to talk to Redis fast. Without a Pod, Unbound would connect to Redis over a container-only network port (e.g., 127.0.0.1:6379). This involves the overhead of the network stack. With a Pod, we can use a Unix Domain Socket (`unbound-redis-socket.container`). Because they share the same namespaces and volume (`cachedb.d.volume`), Unbound can talk to Redis through a file in memory. This is significantly faster and lower latency than network-based communication.

* I mentioned we're going to use a Pod network and a bridge network. Why?
    * The Redis container doesn't need to be visible to Caddy or Pi-hole. It only needs to be visible to Unbound.
    * By putting them in a Pod on a private internal network, we air-gap the database. Even if someone compromised the Caddy container, they couldn't even see the Redis instance because it doesn't exist on the `dns.network`.

</details>

## Unbound's Server Container

* Create [Unbound's `.container` file](./unbound-server.container):

```
micro ~/.config/containers/systemd/unbound/unbound-server.container
```

Input the following:

```
[Container]
ContainerName=unbound-server
Image=docker.io/madnuttah/unbound:latest

Network=dns.network
AddHost=pihole:172.17.0.5
IP=172.17.0.20
Pod=unbound.pod

Volume=%h/.local/share/containers/storage/unbound/log.d/unbound.log:/usr/local/unbound/log.d/unbound.log:rw
Volume=%h/.local/share/containers/storage/unbound/conf.d:/usr/local/unbound/conf.d
Volume=%h/.local/share/containers/storage/unbound/unbound.conf:/usr/local/unbound/unbound.conf:ro
Volume=%h/.local/share/containers/storage/unbound/certs.d:/usr/local/unbound/certs.d
Volume=%h/.local/share/containers/storage/unbound/zones.d/:/usr/local/unbound/zones.d/:rw
Volume=%h/.local/share/containers/storage/unbound/root.hints:/usr/local/unbound/iana.d/root.hints:rw
Volume=%h/.local/share/containers/storage/unbound/root.zone:/usr/local/unbound/iana.d/root.zone
Environment=TZ=America/Boise
Environment=DISABLE_SET_PERMS=true

Memory=256m
HealthCmd=["/usr/local/unbound/sbin/healthcheck.sh", "unbound-healthcheck"]
HealthStartPeriod=15s
HealthInterval=60s
HealthTimeout=30s
HealthRetries=5
Notify=healthy

[Install]
WantedBy=default.target

[Service]
Restart=always
```

> 📝 Note: The `IP=` key can only be applied to a single network in a Quadlet file. So only one specified `Network=` key can be used with `IP=`, or it won't work. In further configuration of Caddy when we add services, we'll see how we get around this.

* Create the [`unbound.conf`](./unbound.conf):

```
micro ~/.local/share/containers/storage/unbound/unbound.conf
```

Input the contents:

```
# Use this anywhere in the file to include other text into this file.
#include: "otherfile.conf"

include: "/usr/local/unbound/conf.d/*.conf"
include: "/usr/local/unbound/zones.d/*.conf"

# Use this anywhere in the file to include other text, that explicitly starts a
# clause, into this file. Text after this directive needs to start a clause.
#include-toplevel: "otherfile.conf"

# The server clause sets the main parameters.
server:
	username: ""
	directory: "/usr/local/unbound"
	chroot: ""
	do-daemonize: no
	tls-cert-bundle: /etc/ssl/certs/ca-certificates.crt
	module-config: "validator cachedb iterator"
```

* Copy the following files and input all of them in `~/.local/share/containers/storage/unbound/conf.d/`:
    * [`trust-anchor.conf`](./conf.d/trust-anchor.conf)
    * [`security.conf`](./conf.d/security.conf)
    * [`remote-control.conf`](./conf.d/remote-control.conf)
    * [`performance.conf`](./conf.d/performance.conf)
    * [`logging.conf`](./conf.d/logging.conf)
    * [`interfaces.conf`](./conf.d/interfaces.conf)
    * [`cachedb.conf`](./conf.d/cachedb.conf)
    * [`access-control.conf`](./conf.d/access-control.conf)

* We need a `root.hints` file and a `root.zone` file or DNSSEC will not work.

```
micro ~/.local/share/containers/storage/unbound/root.hints
```

Input [from here](https://www.internic.net/domain/named.root).


```
micro ~/.local/share/containers/storage/unbound/root.zone
```

Input [from here](https://www.internic.net/domain/root.zone).

> Note: I pulled the information for those files from the IANA. It should not need to be updated regularly, but just in case, [here is the link](https://www.iana.org/domains/root/files).

## Unbound's Redis Socket Container

* Create [Unbound's `redis-socket.container` file](./unbound-redis-socket.container):

```
micro ~/.config/containers/systemd/unbound/unbound-redis-socket.container
```

Input the following:

```
[Container]
Image=busybox
ContainerName=unbound-redis-socket
Pod=unbound.pod
RunInit=true
Exec=/bin/sh -c "rm -f /usr/local/unbound/cachedb.d/redis.sock && chmod 777 /usr/local/unbound/cachedb.d/ && chown -R 999:1000 /usr/local/unbound/cachedb.d/"
Memory=256m

[Install]
WantedBy=default.target

[Service]
Restart=always
```

## Unbound's Database

* * Create [Unbound's database `.container` file](./unbound-db.container):

```
micro ~/.config/containers/systemd/unbound/unbound-db.container
```

Input the following: 

```
[Container]
ContainerName=unbound-db
Image=redis:alpine
Exec=redis-server /usr/local/etc/redis/redis.conf
Pod=unbound.pod
User=1000
Group=1000
Volume=%h/.local/share/containers/storage/unbound/conf.d/redis.conf:/usr/local/etc/redis/redis.conf
Volume=%h/.local/share/containers/storage/unbound/healthcheck.sh:/usr/local/sbin/healthcheck.sh:ro
Memory=512m

HealthCmd=/usr/local/sbin/healthcheck.sh
HealthInterval=10s
HealthRetries=5
HealthStartPeriod=5s
HealthTimeout=30s

[Install]
WantedBy=default.target

[Service]
Restart=always
```

* Create the [healthcheck script](./healthcheck.sh):

```
micro ~/.local/share/containers/storage/unbound/healthcheck.sh
```

Input the following:

```
#!/bin/sh

SOCKET=/usr/local/unbound/cachedb.d/redis.sock
if [[ ! -S "$SOCKET" ]]; then
  echo "⚠ Unix Socket not found"
  exit 1    
else 
  echo "✅ Unix Socket found"
  exit 0
fi
```

Make it executable:

```
chmod +x ~/.local/share/containers/storage/unbound/healthcheck.sh
```

* Create the [`redis.conf`](./conf.d/redis.conf) and input the contents:

```
micro ~/.local/share/containers/storage/unbound/conf.d/redis.conf
```

## Unbound's Internal Network

* Create [Unbound's internal `.network` file](./unbound.network):

```
micro ~/.config/containers/systemd/unbound/unbound.network
```

Input the following:

```
[Network]
Internal=true
```

* Start it:

```
systemctl --user start unbound-network.service
```

## Unbound's Database Cache

* Create [Unbound's database cache `.volume` file](./cachedb.d.volume):

```
micro ~/.config/containers/systemd/unbound/cachedb.d.volume
```

Input the following:

```
[Volume]
VolumeName=cachedb.d
```

## Unbound's Pod

* Create [Unbound's `.pod` file](./unbound.pod):

```
micro ~/.config/containers/systemd/unbound/unbound.pod
```

```
[Pod]
PodName=unbound-pod
Network=unbound.network
Volume=cachedb.d:/usr/local/unbound/cachedb.d/
```

## Covering a Potential Fail

Sometimes Podman wants network files in the same subfolder of the `~/.config/containers/systemd` directory. Create a symlink in the `systemd/unbound` directory.

```
ln -s ~/.config/containers/systemd/dns.network ~/.config/containers/systemd/unbound/dns.network
```

## Run It!

* Refresh the daemon again:

```
systemctl --user daemon-reload
```

* Run the services:

```
systemctl --user start unbound-pod
systemctl --user start pihole
```

* For evidence: See the DNS query cache lowering the response time:

```
podman exec -it unbound-server drill -p 5335 archlinux.org @127.0.0.1 | grep "Query time"
```

First time it will be up to a few hundred milliseconds. Second you run the command, 0 milliseconds or close to it.

---

# Next Steps

* [🏠 VPN for Home Network Access While Away](https://github.com/TheRettom/self-hosted-guide/tree/main/WireGuard) - WireGuard
* [🔒 Password Manager](https://github.com/TheRettom/self-hosted-guide/tree/main/Vaultwarden) - Vaultwarden
* [👥 Powerful Photo Storage](https://github.com/TheRettom/self-hosted-guide/tree/main/Immich) - Immich
