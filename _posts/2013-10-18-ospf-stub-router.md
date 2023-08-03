---
title: OSPF - Stub router
date: 2013-10-18 11:43
comments: true
categories: [OSPF]
tags: [stub router, lsa]
---
Cisco's IOS erbjuder en funktion som kallas "Stub router advertisements" vilket inte nämns i CCNP-materialet så tänkte ta och skriva ett kortare inlägg om det här istället. 

![OSPF-stubrouter](/assets/images/2013/10/ospf-stubrouter.png) 

I ovanstående topologi så går trafiken just nu via R3 för att nå 200.0.0.0/24-nätet.
```
R1#sh ip route 200.0.0.0
Routing entry for 200.0.0.0/16, supernet
 Known via "ospf 1", distance 110, metric 21, type intra area
 Last update from 10.0.13.3 on FastEthernet0/1, 00:00:08 ago
 Routing Descriptor Blocks:
 \* 10.0.13.3, from 4.4.4.4, 00:00:08 ago, via FastEthernet0/1
 Route metric is 21, traffic share count is 1
```
Låt oss nu säga att ex. Cisco har släppt en kritisk IOS-uppdatering vilket kräver att vi startar om R3. Men för att det ej ska påverka trafiken i vårat nät behöver vi först styra om trafiken. I det här lilla nätet hade det ju varit rätt enkelt att gå in och modifiera cost-värden på interfacen, men om vi nu hade haft ett stort nät med massa länkar introducerade Cisco istället en funktion som kallas "Stub router advertisements". Detta gör att vår router automatiskt börjar annonsera alla sina nät med max-metric (65,535).

```
R3(config)#router ospf 1
R3(config-router)#max-metric router-lsa

R3#sh ip ospf
 Routing Process "ospf 1" with ID 3.3.3.3
 Start time: 00:03:49.344, Time elapsed: 00:20:52.136
 Supports only single TOS(TOS0) routes
 Supports opaque LSA
 Supports Link-local Signaling (LLS)
 Supports area transit capability
 **Originating router-LSAs with maximum metric**
 **Condition: always, State: active**
```

Kollar vi återigen i R1 kan vi nu se att trafiken går via R2 istället:

```
R1#sh ip route 200.0.0.0
Routing entry for 200.0.0.0/16, supernet
 Known via "ospf 1", distance 110, metric 41, type intra area
 Last update from 10.0.12.2 on FastEthernet0/0, 00:00:58 ago
 Routing Descriptor Blocks:
 \* 10.0.12.2, from 4.4.4.4, 00:00:58 ago, via FastEthernet0/0
 Route metric is 41, traffic share count is 1

R1#sh ip ospf database router 3.3.3.3
OSPF Router with ID (1.1.1.1) (Process ID 1)
Router Link States (Area 0)
LS age: 175
 Options: (No TOS-capability, DC)
 LS Type: Router Links
 Link State ID: 3.3.3.3
 **Advertising Router: 3.3.3.3**
 LS Seq Number: 80000007
 Checksum: 0xB7BC
 Length: 48
 Number of Links: 2

Link connected to: a Transit Network
 (Link ID) Designated Router address: 10.0.34.3
 (Link Data) Router Interface address: 10.0.34.3
 Number of TOS metrics: 0
 **TOS 0 Metrics: 65535**

Link connected to: a Transit Network
 (Link ID) Designated Router address: 10.0.13.3
 (Link Data) Router Interface address: 10.0.13.3
 Number of TOS metrics: 0
 **TOS 0 Metrics: 65535**
```

Detta kan även användas för att sätta en delay vid uppstart så att routern inte börjar annonsera routes direkt OSPF-processen startat upp och istället ge routern lite tid att först konvergera fullt ut.

```
R1(config-router)#max-metric router-lsa on-startup ?
 <5-86400> Time, in seconds, router-LSAs are originated with max-metric
 wait-for-bgp Let BGP decide when to originate router-LSA with normal metric
```

Vi kan som synes antingen sätta en egen timer, ex 120 sekunder, alternativt om denna router även är en BGP-speaker, avvakta tills BGP meddelar att routern är fullt konvergerad (max 10minuter).