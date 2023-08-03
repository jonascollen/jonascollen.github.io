---
title: BGP - Lab, Configuring a Transit AS/ISP
date: 2013-07-26 19:54
comments: true
categories: [BGP]
tags: [transit as]
---
Tänkte göra ett försök att sätta upp Jeremys Transit AS-lab från hans CCIP BGP-serie. Labben är uppbyggd enligt följande: 

![topology transit lab](/assets/images/2013/07/topology-transit-lab.jpg)
![scenario transit lab](/assets/images/2013/07/scenario-transit-lab.png)
![requirements transit lab 1](/assets/images/2013/07/requirements-transit-lab-1.png) 
![requirements transit lab 2](/assets/images/2013/07/requirements-transit-lab-2.png)

Min lösning
-----------

[![gns3topology](/assets/images/2013/07/gns3topology.png)](/assets/images/2013/07/gns3topology.png)

### Grundkonfig

```
!ISP1 Basic conf
inte s0/0
ip add 17.9.1.1 255.255.255.252
no shut
description To R2
clock rate 256000
int s0/1
ip add 17.9.1.5 255.255.255.252
no shut
description To R2
clock rate 256000
int lo0
ip add 11.11.11.11 255.255.255.255
no shut

!ISP2 Basic conf
inte s0/0
ip add 180.1.5.1 255.255.255.252
no shut
description To R1
clock rate 256000
int s0/1
ip add 180.1.5.5 255.255.255.252
no shut
description To R1
clock rate 256000
int lo0
ip add 22.22.22.22 255.255.255.255
no shut

!R1 Basic conf
int s0/0
ip add 17.9.1.2 255.255.255.252
no shut
description To ISP1
int s0/1
ip add 17.9.1.6 255.255.255.252
no shut
description To ISP1
int s0/2
ip add 10.1.1.5 255.255.255.252
no shut
clock rate 256000
description To R2
int s0/3
ip add 10.1.1.1 255.255.255.252
no shut
clock rate 256000
description To R3
int lo0
ip add 1.1.1.1 255.255.255.255

!R2 Basic conf
int s0/0
ip add 180.1.5.2 255.255.255.252
no shut
description To ISP2
int s0/1
ip add 180.1.5.6 255.255.255.252
no shut
description To ISP2
int s0/2
ip add 10.1.1.6 255.255.255.252
no shut
description To R1
int s0/3
ip add 10.1.1.9 255.255.255.252
no shut
clock rate 256000
description To R4
int lo0
ip add 2.2.2.2 255.255.255.255

!R3 Basic conf
int s0/0
ip add 150.1.0.1 255.255.255.252
no shut
description To Cust1
clock rate 256000
int s0/1
ip add 10.1.1.13 255.255.255.252
no shut
clock rate 256000
description To R4
int s0/3
ip add 10.1.1.2 255.255.255.252
no shut
description To R1
int lo0
ip add 3.3.3.3 255.255.255.255

!R4 Basic conf
int s0/0
ip add 10.1.1.10 255.255.255.252
no shut
description To R2
int s0/1
ip add 10.1.1.14 255.255.255.252
no shut
description To R3
int lo0
ip add 4.4.4.4 255.255.255.255
no shut
!Cust1 Basic conf
int s0/0
ip add 150.1.0.2 255.255.255.252
no shut
description To R3
int lo0
ip add 150.1.1.1 255.255.255.0
```

### Configure IGP

*   Configure network-statements as specific as possible
*   Only advertise between internal routers
*   Use a hello-timer of 1 second / dead-timer of 3 seconds

