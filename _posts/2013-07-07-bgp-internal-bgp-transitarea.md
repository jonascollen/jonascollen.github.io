---
title: BGP - Internal BGP & Transitarea
date: 2013-07-07 00:17
comments: true
categories: [BGP]
tags: [transit area]
---
Kommer skriva några inlägg som går mer in på djupet på det teoretiska men just nu är jag mer sugen på att labba lite. :) Byggde upp denna topologi hämtad från Renee Molenaar's "How to Master CCNP Route":

 [![ibgp ebgp](/assets/images/2013/07/ibgp-ebgp.png)](/assets/images/2013/07/ibgp-ebgp.png)

*   Varje router har ett loopback-interface med sitt router-id (ex. R1 - 1.1.1.1/R2 2.2.2.2)
*   OSPF används som IGP inom AS500
*   R5 & R6 annonserar sin loopback-interface över BGP

R5's grundkonfig:
```
interface Loopback0
 ip address 5.5.5.5 255.255.255.0
 !
 interface FastEthernet0/0
 ip address 172.16.51.5 255.255.255.0
 !
 router bgp 100
 no synchronization
 bgp log-neighbor-changes
 network 5.5.5.0 mask 255.255.255.0
 neighbor 172.16.51.1 remote-as 500
```
R1:
```
interface Loopback0
 ip address 1.1.1.1 255.255.255.0
 !
 interface FastEthernet0/0
 ip address 172.16.51.1 255.255.255.0
 !
 interface Serial0/0
 ip address 172.16.12.1 255.255.255.0
 clock rate 128000
 !
 interface Serial0/1
 ip address 172.16.14.1 255.255.255.0
 clock rate 128000
 !
 router ospf 1
 log-adjacency-changes
 passive-interface FastEthernet0/0
 network 0.0.0.0 255.255.255.255 area 0
 !
 router bgp 500
 no synchronization
 bgp log-neighbor-changes
 neighbor 172.16.51.5 remote-as 100
 no auto-summary
```
R3:
```
interface Loopback0
 ip address 3.3.3.3 255.255.255.0
 !
 interface FastEthernet0/0
 ip address 172.16.36.3 255.255.255.0
 !
 interface Serial0/0
 ip address 172.16.43.3 255.255.255.0
 clock rate 128000
 !
 interface Serial0/1
 ip address 172.16.23.3 255.255.255.0
 clock rate 2000000
 !
 router ospf 1
 log-adjacency-changes
 passive-interface FastEthernet0/0
 network 0.0.0.0 255.255.255.255 area 0
 !
 router bgp 500
 no synchronization
 bgp log-neighbor-changes
 neighbor 172.16.36.6 remote-as 200
 no auto-summary
```
R6:
```
interface Loopback0
 ip address 6.6.6.6 255.255.255.0
 !
 interface FastEthernet0/0
 ip address 172.16.36.6 255.255.255.0
 !
 router bgp 200
 no synchronization
 bgp log-neighbor-changes
 network 6.6.6.0 mask 255.255.255.0
 neighbor 172.16.36.3 remote-as 500
 no auto-summary
```
En show ip route på R5 & R6 visar följande: R5
```
Gateway of last resort is not set
5.0.0.0/24 is subnetted, 1 subnets
 C 5.5.5.0 is directly connected, Loopback0
 172.16.0.0/24 is subnetted, 1 subnets
 C 172.16.51.0 is directly connected, FastEthernet0/0
```
R6
```
Gateway of last resort is not set
6.0.0.0/24 is subnetted, 1 subnets
 C 6.6.6.0 is directly connected, Loopback0
 172.16.0.0/24 is subnetted, 1 subnets
 C 172.16.36.0 is directly connected, FastEthernet0/0
```
Och R1 & R3: R1
```
1.0.0.0/24 is subnetted, 1 subnets
 C 1.1.1.0 is directly connected, Loopback0
 2.0.0.0/32 is subnetted, 1 subnets
 O 2.2.2.2 [110/65] via 172.16.12.2, 00:26:08, Serial0/0
 3.0.0.0/32 is subnetted, 1 subnets
 O 3.3.3.3 [110/129] via 172.16.14.4, 00:26:08, Serial0/1
 [110/129] via 172.16.12.2, 00:26:08, Serial0/0
 4.0.0.0/32 is subnetted, 1 subnets
 O 4.4.4.4 [110/65] via 172.16.14.4, 00:26:08, Serial0/1
 5.0.0.0/24 is subnetted, 1 subnets
 B 5.5.5.0 [20/0] via 172.16.51.5, 00:22:25
 172.16.0.0/24 is subnetted, 6 subnets
 C 172.16.51.0 is directly connected, FastEthernet0/0
 O 172.16.43.0 [110/128] via 172.16.14.4, 00:26:11, Serial0/1
 O 172.16.36.0 [110/138] via 172.16.14.4, 00:26:11, Serial0/1
 [110/138] via 172.16.12.2, 00:26:11, Serial0/0
 O 172.16.23.0 [110/128] via 172.16.12.2, 00:26:11, Serial0/0
 C 172.16.12.0 is directly connected, Serial0/0
 C 172.16.14.0 is directly connected, Serial0/1
```
R3
```
1.0.0.0/32 is subnetted, 1 subnets
 O 1.1.1.1 [110/129] via 172.16.43.4, 00:31:22, Serial0/0
 [110/129] via 172.16.23.2, 00:31:22, Serial0/1
 2.0.0.0/32 is subnetted, 1 subnets
 O 2.2.2.2 [110/65] via 172.16.23.2, 00:31:12, Serial0/1
 3.0.0.0/24 is subnetted, 1 subnets
 C 3.3.3.0 is directly connected, Loopback0
 4.0.0.0/32 is subnetted, 1 subnets
 O 4.4.4.4 [110/65] via 172.16.43.4, 00:31:38, Serial0/0
 6.0.0.0/24 is subnetted, 1 subnets
 B 6.6.6.0 [20/0] via 172.16.36.6, 00:22:24
 172.16.0.0/24 is subnetted, 6 subnets
 O 172.16.51.0 [110/138] via 172.16.43.4, 00:31:51, Serial0/0
 [110/138] via 172.16.23.2, 00:32:50, Serial0/1
 C 172.16.43.0 is directly connected, Serial0/0
 C 172.16.36.0 is directly connected, FastEthernet0/0
 C 172.16.23.0 is directly connected, Serial0/1
 O 172.16.12.0 [110/128] via 172.16.23.2, 00:32:50, Serial0/1
 O 172.16.14.0 [110/128] via 172.16.43.4, 00:32:01, Serial0/0
```
Så R1 känner till R5's loopback, och R3 känner till R6's loopback. Men ska vi lyckas skicka trafik mellan R5 <-> R6 så behöver vi ju även ordna så R1 & R3 sprider denna information mellan varandra. Det är här iBGP (internal BGP) kommer in i bilden. Konfigurationen är i princip densamma, vi använder dock loopback-interfacen för att inte riskera tappa adjacency om vi skulle få problem med någon länk. R1:
```
neighbor 3.3.3.3 remote-as 500
neighbor 3.3.3.3 update-source Loopback0
```
R3:
```
neighbor 1.1.1.1 remote-as 500
neighbor 1.1.1.1 update-source Loopback0
```
Observera att vi måste specificera vilket source-interface vi skall använda, då neighbor-statement måste matchas (specificerar vi inte vilket används utgående interface istället). R5 kan nu se R6s nät och vice versa:
```
R5#sh ip route | beg Gat
 Gateway of last resort is not set
5.0.0.0/24 is subnetted, 1 subnets
 C 5.5.5.0 is directly connected, Loopback0
 6.0.0.0/24 is subnetted, 1 subnets
 B 6.6.6.0 [20/0] via 172.16.51.1, 00:04:34
 172.16.0.0/24 is subnetted, 1 subnets
 C 172.16.51.0 is directly connected, FastEthernet0/0
R6#sh ip route | beg Gat
 Gateway of last resort is not set
5.0.0.0/24 is subnetted, 1 subnets
 B 5.5.5.0 [20/0] via 172.16.36.3, 00:04:49
 6.0.0.0/24 is subnetted, 1 subnets
 C 6.6.6.0 is directly connected, Loopback0
 172.16.0.0/24 is subnetted, 1 subnets
 C 172.16.36.0 is directly connected, FastEthernet0/0
```
Så nu borde vi med andra ord kunna pinga mellan siterna.. Eller?
```
R5#ping 6.6.6.6 source 5.5.5.5
Type escape sequence to abort.
 Sending 5, 100-byte ICMP Echos to 6.6.6.6, timeout is 2 seconds:
 .....
 Success rate is 0 percent (0/5)
R5#traceroute 6.6.6.6 source 5.5.5.5
Type escape sequence to abort.
 Tracing the route to 6.6.6.6
1 172.16.51.1 56 msec 48 msec 32 msec
 2 172.16.14.4 44 msec 52 msec 60 msec
 3 172.16.14.4 !H * !H
```
Tillskillnad från IGP's så betyder det inte att vi kan nå en route bara för att den finns i vårat routing table när vi använder BGP! Vi fastnade i R4, hur ser då routing table ut där, kom ihåg att vi endast satt upp neighbor mellan R1 <-> R3?
```
Gateway of last resort is not set
1.0.0.0/32 is subnetted, 1 subnets
 O 1.1.1.1 [110/65] via 172.16.14.1, 00:51:28, Serial0/1
 2.0.0.0/32 is subnetted, 1 subnets
 O 2.2.2.2 [110/129] via 172.16.43.3, 00:51:18, Serial0/0
 [110/129] via 172.16.14.1, 00:51:18, Serial0/1
 3.0.0.0/32 is subnetted, 1 subnets
 O 3.3.3.3 [110/65] via 172.16.43.3, 00:51:08, Serial0/0
 4.0.0.0/24 is subnetted, 1 subnets
 C 4.4.4.0 is directly connected, Loopback0
 172.16.0.0/24 is subnetted, 6 subnets
 O 172.16.51.0 [110/74] via 172.16.14.1, 00:51:55, Serial0/1
 C 172.16.43.0 is directly connected, Serial0/0
 O 172.16.36.0 [110/74] via 172.16.43.3, 00:52:05, Serial0/0
 O 172.16.23.0 [110/128] via 172.16.43.3, 00:52:06, Serial0/0
 O 172.16.12.0 [110/128] via 172.16.14.1, 00:51:56, Serial0/1
 C 172.16.14.0 is directly connected, Serial0/1
```
Som synes har vi ingen route till varken R5 eller R6. Vi skulle kunna lösa detta genom att redistributa BGP-informationen till OSPF, men hur giltig är den lösningen om vi tänker oss att vi är ute "IRL" och BGP innehåller 350,000k+ routes? :P Vi behöver således sätta upp neighbors med både R4 & R2. Något som är unikt för iBGP är att vi måste konfigurera **fullmesh** när det kommer till neighbor-relationships, detta pga split-horizon för bgp agerar lite speciellt (mer om detta senare). R1
```
router bgp 500
 bgp log-neighbor-changes
 neighbor 2.2.2.2 remote-as 500
 neighbor 2.2.2.2 update-source Loopback0
 neighbor 4.4.4.4 remote-as 500
 neighbor 4.4.4.4 update-source Loopback0
```
R2
```
router bgp 500
 neighbor 1.1.1.1 remote-as 500
 neighbor 1.1.1.1 update-source Loopback0
 neighbor 3.3.3.3 remote-as 500
 neighbor 3.3.3.3 update-source Loopback0
 neighbor 4.4.4.4 remote-as 500
 neighbor 4.4.4.4 update-source Loopback0
```
R3
```
router bgp 500
 neighbor 2.2.2.2 remote-as 500
 neighbor 2.2.2.2 update-source Loopback0
 neighbor 4.4.4.4 remote-as 500
 neighbor 4.4.4.4 update-source Loopback0
```
R4
```
router bgp 500
 neighbor 1.1.1.1 remote-as 500
 neighbor 1.1.1.1 update-source Loopback0
 neighbor 2.2.2.2 remote-as 500
 neighbor 2.2.2.2 update-source Loopback0
 neighbor 3.3.3.3 remote-as 500
 neighbor 3.3.3.3 update-source Loopback0
```

