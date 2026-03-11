## Role

- Windows-facing edge for the lab
- OSPF peer to `R2-BRANCH`
- BGP peer to `R3-EDGE`
- SNMP/syslog source
- NAT toward VMware VMnet8, with **NAT exemption for 192.168.254.0/24**

## Final working configuration

```cisco
enable
conf t
hostname R1-CORE
no ip domain-lookup
service timestamps log datetime msec localtime

interface Ethernet0/0
 description TO-EXT-BRIDGE
 ip address dhcp
 ip nat outside
 no shutdown

interface Ethernet0/1
 description TO-R2-BRANCH
 ip address 10.0.12.1 255.255.255.252
 ip nat inside
 no shutdown

interface Ethernet0/2
 description TO-R3-EDGE
 ip address 10.0.13.1 255.255.255.252
 ip nat inside
 no shutdown

interface Loopback0
 ip address 1.1.1.1 255.255.255.255

router ospf 1
 router-id 1.1.1.1
 network 1.1.1.1 0.0.0.0 area 0
 network 10.0.12.0 0.0.0.3 area 0

router bgp 65001
 bgp log-neighbor-changes
 neighbor 10.0.13.2 remote-as 65002
 network 1.1.1.1 mask 255.255.255.255

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

ip access-list extended NAT-LAB
 deny ip 10.20.10.0 0.0.0.255 192.168.254.0 0.0.0.255
 deny ip 10.20.20.0 0.0.0.255 192.168.254.0 0.0.0.255
 deny ip 10.0.12.0 0.0.0.3 192.168.254.0 0.0.0.255
 deny ip 10.0.13.0 0.0.0.3 192.168.254.0 0.0.0.255
 permit ip 10.20.10.0 0.0.0.255 any
 permit ip 10.20.20.0 0.0.0.255 any
 permit ip 10.0.12.0 0.0.0.3 any
 permit ip 10.0.13.0 0.0.0.3 any

route-map NAT-LAB permit 10
 match ip address NAT-LAB

ip nat inside source route-map NAT-LAB interface Ethernet0/0 overload

end
wr mem
```

## Verification Commands

```cisco
show ip interface brief
show ip ospf neighbor
show ip bgp summary
show ip route ospf
show ip nat translations
show ip nat statistics
show run | include ip nat inside source
show run | include logging
show run | include snmp-server
```

## Important Troubleshooting Note:
The original NAT rule was:

```cisco
ip nat inside source list LAB-NAT interface Ethernet0/0 overload
```

That broke Windows ↔ S1-SVC TCP sessions. The final fix was the route-map NAT above.

If IOS says the old NAT rule cannot be removed because a dynamic mapping is in use, use:

```cisco
show ip nat translations
clear ip nat translation *
```

and temporarily stop Prometheus/blackbox or pause traffic while replacing the rule.