---
title: BGP - AS_Path Prepend & MED
date: 2013-07-24 11:06
comments: true
categories: [BGP]
tags: [as-path prepend, med]
---
AS_Path Prepend
----------------

Steg 4 i BGPs "best path selection"-process är som vi redan vet AS_PATH och vi har tagit upp hur denna process går till i flera tidigare inlägg, bland annat [AS_SEQ Path Attribute](http://Jonas Collén.wordpress.com/2013/07/22/bgp-as_seq-path-attribute-best-path-selection/) & [Path Selection Part II](http://Jonas Collén.wordpress.com/2013/07/23/bgp-path-selection-part-ii-weight/). Genom att använda oss av AS_PATH Prepend får vi möjlighet att manuellt lägga till ytterligare AS i AS_PATH för en route och på så vis få den mindre attraktiv. Detta påverkar dock ej AS_Path Loop Prevention så vi har möjlighet att lägga till samma AS-nummer ett flertal gånger istället. 
![](/assets/images/2013/07/topologylocalpref.jpg) 

Om vi kollar från R7s perspektiv i ovanstående topologi, så har vi  exempelvis möjlighet att få routes vi lär oss från AS500 att se sämre ut genom att lägga till ytterligare AS_Path hopp. En sh ip bgp visar följande just nu: 
![as_path](/assets/images/2013/07/as_path.png) 

För alla externa 200.0.x.0/24-routes (förutom 200.0.3.0/24 pga weight 10 från ett tidigare inlägg) så väljer R7 att gå den kortare AS_Path vägen via AS500 -> AS 100. Låt oss testa lägga till ytterligare AS för 200.0.1.0/24 & 200.0.2.0/24.. Först en access-lista prefix-lista (för att variera oss lite):
```
ip prefix-list AS_PATH seq 5 permit 200.0.1.0/24
ip prefix-list AS_PATH seq 10 permit 200.0.2.0/24
```
Vi skapar sedan en route-map:
```
route-map AS_PATH_PREPEND permit 10
match ip address prefix-list AS_PATH
set as-path prepend 500 500
route-map AS_PATH_PREPEND permit 20
```
Detta bör lägga till ytterligare två AS-hopp (500) för 200.0.1.0/24 & 200.0.2.0/24, och routern bör således föredra att  istället gå via R6(AS200) pga ett AS-hopp mindre. 

![as_path2](/assets/images/2013/07/as_path2.png)

Stämmer bra! Något som inte tas upp i boken (Official Certification Guide - CCNP ROUTE, Wendell Odom) men som kändes rätt intressant - borde vi inte genom detta även kunna påverka hur vi annonserar våra nät till andra AS? Om vi tar från R7's perspektiv igen, tänk om vi köpt en snabbare uppkoppling mot R6/AS200 än mot R4/As500, och således föredrar att våra kunder (ex. AS100) går via den routern istället för att nå 192.168.0.0/24? Vi har ju som kund ingen möjlighet att gå in och konfigurera ett annat företags/isps weight/local preference - men borde vi inte kunna annonsera ytterligare AS-hopp för den vägen vi anser vara "sämre"? Vi testar! Jag har tagit bort route-mapen vi skapade innan och börjar om från noll.. Först en prefix-lista:

`ip prefix-list AS_PATH_INTERNAL permit 192.168.0.0/24`

Sedan en route-map:
```
route-map RMAP-AS_PATH_INTERNAL permit 10
match ip address prefix-list AS_PATH_INTERNAL 
set as-path prepend 50 50
route-map RMAP-AS_PATH_INTERNAL permit 20
```
Och aktiverar den i BGP:
```
router bpg 50
neighbor 172.16.47.4 route-map RMAP-AS_PATH_INTERNAL out
```
Har som sagt ingen aning om detta fungerar eller ej, men kollar vi i R7 så är det ingen skillnad vilket kanske inte är helt lovande.. 

![as_path3](/assets/images/2013/07/as_path3.png)

Men i R4 har det hänt grejjer! :) 
![as_path4](/assets/images/2013/07/as_path41.png) 

Och en trace från AS100/R5 visar att det fungerar som önskat!
```
R5#traceroute 192.168.0.1 source lo5
Type escape sequence to abort.
Tracing the route to 192.168.0.1
1 172.16.51.1 12 msec 132 msec 12 msec
 2 172.16.12.2 140 msec 60 msec 92 msec
 3 172.16.23.3 52 msec 136 msec 20 msec
 4 172.16.36.6 116 msec 80 msec 112 msec
 5 172.16.76.7 92 msec * 108 msec
```
Coolt! Lite märkligt att det inte nämns något i boken om detta dock? Känns som det är ett väldigt smidigt sätt att influera vägvalet för ANDRA företag/AS till dina nät? :) Svaret kommer kanske i nästa del då kapitlet heter "Influencing an Enterprise's **Inbound routes** with MED".

Multi Exit Discriminator (MED)
------------------------------

