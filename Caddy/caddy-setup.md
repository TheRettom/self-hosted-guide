# Caddy Configuration

<details>
<summary><strong>Why use a socket-activated socket?</strong></summary>

There's a few different reasons. With `socket` activation, in our use case of Caddy, only starts when a connection is actually made to its `socket`. This drastically reduces startup time, especially for applications that are not always needed. Ideally, multiple services can be started in parallel without waiting for each other to fully initialize. `systemd` activates the sockets, and the applications start independently when they receive connections. Unbound and Pihole, however, do not support socket-activation in their containers, while Caddy made it possible as of Q4 2024.<br>

The term socket-activated socket sounds a bit weird, eh? In our context, it is managed by systemd and allows a service (like a daemon or application) to be started on demand when a connection is made to its associated socket.  It's a way to defer the startup of a service until it's actually needed, leading to improved performance and resource utilization. As only Pihole's admin panel is the only thing Caddy is reverse_proxying, you're unlikely to be using it a lot. Unbound isn't utilized by Caddy at all, since only Pihole is communicating things with it.<br>

Socket activation integrates seamlessly with `systemd`. It manages the sockets and starts the applications as needed, which simplifies service management and provides better control over application lifecycles. `systemd` can also manage dependencies between services and ensure that applications are started in the correct order.<br>

Security is better versus `bridge networks`, because inactive applications are not exposed to the `network`, reducing the attack surface. Only when a connection is made is the application started and exposed. `systemd` can start applications with different user privileges based on the socket configuration, enhancing security.<br>

Socket activation allows multiple applications to share the same port. `systemd` can determine which application should handle the connection based on the incoming request.
</details>

## Pre-Requisites