Enligt uppgiften skulle vi använda oss av OSPF. Känns inte som något är nytt här så skriver bara ner den konfig jag skriver. Kom ihåg att vi sätter hello/deadtimers per interface! Glöm inte passive-interface default.
```
!R1
router ospf 1
passive-interface default
no passive-interface s0/2
no passive-interface s0/3
router-id 1.1.1.1
network 1.1.1.1 0.0.0.0 area 0
network 17.9.1.2 0.0.0.0 area 0
network 17.9.1.6 0.0.0.0 area 0
network 10.1.1.5 0.0.0.0 area 0
network 10.1.1.1 0.0.0.0 area 0
int s0/2
ip ospf hello-interval 1
ip ospf dead-interval 3
int s0/3
ip ospf hello-interval 1
ip ospf dead-interval 3

!R2
router ospf 1
passive-interface default
no passive-interface s0/2
no passive-interface s0/3
router-id 2.2.2.2
network 2.2.2.2 0.0.0.0 area 0
network 180.1.5.2 0.0.0.0 area 0
network 180.1.5.6 0.0.0.0 area 0
network 10.1.1.6 0.0.0.0 area 0
network 10.1.1.9 0.0.0.0 area 0
int s0/2
ip ospf hello-interval 1
ip ospf dead-interval 3
int s0/3
ip ospf hello-interval 1
ip ospf dead-interval 3

!R3
router ospf 1
passive-interface default
no passive-interface s0/3
no passive-interface s0/1
router-id 3.3.3.3
network 3.3.3.3 0.0.0.0 area 0
network 150.1.0.1 0.0.0.0 area 0
network 10.1.1.2 0.0.0.0 area 0
network 10.1.1.13 0.0.0.0 area 0
int s0/1
ip ospf hello-interval 1
ip ospf dead-interval 3
int s0/3
ip ospf hello-interval 1
ip ospf dead-interval 3

!R4
router ospf 1
passive-interface default
no passive-interface s0/0
no passive-interface s0/1
router-id 4.4.4.4
network 4.4.4.4 0.0.0.0 area 0
network 10.1.1.14 0.0.0.0 area 0
network 10.1.1.10 0.0.0.0 area 0
int s0/0
ip ospf hello-interval 1
ip ospf dead-interval 3
int s0/1
ip ospf hello-interval 1
ip ospf dead-interval 3
```
Kom ihåg att verifiera så att alla nät är nåbara redan nu innan det börjar bli lite mer avancerat. :)

### Full-mesh IGP

*   Peers should fail over based on IGP if any internal links fail
*   Set logical descriptions in BGP
*   Disable BGP synchronization

