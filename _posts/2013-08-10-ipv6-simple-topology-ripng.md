---
title: IPv6 - Simple Topology & RIPng
date: 2013-08-10 19:12
comments: true
categories: [IPv6,RIPng]
---
Detta blir ett kortare inlägg om hur vi konfar upp ett litet IPv6-nät med Global & Link-local adresser. 
[![ipv6-simple-topology](/assets/images/2013/08/ipv6-simple-topology.png)](/assets/images/2013/08/ipv6-simple-topology.png) 

Låt oss börja med R1: För att aktivera IPv6 routing måste vi först lägga till:

```
ipv6 unicast-routing

Interface-konfigen blir sedan:

interface FastEthernet0/0
description To CustLAN
!Global
ipv6 address 2001:db8:6783:1::1/64
!Sätter statisk ipv6 för link-local istället för en genererad EUI64-adress
ipv6 address fe80::1 link-local
no shut
interface Serial0/0
 ipv6 address FE80::1 link-local
 ipv6 address 2001:DB8:6783:12::1/64
 clock rate 256000
no shut

interface Serial0/1
 ipv6 address FE80::1 link-local
 ipv6 address 2001:DB8:6783:13::1/64
 clock rate 256000
no shut
interface Loopback0
ipv6 address 2001:db8:6783:1111::1/64
```
Det är inga problem att ha samma link-local adress på alla interface, kom ihåg att den endast gäller lokalt inom L2-segmentet! Detta ger följande output: 
[![ipv6-biref](/assets/images/2013/08/ipv6-biref.png)](/assets/images/2013/08/ipv6-biref.png)

R2:

```
ipv6 unicast-routing
interface Loopback0
ipv6 address 2001:DB8:6783:2222::2/64

interface FastEthernet0/0
ipv6 address FE80::2 link-local
 ipv6 address 2001:DB8:6783:2::2/64
no shut

interface Serial0/0
ipv6 address FE80::2 link-local
 ipv6 address 2001:DB8:6783:12::2/64
no shut

interface Serial0/1
ipv6 address FE80::2 link-local
 ipv6 address 2001:DB8:6783:23::2/64
 clock rate 256000
no shut
```
R3
```
ipv6 unicast-routing
interface Loopback0
ipv6 address 2001:DB8:6783:3333::3/64
no shut
interface FastEthernet0/0
ipv6 address FE80::3 link-local
ipv6 address 2001:DB8:6783:3::3/64
no shut
interface Serial0/0
ipv6 address FE80::3 link-local
ipv6 address 2001:DB8:6783:23::3/64
no shut
interface Serial0/1
ipv6 address FE80::3 link-local
ipv6 address 2001:DB8:6783:13::3/64
no shut
```
Hur gör vi då om vi vill använda oss av ett routing-protokoll. Låt oss börja med den enklaste, RIPng (RIP Next-Generation). Till skillnad från IPv4 så aktiverar vi RIPng på interfacet istället för via network-statements, samt vilken RIP-instans vi vill ansluta interfacet till. Konfigen är densamma för R1/R2/R3:
```
int fa0/0
ipv6 rip RIP-LAB enable
int s0/0
ipv6 rip RIP-LAB enable
int s0/1
ipv6 rip RIP-LAB enable
int lo0
ipv6 rip RIP-LAB enable
```
Löjligt enkelt. Vi verifierar med hjälp av show ipv6 route rip: 
[![ipv6-route](/assets/images/2013/08/ipv6-route.png)](/assets/images/2013/08/ipv6-route.png) 

Observera att routern använder Link-local adresser som next hop!
```
R1#ping ipv6 2001:db8:6783:2::2
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 2001:DB8:6783:2::2, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 0/5/20 ms
R1#ping ipv6 2001:db8:6783:3::3
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 2001:DB8:6783:3::3, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 0/9/28 ms
```