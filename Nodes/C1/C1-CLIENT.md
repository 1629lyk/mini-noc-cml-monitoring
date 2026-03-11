
## Role

- Client endpoint on `10.20.20.0/24`
- Used to validate inter-LAN routing, DNS, HTTP, and optional DHCP

## Option A - Static bootstrap (recommended first)

Use static addressing first while validating the network:

```bash
sudo ip addr flush dev eth0
sudo ip link set eth0 up
sudo ip addr add 10.20.20.10/24 dev eth0
sudo ip route add default via 10.20.20.1
```

### Verify:

```bash
ip addr show eth0
ip route
ping -c 3 10.20.20.1
ping -c 3 10.20.10.10
wget -S -O- http://10.20.10.10/
nslookup app.lab.local 10.20.10.10
```


## Option B - DHCP client mode (after dnsmasq + relay work)

### R2-BRANCH e0/2 has:

```cisco
ip helper-address 10.20.10.10
```

So after DNS/DHCP is working on S1-SVC, switch C1-CLIENT to DHCP:

```bash
sudo ip addr flush dev eth0
sudo ip link set eth0 up
udhcpc -i eth0
```

### Verify:

```bash
ip addr show eth0
ip route
cat /etc/resolv.conf
ping -c 3 10.20.20.1
ping -c 3 10.20.10.10
wget -S -O- http://app.lab.local/
nslookup app.lab.local 10.20.10.10
```

Recovery note

Like S1-SVC, this Linux host can lose manual interface state during troubleshooting. If connectivity suddenly dies, rebuild the address and route using the static commands above.