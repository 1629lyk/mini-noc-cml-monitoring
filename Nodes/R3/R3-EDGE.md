## Role

- BGP peer to `R1-CORE`
- Simulated upstream / WAN router
- SNMP/syslog source

## Final working configuration

```cisco
enable
conf t
hostname R3-EDGE
no ip domain-lookup
service timestamps log datetime msec localtime

interface Ethernet0/0
 description TO-R1-CORE
 ip address 10.0.13.2 255.255.255.252
 no shutdown

interface Loopback0
 ip address 3.3.3.3 255.255.255.255

ip route 0.0.0.0 0.0.0.0 10.0.13.1

router bgp 65002
 bgp log-neighbor-changes
 neighbor 10.0.13.1 remote-as 65001
 network 3.3.3.3 mask 255.255.255.255

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

## Verification commands

```cisco
show ip interface brief
show ip route
show ip bgp summary
show run | include logging
show run | include snmp-server
```

---

## Failure-test command

### BGP failure
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
