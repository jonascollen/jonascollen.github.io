---
title: Redistribution - Expert Lab
date: 2013-07-06 15:56
comments: true
categories: [EIGRP,Route Manipulation]
---
Tänkte ta mig an labben jag nämnde i [föregående inlägg](http://www.jonascollen.se/posts/redistribution-labs/) från Petr Lapukhov. 

Topologin är följande: 
[![expert redistribution](/assets/images/2013/07/expert-redistribution.png)](/assets/images/2013/07/expert-redistribution.png) 

EIGRP AS 123 has been preconfigured on router BabyDoll,SweetPea and Rocket.

*   OSPF Area 0 (Process 234) has been preconfigured on router SweetPea, Rocket and Blondie.
*   EIGRP AS 356 has been preconfigured on router Rocket, BlueJones and Amber.
*   RIP Version 2 has been preconfigured on router Blondie, Amber and WiseMan.
*   Router BabyDoll's loopback0 interface is advertised in EIGRP AS123.
*   Router SweetPea's, Blondie's and Rocket's loopback0 interfaces are advertised in OSPF Area 0.
*   Router Amber's and BlueJones's loopback0 interfaces are advertised in EIGRP AS356.
*   Router WiseMan's loopback0 interface is advertised in RIPV2.
*   Configure 2-way redistribution on router SweetPea and Rocket between EIGRP AS123 and OSPF.
*   Configure redistribution on router Rocket between EIGRP AS356 and OSPF.
*   Configure redistribution on router Blondie between RIPV2 and OSPF.
*   Ensure you have full connectivity at this moment, every network / loopback should be reachable from any device.

Själva konfigureringen gick smärtfritt och borde inte ställa till några problem vid det här laget. Troubleshooting:

1.  Remove the "network 1.0.0.0" command on router BabDoll and replace it by redistributing the loopback0 interface into EIGRP AS123.
2.  Do a traceroute from router SweetPea and Rocket to 1.1.1.1, you notice packets are sent through the OSPF network. Ensure they take the most optimal route to the destination.
3.  Ensure all traffic will be sent using the FastEthernet links. The Frame-Relay links should only be used as a backup when the FastEthernet links are down.

Steg 1 & 2
----------

Easy stuff, konfade följande:
```
BabyDoll(config-router)#no network 1.0.0.0
BabyDoll(config-router)#redistribute connected
```
En show ip route från SweetPea & Rocket visar dock följande: SweetPea

```
Gateway of last resort is not set
C 192.168.123.0/24 is directly connected, FastEthernet0/0
 **1.0.0.0/24 is subnetted, 1 subnets**
**D EX 1.1.1.0 [170/156160] via 192.168.123.1, 00:02:08, FastEthernet0/0**
 2.0.0.0/24 is subnetted, 1 subnets
C 2.2.2.0 is directly connected, Loopback0
 3.0.0.0/24 is subnetted, 1 subnets
O 3.3.3.0 [110/65] via 192.168.234.3, 00:31:32, Serial1/0
O E2 192.168.45.0/24 [110/20] via 192.168.234.4, 00:31:32, Serial1/0
 4.0.0.0/24 is subnetted, 1 subnets
O 4.4.4.0 [110/65] via 192.168.234.4, 00:31:33, Serial1/0
 5.0.0.0/24 is subnetted, 1 subnets
O E2 5.5.5.0 [110/20] via 192.168.234.3, 00:31:33, Serial1/0
 6.0.0.0/24 is subnetted, 1 subnets
O E2 6.6.6.0 [110/20] via 192.168.234.3, 00:31:37, Serial1/0
 7.0.0.0/24 is subnetted, 1 subnets
O E2 7.7.7.0 [110/20] via 192.168.234.4, 00:31:38, Serial1/0
C 192.168.234.0/24 is directly connected, Serial1/0
O E2 192.168.35.0/24 [110/20] via 192.168.234.3, 00:31:38, Serial1/0
```
Rocket
```
Gateway of last resort is not set
C 192.168.123.0/24 is directly connected, FastEthernet0/0
 **1.0.0.0/24 is subnetted, 1 subnets**
**O E2 1.1.1.0 [110/20] via 192.168.234.2, 00:14:50, Serial2/0**
 2.0.0.0/24 is subnetted, 1 subnets
O 2.2.2.0 [110/65] via 192.168.234.2, 00:44:09, Serial2/0
 3.0.0.0/24 is subnetted, 1 subnets
C 3.3.3.0 is directly connected, Loopback0
O E2 192.168.45.0/24 [110/20] via 192.168.234.4, 00:44:09, Serial2/0
 4.0.0.0/24 is subnetted, 1 subnets
O 4.4.4.0 [110/65] via 192.168.234.4, 00:44:10, Serial2/0
 5.0.0.0/24 is subnetted, 1 subnets
D 5.5.5.0 [90/156160] via 192.168.35.5, 00:54:56, FastEthernet1/0
 6.0.0.0/24 is subnetted, 1 subnets
D 6.6.6.0 [90/156160] via 192.168.35.6, 00:54:57, FastEthernet1/0
 7.0.0.0/24 is subnetted, 1 subnets
O E2 7.7.7.0 [110/20] via 192.168.234.4, 00:44:11, Serial2/0
C 192.168.234.0/24 is directly connected, Serial2/0
C 192.168.35.0/24 is directly connected, FastEthernet1/0
```
Så Sweetpea tar för tillfället rätt väg direkt till R1, men Rocket går via OSPF/Frame-Relay switchen -> R2 -> R1 - **suboptimal routing**! 
Varför det blir såhär har vi gått igenom många gånger tidigare, OSPF har lägre AD än vad EIGRP har, så Rocket ersätter sin egna route med den SweetPea annonserar.  Vi testar med en enkel lösning först och höjer AD för OSPFs externa routes.
```
router ospf 234
distance ospf external 180
```
Vi konfar detta på både SweetPea & Rocket, de använder nu routen som annonseras av EIGRP istället och går direkt till R1.
```
1.0.0.0/24 is subnetted, 1 subnets
D EX 1.1.1.0 [170/156160] via 192.168.123.1, 00:03:58, FastEthernet0/0
```
Detta leder dock till rätt konstiga problem för andra nät! Om vi kör en sh ip route på SweetPea visas följande:
```
Gateway of last resort is not set
C 192.168.123.0/24 is directly connected, FastEthernet0/0
 1.0.0.0/24 is subnetted, 1 subnets
D EX 1.1.1.0 [170/156160] via 192.168.123.1, 00:07:03, FastEthernet0/0
 2.0.0.0/24 is subnetted, 1 subnets
C 2.2.2.0 is directly connected, Loopback0
 3.0.0.0/24 is subnetted, 1 subnets
O 3.3.3.0 [110/65] via 192.168.234.3, 00:07:03, Serial1/0
**D EX 192.168.45.0/24** 
 **[170/1709312] via 192.168.123.3, 00:07:01, FastEthernet0/0**
 **4.0.0.0/24 is subnetted, 1 subnets**
O 4.4.4.0 [110/65] via 192.168.234.4, 00:07:04, Serial1/0
 5.0.0.0/24 is subnetted, 1 subnets
**O E2 5.5.5.0 [180/20] via 192.168.234.3, 00:07:04, Serial1/0**
 **6.0.0.0/24 is subnetted, 1 subnets**
**O E2 6.6.6.0 [180/20] via 192.168.234.3, 00:07:05, Serial1/0**
 **7.0.0.0/24 is subnetted, 1 subnets**
**O E2 7.7.7.0 [180/20] via 192.168.234.3, 00:06:44, Serial1/0**
**C 192.168.234.0/24 is directly connected, Serial1/0**
**O E2 192.168.35.0/24 [180/20] via 192.168.234.3, 00:07:05, Serial1/0**
```
SweetPea tror nu att den ska gå via Rocket för att nå exempelvis 192.168.45.0. En traceroute visar dock följande:
```
SweetPea#traceroute 7.7.7.7
Type escape sequence to abort.
Tracing the route to 7.7.7.7
1 192.168.234.3 24 msec 24 msec 20 msec
 2 * * * 
 3 192.168.234.3 60 msec 60 msec 36 msec
 4 * * * 
 5 192.168.234.3 88 msec 124 msec 52 msec
 6 * * * 
 7 192.168.234.3 112 msec 88 msec 116 msec
 8 * * *
```
Kollar vi Rocket så ser vi varför det loopas:
```
7.0.0.0/24 is subnetted, 1 subnets
D EX 7.7.7.0 [170/1709312] via 192.168.123.2, 00:05:42, FastEthernet0/0
```
[![expert redistribution 2](/assets/images/2013/07/expert-redistribution-2.png)](/assets/images/2013/07/expert-redistribution-2.png) 

SweetPea skickar till Rocket som i sin tur skickar tillbaka till SweetPea. Här körde jag fast rätt rejält och tog istället och läste Petr Lapukhovs blogg om detta, se [http://blog.ine.com/2008/02/09/understanding-redistribution-part-i/](http://blog.ine.com/2008/02/09/understanding-redistribution-part-i/). Där anger han bl.a. två regler vi alltid ska utgå ifrån: 1 - Router should always prefer internal prefix information over any external information. 2 - Split-Horizon – Never redistribute a prefix injected from domain A into B back to domain A Han menar också att vi ska tänka på att OSPF endast är en transit-area/core, och bör således ha bäst information generellt sätt. Vi bör därför se till att den föredrar OSPFs uppdateringar istället för EIGRPs som endast används "ute i kanterna". OSPF's AD för interna routes är 110 som standard, EIGRP har 90. Vi behöver således sänka AD't till <90:
```
router ospf 234
distance ospf intra-area 80 
distance ospf inter-area 80 
distance ospf external 160
```
Vi behöver ju dock få R2 & R3 att hellre använda EIGRP för 1.1.1.0/24-nätet (R1's loopback), detta kan vi lösa genom en access-lista istället:
```
access-list 1 permit 1.1.1.0
router ospf 234
distance 171 0.0.0.0 255.255.255.255 1
```
Detta ändrar distance för den specifika routern till 171, R2 & R3 kommer således föredra EIGRP External som har 170 per default. Kollar vi återigen sh ip route så ser det nu mycket bättre ut: SweetPea
```
Gateway of last resort is not set
C 192.168.123.0/24 is directly connected, FastEthernet0/0
 1.0.0.0/24 is subnetted, 1 subnets
D EX 1.1.1.0 [170/156160] via 192.168.123.1, 00:03:30, FastEthernet0/0
 2.0.0.0/24 is subnetted, 1 subnets
C 2.2.2.0 is directly connected, Loopback0
 3.0.0.0/24 is subnetted, 1 subnets
O 3.3.3.0 [80/65] via 192.168.234.3, 00:03:30, Serial1/0
O E2 192.168.45.0/24 [160/20] via 192.168.234.4, 00:03:30, Serial1/0
 4.0.0.0/24 is subnetted, 1 subnets
O 4.4.4.0 [80/65] via 192.168.234.4, 00:03:30, Serial1/0
 5.0.0.0/24 is subnetted, 1 subnets
O E2 5.5.5.0 [160/20] via 192.168.234.3, 00:03:30, Serial1/0
 6.0.0.0/24 is subnetted, 1 subnets
O E2 6.6.6.0 [160/20] via 192.168.234.3, 00:03:30, Serial1/0
 7.0.0.0/24 is subnetted, 1 subnets
O E2 7.7.7.0 [160/20] via 192.168.234.4, 00:03:30, Serial1/0
C 192.168.234.0/24 is directly connected, Serial1/0
O E2 192.168.35.0/24 [160/20] via 192.168.234.3, 00:03:30, Serial1/0
```
Rocket
```
Gateway of last resort is not set
C 192.168.123.0/24 is directly connected, FastEthernet0/0
 1.0.0.0/24 is subnetted, 1 subnets
D EX 1.1.1.0 [170/156160] via 192.168.123.1, 00:03:41, FastEthernet0/0
 2.0.0.0/24 is subnetted, 1 subnets
O 2.2.2.0 [80/65] via 192.168.234.2, 00:03:41, Serial2/0
 3.0.0.0/24 is subnetted, 1 subnets
C 3.3.3.0 is directly connected, Loopback0
O E2 192.168.45.0/24 [160/20] via 192.168.234.4, 00:03:41, Serial2/0
 4.0.0.0/24 is subnetted, 1 subnets
O 4.4.4.0 [80/65] via 192.168.234.4, 00:03:41, Serial2/0
 5.0.0.0/24 is subnetted, 1 subnets
D 5.5.5.0 [90/156160] via 192.168.35.5, 00:03:41, FastEthernet1/0
 6.0.0.0/24 is subnetted, 1 subnets
D 6.6.6.0 [90/156160] via 192.168.35.6, 00:03:41, FastEthernet1/0
 7.0.0.0/24 is subnetted, 1 subnets
O E2 7.7.7.0 [160/20] via 192.168.234.4, 00:03:41, Serial2/0
C 192.168.234.0/24 is directly connected, Serial2/0
C 192.168.35.0/24 is directly connected, FastEthernet1/0
```
Vi har inte längre några routing loopar eller suboptimal routing för 1.1.1.0/24-nätet.  En traceroute till 7.7.7.7 borde således fungera nu:
```
SweetPea#traceroute 7.7.7.7
Type escape sequence to abort.
Tracing the route to 7.7.7.7
1 192.168.234.4 48 msec 32 msec 20 msec
 2 192.168.45.7 64 msec * 24 msec
Rocket#traceroute 7.7.7.7
Type escape sequence to abort.
Tracing the route to 7.7.7.7
1 192.168.234.4 36 msec 24 msec 8 msec
 2 192.168.45.7 16 msec
```
Stabilt! Vi ser rätt tydligt vad farligt det kan vara att gå in och ändra AD som gäller generellt för hela routern, utan detta bör göras specifikt för det önskade nätet endast för att undvika märkliga problem.

Steg 3
------

> Ensure all traffic will be sent using the FastEthernet links. The [Frame-Relay](http://gns3vault.com/administrator/index.php?option=com_sitelinkx&controller=sitelinkxlist&task=edit&cid[]=27) links should only be used as a backup when the FastEthernet links are down.

Som det är i nuläget använder vi alltid OSPF som transit-area för att ta mellan näten, med ett undantag - R6 <-> R1. En sh ip route på BlueJones(R6) visar följande:
```
Gateway of last resort is not set
2.0.0.0/24 is subnetted, 1 subnets
D EX 2.2.2.0 [170/1709312] via 192.168.35.3, 00:13:15, FastEthernet0/0
 3.0.0.0/24 is subnetted, 1 subnets
D EX 3.3.3.0 [170/1709312] via 192.168.35.3, 02:03:27, FastEthernet0/0
D EX 192.168.45.0/24 [170/1709312] via 192.168.35.3, 00:13:15, FastEthernet0/0
 4.0.0.0/24 is subnetted, 1 subnets
D EX 4.4.4.0 [170/1709312] via 192.168.35.3, 00:13:15, FastEthernet0/0
 5.0.0.0/24 is subnetted, 1 subnets
D 5.5.5.0 [90/156160] via 192.168.35.5, 02:12:26, FastEthernet0/0
 6.0.0.0/24 is subnetted, 1 subnets
C 6.6.6.0 is directly connected, Loopback0
 7.0.0.0/24 is subnetted, 1 subnets
D EX 7.7.7.0 [170/1709312] via 192.168.35.3, 00:13:16, FastEthernet0/0
D EX 192.168.234.0/24 
 [170/1709312] via 192.168.35.3, 02:03:29, FastEthernet0/0
C 192.168.35.0/24 is directly connected, FastEthernet0/0
```
Den ser helt enkelt inte 1.1.1.0/24-nätet. Alla andra routrar känner dock till nätet, R2 & R3 har lärt sig via EIGRP, R4  via OSPF från R1&R2, R5& R7 via RIP från R4, Det bör ju således vara R3 eller R4's jobb att informera R6, problemet är dock att vi endast redistributar mellan OSPF AS 234 & EIGRP 356 i Rocket. Vi löser detta genom att göra en two-way dist. mellan AS 123 & 356.
```
router eigrp 123
redistribute eigrp 356 metric 1500 1 255 1 1500
router eigrp 356
redistribute eigrp 123 metric 1500 1 255 1 1500
BlueJones#sh ip route 1.1.1.0
Routing entry for 1.1.1.0/24
 Known via "eigrp 356", distance 170, metric 1709312, type external
 Redistributing via eigrp 356
 Last update from 192.168.35.3 on FastEthernet0/0, 00:01:15 ago
 Routing Descriptor Blocks:
 * 192.168.35.3, from 192.168.35.3, 00:01:15 ago, via FastEthernet0/0
 Route metric is 1709312, traffic share count is 1
 Total delay is 110 microseconds, minimum bandwidth is 1500 Kbit
 Reliability 255/255, minimum MTU 1500 bytes
 Loading 1/255, Hops 1
```
Sweet!

Advanced
--------

Vi har nu fullt flöde genom nätet, vi har dock fortfarande kvar uppgiften att flytta "transit-arean" från OSPF till EIGRP 356. och endast använda OSPF som backup. Det är nu vi kommer till CCIE-nivån av redistribution-tekniker  här kan jag ärligt säga att jag inte lyckades hitta någon lösning som fungerade. Petr har som tur var dock gjort en bloggpost om hur olika "redistribueringsproblem" kan lösas och finns att hitta [här](http://blog.ine.com/2008/03/17/understanding-redistribution-part-iii/). Kommer använda hans post som guide för att ha en chans att slutföra detta, med förhoppning att kanske om ett år så fixar jag detta på egen hand! :) Det vi vill göra är att ge route's som kommer från EIGRP356 mer "önskvärda" än de vi lär oss från OSPF & RIP. Routrarna vi behöver modifiera är därmed "border routers" (R2,R3, R4 & R5).

> The OSPF and EIGRP 356 domains are both transit. That means, when redistributing from the OSPF domain to EIGRP 356 we should accept the OSPF native prefixes (per the ACLs configured above) in addition to the transit RIP and EIGRP 123 prefixes. The latter prefixes could also be injected into EIGRP 356 natively, from the respective routing domains. Therefore, when accepting the transit routes from the OSPF domain, we should give them the metric value, which makes the prefixes less preferable. For out example, we will use EIGRP metric “1 100 1 1 1” for the “non-native” injected prefixes, which is less preferable than our default “1 1 1 1 1” due to the higher “delay” component value

Vi behöver först skapa access-listor som delar upp de olika näten efter vilken "domain" de kommer ifrån.
```
ip access-list standard RIP_PREFIXES
 permit 192.168.45.0
 permit 7.7.7.0
ip access-list standard OSPF_PREFIXES
 permit 192.168.234.0
 permit 2.2.2.0
 permit 3.3.3.0
 permit 4.4.4.0
ip access-list standard EIGRP_123_PREFIXES
 permit 192.168.123.0
 permit 1.1.1.0
ip access-list standard EIGRP_356_PREFIXES
 permit 192.168.35.0/24
 permit 6.6.6.0
 permit 5.5.5.0
 permit 3.3.3.0
```
### R3

Vi tar sedan och skapar route-maps, vi börjar med R3: För OSPF till EIGRP 356:
```
route-map OSPF_TO_EIGRP_356 permit 10
 match ip address OSPF_PREFIXES
 set metric 1 1 1 1 1
!
route-map OSPF_TO_EIGRP_356 permit 20
 match ip address EIGRP_123_PREFIXES
 set metric 1 100 1 1 1
!
route-map OSPF_TO_EIGRP_356 permit 30
 match ip address RIP_PREFIXES
 set metric 1 100 1 1 1
```
Vi ger med andra ord ett högre metric för RIP & OSPF-näten när de går via OSPF-domänen. genom att modifera K2-värdet till 100. Vi måste ju dock även göra det åt andra hållet (EIGRP 356 -> OSPF):
```
route-map EIGRP_356_TO_OSPF permit 10
 match ip address EIGRP_356_PREFIXES
!
route-map EIGRP_356_TO_OSPF permit 20
 match ip address RIP_PREFIXES
 set metric 100
```
Vi behåller en låg metric för 356-näten men vi vill ju få RIP-näten mindre attraktiva. Vi behöver givetvis inte ha med EIGRP AS123 här då vi inte vill redistributa det från 356 -> OSPF. Mellan EIGRP 356 <-> 123 modifierar vi inte metricen då de både redan använder sig av samma routing-protokoll, vi vill dock ge RIP-prefixen ett sämre metric:
```
route-map EIGRP_123_TO_EIGRP_356 permit 10
 match ip address EIGRP_123_PREFIXES
!
route-map EIGRP_356_TO_EIGRP_123 permit 10
match ip address EIGRP_356_PREFIXES
!
route-map EIGRP_356_TO_EIGRP_123 permit 20
match ip address RIP_PREFIXES
set metric 1 1 1 1 1
```
Tillsist har vi EIGRP 123 <- > OSPF kvar:
```
route-map OSPF_TO_EIGRP_123 permit 10
 match ip address OSPF_PREFIXES
 set metric 1 1 1 1 1
!
route-map OSPF_TO_EIGRP_123 permit 20
 match ip address RIP_PREFIXES
 set metric 1 100 1 1 1
!
route-map EIGRP_123_TO_OSPF permit 10
match ip address EIGRP_123_PREFIXES
```
Vi aktiverar sedan route-mapsen via:
```
router ospf 234
 redistribute eigrp 356 route-map EIGRP_356_TO_OSPF subnets
 redistribute eigrp 123 route-map EIGRP_123_TO_OSPF subnets
!
router eigrp 123
 redistribute ospf 234 route-map OSPF_TO_EIGRP_123
 redistribute eigrp 356 route-map EIGRP_356_TO_EIGRP_123
!
router eigrp 356
 redistribute eigrp 123 route-map EIGRP_123_TO_EIGRP_356
 redistribute ospf 234 route-map OSPF_TO_EIGRP_356
```
One down.. Three to go! :)

### R2

Här finns endast EIGRP 123 & OSPF 234 domänen. För att möjliggöra "backup"-routing via OSPF-domänen om länken till R3 går ner behöver vi även redistributa in EIGRP356 & RIP-routes till EIGRP 123. Värt att tillägga är dock att för RIP-näten så kommer de ju även senare annonseras via R4->R3 -> EIGRP AS123, vi behöver därför sätta ett "bättre" metric när de kommer från OSPF-domänen. För EIGRP123 -> OSPF behöver vi endast redist. de interna näten. Vi återanvänder ACL'en från tidigare men route-mapsen blir enligt följande:
```
route-map OSPF_TO_EIGRP_123 permit 10
 match ip address OSPF_PREFIXES
 set metric 1 1 1 1 1
!
route-map OSPF_TO_EIGRP_123 permit 20
 match ip address RIP_PREFIXES
 set metric 1 100 1 1 1
!
route-map OSPF_TO_EIGRP_123 permit 30
 match ip address EIGRP_356_PREFIXES
 set metric 1 1 1 1 1
!
route-map EIGRP_123_TO_OSPF permit 10
match ip address EIGRP_123_PREFIXES
Inga konstigheter här, vi aktiverar redist. genom:
router eigrp 123
 redistribute ospf 234 route-map OSPF_TO_EIGRP_123
!
router ospf 234
 redistribute eigrp 123 route-map EIGRP_123_TO_OSPF subnets
```
### R4

Här behöver vi endast hantera OSPF <-> RIP. Vi har redan konfigurerat så R3 annonserar RIP-näten till OSPF med en metric på 100, vi kan därför lämna den som default här. För OSPF in till RIP sätter vi en metric på 8  (hopcount), och för "transit-prefixen" EIGRP 123 & EIGRP 356 sätter vi ett högre värde. Vi kan då ge en bättre väg från R5 till dessa nät senare.
```
route-map RIP_TO_OSPF permit 10
 match ip address RIP_PREFIXES 
!
route-map OSPF_TO_RIP permit 10
 match ip address OSPF_PREFIXES
 set metric 8
!
route-map OSPF_TO_RIP permit 20
 match ip address EIGRP_123_PREFIXES
 set metric 12
!
route-map OSPF_TO_RIP permit 30
 match ip address EIGRP_356_PREFIXES
 set metric 12
Och vi aktiverar redist med:
router rip
 redistribute ospf 234 route-map OSPF_TO_RIP
!
router ospf 234
 redistribute rip route-map RIP_TO_OSPF subnets
```
### R5

Last up! Här kommer vi redistributa mellan EIGRP 356 & RIP. Vi behåller vårat standard metric på 8 för de interna RIP-näten samt EIGRP-näten från AS123 & 356 för att göra denna väg mer attraktiv än att gå via OSPF, som vi sätter till 12.
```
route-map RIP_TO_EIGRP_356 permit 10
 match ip address RIP_PREFIXES
 set metric 1 1 1 1 1
!
route-map EIGRP_356_TO_RIP permit 10
 match ip address EIGRP_356_PREFIXES
 set metric 8
!
route-map EIGRP_356_TO_RIP permit 20
 match ip address OSPF_PREFIXES
 set metric 12
!
route-map EIGRP_356_TO_RIP permit 30
 match ip address EIGRP_123_PREFIXES
 set metric 8
```
Tro det eller ej men vi är tyvärr inte klara än! :D Vi går fortfarande över OSPF-domänen för vissa av näten. Men genom att justera AD-värdena för respektive router ska vi nog kunna lösa detta nu! Den uppgiften vi hade från GNS3Vault skiljde sig från den lösningen Petr skrev om så här fanns det ingen hjälp att få, men detta var ju löjligt basic om vi jämför med allt vi gjort tidigare. Konfigen blev följande:
```
R2
router ospf 234
distance ospf external 190
R3
router ospf 234
distance ospf external 190
R4
router ospf 234
distance ospf external 190

R5
router rip
distance 190
```
Jag fick även gå in och modifiera route-mapen på R2 då pga vi manuellt satt metrics så började R1/BabyDoll lastbalansera mellan R2&R3 för att nå EIGRP AS356- & RIP-domänerna.
```
route-map OSPF_TO_EIGRP_123 permit 10
 match ip address OSPF_PREFIXES
 set metric 1 100 1 1 1
!
route-map OSPF_TO_EIGRP_123 permit 20
 match ip address RIP_PREFIXES
 set metric 1 100 1 1 1
!
route-map OSPF_TO_EIGRP_123 permit 30
 match ip address EIGRP_356_PREFIXES
 set metric 1 100 1 1 1
!
route-map EIGRP_123_TO_OSPF permit 10
 match ip address EIGRP_123_PREFIXES
```
### Verifiering

Från R1 - > R7:
```
BabyDoll#traceroute 7.7.7.7
Type escape sequence to abort.
Tracing the route to 7.7.7.7
1 192.168.123.3 52 msec 24 msec 64 msec
 2 192.168.35.5 44 msec 92 msec 20 msec
 3 192.168.45.7 84 msec * 72 msec
```
R2 -> R5:
```
SweetPea#traceroute 5.5.5.5
Type escape sequence to abort.
Tracing the route to 5.5.5.5
1 192.168.123.3 20 msec 56 msec 28 msec
 2 192.168.35.5 44 msec * 20 msec
```
R4 -> R2:
```
Blondie#traceroute 2.2.2.2
Type escape sequence to abort.
Tracing the route to 2.2.2.2
1 192.168.234.2 32 msec * 48 msec
```
*Kom ihåg att vi tillät att trafik med destination direkt till OSPF-domänen ej behövde cirkulera hela nätet. R4 - > R1:
```
Blondie#traceroute 1.1.1.1
Type escape sequence to abort.
Tracing the route to 1.1.1.1
1 192.168.45.5 36 msec 24 msec 20 msec
 2 192.168.35.3 36 msec 20 msec 28 msec
 3 192.168.123.1 28 msec * 60 msec
```
Så vackert att man nästan blir tårögd! ;) Detta avslutar det här inlägget och nu känns det allt som det får räcka med redistribution ett tag framöver, next up - BGP! :) Vill du testa själv så har GNS3Vault en färdig grundtopologi att hämta [här](http://gns3vault.com/Redistribution/expert-redistribution.html).
