# Caddy with Crowdsec

<details>
<summary><strong>Why use systemd socket activation?</strong></summary>

There's a few different reasons. With `socket` activation, in our use case of Caddy, only starts when a connection is actually made to its `socket`. This drastically reduces startup time, especially for applications that are not always needed. Ideally, multiple services can be started in parallel without waiting for each other to fully initialize. `systemd` activates the sockets, and the applications start independently when they receive connections. Unbound and Pihole, however, do not support socket-activation in their containers, while Caddy made it possible as of Q4 2024.

The socket is managed by systemd and allows a service (like a daemon or application) to be started on demand when a connection is made to its associated socket.  It's a way to defer the startup of a service until it's actually needed, leading to improved performance and resource utilization. As only Pihole's admin panel is the only thing Caddy is reverse_proxying, you're unlikely to be using it a lot. Unbound isn't utilized by Caddy at all, since only Pihole is communicating things with it.

Socket activation integrates seamlessly with `systemd`. It manages the sockets and starts the applications as needed, which simplifies service management and provides better control over application lifecycles. `systemd` can also manage dependencies between services and ensure that applications are started in the correct order.

Security is better versus `bridge networks`, because inactive applications are not exposed to the `network`, reducing the attack surface. Only when a connection is made is the application started and exposed. `systemd` can start applications with different user privileges based on the socket configuration, enhancing security.

Socket activation allows multiple applications to share the same port. `systemd` can determine which application should handle the connection based on the incoming request.

__Tl;dr: Systemd manages the 'listening' part of the network. Instead of Caddy sitting in the background constantly holding ports 80 and 443, Systemd holds them. When a packet hits your server, Systemd 'hands' that active connection to Podman. This allows for zero-downtime restarts and better resource management.__

</details>

## Pre-Requisites