Steg 1 har vi redan löst delvis genom att konfigurera Loopbacks på R1-R4 som vi annonserar via OSPF. När vi sätter upp iBGP-relations nu så behöver vi endast använda loopback-adresserna istället för de fysiska länknäten. Kom ihåg att även ändra update-source (om du inte vet varför, se tidigare inlägg om [iBGP transit-area](http://roadtoccie.se/2013/07/07/bgp-internal-bgp-transitarea/))! Steg 2 fixar vi genom descriptions per neighbor-statement, och BGP synchronization är avslaget per default i den IOS jag använder (v12.4(15)T13) (no synchronization).
```
!R1 iBGP
router bgp 500
neighbor 2.2.2.2 remote-as 500
neighbor 2.2.2.2 description iBGP to R2
neighbor 2.2.2.2 update-source Lo0
neighbor 3.3.3.3 remote-as 500
neighbor 3.3.3.3 description iBGP to R3
neighbor 3.3.3.3 update-source Lo0
neighbor 4.4.4.4 remote-as 500
neighbor 4.4.4.4 description iBGP to R4
no synchronization

!R2
router bgp 500
neighbor 1.1.1.1 remote-as 500
neighbor 1.1.1.1 description iBGP to R1
neighbor 1.1.1.1 update-source Lo0
neighbor 3.3.3.3 remote-as 500
neighbor 3.3.3.3 description iBGP to R3
neighbor 3.3.3.3 update-source Lo0
neighbor 4.4.4.4 remote-as 500
neighbor 4.4.4.4 description iBGP to R4
neighbor 4.4.4.4 update-source Lo0
no synchronization

!R3
router bgp 500
neighbor 1.1.1.1 remote-as 500
neighbor 1.1.1.1 description iBGP to R1
neighbor 1.1.1.1 update-source Lo0
neighbor 2.2.2.2 remote-as 500
neighbor 2.2.2.2 description iBGP to R2
neighbor 2.2.2.2 update-source Lo0
neighbor 4.4.4.4 remote-as 500
neighbor 4.4.4.4 description iBGP to R4
neighbor 2.2.2.2 update-source Lo0
no synchronization

!R4
router bgp 500
neighbor 1.1.1.1 remote-as 500
neighbor 1.1.1.1 description iBGP to R1
neighbor 1.1.1.1 update-source Lo0
neighbor 2.2.2.2 remote-as 500
neighbor 2.2.2.2 description iBGP to R2
neighbor 2.2.2.2 update-source Lo0
neighbor 3.3.3.3 remote-as 500
neighbor 3.3.3.3 description iBGP to R3
neighbor 3.3.3.3 update-source Lo0
no synchronization
```
En sh ip bgp summary visar att vi är på rätt spår. :)
```
R1#sh ip bgp summary 
BGP router identifier 1.1.1.1, local AS number 500
BGP table version is 1, main routing table version 1
Neighbor V AS MsgRcvd MsgSent TblVer InQ OutQ Up/Down State/PfxRcd
2.2.2.2 4 500 6 7 1 0 0 00:03:47 0
3.3.3.3 4 500 4 5 1 0 0 00:01:55 0
4.4.4.4 4 500 4 4 1 0 0 00:00:12 0
```
### Configure eBGP-peers between NL Fast & ISP1-2 and Customer

*   On redudant links, peers using loopback and provide loadbalancing
*   Configure authentication
*   Set logical descriptions for neighbors

Steg 1 skiljer sig inte jättemycket från vad vi nyss gjorde när vi satte upp iBGP-mesh. Som du förhoppningsvis kommer ihåg tillåter BGP endast en öppen session mot en neighbor, för redundanta länkar bör vi således använda Loopback-interface för att inte vara låsta till någon specifik länk. Kom ihåg att ändra update-source! Lastbalansering skapar vi enklast via två statiska routes som båda pekar på ISP1/2s loopback-adress och vice-versa. Autentisering har jag inte skrivit någon post om ännu men det är extremt enkelt att sätta upp i BPG som du kommer se nedan.
```
!R1 eBGP

ip route 11.11.11.11 255.255.255.255 17.9.1.1
ip route 11.11.11.11 255.255.255.255 17.9.1.5

router bgp 500
neighbor 11.11.11.11 remote-as 200
neighbor 11.11.11.11 description ISP1
neighbor 11.11.11.11 password cisco
neighbor 11.11.11.11 update-source lo0

!ISP1

ip route 1.1.1.1 255.255.255.255 17.9.1.2
ip route 1.1.1.1 255.255.255.255 17.9.1.6

router bgp 200
neighbor 1.1.1.1 remote-as 500
neighbor 1.1.1.1 description ISP1
neighbor 1.1.1.1 password cisco
neighbor 1.1.1.1 update-source Lo0

!R2

ip route 22.22.22.22 255.255.255.255 180.1.5.1
ip route 22.22.22.22 255.255.255.255 180.1.5.5

router bgp 500
neighbor 22.22.22.22 remote-as 300
neighbor 22.22.22.22 description ISP1
neighbor 22.22.22.22 password cisco
neighbor 22.22.22.22 update-source lo0
neighbor 22.22.22.22 ebgp-multihop 2

!ISP2

ip route 2.2.2.2 255.255.255.255 180.1.5.2
ip route 2.2.2.2 255.255.255.255 180.1.5.6

router bgp 300
neighbor 2.2.2.2 remote-as 500
neighbor 2.2.2.2 description ISP1
neighbor 2.2.2.2 password cisco
neighbor 2.2.2.2 update-source Lo0
```
Här fastande jag dock ett tag, av någon anledning fick jag inte upp någon adjacency? Varken debug ip bgp all eller wireshark gav några hintar om vad felet kunde tänkas vara. En show ip bgp summary visade endast att neighbor x.x.x.x var fast i "idle". Tillslut kom jag dock på vad det var, gör du? **Försök att finna lösningen själv**! facit i vit text (markera för svar): eBGP-multihop! TTL sätts som bekant till 1 för eBGP-relationsships, när vi använder Loopbacks behöver vi därför modifera detta via neighbor x.x.x.x ebgp-multihop 2. Har en tidigare post dedikerad om just detta om detta koncept är nytt för dig och finns att läsa [här](http://roadtoccie.se/2013/07/22/bgp-ebgp-multihop/).

### Announce networks in BGP appropriately

*   ISP1 & 2 should use filtered redistribution to announce their networks. Only networks located on the loopback-interfaces of ISP1&2 should enter the BGP-table
*   The Cust1-router should advertise its network using the network-statement
*   R1 & R2 should advertise the WAN-link subnet (currently 150.1.0.0/30) as 150.1.0.0/24

Här är det nog lika bra vi bryter ner det i mindre delar. Vi börjar med **filtered redistribution!** Verkar som att det saknades i topologin jeremy skapade, men ISP1 ska tydligen ha mer än det loopback-nätet vi använder för att sätta upp eBGP. Jag skapade därför 5 nät på respektive router:
```
!R1
int lo1
ip add 200.10.0.1 255.255.255.0
int lo2
ip add 200.10.1.1 255.255.255.0
int lo3
ip add 200.10.2.1 255.255.255.0
int lo4
ip add 200.10.3.1 255.255.255.0
int lo5
ip add 200.10.4.1 255.255.255.0

!R2
int lo1
ip add 200.20.0.1 255.255.255.0
int lo2
ip add 200.20.1.1 255.255.255.0
int lo3
ip add 200.20.2.1 255.255.255.0
int lo4
ip add 200.20.3.1 255.255.255.0
int lo5
ip add 200.20.4.1 255.255.255.0
```
Vi hade enkelt kunnat lösa detta genom att använda network-statements för Loopback-näten, men i detta fall så önskas filtrering. Finns säkert många olika lösningar på detta men såhär tänkte jag: Vi skapar först en prefix-lista:
```
!ISP1
 ip prefix-list LOOPBACKS seq 5 permit 200.10.0.0/24
 ip prefix-list LOOPBACKS seq 10 permit 200.10.1.0/24
 ip prefix-list LOOPBACKS seq 15 permit 200.10.2.0/24
 ip prefix-list LOOPBACKS seq 20 permit 200.10.3.0/24
 ip prefix-list LOOPBACKS seq 25 permit 200.10.4.0/24

 !ISP2
 ip prefix-list LOOPBACKS seq 5 permit 200.20.0.0/24
 ip prefix-list LOOPBACKS seq 10 permit 200.20.1.0/24
 ip prefix-list LOOPBACKS seq 15 permit 200.20.2.0/24
 ip prefix-list LOOPBACKS seq 20 permit 200.20.3.0/24
 ip prefix-list LOOPBACKS seq 25 permit 200.20.4.0/24
```
Sedan en route-map som endast tillåter nät som matchar vår prefix-lista, passar även på att sätta origin till igp när vi ändå är igång:
```
!ISP1 & 2
route-map LOOPBACK_FILTER permit 10
match ip address prefix-list LOOPBACKS
set origin igp
```
Och vi använder sedan route-mapen som filter när vi aktiverar redistribution av nät som är "Connected":
```
!ISP1 & 2
router bgp 200 / 300
redistribute connected route-map LOOPBACK_FILTER
```
Det borde fixa biffen! En debug visar att vi är på rätt spår:
```
*Mar 1 02:18:30.707: BGP: Applying map to find origin for 200.10.4.0/24
*Mar 1 02:18:30.735: BGP: Applying map to find origin for 200.10.0.0/24
*Mar 1 02:18:30.739: BGP: Applying map to find origin for 200.10.1.0/24
*Mar 1 02:18:30.743: BGP: Applying map to find origin for 200.10.2.0/24
*Mar 1 02:18:30.747: BGP: Applying map to find origin for 200.10.3.0/24
```
Och en sh ip bgp på R4 visar att allt fungerar som det ska! 
[![R4-bgplab](/assets/images/2013/07/r4-bgplab.png)](/assets/images/2013/07/r4-bgplab.png)

Next up var :

*   **The Cust1-router should advertise its network using the network-statement**

Denna router har just nu endast grundkonfig så vi får fixa allt grundläggande också! Då vi ej har en redundant lina behöver vi ej "krångla" med loopbacks/ebgp multihop. Observera också att kunden använder ett Privat-AS, vilket kan jämföras med privata ip-adresser, och bör således aldrig annonseras ut till internet(ISP1/2).
```
!R3
 router bgp 500
 neighbor 150.1.0.2 remote-as 64512
 neighbor 150.1.0.2 description Cust1
 neighbor 150.1.0.2 password cisco

!Cust1 (private AS)
 router bgp 64512
 no synchronization
 network 150.1.1.0 mask 255.255.255.0
 neighbor 150.1.0.1 remote-as 500
 neighbor 150.1.0.1 description NLFastISP
 neighbor 150.1.0.1 password cisco
```
Ser bra ut! Observera dock att vi just nu skickar med det privata as-numret till ISP:en. [![isp1](/assets/images/2013/07/isp1.png)](/assets/images/2013/07/isp1.png) Detta är inget som nämns i CCNP-boken så fick ta och googla lite, skönt nog var det väldigt enkelt att fixa!
```
!R1
router bgp 500
neighbor 11.11.11.11 remove-private-as

!R2
router bgp 500
neighbor 22.22.22.22 remote-private-as
```
*   **R1 & R2 should advertise the WAN-link subnet (currently 150.1.0.0/30) as 150.1.0.0/24**

Enklast tycker jag borde vara att göra enligt följande:
```
!R3
ip route 150.1.0.0 255.255.255.0 null0
router bgp 500
network 150.1.0.0 mask 255.255.255.0
```
Svårare än så behöver det nog inte vara.. 
[![isp1-null0](/assets/images/2013/07/isp1-null0.png)](/assets/images/2013/07/isp1-null0.png) Woho!! :)

### Verify

*   Verify that all expected neighbors have formed - check!
*   Verify that all expected routes appear - check!
*   ISP1 & 2 should be able to ping: Cust1 routes & NL FastISP WAN-link
```
ISP1#ping 150.1.0.1 source lo0
Success rate is 0 percent (0/5)
ISP1#ping 150.1.1.1 source lo0
Success rate is 0 percent (0/5)
```
Crap.. :) Let the troubleshooting begin! En traceroute visar att vi fastnar i R3:
```
ISP1#traceroute 150.1.0.2
Type escape sequence to abort.
Tracing the route to 150.1.0.2
1 17.9.1.6 36 msec
 17.9.1.2 48 msec
 17.9.1.6 72 msec
 2 10.1.1.2 8 msec 40 msec 124 msec
 3 * * *
```
Om du varit väldigt uppmärksam på skärmdumparna jag visat under labben här så kanske du redan upptäckt ett stort problem. Om vi kollar en skärmdump från R4 exempelvis: [![r4-bgplab2](/assets/images/2013/07/r4-bgplab2.png)](/assets/images/2013/07/r4-bgplab2.png) Endast 150.1.0.0/24 & 150.1.1.0/24 har valts till "best"-route och har därför inte lagt in i routing table! Det är även därför vi inte kan se ISP2's routes i ISP1's table. [![isp1-null0](/assets/images/2013/07/isp1-null0.png)](/assets/images/2013/07/isp1-null0.png) Problemet är helt enkelt att routrarna **EJ** har en route för den next-hop adressen som annonseras (förutom R1<->ISP1 och R2<->ISP2 tack vare sina statiska routes)! Jag löste det enligt följande:
```
!R1
access-list 1 permit 11.11.11.11 0.0.0.0 
route-map LOOP_REDIST permit 10
match ip address 1
router ospf 1
redistribute static subnets route-map LOOP_REDIST

!R2
access-list 1 permit 22.22.22.22 0.0.0.0 
route-map LOOP_REDIST permit 10
match ip address 1
router ospf 1
redistribute static subnets route-map LOOP_REDIST
```
Let's watch the magic happen! :) 

