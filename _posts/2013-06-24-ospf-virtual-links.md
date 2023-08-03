---
title: OSPF - Virtual Links
date: 2013-06-24 19:06
comments: true
categories: [OSPF]
tags: [virtual-link]
---
Detta blir ett kort inlägg om Virtual Link, vilket som jag redan nämnt i några tidigare inlägg används för att skapa en sorts tunnel mellan ABR's när en ny area inte har direktkontakt med area 0 (som annars är ett krav för att kommunikationen skall fungera). 
[![ospf virtuallink](/assets/images/2013/06/ospf-virtuallink.png)](/assets/images/2013/06/ospf-virtuallink.png) 

Om vi använder oss av ovanstående topologi så kan vi testa detta i praktiken. Som synes har area 20 ingen direktanslutning till area 0 just nu, hur påverkar det routing-uppdateringarna? Vi kan se i R2 att den bildat neighbor adjacency mot både R1 & R2:

```
Neighbor ID Pri State Dead Time Address Interface
1.1.1.1 1 FULL/DR 00:00:33 10.0.0.1 FastEthernet0/1
3.3.3.3 1 FULL/DR 00:00:39 172.16.0.2 FastEthernet0/0
```
Men en "**sh ip route**" ser dock mindre bra ut.. R1
```
10.0.0.0/30 is subnetted, 1 subnets
C 10.0.0.0 is directly connected, FastEthernet0/1
C 192.168.0.0/24 is directly connected, Loopback0
```
R2
```
C 172.16.0.0/16 is directly connected, FastEthernet0/0
 10.0.0.0/30 is subnetted, 1 subnets
C 10.0.0.0 is directly connected, FastEthernet0/1
O IA 192.168.0.0/24 \[110/11\] via 10.0.0.1, 00:04:05, FastEthernet0/1
```
R3

`C    172.16.0.0/16 is directly connected, FastEthernet0/0`

R3 får inga routes, och R2 lär sig bara om area 0. Det är här virtual-link kommer in i bilden, genom att skapa en tunnel över area 10 kan vi "lura" area 20 att den är "directly connected". Vi skapar länken mellan ABR's i själva "transit arean", i detta fall mellan R1 & R2, med följande kommando: R1
```
router ospf 1
area 10 virtual-link 2.2.2.2
```
R2
```
router ospf 1
area 10 virtual-link 1.1.1.1
```
Observera att det ej är ip-adressen till ABR vi anger utan dess **router-id**!
```
R1#sh ip ospf virtual-links 
Virtual Link OSPF\_VL0 to router 2.2.2.2 is up
 Run as demand circuit
 DoNotAge LSA allowed.
 **Transit area 10**, via interface FastEthernet0/1, Cost of using 10
 Transmit Delay is 1 sec, State POINT\_TO\_POINT,
 Timer intervals configured, Hello 10, Dead 40, Wait 40, Retransmit 5
 Hello due in 00:00:04
 Adjacency State FULL (Hello suppressed)
 Index 1/2, retransmission queue length 0, number of retransmission 0
 First 0x0(0)/0x0(0) Next 0x0(0)/0x0(0)
 Last retransmission scan length is 0, maximum is 0
 Last retransmission scan time is 0 msec, maximum is 0 msec
```
Virtual-link skapar även ytterligare en neighbor-adjacency:
```
\*Mar  1 00:30:48.279: %OSPF-5-ADJCHG: Process 1, Nbr 1.1.1.1 on OSPF\_VL1 from LOADING to FULL, Loading Done
R2(config-router)#do sh ip ospf neig
Neighbor ID Pri State Dead Time Address Interface
**1.1.1.1 0 FULL/ - - 10.0.0.1 OSPF\_VL1**
1.1.1.1 1 FULL/DR 00:00:32 10.0.0.1 FastEthernet0/1
3.3.3.3 1 FULL/DR 00:00:38 172.16.0.2 FastEthernet0/0
```
Och kontrollerar vi OSPF-databasen kan vi se att virtual-linken räknas som ett Type 1-LSA och har tillägget DNA (Do Not Age).
```
OSPF Router with ID (2.2.2.2) (Process ID 1)
Router Link States (Area 0)
Link ID ADV Router Age Seq# Checksum Link count
**1.1.1.1 1.1.1.1 1 (DNA) 0x80000005 0x002568 2**
2.2.2.2 2.2.2.2 431 0x80000005 0x00BB47 1
```
[![virtual link lsa](/assets/images/2013/06/virtual-link-lsa.png)](/assets/images/2013/06/virtual-link-lsa.png)

Om du glömt bort hur LSA Type 1 byggs upp med de olika **Link Type's** som finns etc, repetera [denna post](http://Jonas Collén.wordpress.com/2013/06/16/ospf-lsa-types/) om LSA-types. Vilken effekt har detta då på R1 & R3's routing tabell? Låt oss kolla.. 

R1
```
Gateway of last resort is not set
O IA 172.16.0.0/16 \[110/20\] via 10.0.0.2, 00:07:56, FastEthernet0/1
 10.0.0.0/30 is subnetted, 1 subnets
C 10.0.0.0 is directly connected, FastEthernet0/1
C 192.168.0.0/24 is directly connected, Loopback0
```
R3
```
Gateway of last resort is not set
C 172.16.0.0/16 is directly connected, FastEthernet0/0
 10.0.0.0/30 is subnetted, 1 subnets
O IA 10.0.0.0 \[110/20\] via 172.16.0.1, 00:08:16, FastEthernet0/0
O IA 192.168.0.0/24 \[110/21\] via 172.16.0.1, 00:08:07, FastEthernet0/0
```
Svårare än så är det inte. Men just användandet av Virtual-links räknas som en "ful-lösning"/plåster och är inte något du ska sträva efter att ha i ditt nät.
