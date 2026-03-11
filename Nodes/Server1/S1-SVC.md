# S1-SVC build (DNS + HTTP + optional DHCP server)

The final service-node guide is in nodes/S1-SVC.md


## Summary

* static IPv4 on eth0: `10.20.10.10/24`
* default route: `10.20.10.1`
* dnsmasq on `port 53`
* nginx on `port 80`

web root serves a real index.html returning 200 OK

---

## Role

- DNS service (`dnsmasq`)
- Optional DHCP server for `C1-CLIENT` via relay on `R2-BRANCH`
- HTTP service (`nginx`)
- Main application/service target for blackbox checks

## 1. Restore / apply IPv4 every time the node loses it

If the node loses its IPv4 config, reapply this first:

```bash
sudo ip addr flush dev eth0
sudo ip link set eth0 up
sudo ip addr add 10.20.10.10/24 dev eth0
sudo ip route add default via 10.20.10.1
```

---

### Verify 

```bash
ip addr show eth0
ip route
ping -c 3 10.20.10.1
```

### Expected:

```bash
eth0 has 10.20.10.10/24

default route via 10.20.10.1
```

---

## Fix Alpine repositories if package install fails

If apk update fails because of DNS / internet, fix routing first. If repository entries need correction, use:
```bash
cat /etc/alpine-release
cat /etc/apk/repositories
```

The lab was built on Alpine 3.21, so repo entries used v3.21.

### Install services
```bash
sudo apk update
sudo apk add dnsmasq nginx tcpdump
```

---

## dnsmasq config

```bash
cat <<'EOF' | sudo tee /etc/dnsmasq.conf
interface=eth0
listen-address=10.20.10.10
bind-interfaces

domain-needed
bogus-priv
domain=lab.local
expand-hosts

# Local DNS records
address=/app.lab.local/10.20.10.10
address=/dns.lab.local/10.20.10.10

# Optional DHCP scope for C1 via relay on R2
dhcp-authoritative
dhcp-range=10.20.20.100,10.20.20.150,255.255.255.0,12h
dhcp-option=option:router,10.20.20.1
dhcp-option=option:dns-server,10.20.10.10
EOF
```
---

## nginx web root and index page

### Create the web root:

```bash
sudo mkdir -p /var/www/localhost/htdocs

cat <<'EOF' | sudo tee /var/www/localhost/htdocs/index.html
<html>
  <body>
    <h1>Mini-NOC service OK</h1>
    <p>S1-SVC is serving HTTP correctly.</p>
  </body>
</html>
EOF
```

---


## Replace Alpine's default 404 site

The stock nginx config returned 404 for everything. Replace it with this:

```bash
cat <<'EOF2' | sudo tee /etc/nginx/http.d/default.conf
server {
    listen 80 default_server;
    listen [::]:80 default_server;

    server_name _;

    root /var/www/localhost/htdocs;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
EOF2
```

### Start services
```bash
sudo rc-service dnsmasq restart
sudo rc-service nginx restart
sudo rc-update add dnsmasq default
sudo rc-update add nginx default
```

### Validation commands
```bash
rc-service dnsmasq status
rc-service nginx status
ss -lntu | egrep ':53|:80'
wget -S -O- http://127.0.0.1/
nslookup app.lab.local 10.20.10.10
```

### Healthy state:

dnsmasq started

nginx started

listeners on :53 and :80

local wget returns HTTP/1.1 200 OK

local DNS lookup works

### Failure-test commands

#### DNS failure
```bash
sudo rc-service dnsmasq stop
```

#### Restore:

```bash
sudo rc-service dnsmasq start
HTTP failure
sudo rc-service nginx stop
sudo rc-service nginx start
```


### Useful packet-capture command

To see whether Windows reaches the service host:

```bash
sudo tcpdump -ni eth0 host 192.168.254.1 and tcp port 80
```