[![r4-bgplab3](/assets/images/2013/07/r4-bgplab3.png)](/assets/images/2013/07/r4-bgplab3.png) 

Och från ISP:
[![isp2](/assets/images/2013/07/isp2.png)](/assets/images/2013/07/isp2.png) 

Vackert! Vi bör dock se till att Customer1 får en default-route, vi vill ju inte hålla på och annonsera våra interna nät dit. Detta löser vi enklast genom följande:
```
!R3
router bgp 500
#neighbor 150.1.0.2 default-originate
```
Detta skickar en default-route till Cust1 via BGP! Enkelt. :) 

[![custisp](/assets/images/2013/07/custisp.png)](/assets/images/2013/07/custisp.png) Vi kan nu pinga som önskat!
```
ISP1#ping 150.1.1.1 source lo1
Success rate is 100 percent (5/5), round-trip min/avg/max = 44/63/92 ms

ISP2#ping 150.1.1.1 source lo1
Success rate is 100 percent (5/5), round-trip min/avg/max = 52/83/140 ms
```
Finally! Ytterligare något man skulle kunna fixa till är att blockera ISP1 från att lära sig ISP2's routes via vårat AS och vice versa, då detta gör oss till en transit-area även för dem - något vi absolut inte vill! Men klockan börjar bli väldigt sent och jag ska upp tidigt och jobba imorgon så tar och avslutar här. Problemet hade vi hur som enkelt löst genom route-maps, och endast tillåtit de nät vi specificerar att annonseras ut till AS200 & 300.
