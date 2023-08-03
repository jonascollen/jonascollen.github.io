---
title: Policy-based Routing - The Basics
date: 2013-07-02 14:14
comments: true
categories: [Policy-based routing, Route Manipulation]
tags: [pbr]
---
Detta blir ett kort inlägg om policy-based routing där vi återigen använder oss av route maps för att modifiera routingen. Satte upp följande topologi med OSPF som routing-protokoll: 
[![Policy-based routing](/assets/images/2013/07/policy-based-routing.png)](/assets/images/2013/07/policy-based-routing.png) 

Låt oss säga att vi nu vill göra en form av lastbalansering och skicka trafik som ska till 10.1.1.0/24 & 10.1.2.0/24 via R3, och trafik till 10.1.3.0/24 & 10.1.4.0/24 via R4. Möjligtvis hade vi kunnat åstadkomma detta genom att justera metrics i en evighet men det går att lösa betydligt enklare med hjälp av "Policy-based routing". Vi skapar först två access-listor:

```
access-list 100 permit ip any 10.1.1.0 0.0.0.255
access-list 100 permit ip any 10.1.2.0 0.0.0.255
access-list 110 permit ip any 10.1.3.0 0.0.0.255
access-list 110 permit ip any 10.1.4.0 0.0.0.255
```
Och sedan tillhörande route-map:
```
route-map PBR permit 10
 match ip address 100
 set ip next-hop 10.23.0.3

route-map PBR permit 20
 match ip address 110
 set ip next-hop 10.24.0.4
```
Till skillnad från gårdagens distribution-lab så aktiverar vi policy-routing på önskat interface, i detta fall R2's fa1/0 mot R1:
```
interface FastEthernet1/0
 no switchport
 ip address 10.12.0.2 255.255.255.0
 ip policy route-map PBR
```
Detta gäller dock endast trafik som flödar GENOM routern, om vi även vill applicera en policy på trafik som genereras från den egna router används kommandot: _**ip local policy route-map PBR**_. Gör vi nu en traceroute från R1 kan vi se effekten detta har:
```
Tracing the route to 10.1.1.1
1 10.12.0.2 40 msec 40 msec 32 msec
 **2 10.23.0.3 56 msec 52 msec 40 msec**
 3 10.35.0.5 84 msec * 56 msec
Tracing the route to 10.1.2.1
1 10.12.0.2 60 msec 40 msec 24 msec
 **2 10.23.0.3 60 msec 44 msec 24 msec**
 3 10.35.0.5 68 msec * 68 msec
Tracing the route to 10.1.3.1
1 10.12.0.2 32 msec 64 msec 28 msec
 **2 10.24.0.4 60 msec 44 msec 44 msec**
 3 10.45.0.5 44 msec * 64 msec
Tracing the route to 10.1.4.1
1 10.12.0.2 44 msec 56 msec 20 msec
 **2 10.24.0.4 60 msec 24 msec 52 msec**
 3 10.45.0.5 52 msec * 44 msec
```
Genom debug-kommandot "**debug ip policy**" kan vi se vad som sker när vi pingar från R1 (10.12.0.1) till R5's 10.1.3.1 in action:
```
*Mar 1 00:33:04.367: IP: s=10.12.0.1 (FastEthernet1/0), d=10.1.3.1, len 28, FIB policy match
*Mar 1 00:33:04.367: IP: s=10.12.0.1 (FastEthernet1/0), d=10.1.3.1, g=10.24.0.4, len 28, FIB policy routed
*Mar 1 00:33:04.411: IP: s=10.12.0.1 (FastEthernet1/0), d=10.1.3.1, len 28, FIB policy match
*Mar 1 00:33:04.411: IP: s=10.12.0.1 (FastEthernet1/0), d=10.1.3.1, g=10.24.0.4, len 28, FIB policy routed
```
Easy! :)
