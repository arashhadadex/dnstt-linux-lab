# Building a DNS Tunnel on Linux: Installing dnstt and danted
When traditional VPNs failed, I turned to DNS tunneling with dnstt. This article covers repository inspection, DNS delegation, Dante SOCKS5 proxy setup, and debugging a working tunnel.

## Original Project

This repository documents my experience deploying and experimenting with the dnstt DNS tunneling project. Special thanks to the creator and maintainer of the dnstt project for providing the original DNS tunneling implementation and documentation.

Original project website:
https://www.bamsoftware.com/git/dnstt.git

https://www.inet.no/dante/

https://dnstt.network/

GitHub:
https://github.com/bugfloyd/dnstt-deploy

Technologies we use:
- dnstt
- Dante SOCKS5
- Cloudflare DNS
- Ubuntu Linux
- systemd
- Go

You can also visit my website where i explained the step-by-step with details:

https://datatodeploy.com/dns-tunneling-with-dnstt/

## How dnstt works?
The client (Your device) encodes traffic into DNS packets. The server receives thos DNS packets, decodes them, and forwards the traffic thorugh a SOCKS proxy. DNS becomes a transport layer. Slow, but still work in restricted environments.


Your Browser > SOCKS5 Proxy > dnstt-client > DNS Queries > dnstt-server > Internet



## Configuring DNS Through Cloudflare
login to your cloudflare account and click on DNS records, add the following records:

- Type Name Points to
- A tns.example.com 186.112.111.6
- AAA tns.example.com 1002:cd8::2 (if IPV6 available)
- NS t.example.com tns.example.com

## Install Requirements
### Install Go

``` 
sudo apt update
sudo apt install golang-go git -y
```

Then check:

```
go version
```

### Install dnstt

```
git clone https://www.bamsoftware.com/git/dnstt.git
cd dnstt/dnstt-server
```

### Inspecting GitHub Repository

After cloning the repostory and before installing anything on our server with go build, it's a good pratice to inspect the repository carefully. 

### First look at the overall layout:

```
tree -L 2
```

If tree is not installed:

```
sudo apt install tree
```

### Then search for potentially dangerous commands:

```
grep -RniE "iptables|ufw|systemctl|curl|wget|docker.sock" .
```

Check whether the repository contains a docker-compose.yml , Caddyfile , Dockerfile , or any automatic installation scripts. 

## Building Project

```
go build

Expected output:
go: downloading github.com/xtaci/kcp-go/v5 v5.6.8 
go: downloading github.com/xtaci/smux v1.5.24 
go: downloading github.com/klauspost/reedsolomon v1.12.0 
go: downloading github.com/pkg/errors v0.9.1 
go: downloading github.com/templexxx/xorsimd v0.4.2 
go: downloading github.com/tjfoc/gmsm v1.4.1 
go: downloading golang.org/x/crypto v0.21.0 
go: downloading golang.org/x/net v0.23.0 
go: downloading github.com/flynn/noise v1.0.0 
go: downloading github.com/klauspost/cpuid/v2 v2.2.6 
go: downloading github.com/templexxx/cpu v0.1.0 
go: downloading golang.org/x/sys v0.18.0
```

Once the build finished, you have:

- dnstt-server
- dnstt-client

## Generate keys

Inside dnstt-server directory:

```
./dnstt-server -gen-key -privkey-file server.key -pubkey-file server.pub
```

This creates:
- server.key
- server.pub

## Installing Dante SOCKS Proxy

dnstt only handles the transport layer. danted play a SOCKS proxy role to forward traffic to the internet.

###  Install Dante SOCKS server

```
sudo apt update
sudo apt install dante-server -y
```

### Check service status

If configuration is correct, it show active.

```
sudo systemctl status danted

Result:

● danted.service - SOCKS (v4 and v5) proxy daemon (danted)
     Loaded: loaded (/lib/systemd/system/danted.service; enabled; vendor preset: enabled)
     Active: active (running) since Sat 2026-05-16 08:19:55 EEST; 24h ago
       Docs: man:danted(8)
             man:danted.conf(5)
   Main PID: 211903 (danted)
      Tasks: 20 (limit: 4552)
     Memory: 8.0M
        CPU: 165ms
     CGroup: /system.slice/danted.service
             ├─211903 /usr/sbin/danted
             ├─211904 "danted: monitor" ""
             ├─211907 "danted: request" ""
             └─212627 "danted: io-chil" ""

May 16 08:19:55 node01 systemd[1]: Starting SOCKS (v4 and v5) proxy daemon (danted)...
May 16 08:19:55 node01 systemd[1]: Started SOCKS (v4 and v5) proxy daemon (danted).
May 16 08:19:55 node01 danted[211903]: info: Dante/server[1/1] v1.4.2 running
```

## Verify port listening

After configuring Dante, I checked whether it was listening correctly:

```
sudo ss -tulpn | grep 1080

Result:
tcp   LISTEN 0      511        127.0.0.1:1080       0.0.0.0:*    users:(("danted",pid=211903,fd=8))  
```

Before running dnstt-server, make ensure nothing else already occupied port 53.

```
sudo ss -tulpn | grep :53

Result:
udp   UNCONN 0      0            0.0.0.0:53         0.0.0.0:*    users:(("dnstt-server",pid=212818,fd=6))
```


## Running dnstt Manually

Instead of immediately creating a service, first ran the server manually. It's much easier to debug.

```
sudo ./dnstt-server -udp :53 -privkey-file server.key t.example.com 127.0.0.1:1080

Output:
May 16 08:36:16 node01 systemd[1]: Started dnstt DNS Tunnel Server.
May 16 08:36:16 node01 dnstt-server[212818]: 2026/05/16 05:36:16 pubkey 763d2bs16a6624gf03081c5fdcd03e7426228be6w642b91n0b8213d2cp7a0o7i
May 16 08:36:16 node01 dnstt-server[212818]: 2026/05/16 05:36:16 effective MTU 932
```

## Installing the Client Side

Common platforms:
- Linux x64: dnstt-client-linux-amd64
- Windows x64: dnstt-client-windows-amd64.exe
- macOS Intel: dnstt-client-darwin-amd64
- macOS Apple Silicon: dnstt-client-darwin-arm64


On macOS, I downloaded the precompiled ARM binary:

```
dnstt-client-darwin-arm64

Rename it:
mv dnstt-client-darwin-arm64 dnstt-client
```

Then lauch the client:

```
./dnstt-client -udp SERVER_IP:53 -pubkey PUBLIC_KEY t.example.com 127.0.0.1:1080

If you saved your key in a file then: -pubkey-file pub.key


Output:
2026/05/17 07:19:05 uTLS fingerprint Firefox 120
2026/05/17 07:19:05 effective MTU 132
2026/05/17 07:19:05 begin session 77a988c6
```

Next configured the Firefox browser to use:
- SOCKS5
- localhost (127.0.0.1)
- port 1080
- Open Settings, search for proxies, then add manual proxy.