* You need a domain. A free one can be acquired by registering through [DuckDNS](https://www.duckdns.org). Because this is your first time running a server, I doubt you want to dedicate money toward a domain right now. Why would you want to buy your own domain?
    * In the case you don't trust DuckDNS, which there is no evidence to not trust them.
    * If you want to use a shorter domain name to access services, getting your own domain can be worth it.
    * Higher security. The DuckDNS API is unregulated, whereas a free DNS service like Cloudflare allows you to limit API permissions with granular control.
Simply for more privacy, custom naming schemes, and me being in control of my domain and its `DNS`, I got my own domain. You can get them as cheap as $5 a year, sometimes even less. I literally paid no more than $3 for my first year, and after that went up a little. It all depends on the demand for the name, and the `tld` (top-level domain, like `.com`/`.net`/`.org`) is probably the biggest factor in the pricing. There are a few domain providers out there, but I decided to go with [Namecheap](https://www.namecheap.com/). Research and compare on your own.

## Create the Directories

We need the data directory for Caddy. I also want to organize Quadlet service files into folders for organization purposes. You don't have to, but when you have multiple services, especially ones with lots of attached microservices, it's easier to do it this way.

* Create the directories:

```
mkdir -p ~/.local/share/containers/storage/caddy
mkdir -p ~/.config/containers/systemd/caddy
```

Your firewall must allow incoming connects at ports 80 and 443 so that Caddy can do its job. We want minimal packages, so we're sticking with `nftables`. 
`iptables` has been replaced with [nftables](https://wiki.archlinux.org/title/Nftables), but the commands for iptables still work. I'm going to utilize `nftables` commands here.<br>
I highly recommend you read up on the service you're using before you blindly enter commands.
<br>

```
sudo nft add rule ip filter input tcp dport 80 accept && sudo nft add rule ip filter input tcp dport 443 accept && sudo nft add rule ip filter input udp dport 443 accept && sudo nft add rule ip filter input tcp dport 53 accept && sudo nft add rule ip filter input udp dport 53 accept
```

If you have not yet worked with Caddy, then you need to do a few things. The default location for Podman's image data is in `~/.config/containers/storage`. To keep things organized, I like to create subfolders for each service. Start with:

```
mkdir -p ~/.config/containers/storage/caddy
```

We also need to create the directory for where our socket is going.
```
mkdir -p ~/.config/systemd/user
```
</details>




There are three ways to get a custom version of Caddy. Caddy alone can't do much, it needs plugins to work with DNS providers, utilize DynamicDNS, and work with Crowdsec. I'm listing all of them, but **for the rest of the guide, I'm only working with option 1.** I will still briefly list what is needed to have it work in options 2 and 3.

1: Create a custom Caddy image with Dockerfile.

2: You can use [xcaddy](https://github.com/caddyserver/xcaddy) to build a `caddy` binary with the necessary plugins (like `DNSDuckDNS`,  `DNS/Cloudfare` or `DNS/Namecheap`), and then you'll mount the binary in our `caddy.container` file.

Make it executable with:

```
chmod +x ~/.config/containers/storage/caddy/caddy
```

3: [Download](https://caddyserver.com/download) the `caddy` binary with the needed plugins, and then you'll mount the binary in our `caddy.container` file. The binary likely downloaded to your `Downloads` directory. Use this to rename it and move it to where we need it:

```
mv ~/Downloads/caddy_linux_amd64_custom ~/.config/containers/storage/caddy/caddy
```

Make it executable with:

```
chmod +x ~/.config/containers/storage/caddy/caddy
```




# Create the Socket
Thanks to [Erik Sjölund](https://github.com/eriksjolund) for [making this easy](https://github.com/eriksjolund/podman-caddy-socket-activation/tree/main) to be done. Create a `caddy.socket` file in `~/.config/systemd/user` with:

```
nano ~/.config/systemd/user/caddy.socket
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
For an explanation of systemd specifier "%t", see https://www.freedesktop.org/software/systemd/man/latest/systemd.unit.html



<details>
<summary>A few things to cover if you want to know. If not, continue on.</summary>

* A hash (`#`) is used to make a line a comment, so [Podman's generator](https://docs.podman.io/en/latest/markdown/podman-systemd.unit.5.html) does not read the values and incorporate it into the .service file.<br>

* Ending the file in the `~/.config/containers/systemd` directory with `.container`, `.network`, `.pod`, `.volume`, and `.kube` allow Podman Quadlets to translate the contents into a `systemd` `.service` file

* The [IANA](https://en.wikipedia.org/wiki/Internet_Assigned_Numbers_Authority)-reserved ranges for private networks are:
*10.0.0.0/8: (10.0.0.0 to 10.255.255.255)*
*172.16.0.0/12: (172.16.0.0 to 172.31.255.255)*
*192.168.0.0/16: (192.168.0.0 to 192.168.255.255)*
Any of these work as long as it's consistent in your network configuration.<br>

* `.tld` is top level domain. (.com,.net,.org)<br>

* `~/` simply references to the home directory of the user.<br>

* `%h` also refers to the home directory of the user executing:
```
systemd --user start caddy.service
```
<br>

* The `Network=` directive points to a file with the name in the same `~/.config/containers/systemd` directory.<br>

* `Notify=true` provides a way for services to communicate their state changes (starting, running, stopping, etc.) back to systemd.  This is done through a Unix socket, often referred to as the "notify socket."  When `Notify=true` is set, the service is expected to use this socket to send notifications to systemd. This is necessary with our socket-activated socket being passed into the container.<br>

* `[Install]` contains installation directives that tells systemd how to enable and disable the generated caddy.service service.<br>

* The `WantedBy=` directive specifies one or more targets that the service should be "pulled in" or "wanted by" when those targets are activated.<br>

* `default.target` is a special target in systemd.  It represents the default multi-user system state.  In simpler terms, it's what systemd aims to achieve when the system boots into normal operating mode (not single-user mode or other special modes).<br>

* `[Service]` contains directives that control the behavior of the service itself, such as how it's started, stopped, and restarted.<br>

* The `Restart=` directive specifies under what conditions systemd should automatically restart the service.<br>

* The `always` value means that systemd should restart the service regardless of why it stopped.  This includes normal exits, crashes, kill signals, and timeouts.
</details>
<hr>




# Create the Caddyfile
Create a `Caddyfile`

<details>
<summary>More details for this section. If you don't care, continue on.</summary>
<br>

* Remember that I included the [DNS/Namecheap plugin](https://github.com/caddy-dns/namecheap) with my Caddy build. If you're using another provider, such as [Cloudflare](https://github.com/caddy-dns/cloudflare), use those parameters.
<hr>

* There are a LOT of ways to configure a `Caddyfile`. [Read here](https://caddyserver.com/docs/caddyfile) if you decide you'd like to make any revisions or additions.
<hr>

* `domain.tld` (or `localhost` if used) is the `site block` in the Caddyfile.
<hr>

* The `bind` [directive](https://caddyserver.com/docs/caddyfile/directives/bind) overrides the interface to which the server's socket should bind. Normally, the listener binds to the empty (wildcard) interface. However, you may force the listener to bind to another hostname or IP instead. This directive accepts only a host, not a port. The port is determined by the site address (defaulting to 443).
<hr>

* `fd/#` refers to file descriptor. Those are passed from the service manager (like `systemd`) to the Caddy server, allowing it to accept connections without binding to specific addresses. Caddy recognizes these file descriptors through `environment variables` provided by `systemd`, which indicate the number and names of the sockets it should use.<br>
So file descriptors like `fd/3`, `fd/4` and `fd/5` are typically assigned to standard ports such as :80 (`HTTP`) and :443 (`HTTPS`). `systemd` creates these file descriptors and passes them to the Caddy server when it starts. Caddy then uses these file descriptors to listen for incoming connections on the specified ports without needing to bind to them directly. This allows for more efficient management of services and resources, for example, socket activation.<br>
The exact meaning of `fd/6` can vary depending on the configuration of the service and the number of sockets that have been set up. In the context of Caddy, `fd/6` can be used as a file descriptor for the `admin` API when socket activation is enabled. The `admin` API allows for dynamic configuration and management of the Caddy server while it is running.
<hr>

* The protocols `h1`, `h2`, and `h3` refer to the different versions of the HTTP protocol that the server can support:<br>

    `h1`: This stands for HTTP/1.1, which is the traditional version of the HTTP protocol. It introduced persistent connections and chunked transfer encoding, among other features.<br>

    `h2`: This refers to HTTP/2, which is a major revision of the HTTP protocol. It improves performance by allowing multiple streams of data to be sent over a single connection, reducing latency and improving loading times. It also includes features like header compression and prioritization of requests.<br>

    `h3`: This denotes HTTP/3, which is the latest version of the HTTP protocol. It is built on top of QUIC (Quick UDP Internet Connections), a transport layer network protocol that aims to reduce latency and improve security. HTTP/3 further enhances performance, especially in scenarios with high packet loss or latency.

* Ideally, `header_up X-Forwarded-For {http.request.header.X-Real-IP}` would provide the source IP, thus allowing proper hostname resolution for clients. This does not work in this configuration. An alternative may be posted at a later point.

</details>
