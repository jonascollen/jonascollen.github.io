---
title: BGP - Path Selection Part II & Weight
date: 2013-07-23 19:50
comments: true
categories: [BGP, CCNP, Path Selection, Route Manipulation, route-map, weight]
---
Detta blir en fortsättning på inlägget "[AS_SEQ Path Attribute & Path Selection](http://Jonas Collén.wordpress.com/2013/07/22/bgp-as_seq-path-attribute-best-path-selection/)" så vi kan gå in lite mer på djupet över hur just BGPs best path selection går till. För att bestämma vilken route som är bäst används följande kriterier i turordning tills den hittat en avgörande faktor, dvs om alla parametrar är identiska mellan två vägar för en specifik destination så är det tillslut den med lägst Router-ID som vinner. :)

1.  Largest Weight (cisco proprietary)
2.  Highest Local Preference
3.  Locally Originated
4.  Shortest AS Path
5.  Lowest Origin Type (i < e < ?)*
6.  Lowest MED (metric)
7.  eBGP over iBGP
8.  Lowest IGP metric to neighbor (Max. paths check - default 1)
9.  Older route
10.  Lowest Router-ID

_*i = Injected from an IGP (using network-command), e = Injected from Exterior Gateway Protocol (EGP), ? = Undetermined_ Det finns dock ett Steg 0 också, "Next hop: reachable?", om routern inte kan nå den specificerade next-hop adressen kommer routern räknas som invalid. För att lättare komma ihåg detta föreslår Cisco följande "ordramsa(?)" - N WLLA OMNI. Ytterligare något som är bra att känna till är att Cisco särskiljer på rekommenderade metoder beroende på om vi vill modifiera "outbound"-eller "inbound"-routes. För outbound rekommenderas:

*   Weight
*   Local Preference
*   AS_PATH

Och för inbound:

*   MED (metric)

Låt oss använda följande topologi igen och kontrollera hur valet gått till för exempelvis routen 200.0.3.0/24 i R7: 
![bgp AS-path](/assets/images/2013/07/bgp-as-path.png) 
![pathselection1](/assets/images/2013/07/pathselection1.png) 

Först har vi weight, men vi kan se att värdet är satt till "0" för båda alternativen, ingen vinnare där med andra ord. Nästa steg är "Highest Local Preference", men inte heller där finns det något värde satt, Tredje punkten, "Locally Originated" får vi inte heller någon match på då det är en extern route. För fjärde punkten i listan - "Shortest AS Path", kan vi dock se skillnad. Routen med next-hop adressen 172.16.47.4 har ett mindre AS-hopp än alternativet som går via AS200- > AS500 -> AS100. Denna route blir därför vald till "best route" och installeras i routerns routing table:
```
R7#sh ip route 200.0.3.0
Routing entry for 200.0.3.0/24
 Known via "bgp 50", distance 20, metric 0
 Tag 500, type external
 Last update from 172.16.47.4 05:53:45 ago
 Routing Descriptor Blocks:
 *** 172.16.47.4, from 172.16.47.4, 05:53:45 ago**
 Route metric is 0, traffic share count is 1
 AS Hops 2
 Route tag 500
```
Hur bär vi då oss åt om vi vill modifiera detta? I CCNP route räknar Cisco med att vi ska ha koll på följande fyra:

*   Weight
*   Local preference
*   AS_Path Lenght
*   MED (metric)

Vi börjar med den enklaste(?)..

Largest Weight
--------------

*   Is not a Path Attribute
*   Purpose - Identifies a single router's best route
*   Scope - Set on inbound route Updates; influences only that one router's choice
*   Range - 0 through 65-535 (2^16-1)
*   Which is best? - Bigger value is better
*   Default - 0 for learned routes, 32,768 for locally injected routes
*   Defining a new default - Not supported
*   Configuration - neighbor route-map (per prefix), neighbor weight (all routes from this neighbor)

Detta option är "Cisco proprietary" och räknas ej som ett "Path Attribute". Vi behöver dock inte bry oss i om våra neighbors också använder Cisco's hårdvara då Weight endast används lokalt! Om vi använder oss av samma topologi återigen, låt oss säga att vi hellre föredrar att trafiken från R7 tar den längre vägen via AS200 -> AS500 -> AS100 (då vi kanske har en snabbare uppkoppling mellan de kontoren). Konfigurationen blir då följande:
```
R7(config)#router bgp 50
R7(config-router)#neighbor 172.16.76.6 weight ?
 <0-65535> default weight
R7(config-router)#neighbor 172.16.76.6 weight 5
R7(config-router)#end
R7#clear ip bgp *
```
Kom ihåg att vi behöver starta om BGP-processen! Detta leder till följande resultat: 

![pathselection-weight](/assets/images/2013/07/pathselection-weight.png) 

R7 föredrar nu att gå via R6 trots att det är ett AS-hopp mer än direkt via R4/AS500. Hur gör vi då om vi endast vill modifiera detta för en/flera specifika routes och inte alla? Route-maps! Vi tar bort ändringen ovan och försöker istället utföra samma sak men endast för routen 200.0.3.0/24. Som du säkert gissat behöver vi först skapa en access-lista:

`access-list 1 permit 200.0.3.0`

Och sedan en route-map:
```
route-map WEIGHT-FILTER permit 10
match ip address 1
set weight 10
```
Vi applicerar sedan route-map:en i vårat **neighbor**-statement(!)
```
router bgp 50
neighbor 172.16.76.6 route-map WEIGHT-FILTER in
do clear ip bgp *
```
That's it, detta ger följande resultat: 

![route-map weight](/assets/images/2013/07/route-map-weight.png)

Fast vänta lite nu.. Jämför detta med det resultat vi fick innan vi applicerade route-mapen ovan. Vi saknar nu de alternativa vägarna för näten 5.5.5.0/24, 6.6.6.0/24 & 200.0.2.0/24! Du kanske redan listat ut varför det blir såhär, om inte föreslår jag att du tar en paus och försöker lösa det på egen hand innan du läser vidare. I den access-lista vi skapade matchade vi endast nätet 200.0.3.0/24, så det är kanske inte så konstigt att vi endast ser detta nätet nu. Men vi kan ju inte gärna lägga till resterande nät i acl:en då vi endast ville ändra weight för 200.0.3.0? Svaret ligger i route-map:en! Just nu har vi ju följande konfiguration:
```
access-list 1 permit 200.0.3.0
!
route-map WEIGHT-FILTER permit 10
match ip address 1
set weight 10
```

Det finns ju bevisligen inget statement för de övriga näten. Vi fixar detta enkelt med att skapa ett match-any statement som tillåter allt!

`route-map WEIGHT-FILTER permit 20`

Vi behöver inget match-statement då vi vill tillåta allt som inte redan matchats under sequence 10/ACL 1. Vi kör återigen en clear ip bgp * och håller tummarna.. 

![route-map weight2](/assets/images/2013/07/route-map-weight2.png)

Kungligt! Nästa inlägg blir en fortsättning på samma ämne men då kollar vi istället på Local Preference & AS PATH_LENGHT.
