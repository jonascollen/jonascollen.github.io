---
title: MPLS & MP-BGP Del 2
date: 2014-01-21 20:55
comments: true
categories: [BGP, MPLS]
---
Tänkte fortsätta på det tidigare inlägget [CCDP – MPLS & MP-BGP](http://www.jonascollen.se/posts/ccdp-mpls-mp-bgp/) där vi satte upp ett core-nät med MPLS & MP-BGP, och visa ett simplet exempel på hur vi installerar två kundsiter som kopplas samman via VPN (VRF) för att utbyta dynamisk routing. 

![Customer-MPLS](/assets/images/2014/01/customer-mpls.png)
 Företaget använder sig av OSPF internt där varje regionalkontor placeras i en egen area (i detta fall 461), och upplänken mot PE/Core i area 0. 
 
 **NL PE KistaRed** 
 Vi skapar först en VRF-instans som är unik för kunden, i detta fall vrf LIGHT med "route distinguisher" till 300:10. Routes märkta med just 300:10 importeras till denna routers vrf-instans (LIGHT) samtidigt som vi exporterar lokala vrf-routes till samma instans för annonsering vidare till övriga PEs.

```
ip vrf LIGHT
 rd 300:10
 route-target both 300:10
```
Konfigurera upp lämpligt interface med tillhörande VRF:

```
interface s0/3
 description LIGHT-Internal Kista
 ip vrf forwarding LIGHT
 ip address 37.46.2.1 255.255.255.252
```

Då PE-routern ska peera med kundens OSPF-instans krävs lite av en specialare, tänk på att vi redan kör OSPF som IGP inom core-nätet och där vill vi absolut ej ha in några routes från denna kund. Vi kan däremot köra flera samtidiga OSPF-instanser som vi sedan placerar i olika vrf:er, i detta fall en för vrf LIGHT. Redistribution krävs mellan OSPF & MP-BGP instansen för att KistaRed ska annonsera andra vrf LIGHT OSPF-routes den lär sig via just MP-BGP från andra PEs.

```
router ospf 310 vrf LIGHT 
 router-id 37.46.2.1
 auto-cost reference-bandwidth 20000000
 **redistribute bgp 46 subnets**
 network 37.46.2.0 0.0.0.3 area 0
```

Vi skapar sedan en MP-BGP instans för kunden, samma sak här - använd redistribution:

```
router bgp 46
 address-family ipv4 vrf LIGHT
  redistribute ospf 310 vrf LIGHT
  no synchronization
  exit-address-family
```

**NL PE KistaRed**

```
ip vrf LIGHT
 rd 300:10
 route-target both 300:10

interface s0/3
 description LIGHT-Internal Kiruna
 ip vrf forwarding LIGHT
 ip address 37.46.2.5 255.255.255.252

router ospf 310 vrf LIGHT
 router-id 37.46.2.5
 auto-cost reference-bandwidth 20000000
 redistribute bgp 46 subnets
 network 37.46.2.4 0.0.0.3 area 0

router bgp 46
 address-family ipv4 vrf LIGHT
  redistribute ospf 310 vrf LIGHT
  no synchronization
  exit-address-family
```

**NLKista-Edge** 

Edge-routern behöver sedan endast konfigureras med OSPF-instansen för att det ska hoppa igång.

```
!Downlink to CustomerLAN, demo with Loopback
interface Loopback0
 description to CustomerLAN Kista
 ip address 10.46.0.1 255.255.254.0
 ip ospf network point-to-point

interface Serial1/3
 description to ISP-NL-KistaRed
 ip address 37.46.2.2 255.255.255.252

interface Serial1/2
 description to ISP-NL-KistaBlue
 ip address 37.46.2.6 255.255.255.252

router ospf 310
 router-id 37.46.128.1
 auto-cost reference-bandwidth 20000000
 network 37.46.2.0 0.0.0.3 area 0 
 network 37.46.2.4 0.0.0.3 area 0
 network 10.46.0.0 0.0.63.255 area 461
```

**Verifikation**
![Customer-MPLSKiruna](/assets/images/2014/01/customer-mplskiruna.png?w=630)

Ett till kontor sattes upp i Kiruna med identisk konfig för att kunna köra lite tester. Ping från Kista- till Kiruna-kontoret:

```
NL-Office-Kista#ping 10.46.64.1 source lo0
Success rate is 100 percent (5/5), round-trip min/avg/max = 28/43/60 ms
```

Traceroute mellan Kista- & Kiruna-kontoret:

```
NL-Office-Kista#traceroute 10.46.64.1 source lo0
Type escape sequence to abort.
Tracing the route to 10.46.64.1
1 37.46.2.5 36 msec
37.46.2.1 20 msec
37.46.2.5 28 msec
2 37.46.2.9 44 msec
37.46.2.13 48 msec
37.46.2.9 60 msec
3 37.46.2.14 100 msec
37.46.2.10 72 msec
37.46.2.14 24 msec
```

Routing-tabellen för Kista-kontoret:
```
NL-Office-Kista#sh ip route | beg Gateway
37.0.0.0/30 is subnetted, 4 subnets
O IA 37.46.2.8 \[110/65\] via 37.46.2.5, 00:02:54, Serial1/2
\[110/65\] via 37.46.2.1, 00:02:54, Serial1/3
O IA 37.46.2.12 \[110/65\] via 37.46.2.5, 00:02:54, Serial1/2
\[110/65\] via 37.46.2.1, 00:02:54, Serial1/3
C 37.46.2.0 is directly connected, Serial1/3
C 37.46.2.4 is directly connected, Serial1/2
10.0.0.0/18 is subnetted, 2 subnets
C 10.46.0.0 is directly connected, Loopback0
**O IA 10.46.64.0 \[110/129\] via 37.46.2.5, 00:01:56, Serial1/2**
**\[110/129\] via 37.46.2.1, 00:01:56, Serial1/3**
```

PE

```
KistaBlue#sh ip bgp vpnv4 vrf LIGHT
BGP table version is 15, local router ID is 37.46.255.102
Status codes: s suppressed, d damped, h history, \* valid, > best, i - internal,
 r RIB-failure, S Stale
Origin codes: i - IGP, e - EGP, ? - incomplete
Network Next Hop Metric LocPrf Weight Path
Route Distinguisher: 300:10 (default for vrf LIGHT)
\* i10.46.0.0/23 37.46.255.101 65 100 0 ?
\*> 37.46.2.6 65 32768 ?
\* i10.46.2.0/23 37.46.255.103 65 100 0 ?
\*>i 37.46.255.104 65 100 0 ?
\* i37.46.2.0/30 37.46.255.101 0 100 0 ?
\*> 37.46.2.6 128 32768 ?
\* i37.46.2.4/30 37.46.255.101 128 100 0 ?
\*> 0.0.0.0 0 32768 ?
\*>i37.46.2.8/30 37.46.255.103 0 100 0 ?
\* i 37.46.255.104 128 100 0 ?
\* i37.46.2.12/30 37.46.255.103 128 100 0 ?
\*>i 37.46.255.104 0 100 0 ?

KistaBlue#sh ip route vrf LIGHT | beg Gateway
Gateway of last resort is not set
37.0.0.0/30 is subnetted, 4 subnets
B 37.46.2.8 \[200/0\] via 37.46.255.103, 00:22:17
B 37.46.2.12 \[200/0\] via 37.46.255.104, 00:22:32
O 37.46.2.0 \[110/128\] via 37.46.2.6, 00:22:59, Serial0/3
C 37.46.2.4 is directly connected, Serial0/3
 10.0.0.0/23 is subnetted, 2 subnets
**O IA 10.46.0.0 \[110/65\] via 37.46.2.6, 00:22:59, Serial0/3
B 10.46.2.0 \[200/65\] via 37.46.255.104, 00:22:32**
```

Svårare än så är det faktiskt inte. :)
