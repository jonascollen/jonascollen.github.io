---
title: BGP - Local Preference & RIB-failures
date: 2013-07-23 22:50
comments: true
categories: [BGP, Route Manipulation]
---
Vi fortsätter på temat BGP Path Selection från föregående [posten](http://www.jonascollen.se/posts/bgp-path-selection-part-ii-weight/) där vi kollade närmare på route manipulering via route-maps & weight-attributet. 
![](/assets/images/2013/07/topologylocalpref.jpg)

Local Preference
----------------

*   Is a Path Attribute
*   Purpose - Identifies the best exit point from the AS to reach a given prefix
*   Scope - Throughout the AS in which it was set; not advertised to eBGP peers
*   Range - 0 through 4,294,967,295 (2^32 - 1)
*   Which is best? - Higher value is better
*   Default - 100
*   Changing the default - Using the bgp default local-preference _n_
*   Configuration - Via neighbor route-map; in option is required for updates from an eBGP peer

Efter Weight under BGP's "best path selection"-process så hittar vi Local Preference, som tillskillnad från just Weight är det BGP Path Attribute. Detta betyder att det inte används endast lokalt av den routern vi konfigurerar det på utan att det annonseras vidare till andra. För Local Preference så är det dock **endast till iBGP-neighbors** vi annonserar detta! Detta ger oss möjligheten att inom ett enskilt AS ge oss möjligheten att annonsera till alla iBGP-neighbors en önskad "exit point" för en specifik route vi lärt oss via eBGP. Själva utförandet görs genom en route-map på de inkommande routinguppdateringarna från vår eBGP-peer. Om vi tittar närmare på ovanstående topologi så har vi ett full mesh-nät inom AS500 mellan R1,R2,R3 & R4. BGP table ser för tillfället ut enligt följande för R1: 
[![bgp localpreference](/assets/images/2013/07/bgp-localpreference.png)](/assets/images/2013/07/bgp-localpreference.png) 

Och R2:
[![bgp localpreference2](/assets/images/2013/07/bgp-localpreference2.png)](/assets/images/2013/07/bgp-localpreference2.png)

Du kanske redan har sett något som ser lite konstigt ut? För R2 så har vi default-värdet 100 på Local Preference för samtliga routes, men inte på R1? Vi har även routes markerade med r för R2, men mer om detta senare. Detta beror på att som tidigare nämnt så används Local Preference endast inom iBGP, routes vi lärt oss från eBGP (R1) skickas endast med ett null värde och routern listar därför inget alls för dessa. R2, som inte har någon eBGP-neighbor utan lär sig alla routes via iBGP får dock default-värdet 100 för samtliga. För att testa detta rent praktiskt - låt oss försöka konfigurera så att alla iBGP-peers i AS500 väljer omvägen via AS200 för att nå nätet 192.168.0.0/24, istället för direkt via R4 -> AS50. Vilken router är det då vi ska utföra ändringen i? Om Local Preference justeras via en route-map på inkommande routing-uppdateringar från en eBGP-neighbor, så måste det ju vara i R3 vi ska ändra detta? Vi börjar med att skapa en.... access-lista.

`access-list 1 permit 192.168.0.0`

Vi skapar sedan en route-map:
```
route-map LOCAL-PREF permit 10
match ip address 1
**set local-preference 150**
route-map LOCAL-PREF permit 20
router bgp 500
neighbor 172.16.36.6 route-map LOCAL-PREF in
do clear ip bgp *
```
Om allt fungerar som det ska så skall nu R3 annonsera detta till resterande iBGP-neighbors (R1,R2 & R4) som tack vare den högre Local pref. kommer välja att gå via R3(AS500) -> R6(AS200) -> R7(AS50) för att nå 192.168.0.0/24. 
[![bgp localpref3](/assets/images/2013/07/bgp-localpref3.png)](/assets/images/2013/07/bgp-localpref3.png) 

Sick! :) Vi hade givetvis även kunnat ändra detta i R4 också, men då istället sänkt Local Preference <100 för att få samma effekt.

RIB-Failure
-----------