* You need a domain. A free one can be acquired by registering through [DuckDNS](https://www.duckdns.org). Because this is your first time running a server, I doubt you want to dedicate money toward a domain right now. Why would you want to buy your own domain?
    * In the case you don't trust DuckDNS, which there is no evidence to not trust them.
    * If you want to use a shorter domain name to access services, getting your own domain can be worth it.
    * Higher security. The DuckDNS API is unregulated, whereas a free DNS service like Cloudflare allows you to limit API permissions with granular control.

Simply for more privacy, custom naming schemes, and me being in control of my domain and its `DNS`, I got my own domain. You can get them as cheap as $5 a year, sometimes even less. I literally paid no more than $3 for my first year, and after that went up a little. It all depends on the demand for the name, and the `tld` (top-level domain, like `.com`/`.net`/`.org`) is probably the biggest factor in the pricing. There are a few domain providers out there, but I decided to go with [Namecheap](https://www.namecheap.com/). Research and compare on your own.

<details>
<summary>Getting a DuckDNS domain</summary>
 
 DuckDNS is a free domain/DNS provider. They're so kind as to allow you to create a domain so others don't have to type in an IP to access your service, and more importantly, allow Caddy to create SSL certificates so you can utilize HTTPS. All for free. And yes, it is a Trust Us Bro™ service. I actually do trust them. Otherwise, you can buy your own domain.

* Go to [DuckDNS](https://www.duckdns.org/) and sign in whatever way you want. On the domains section, type in whatever sub-domain you'd like to use for Caddy to get certificates. It obviously needs to be unique, as many others use DuckDNS. Click add domain. It should automatically detect your public IP and use that for the IP.

Wow. That was easy.

</details>

## Create the Directories

We need the data directory for Caddy. I also want to organize Quadlet service files into folders for organization purposes. You don't have to, but when you have multiple services, especially ones with lots of attached microservices, it's easier to do it this way. We also need to create the directory for the `.socket` in `~/.config/systemd/user`.

* Create the directories:

```
mkdir -p ~/.local/share/containers/storage/caddy && mkdir -p ~/.config/containers/systemd/caddy && mkdir -p ~/.config/systemd/user && mkdir ~/.local/share/containers/storage/caddy/caddy-config && mkdir ~/.local/share/containers/storage/caddy/caddy-data && mkdir ~/.local/share/containers/storage/caddy/log.d && mkdir ~/.config/containers/systemd/crowdsec && mkdir -p ~/.local/share/containers/storage/crowdsec/data
```

## Open the Firewall

Your firewall must allow incoming connects at ports 80 and 443 so that Caddy can do its job. We chose minimal packages, so we're sticking with `nftables`.

* Run:

```
sudo micro /etc/nftables.conf
```

Under `chain input`, add the following:

```
table inet filter {                                                                                                                                         
  chain input {
      ... existing rules ...                                                                                            
    tcp dport { 80, 443 } accept comment "allow caddy http/https"                                                                                           
    udp dport 443 accept comment "allow caddy quic/http3"  
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

### Testing the Firewall

This is optional. After reloading `nftables`, you can verify the rules are active by running `sudo nft list ruleset`. To test if the ports are actually open from the outside, use a tool like `nmap` or a web-based port checker.

## Getting Caddy with Plugins

There are three ways to get a custom version of Caddy. Caddy alone can't do much, it needs plugins to work with DNS providers, utilize DynamicDNS, and work with Crowdsec. I'm listing all of the ways to get Caddy, but **for the rest of the guide, I'm only working with option 1.** I will still briefly list what is needed to have it work in options 2 and 3.

<details>
<summary><strong>1: Create a custom Caddy image with Containerfile</strong></summary>

<br>

* Create a Containerfile/Dockerfile in `~/.local/share/containers/storage/caddy`:

```
micro ~/.local/share/containers/storage/caddy/Containerfile
```

We want Crowdsec for automated security rules. Assuming you're using DuckDNS and you don't have a static IP assigned to your home, we need the following:

```
FROM caddy:2.11.1-builder AS builder

RUN xcaddy build \
    --with github.com/caddy-dns/duckdns \
    --with github.com/mholt/caddy-dynamicdns \
    --with github.com/hslatman/caddy-crowdsec-bouncer

FROM caddy:2.11.1

COPY --from=builder /usr/bin/caddy /usr/bin/caddy
```

> 📝 Note: Here is the list for [all DNS Providers](https://github.com/caddy-dns). Simply replace the URL in the Dockerfile with the provider you use for DNS. Here is the [list of Caddy versions](https://github.com/caddyserver/caddy/releases).

* Create the local image from the Containerfile:

```
podman build -t localhost/caddy-custom:latest ~/.local/share/containers/storage/caddy
```

</details>

<details>
<summary><strong>2: Create a custom Caddy binary with xcaddy</strong></summary>

[xcaddy](https://github.com/caddyserver/xcaddy) builds a `caddy` binary with the necessary plugins (like `DNSDuckDNS`,  `DNS/Cloudfare` or `DNS/Namecheap`). You do need `Go` to do this option.

* Get `go`:

```
sudo pacman -Sy go
```

* Get `xcaddy`:

```
yay xcaddy
```

It'll ask:

```
2 aur/xcaddy-bin 0.4.5-1 (+8 0.38) 
    Build Caddy with plugins
1 aur/xcaddy 0.4.5-1 (+2 0.04) 
    Build Caddy with plugins
==> Packages to install (eg: 1 2 3, 1-3 or ^4)
```

Press `1` and `Enter`, we don't want the `bin` version. Follow the steps.

* After installing, build the binary:

```
xcaddy build --with github.com/caddy-dns/duckdns --with github.com/mholt/caddy-dynamicdns --with github.com/hslatman/caddy-crowdsec-bouncer
```

* Move it:

```
mv caddy ~/.config/containers/storage/caddy/caddy
```

* Make it executable with:

```
chmod +x ~/.config/containers/storage/caddy/caddy
```

</details>

<details>
<summary><strong>3: Create a custom Caddy binary by downloading</strong></summary>

* [Download](https://caddyserver.com/download) the `caddy` binary with the needed plugins.

* Move it to where we need it:

```
mv ~/Downloads/caddy_linux_amd64_custom ~/.config/containers/storage/caddy/caddy
```

* Make it executable with:

```
chmod +x ~/.config/containers/storage/caddy/caddy
```

</details>

## Create the Socket
Thanks to [Erik Sjölund](https://github.com/eriksjolund) for [making this easy](https://github.com/eriksjolund/podman-caddy-socket-activation/tree/main) to be done. Create a `caddy.socket` file in `~/.config/systemd/user` with:

```
micro ~/.config/systemd/user/caddy.socket
```
and input:

```
[Socket]
BindIPv6Only=both

### sockets for the HTTP reverse proxy
# fd/3
ListenStream=[::]:80

# fd/4
ListenStream=[::]:443

# fdgram/5
ListenDatagram=[::]:443

### socket for the admin API endpoint
# fd/6
ListenStream=%t/caddy.sock
SocketMode=0600

[Install]
WantedBy=sockets.target
```
> Note: For an explanation of systemd specifier "%t", see https://www.freedesktop.org/software/systemd/man/latest/systemd.unit.html

## Create the `caddy.container` File

* Create Caddy's [`.container`](./caddy.container) file with
```
micro ~/.config/containers/systemd/caddy/caddy.container
```
and then put in the following:

```
[Unit]
AssertPathExists=%h/.local/share/containers/storage/caddy/Caddyfile

[Container]
ContainerName=caddy
#If using a binary, uncomment the normal Caddy image and remove the custom image.
#Image=caddy:latest
Image=localhost/caddy-custom
Exec=/usr/bin/caddy run --config /etc/caddy/Caddyfile
# Add your email below
Environment=EMAIL=
Environment=LOG_FILE=/data/access.log
Secret=DUCKDNS_API_TOKEN,type=env,target=DUCKDNS_API_TOKEN
Secret=CROWDSEC_API_KEY,type=env,target=CROWDSEC_API_KEY

# If you ran xcaddy or downloaded it, uncomment the volume mount for the caddy binary.
# Otherwise, delete the line below and these comments.
#Volume=%h/.local/share/containers/storage/caddy/caddy:/usr/bin/caddy
Volume=%h/.local/share/containers/storage/caddy/Caddyfile:/etc/caddy/Caddyfile
Volume=%h/.local/share/containers/storage/caddy/caddy-config:/config
Volume=%h/.local/share/containers/storage/caddy/caddy-data:/data
Volume=%h/.local/share/containers/storage/caddy/log.d:/data/log.d
Notify=true
Memory=256m

Network=crowdsec.network
AddHost=crowdsec:172.17.1.4

HealthCmd=caddy validate --config /etc/caddy/Caddyfile || exit 1
HealthInterval=30s
HealthRetries=3

[Install]
WantedBy=default.target

[Service]
ExecReload=/usr/bin/podman exec caddy /usr/bin/caddy reload --config /etc/caddy/Caddyfile --force
```


<details>
<summary>A few things to cover if you want to know. If not, continue on.</summary>

* A hash (`#`) is used to make a line a comment, so [Podman's generator](https://docs.podman.io/en/latest/markdown/podman-systemd.unit.5.html) does not read the values and incorporate it into the .service file.

* Ending the file in the `~/.config/containers/systemd` directory with `.container`, `.network`, `.pod`, `.volume`, and `.kube` allow Podman Quadlets to translate the contents into a `systemd` `.service` file

* The [IANA](https://en.wikipedia.org/wiki/Internet_Assigned_Numbers_Authority)-reserved ranges for private networks are:
    * 10.0.0.0/8: (10.0.0.0 to 10.255.255.255)
    * 172.16.0.0/12: (172.16.0.0 to 172.31.255.255)
    * 192.168.0.0/16: (192.168.0.0 to 192.168.255.255)
Any of these work as long as it's consistent in your network configuration, and we WILL be using them.

* `.tld` is top level domain. (.com,.net,.org)<br>

* `~/` simply references to the home directory of the user.<br>

* `%h` also refers to the home directory of the user executing `systemd --user start caddy.service`

* The `Network=` directive points to a file with the name in the same `~/.config/containers/systemd` directory.

* `Notify=true` provides a way for services to communicate their state changes (starting, running, stopping, etc.) back to systemd.  This is done through a Unix socket, often referred to as the "notify socket."  When `Notify=true` is set, the service is expected to use this socket to send notifications to systemd. This is necessary with our socket-activated socket being passed into the container.

* `[Install]` contains installation directives that tells systemd how to enable and disable the generated caddy.service service.

* The `WantedBy=` directive specifies one or more targets that the service should be "pulled in" or "wanted by" when those targets are activated.

* `default.target` is a special target in systemd.  It represents the default multi-user system state.  In simpler terms, it's what systemd aims to achieve when the system boots into normal operating mode (not single-user mode or other special modes).

* `[Service]` contains directives that control the behavior of the service itself, such as how it's started, stopped, and restarted.

</details>

## Create the Caddyfile

* Create your [`Caddyfile`](./Caddyfile) with:
```
micro ~/.config/containers/storage/caddy/Caddyfile
```
and input the following, replacing `yoursubdomain` with your created DuckDNS subdomain:

```
{
        crowdsec {
                api_url http://crowdsec:8080
                api_key {env.CROWDSEC_API_KEY}
        }
        acme_dns duckdns {env.DUCKDNS_API_TOKEN}
        dynamic_dns {
                provider duckdns {env.DUCKDNS_API_TOKEN}
                domains {
                        duckdns.org yoursubdomain
                }
                versions ipv4
        }
}

yoursubdomain.duckdns.org {
        crowdsec
        bind fd/3 {
                protocols h1
        }
        bind fd/4 {
                protocols h1 h2
        }
        bind fdgram/5 {
                protocols h3
        }
}
```

<details>
<summary>More details for this section. If you don't care, continue on.</summary>

* There are a LOT of ways to configure a `Caddyfile`. [Read here](https://caddyserver.com/docs/caddyfile) if you decide you'd like to make any revisions or additions.

* `yoursubdomain.duckdns.org` is the `site block` in the Caddyfile.

* The `bind` [directive](https://caddyserver.com/docs/caddyfile/directives/bind) overrides the interface to which the server's socket should bind. Normally, the listener binds to the empty (wildcard) interface. However, you may force the listener to bind to another hostname or IP instead. This directive accepts only a host, not a port. The port is determined by the site address (defaulting to 443).

* `fd/#` refers to file descriptor. Those are passed from the service manager (like `systemd`) to the Caddy server, allowing it to accept connections without binding to specific addresses. Caddy recognizes these file descriptors through `environment variables` provided by `systemd`, which indicate the number and names of the sockets it should use.<br>
So file descriptors like `fd/3`, `fd/4` and `fd/5` are typically assigned to standard ports such as :80 (`HTTP`) and :443 (`HTTPS`). `systemd` creates these file descriptors and passes them to the Caddy server when it starts. Caddy then uses these file descriptors to listen for incoming connections on the specified ports without needing to bind to them directly. This allows for more efficient management of services and resources, for example, socket activation.<br>
The exact meaning of `fd/6` can vary depending on the configuration of the service and the number of sockets that have been set up. In the context of Caddy, `fd/6` can be used as a file descriptor for the `admin` API when socket activation is enabled. The `admin` API allows for dynamic configuration and management of the Caddy server while it is running.

* The protocols `h1`, `h2`, and `h3` refer to the different versions of the HTTP protocol that the server can support:
    * `h1`: This stands for HTTP/1.1, which is the traditional version of the HTTP protocol. It introduced persistent connections and chunked transfer encoding, among other features.<br>
    * `h2`: This refers to HTTP/2, which is a major revision of the HTTP protocol. It improves performance by allowing multiple streams of data to be sent over a single connection, reducing latency and improving loading times. It also includes features like header compression and prioritization of requests.<br>
    * `h3`: This denotes HTTP/3, which is the latest version of the HTTP protocol. It is built on top of QUIC (Quick UDP Internet Connections), a transport layer network protocol that aims to reduce latency and improve security. HTTP/3 further enhances performance, especially in scenarios with high packet loss or latency.

</details>

## Create Caddy's Podman Secret

You probably noticed the previous mentions of `Secret`. That's because we don't want anyone seeing the values of our sensitive information, just out there in the open like that.

* Create the secret. Edit with your API token and run:

```
DUCKDNS_API_TOKEN="put-your-token-string-here" podman secret create --env=true DUCKDNS_API_TOKEN DUCKDNS_API_TOKEN
```

## Configure Crowdsec

* Create Crowdsec's [`.container`](./crowdsec.container) file:

```
micro ~/.config/containers/systemd/crowdsec/crowdsec.container
```

Input:

```
[Container]
ContainerName=crowdsec
Image=crowdsecurity/crowdsec:latest
Memory=256m
Environment=COLLECTIONS=crowdsecurity/caddy crowdsecurity/http-cve crowdsecurity/whitelist-good-actors
#Secret=BOUNCER_KEY_CADDY,target=BOUNCER_KEY_CADDY
Network=crowdsec.network
IP=172.17.1.4
Volume=%h/.local/share/containers/storage/crowdsec:/etc/crowdsec
Volume=%h/.local/share/containers/storage/crowdsec/data:/var/lib/crowdsec/data/
Volume=%h/.local/share/containers/storage/caddy/log.d:/var/log:ro


[Install]
WantedBy=default.target

[Service]
Restart=always
```

* The `Restart=always` value means that systemd should restart the service regardless of why it stopped.  This includes normal exits, crashes, kill signals, and timeouts.

* Create Crowdsec's [`.network`](./crowdsec.network) file:

```
micro ~/.config/containers/systemd/crowdsec/crowdsec.network
```

Input:

```
[Network]
Subnet=172.17.1.0/24
```

* Reload the generator:

```
systemctl --user daemon-reload
```

* Create the network:

```
systemctl --user start systemd-crowdsec-network
```

* Run the container:

```
systemctl --user start crowdsec
```

* Add Caddy Bouncer:

```
podman exec -it crowdsec cscli bouncers add caddy-bouncer
```

Copy the output.

* Create the secrets:

```
BOUNCER_KEY_CADDY="your-key" podman secret create --env=true BOUNCER_KEY_CADDY BOUNCER_KEY_CADDY && CROWDSEC_API_KEY="your-key" podman secret create --env=true CROWDSEC_API_KEY CROWDSEC_API_KEY
```

* The output will need to be added as a Podman secret. Stop Crowdsec:

```
systemctl --user stop crowdsec
```

Uncomment the `Secret=` key in `crowdsec.container`.

* Restart Crowdsec and enable Caddy's socket to start Caddy:

```
systemctl --user daemon-reload && systemctl --user start crowdsec && systemctl --enable caddy.socket
```

## Next Step

* [🔰 DNS Setup for Privacy and Security](https://github.com/TheRettom/self-hosted-guide/DNS/README.md) - Pi-hole and Unbound
