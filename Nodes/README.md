## Router Configurations 

Refer the Readme files in the respective folders

## Critical Linux-host lesson

### If S1-SVC or C1-CLIENT suddenly stop responding, first re-check:

```bash
ip addr show eth0
ip route
```

If S1-SVC loses its IPv4 config, reapply:

```bash
sudo ip addr flush dev eth0
sudo ip link set eth0 up
sudo ip addr add 10.20.10.10/24 dev eth0
sudo ip route add default via 10.20.10.1
```

---

## Windows Routes into the lab

Add persistent routes on Windows:

```powershell
route -p add 10.20.10.0 mask 255.255.255.0 192.168.254.130
route -p add 10.20.20.0 mask 255.255.255.0 192.168.254.130
route -p add 1.1.1.1 mask 255.255.255.255 192.168.254.130
route -p add 2.2.2.2 mask 255.255.255.255 192.168.254.130
route -p add 3.3.3.3 mask 255.255.255.255 192.168.254.130
```

If PowerShell says The object already exists, that is fine.

---

## Baseline validation sequence

Do this before any failure testing.

### Router health

#### On R1-CORE:

```cisco
show ip interface brief
show ip ospf neighbor
show ip bgp summary
show ip route ospf
```

##### Expected:

* OSPF Full with R2-BRANCH
* BGP Established with R3-EDGE
* OSPF routes to `10.20.10.0/24` and `10.20.20.0/24`


#### S1-SVC
```bash
ip addr show eth0
ip route
ss -lntu | egrep ':53|:80'
wget -S -O- http://127.0.0.1/
nslookup app.lab.local 10.20.10.10
```

##### Expected:

* eth0 has `10.20.10.10/24`
* default route via `10.20.10.1`
* listeners on `:53` and `:80`
* local HTTP returns `200 OK`

#### C1-CLIENT

```bash
ping -c 3 10.20.20.1
ping -c 3 10.20.10.10
wget -S -O- http://10.20.10.10/
nslookup app.lab.local 10.20.10.10
```


#### Windows Powershell

```powershell
Test-NetConnection 10.20.10.10 -Port 80
curl.exe -i http://10.20.10.10/
nslookup app.lab.local 10.20.10.10
```

##### Expected:

* TcpTestSucceeded : `True`
* HTTP response from `10.20.10.10`
* DNS resolution of `app.lab.local`


