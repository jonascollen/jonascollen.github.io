---
title: BGP - Advanced BGP Lab, del 1
date: 2013-11-28 21:06
comments: true
categories: [BGP]
---
Var ett tag sedan jag konfigurerade något nu så fick lite abstinens... ;) Tänkte göra ett försök på Renée's "[Advanced BGP-Lab](http://gns3vault.com/BGP/bgp-advanced.html)" ikväll från www.gns3vault.com. Topologin ser ut enligt följande: 
[![bgp-advanced-lab](/assets/images/2013/11/bgp-advanced-lab.png)](/assets/images/2013/11/bgp-advanced-lab.png)

 _R1 - R2: 192.168.12.X /_ _R1 - R3: 192.168.13.X /_ _R3 - R4: 192.168.34.X_ _(where X = Router number) etc.._ _Every router has a Loopback0 interface: X.X.X.X (where X = Router number)_

*   Configure each Autonomous System (AS) with a different IGP: AS100: [RIP](http://gns3vault.com/Labs/RIP/) AS300: [OSPF](http://gns3vault.com/Labs/OSPF/) AS200: [EIGRP](http://gns3vault.com/Labs/EIGRP/) AS400: OSPF
*   Do not configure the IGP on the interfaces connecting to another AS. For example; R3 should not send any RIP routing updates towards R4.
*   Make sure the loopbacks are advertised in the IGP's.
*   Configure BGP on every router, make sure you have the right [IBGP](http://gns3vault.com/BGP/ibgp-internal-bgp.html) and [EBGP](http://gns3vault.com/BGP/ebgp-external-bgp.html) configurations. AS300 has to be configured as a confederation.
*   R1 has to be configured as a [route-reflector](http://gns3vault.com/BGP/bgp-route-reflectors.html) for R2 and R3.
*   Configure on all routers that BGP updates are sourced from the Loopback0 interface.
*   Configure BGP authentication between R7 and R11, use password VAULT
*   Make sure all BGP neighbor relationships are working before you continue with the next steps.
*   Advertise all physical and loopback interfaces in BGP, you are not allowed to use the "network" command to achieve this.
*   Achieve full connectivity, every IP address should be pingable. Use a [TCLSH](http://gns3vault.com/Network-Services/tclsh-scripting.html) script to do this.
*   When R4 sends a ping to the loopback interface of R1 it should choose the path through R2. You are only allowed to make changes on R3.
*   Create another loopback interface on R1 with ip address 172.16.1.1 /24, advertise this in RIP.
*   When R4 sends a ping to the 172.16.1.1 address it should take the path through R3, you are only allowed to make changes on R4.
*   When R6 sends a ping towards the loopback interface on R11 it should go through AS300.
*   R7 should prefer the path through R11 for all external networks except for 172.16.1.1.
*   Configure AS300 so it is no longer a transit AS for AS200 to reach 172.16.1.1 in AS100. AS400 should not be influenced.

Får se hur långt vi hinner idag, kommer troligtvis att bli (minst) två inlägg istället. Men låt oss börja med basics, sätta upp IGPs inom respektive AS.

### IGP Routing

*   Configure each Autonomous System (AS) with a different IGP: AS100: [RIP](http://gns3vault.com/Labs/RIP/) AS300: [OSPF](http://gns3vault.com/Labs/OSPF/) AS200: [EIGRP](http://gns3vault.com/Labs/EIGRP/) AS400: OSPF
*   Do not configure the IGP on the interfaces connecting to another AS. For example; R3 should not send any RIP routing updates towards R4.
*   Make sure the loopbacks are advertised in the IGP's.

Finns inte direkt något speciellt att tänka på här. Använd passive-interface default enligt best practice och kom ihåg att annonsera Loopback för respektive router. 

# RIP

**R1**
```
router rip
 version 2
 passive-interface default
 no passive-interface FastEthernet0/0
 no passive-interface FastEthernet1/0
 network 1.0.0.0
 network 192.168.12.0
 network 192.168.13.0
 no auto-summary
```
**R2**
```
 router rip
 version 2
 passive-interface default
 no passive-interface FastEthernet0/0
 network 2.0.0.0
 network 192.168.12.0
 no auto-summary
```
**R3**
```
 router rip
 version 2
 passive-interface default
 no passive-interface FastEthernet0/0
 network 3.0.0.0
 network 192.168.13.0
 no auto-summary
```

# OSPF / AS300

**R4**
```
 router ospf 300
 router-id 4.4.4.4
 log-adjacency-changes
 passive-interface default
 no passive-interface FastEthernet2/0
 network 4.4.4.0 0.0.0.255 area 0
 network 192.168.45.0 0.0.0.255 area 0

interface Loopback0
 ip ospf network point-to-point
```
 **R5**
 ```
router ospf 300
 router-id 5.5.5.5
 log-adjacency-changes
 passive-interface default
 no passive-interface FastEthernet0/0
 no passive-interface FastEthernet1/0
 network 5.5.5.0 0.0.0.255 area 0
 network 192.168.45.0 0.0.0.255 area 0
 network 192.168.58.0 0.0.0.255 area 0

 interface Loopback0
 ip ospf network point-to-point
```

 **R8**
 ```
 router ospf 300
 router-id 8.8.8.8
 log-adjacency-changes
 passive-interface default
 no passive-interface FastEthernet0/0
 no passive-interface FastEthernet1/0
 network 8.8.8.0 0.0.0.255 area 0
 network 192.168.58.0 0.0.0.255 area 0
 network 192.168.89.0 0.0.0.255 area 0

 interface Loopback0
 ip ospf network point-to-point
```

**R9**
```
 router ospf 300
 router-id 9.9.9.9
 log-adjacency-changes
 passive-interface default
 no passive-interface FastEthernet1/0
 network 9.9.9.0 0.0.0.255 area 0
 network 192.168.89.0 0.0.0.255 area 0

interface Loopback0
 ip ospf network point-to-point
```

Anledningen till att jag lagt till "ospf network point-to-point" är för att OSPF per default annars sätter network-type till stub för Loopbacks och endast annonserar våra nät som en /32. 

# EIGRP / AS200

**R6**
```
 router eigrp 200
 passive-interface default
 no passive-interface FastEthernet0/0
 network 7.7.7.0 0.0.0.255
 network 192.168.67.0
 no auto-summary
```
**R7**
```
 router eigrp 200
 passive-interface default
 no passive-interface FastEthernet1/0
 network 6.6.6.0 0.0.0.255
 network 192.168.67.0
 no auto-summary
```

# OSPF / AS400

**R10**
```
router ospf 400
 router-id 10.10.10.10
 log-adjacency-changes
 passive-interface default
 no passive-interface FastEthernet0/0
 network 10.10.10.0 0.0.0.255 area 0
 network 192.168.110.0 0.0.0.255 area 0

interface Loopback0
 ip ospf network point-to-point
```

**R11**
```
router ospf 400
 router-id 11.11.11.11
 log-adjacency-changes
 passive-interface default
 no passive-interface FastEthernet1/0
 network 11.11.11.0 0.0.0.255 area 0
 network 192.168.110.0 0.0.0.255 area 0

interface Loopback0
 ip ospf network point-to-point
```

# BGP
---

### IBGP & Route-reflector

*   Configure BGP on every router
*   Configure on all routers that BGP updates are sourced from the Loopback0 interface.
*   R1 has to be configured as a [route-reflector](http://gns3vault.com/BGP/bgp-route-reflectors.html) for R2 and R3.

Allt är relativt standard, kom ihåg att ändra update-source när vi använder Loopback-adresser för peering.

**R1 - Kom ihåg route-reflector mot R2 & R3!**

```
router bgp 100
 no synchronization
 bgp log-neighbor-changes
 neighbor 2.2.2.2 remote-as 100
 neighbor 2.2.2.2 update-source Loopback0
 **neighbor 2.2.2.2 route-reflector-client**
 neighbor 3.3.3.3 remote-as 100
 neighbor 3.3.3.3 update-source Loopback0
 **neighbor 3.3.3.3 route-reflector-client**
 no auto-summary
```
**R2**
```
router bgp 100
 no synchronization
 bgp log-neighbor-changes
 neighbor 1.1.1.1 remote-as 100
 neighbor 1.1.1.1 update-source Loopback0
 no auto-summary
```
**R3**
```
router bgp 100
 no synchronization
 bgp log-neighbor-changes
 neighbor 1.1.1.1 remote-as 100
 neighbor 1.1.1.1 update-source Loopback0
 no auto-summary
```

Blir lite tjatigt att ta med i princip identisk konfig för R6, R7, R10 & R11 så hoppar över det. R4, R5, R8 & R9 tar vi nästa punkt då det ska konfigureras confederations istället.

# Confederations

*   AS300 has to be configured as a confederation.

Vi tar och börjar med att sätta upp två confederations inom AS300, Sub-AS 10 & 20. Om du vill läsa mer om confederations har jag ett gammalt inlägg som går igenom detta relativt grundligt [BGP – Confederations & Route Reflectors](http://www.jonascollen.se/posts/bgp-confederations-route-reflectors/).  Vanligtvis använder man privata AS-nummer för detta men tar och följer labben slaviskt nu.. :) 

**R4**
```
router bgp 10
 no synchronization
 bgp log-neighbor-changes
 bgp confederation identifier 300
 bgp confederation peers 20
 neighbor 5.5.5.5 remote-as 10
 neighbor 5.5.5.5 update-source Loopback0
 no auto-summary
```
**R5**
```
router bgp 10
 no synchronization
 bgp log-neighbor-changes
 bgp confederation identifier 300
 bgp confederation peers 20
 neighbor 4.4.4.4 remote-as 10
 neighbor 4.4.4.4 update-source Loopback0
 neighbor 8.8.8.8 remote-as 20
 neighbor 8.8.8.8 update-source Loopback0
 no auto-summary
```
**R8**
```
router bgp 20
 no synchronization
 bgp log-neighbor-changes
 bgp confederation identifier 300
 bgp confederation peers 10
 neighbor 5.5.5.5 remote-as 10
 neighbor 5.5.5.5 update-source Loopback0
 neighbor 9.9.9.9 remote-as 20
 neighbor 9.9.9.9 update-source Loopback0
 no auto-summary
```
**R9**
```
router bgp 20
 no synchronization
 bgp log-neighbor-changes
 bgp confederation identifier 300
 bgp confederation peers 10
 neighbor 8.8.8.8 remote-as 20
 neighbor 8.8.8.8 update-source Loopback0
 no auto-summary
```

Det var all konfig.. Men tyvärr fungerade det inte riktigt som önskat. IBGP kom upp utan problem men av någon anledning får jag ej upp någon neighbor mellan R5 <-> R8.

```
R5#sh ip bgp
 *Mar 1 06:30:59.486: %SYS-5-CONFIG_I: Configured from console by consolesummary
 BGP router identifier 5.5.5.5, local AS number 10
 BGP table version is 1, main routing table version 1
Neighbor V AS MsgRcvd MsgSent TblVer InQ OutQ Up/Down State/PfxRcd
 4.4.4.4 4 10 27 27 1 0 0 00:23:08 0
 **8.8.8.8 4 20 0 0 0 0 0 never Idle**
```

En debug ip bgp events gav inte heller någon vettig information. Tillslut kom jag äntligen på vad felet var: ebgp-multihop! TTL för EBGP är ju per default satt till 1, så när paketet när R5/R8 kastas paketet istället för att vidarebefordras till Loopback. Detta var inte direkt första gången jag åkt dit på den grejjen.. :)

```
R5
 neighbor 8.8.8.8 ebgp-multihop 2
 R8
 neighbor 5.5.5.5 ebgp-multihop 2
```

Nu ser det bättre ut!
```
R8#sh ip bgp summary
 BGP router identifier 8.8.8.8, local AS number 20
 BGP table version is 1, main routing table version 1
Neighbor V AS MsgRcvd MsgSent TblVer InQ OutQ Up/Down State/PfxRcd
 5.5.5.5 4 10 4 4 1 0 0 00:00:29 0
 9.9.9.9 4 20 15 15 1 0 0 00:11:38 0
```

Detta får räcka för idag, känns som inlägget blir lite väl långt annars. Fortsättning följer!