När BGP är klar med sin "BGP best path"-algoritm och har valt vilken väg som är bäst för en specifikt destination, har jag tidigare skrivit att detta skickas vidare till routerns routing table, detta är dock en sanning med modifikation.. BGP skickar nämligen den bästa routen till en annan process vid namnet "IOS Routing Table Manager" (_RTM_). IOS RTM väljer sedan den bästa routen för en destination mellan flera olika konkurrerande källor. En route kan ju som du bör veta vid det här laget även läras via IGP/BGP/Connected/Static route. IOS samlar den bästa routen från varje process och skickar detta vidare till RTM som sedan i sin tur väljer vilken route som är bäst. För att lyckas med detta används "Administrative Distance" (detta borde du verkligen kunna om du läst CCNA ;)..)

*   Connected - AD 0
*   Static - AD 1
*   EIGRP Summary route - AD 5
*   BGP - AD 20
*   EIGRP (internal) - AD 90
*   IGRP - AD 100
*   OSPF - AD 110
*   ISIS - AD 115
*   RIP - AD 120
*   ODR (On-Demand Routing) - AD 160
*   EIGRP (External) - AD 170
*   iBGP - AD 200
*   Unreachable - AD 255

I de flesta fall borde vi aldrig se en route vi lärt oss via BGP även finns i vår IGP-process eller som Connected (detta blir mer ett problem när vi senare börjar grotta ner oss i MPLS VPNs med BGP/IGP-redistribution). Men hur som helst, OM det skulle hända så finns det ett inbyggt kommando som kan vara bra att känna till - sh ip bgp rib-failures. Om vi går tillbaka och kollar R2's BGP-table igen från föregående exempel: 
[![bgp localpreference2](/assets/images/2013/07/bgp-localpreference2.png)](/assets/images/2013/07/bgp-localpreference2.png) 

En sh ip bgp rib-failures visar följande:
```
R2#sh ip bgp rib-failure 
Network Next Hop RIB-failure RIB-NH Matches
200.0.0.0 1.1.1.1 Higher admin distance n/a
200.0.1.0 1.1.1.1 Higher admin distance n/a
200.0.2.0 1.1.1.1 Higher admin distance n/a
200.0.3.0 1.1.1.1 Higher admin distance n/a
```
Detta listar routes som BGP anser vara bäst, men som RTM valt att ej lägga till i routing table - anledningen är tydlig här, iBGP har en AD på 200, och routern har valt att lägga till routes från en annan "källa" med lägre AD. show ip route visar boven:
```
R2#sh ip route | i O E2 
O E2 200.0.0.0/24 [110/1] via 172.16.12.1, 00:03:47, Serial0/0
O E2 200.0.1.0/24 [110/1] via 172.16.12.1, 00:03:47, Serial0/0
O E2 200.0.2.0/24 [110/1] via 172.16.12.1, 00:03:47, Serial0/0
O E2 200.0.3.0/24 [110/1] via 172.16.12.1, 00:03:47, Serial0/0
```
RTM har fått information om samma routes från OSPF-processen med en AD på 110 och därför valt detta istället. Detta orsakades på grund av ett konfigurationsfel i R1 där vi redistributade klassfulla BGP-nät in till OSPF-processen, efter att ha tagit bort detta försvann även rib-failure:
```
R2#sh ip bgp 
BGP table version is 86, local router ID is 2.2.2.2
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal,
 r RIB-failure, S Stale
Origin codes: i - IGP, e - EGP, ? - incomplete
Network Next Hop Metric LocPrf Weight Path
*>i5.5.5.0/24 1.1.1.1 0 100 0 100 i
*>i6.6.6.0/24 3.3.3.3 0 100 0 200 i
*>i10.0.0.0/16 1.1.1.1 0 100 0 100 i
*>i192.168.0.0 3.3.3.3 0 150 0 200 50 i
*>i200.0.0.0 1.1.1.1 0 100 0 100 i
*>i200.0.1.0 1.1.1.1 0 100 0 100 i
*>i200.0.2.0 1.1.1.1 0 100 0 100 i
*>i200.0.3.0 1.1.1.1 0 100 0 100 i
```
Vackert!
