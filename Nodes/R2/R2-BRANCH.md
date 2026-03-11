## Role

- OSPF peer to `R1-CORE`
- Gateway for both inside LANs
- DHCP relay for `C1-CLIENT`
- SNMP/syslog source

## Final working configuration

```cisco
enable
conf t
hostname R2-BRANCH
no ip domain-lookup
service timestamps log datetime msec localtime

interface Ethernet0/0
 description TO-R1-CORE
 ip address 10.0.12.2 255.255.255.252
 no shutdown

interface Ethernet0/1
 description TO-S1-SVC
 ip address 10.20.10.1 255.255.255.0
 no shutdown

interface Ethernet0/2
 description TO-C1-CLIENT
 ip address 10.20.20.1 255.255.255.0
 ip helper-address 10.20.10.10
 no shutdown

interface Loopback0
 ip address 2.2.2.2 255.255.255.255

ip route 0.0.0.0 0.0.0.0 10.0.12.1

router ospf 1
 router-id 2.2.2.2
 network 2.2.2.2 0.0.0.0 area 0
 network 10.0.12.0 0.0.0.3 area 0
 network 10.20.10.0 0.0.0.255 area 0
 network 10.20.20.0 0.0.0.255 area 0

logging buffered 64000 informational
logging host 192.168.254.1 transport udp port 1514
logging trap informational
logging source-interface Loopback0
logging origin-id hostname

snmp-server community public RO
snmp-server ifindex persist
snmp-server contact lab
snmp-server location CML
snmp-server trap-source Loopback0
snmp-server enable traps snmp linkdown linkup coldstart warmstart

end
wr mem
```

---

## Verification Commands
```cisco
show ip interface brief
show ip ospf neighbor
show ip ospf interface brief
show ip route
show run interface Ethernet0/2
show run | section router ospf
show run | include logging
show run | include snmp-server
```

---

## Failure-test commands

### Interface flap

```cisco
conf t
interface Ethernet0/2
 shutdown
 no shutdown
end
```

### OSPF failure
```cisco
conf t
interface Ethernet0/0
 shutdown
end
```

### Restore:

```cisco
conf t
interface Ethernet0/0
 no shutdown
end
```