Verifiering:

```
R1#sh ip bgp summary
 BGP router identifier 1.1.1.1, local AS number 500
 BGP table version is 27, main routing table version 27
 2 network entries using 240 bytes of memory
 2 path entries using 104 bytes of memory
 3/2 BGP path/bestpath attribute entries using 372 bytes of memory
 2 BGP AS-PATH entries using 48 bytes of memory
 0 BGP route-map cache entries using 0 bytes of memory
 0 BGP filter-list cache entries using 0 bytes of memory
 Bitfield cache entries: current 2 (at peak 3) using 64 bytes of memory
 BGP using 828 total bytes of memory
 BGP activity 14/12 prefixes, 14/12 paths, scan interval 60 secs
Neighbor V AS MsgRcvd MsgSent TblVer InQ OutQ Up/Down State/PfxRcd
 2.2.2.2 4 500 9 10 27 0 0 00:05:08 0
 3.3.3.3 4 500 45 45 27 0 0 00:16:53 1
 4.4.4.4 4 500 7 8 27 0 0 00:03:19 0
 172.16.51.5 4 100 56 66 27 0 0 00:52:19 1
```
Nu borde vi ha lite bättre action! Ping från R5 -> R6:
```
R5#ping 6.6.6.6 source 5.5.5.5
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 6.6.6.6, timeout is 2 seconds:
Packet sent with a source address of 5.5.5.5 
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 76/139/184 ms
R5#traceroute 6.6.6.6 source 5.5.5.5
Type escape sequence to abort.
Tracing the route to 6.6.6.6
1 172.16.51.1 56 msec 52 msec 60 msec
 2 172.16.12.2 136 msec 100 msec 68 msec
 3 172.16.23.3 96 msec 76 msec 144 msec
 4 172.16.36.6 172 msec * 256 msec
```
En sh ip route på R3 visar följande:
```
Gateway of last resort is not set
1.0.0.0/32 is subnetted, 1 subnets
 O 1.1.1.1 [110/129] via 172.16.43.4, 01:04:09, Serial0/0
 [110/129] via 172.16.23.2, 01:04:09, Serial0/1
 2.0.0.0/32 is subnetted, 1 subnets
 O 2.2.2.2 [110/65] via 172.16.23.2, 01:03:59, Serial0/1
 3.0.0.0/24 is subnetted, 1 subnets
 C 3.3.3.0 is directly connected, Loopback0
 4.0.0.0/32 is subnetted, 1 subnets
 O 4.4.4.4 [110/65] via 172.16.43.4, 01:04:25, Serial0/0
 **5.0.0.0/24 is subnetted, 1 subnets**
 **B 5.5.5.0 [200/0] via 172.16.51.5, 00:20:34**
 6.0.0.0/24 is subnetted, 1 subnets
 B 6.6.6.0 [20/0] via 172.16.36.6, 00:55:10
 172.16.0.0/24 is subnetted, 6 subnets
 **O 172.16.51.0 [110/138] via 172.16.43.4, 01:04:37, Serial0/0
 [110/138] via 172.16.23.2, 01:05:37, Serial0/1**
 C 172.16.43.0 is directly connected, Serial0/0
 C 172.16.36.0 is directly connected, FastEthernet0/0
 C 172.16.23.0 is directly connected, Serial0/1
 O 172.16.12.0 [110/128] via 172.16.23.2, 01:05:37, Serial0/1
 O 172.16.14.0 [110/128] via 172.16.43.4, 01:04:47, Serial0/0
```
Se dock nexthop-adressen! Till skillnad från IGPs som EIGRP/OSPF så ändras inte nexthop-adressen när en route annonseras inom iBGP. Just nu annonserar vi ju dock länknäten inom OSPF,  med hur löser vi detta om så inte är fallet? Vi konfar om OSPF så endast länknäten inom AS500 & loopback-interfacen annonseras. En show ip route på R1 ser nu ut såhär:
```
Gateway of last resort is not set
1.0.0.0/24 is subnetted, 1 subnets
 C 1.1.1.0 is directly connected, Loopback0
 2.0.0.0/32 is subnetted, 1 subnets
 O 2.2.2.2 [110/65] via 172.16.12.2, 00:02:39, Serial0/0
 3.0.0.0/32 is subnetted, 1 subnets
 O 3.3.3.3 [110/129] via 172.16.14.4, 00:02:29, Serial0/1
 [110/129] via 172.16.12.2, 00:02:29, Serial0/0
 4.0.0.0/32 is subnetted, 1 subnets
 O 4.4.4.4 [110/65] via 172.16.14.4, 00:02:19, Serial0/1
 5.0.0.0/24 is subnetted, 1 subnets
 B 5.5.5.0 [20/0] via 172.16.51.5, 01:28:21
 172.16.0.0/24 is subnetted, 5 subnets
 C 172.16.51.0 is directly connected, FastEthernet0/0
 O 172.16.43.0 [110/128] via 172.16.14.4, 00:04:18, Serial0/1
 O 172.16.23.0 [110/128] via 172.16.12.2, 00:04:28, Serial0/0
 C 172.16.12.0 is directly connected, Serial0/0
 C 172.16.14.0 is directly connected, Serial0/1
```
Observera att vi inte längre ser 6.6.6.0/24-nätet! Anledningen till detta är att R1 inte kan resolve'a next-hop adressen som R6 annonserar (172.16.36.6) och den ignorerar därför routen! Samma sak gäller givetvis för R3. Vi kan lösa detta på två sätt, antingen genom att annonsera ut länknätet via ex. OSPF som vi gjorde tidigare, eller kommandot "next-hop-self". R3 kommer då ersätta next-hop adressen 172.16.36.6 till sin egen loopback 3.3.3.3, och då denna adress finns i routing table borde vi återigen se nätet i R1. Låt oss testa.. R1
```
neighbor 2.2.2.2 next-hop-self
 neighbor 3.3.3.3 next-hop-self
 neighbor 4.4.4.4 next-hop-self
```
R3
```
neighbor 1.1.1.1 next-hop-self
 neighbor 2.2.2.2 next-hop-self
 neighbor 4.4.4.4 next-hop-self
```
En show ip route på R1 visar nu följande:
```
Gateway of last resort is not set
1.0.0.0/24 is subnetted, 1 subnets
 C 1.1.1.0 is directly connected, Loopback0
 2.0.0.0/32 is subnetted, 1 subnets
 O 2.2.2.2 [110/65] via 172.16.12.2, 00:09:06, Serial0/0
 3.0.0.0/32 is subnetted, 1 subnets
 O 3.3.3.3 [110/129] via 172.16.14.4, 00:08:56, Serial0/1
 [110/129] via 172.16.12.2, 00:08:56, Serial0/0
 4.0.0.0/32 is subnetted, 1 subnets
 O 4.4.4.4 [110/65] via 172.16.14.4, 00:08:48, Serial0/1
 5.0.0.0/24 is subnetted, 1 subnets
 B 5.5.5.0 [20/0] via 172.16.51.5, 01:34:47
 **6.0.0.0/24 is subnetted, 1 subnets**
 **B 6.6.6.0 [200/0] via 3.3.3.3, 00:00:31**
 172.16.0.0/24 is subnetted, 5 subnets
 C 172.16.51.0 is directly connected, FastEthernet0/0
 O 172.16.43.0 [110/128] via 172.16.14.4, 00:10:45, Serial0/1
 O 172.16.23.0 [110/128] via 172.16.12.2, 00:10:55, Serial0/0
 C 172.16.12.0 is directly connected, Serial0/0
 C 172.16.14.0 is directly connected, Serial0/1
```
Låt oss testa en pinga från R5 - R6:
```
R5#ping 6.6.6.6 source 5.5.5.5
Type escape sequence to abort.
 Sending 5, 100-byte ICMP Echos to 6.6.6.6, timeout is 2 seconds:
 Packet sent with a source address of 5.5.5.5
 !!!!!
 Success rate is 100 percent (5/5), round-trip min/avg/max = 76/139/184 ms
```
Wohoo! :) Känns som det får avsluta första inlägget om BGP, känns som detta kommer bli riktigt kul nördig som man är.. :)
