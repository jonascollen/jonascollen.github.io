---
title: TSHOOT - Part I, Internet & BGP
date: 2013-09-18 10:19
comments: true
categories: [Troubleshoot]
---
![bgp-internet](/assets/images/2013/09/bgp-internet1.png)
Tänkte fokusera på att sätta upp ovanstående del i detta inlägg vilket väl förhoppningsvis ska vara ganska basic. Kan väl börja med webservern.

```
Webserver(config)#int fa0/0
Webserver(config-if)#ip add 209.65.200.241 255.255.255.248
Webserver(config-if)#no shut
Webserver(config-if)#exit
Webserver(config)#no ip routing
Webserver(config)#ip default-gateway 209.65.200.242
```
ISP:n har vi lite mer att fixa med.
```
ISP(config)#inte fa0/0
ISP(config-if)#ip add 209.65.200.242 255.255.255.248
ISP(config-if)#no shut
ISP(config-if)#descrip Webserver
ISP(config-if)#inte fa0/1
ISP(config-if)#ip add 209.65.200.226 255.255.255.252
ISP(config-if)#no shut
ISP(config-if)#descrip to C
ISP(config-if)#descrip to CustomerA
ISP(config-if)#exit

ISP(config)#router bgp 65002
ISP(config-router)#neighbor 209.65.200.225 remote-as 65001
ISP(config-router)#neighbor 209.65.200.225 shutdown
ISP(config-router)#neighbor 209.65.200.225 description CustomerA
ISP(config-router)#neighbor 209.65.200.225 default-originate
ISP(config-router)#redistribute connected route-map SetOrigin
ISP(config-router)#exit

ISP(config)#access-list 1 permit 209.65.200.240
ISP(config)route-map SetOrigin permit 10
ISP(config-route-map)#match ip address 1
ISP(config-route-map)#set origin igp
ISP(config-route-map)#route-map SetOrigin permit 20
ISP(config-route-map)#exit
ISP(config)#
```
Att sätta igp origin är väl egentligen inte nödvändigt men det blir åtminstone snyggare än att ha ett "?". ;) R1
```
R1(config)#int fa0/1
R1(config-if)#ip add 209.65.200.225 255.255.255.248
R1(config-if)#no shut
R1(config-if)#description to ISP
R1(config-if)#exit

R1(config)#router bgp 65001
R1(config-router)#neighbor 209.65.200.226 remote-as 65002
R1(config-router)#neighbor 209.65.200.226 description to ISP
```
Vi kan nu testa ta upp BGP-sessionen på ISPn och förhoppningsvis ska vi väl ha full connectivity mellan Webservern och R1 efter det.
```
R1#sh ip route
Codes: C - connected, S - static, R - RIP, M - mobile, B - BGP
 D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area
 N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
 E1 - OSPF external type 1, E2 - OSPF external type 2
 i - IS-IS, su - IS-IS summary, L1 - IS-IS level-1, L2 - IS-IS level-2
 ia - IS-IS inter area, \* - candidate default, U - per-user static route
 o - ODR, P - periodic downloaded static route
Gateway of last resort is 209.65.200.226 to network 0.0.0.0
209.65.200.0/24 is variably subnetted, 3 subnets, 2 masks
B 209.65.200.240/29 \[20/0\] via 209.65.200.226, 00:03:13
B 209.65.200.224/30 \[20/0\] via 209.65.200.226, 00:03:13
C 209.65.200.224/29 is directly connected, FastEthernet0/1
B\* 0.0.0.0/0 \[20/0\] via 209.65.200.226, 00:00:02
```
Verifiering:
```
R1#sh ip bgp
BGP table version is 4, local router ID is 209.65.200.225
Status codes: s suppressed, d damped, h history, \* valid, > best, i - internal,
 r RIB-failure, S Stale
Origin codes: i - IGP, e - EGP, ? - incomplete
Network Next Hop Metric LocPrf Weight Path
\*> 0.0.0.0 209.65.200.226 0 0 65002 i
\*> 209.65.200.224/30
 209.65.200.226 0 0 65002 ?
\*> 209.65.200.240/29
 209.65.200.226 0 0 65002 i
R1#ping 209.65.200.241
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 209.65.200.241, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 24/40/64 ms
```