*   Is a path attribute
*   Purpose - Allows an AS to tell a neighboring AS the best way to forward packets into the first AS
*   Scope - Advertised by one AS into another, propagated inside the AS, but not send to any other autonomous systems
*   Range - 0 to 4,294,967,295 (2^32 - 1)
*   Which is best ? Smaller is better
*   Default - 0
*   Configuration - Via neighbor route-map out, using the set metric in the route-map

Boken tar upp ett problem med mina tankegångar ovan - är du ett mindre företag är risken stor att de publika nät du annonserar till "internet" summeras hos din ISP tillsammans med många andra kunder. Vilket i sin tur gör att alla potentiella justeringar du gör "försvinner", vi kan dock fortfarande påverka det "sista" hoppet, mellan dig som kund och din ISP(er). Det är här MED kommer in i bilden, det ger oss nämligen möjlighet att berätta för ISP:en vilken väg (exit point) som är att föredra in till ditt nät. MED skapades till en början för dual-homed nät, men har på senare tid expanderats och kan nu även användas för dual-multihomed lösningar! MED skickas till vår eBGP-neighbor och sprids sedan vidare till dess iBGP-peers. Informationen skickas dock **EJ** vidare till andra AS! Endast modifierandet av AS-path som vi gjorde tidigare kan göra detta. 

[![MED](/assets/images/2013/07/med.png)](/assets/images/2013/07/med.png) 

Om vi använder ovanstående topologi så ser BGP-table ut enligt följande innan vi gör några modifieringar: 
[![med-bgptable](/assets/images/2013/07/med-bgptable.png)](/assets/images/2013/07/med-bgptable.png) 
[![med-bgptable2](/assets/images/2013/07/med-bgptable2.png)](/assets/images/2013/07/med-bgptable2.png) 

Och en traceroute från Internet till 10.0.12.0/24 visar att trafiken just nu väljer att gå via R5 -> R3 -> R1 -> R2.
```
Internet#traceroute 10.0.12.2 source 200.0.0.1
Type escape sequence to abort.
 Tracing the route to 10.0.12.2
1 140.0.0.2 12 msec 68 msec 36 msec
 2 172.0.35.3 40 msec 36 msec 48 msec
 3 191.0.0.1 112 msec 64 msec 84 msec
 4 10.0.12.2 [AS 500] 116 msec * 100 msec
```
Om vi nu säger att företaget har köpt följande länkhastigheter av sin ISP: R1 -> R3 - 10Mbit R1 -> R4 - 2Mbit R2 - R4 - 100Mbit Så är det ju klart och tydligt att den väg trafiken från internet tar till vårat nät är klart icke-optimalt. Det blir då rätt självklart att vi på något sätt vill försöka influera vår eBGP-neighbor att istället gå via 100Mbits-länken. Kom ihåg att för MED så är lägst värde bäst, till skillnad mot Weight & Local preference. Vi börjar med att konfa R1:
```
ip prefix-list INTERNAL permit 10.0.12.0/24
route-map SET-MED-R3 permit 10
 match ip address prefix-list INTERNAL
 set metric 50
 route-map SET-MED-R4 permit 10
 match ip address prefix-list INTERNAL
 set metric 100
router bgp 500
 neighbor 191.0.0.2 route-map SET-MED-R3 out
 neighbor 191.0.0.6 route-map SET-MED-R4 out
```
Och sedan för R2:
```
ip prefix-list INTERNAL permit 10.0.12.0/24
 route-map SET-MED-R4 permit 10
 match ip address prefix-list INTERNAL
 set metric 10
router bgp 500
 neighbor 191.0.0.10 route-map SET-MED-R4 out
```
Detta ger följande resultat i R3: 
[![med-r3](/assets/images/2013/07/med-r31.png)](/assets/images/2013/07/med-r31.png) 

R4:
[![med-r4](/assets/images/2013/07/med-r41.png)](/assets/images/2013/07/med-r41.png) 

R5: 
[![med-r5](/assets/images/2013/07/med-r51.png)](/assets/images/2013/07/med-r51.png) 

Inte allt för krångligt! Om vi gör en ny traceroute från Internet kan vi nu se att den tar den "snabbare" vägen:
```
Internet#traceroute 10.0.12.2 source 200.0.0.1
Type escape sequence to abort.
 Tracing the route to 10.0.12.2
1 140.0.0.2 44 msec 60 msec 8 msec
 2 172.0.45.4 44 msec 100 msec 88 msec
 3 191.0.0.9 56 msec * 16 msec
```
Men som synes nedan så skickas **EJ** MED-attributet vidare till nästa AS (25000) då vi ej kan se några värden längre. 

[![med-internet](/assets/images/2013/07/med-internet.png)](/assets/images/2013/07/med-internet.png) 

Vackert! Nu har vi endast maximum paths kvar att gå igenom innan vi är "klara" med CCNP's BGP-del, även om vi gått ganska långt utanför "CCNP-scopet" i tidigare poster också. :P Kommer nog köra vidare på samma spår och fortsätta med BGP då det känns som vi fortfarande bara skrapat lite på ytan av vad som är möjligt. Behöver bara leta upp en ny bok som täcker det materialet (CCIE Certification Guide eller TCP/IP Routing vol2 misstänker jag).. :)
