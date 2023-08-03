---
title: Route Filtering - The Basics
date: 2013-06-26 16:00
comments: true
categories: [OSPF, Route Manipulation]
tags: [acl,route-map,filtering]
---
Genom att använda oss av Route Filtering har vi möjlighet att själva bestämma vilka route's vi vill tillåta i vårat routing table från de olika routing-protokollen, och även vad vi annonserar ut till övriga routrar. När vi applicerar ett filter på inkommande trafik använder routern följande flödesschema: 
[![routefilter](/assets/images/2013/06/routefilter1.png)](/assets/images/2013/06/routefilter1.png) 

Som synes agerar route filtret precis som en access-lista med en explicit deny any. Låt oss använda följande topologi för att testa oss fram: 

[![routefilter ospf](/assets/images/2013/06/routefilter-ospf.png)](/assets/images/2013/06/routefilter-ospf.png) 

En sh ip route på R5 ger just nu följande:

```
20.0.0.0/24 is subnetted, 4 subnets
 O E2 20.0.0.0 [110/20] via 192.168.0.1, 00:00:01, FastEthernet0/1
 O E2 20.0.1.0 [110/20] via 192.168.0.1, 00:00:01, FastEthernet0/1
 O E2 20.0.2.0 [110/20] via 192.168.0.1, 00:00:01, FastEthernet0/1
 O E2 20.0.3.0 [110/20] via 192.168.0.1, 00:00:01, FastEthernet0/1
 O IA 172.16.0.0/16 [110/40] via 192.168.0.1, 00:00:01, FastEthernet0/1
 10.0.0.0/25 is subnetted, 2 subnets
 O IA 10.0.0.0 [110/30] via 192.168.0.1, 00:00:01, FastEthernet0/1
 O IA 10.0.0.128 [110/20] via 192.168.0.1, 00:00:02, FastEthernet0/1
 C 192.168.0.0/24 is directly connected, FastEthernet0/1
 30.0.0.0/24 is subnetted, 2 subnets
 S 30.0.0.0 is directly connected, Null0
 S 30.0.1.0 is directly connected, Null0
```
För att aktivera filtrering använder vi oss av något som kallas "distribute-list" under routing-processen,
```
R5(config)#router ospf 1
R5(config-router)#distribute-list ?
 <1-199> IP access list number
 <1300-2699> IP expanded access list number
 WORD Access-list name
 gateway Filtering incoming updates based on gateway
 prefix Filter prefixes in routing updates
 route-map Filter prefixes based on the route-map
```
Som synes har vi här flera olika möjligheter, vi börja med den enklaste!

Filtrering via Access-list
--------------------------

Låt oss testa filtrera bort 20.0.1.0/24 & 20.0.3.0/24 på R5. Vi skapar först en vanlig access-lista:
```
access-list 1 deny 20.0.1.0 0.0.0.255
access-list 1 deny 20.0.3.0 0.0.0.255
```
Vi använder oss sedan av "distribute-list" för att aktivera filtreringen:
```
router ospf 1
distribute-list 1 in
```
Om vi endast skriver ovanstående så gäller filtret på alla inkommande interface där vi aktiverat OSPF, men vi kan även skriva ett specifikt interface. En sh ip route borde ju nu visa alla route's utom 20.0.1.0 & 20.0.3.0:
```
Gateway of last resort is not set
C 192.168.0.0/24 is directly connected, FastEthernet0/1
 30.0.0.0/24 is subnetted, 2 subnets
S 30.0.0.0 is directly connected, Null0
S 30.0.1.0 is directly connected, Null0
```
Hum...? Förhoppningsvis kanske du redan förstått varför det inte fungerar, kom ihåg den explicita deny any som finns för access-listor! Vi justerar access-listan till följande istället:
```
access-list 1 deny 20.0.1.0 0.0.0.255
access-list 1 deny 20.0.3.0 0.0.0.255
access-list 1 permit any
```
En sh ip route visar nu:
```
Gateway of last resort is not set
20.0.0.0/24 is subnetted, 2 subnets
O E2 20.0.0.0 [110/20] via 192.168.0.1, 00:00:02, FastEthernet0/1
O E2 20.0.2.0 [110/20] via 192.168.0.1, 00:00:02, FastEthernet0/1
O IA 172.16.0.0/16 [110/40] via 192.168.0.1, 00:00:02, FastEthernet0/1
 10.0.0.0/25 is subnetted, 2 subnets
O IA 10.0.0.0 [110/30] via 192.168.0.1, 00:00:02, FastEthernet0/1
O IA 10.0.0.128 [110/20] via 192.168.0.1, 00:00:02, FastEthernet0/1
C 192.168.0.0/24 is directly connecte
*Mar 1 00:42:22.831: %SYS-5-CONFIG_I: Configured from console by consoled, FastEthernet0/1
 30.0.0.0/24 is subnetted, 2 subnets
S 30.0.0.0 is directly connected, Null0
S 30.0.1.0 is directly connected, Null0
```
2easy! :)

Filtrering via Prefix-List
--------------------------

Prefx-list liknar access-listor då vi även här använder oss av permit/deny men är en betydligt mer "kraftfull" variant. Om vi exempelvis vill filtrera bort 20.0.1.0/24 igen så hade vi med en access-lista skrivit:
```
access-list 1 deny 20.0.1.0 0.0.0.255
access-list 1 permit any
```
Om vi vill göra samma sak med en prefix-list så skriver vi istället följande:

`ip prefix-list OSPF-FILTER deny 20.0.1.0/24`

Härligt nog slipper vi skriva wildcardmask den här gången. :) Det som gör prefix-list det lilla extra är den inbyggda funktionen ge (greater than or equal to) & le (less than or equal to). Se följande exempel:

`ip prefix-list OSPF-FILTER deny 20.0.0.0/24 le 25`

Detta innebär att alla route's som faller inom spannet 20.0.0.0/24 och har "**L**ess than or **E**qual to" en /25-mask filtreras bort. Vi kommer med andra ord tillåta /26-32-nät inom samma spann! Förstå vad omständigt det skulle vara att göra samma sak med en vanlig access-lista..  Ytterligare ett exempel:

`ip prefix-list OSPF-FILTER permit 10.0.0.0/8 ge 16 le 24`

Detta tillåter route's som faller inom 10.0.0.0/8-spannet och har en subnät-mask som är "**G**reater than or **E**qual to" /16 men måste samtidigt vara "**L**ess than or **E**qual to" /24. Detta innebär att exempelvis 10.0.24.0/24 kommer tillåtas, men 10.10.0.4/30 kommer blockeras då nätet är för litet.  Vad händer med ett nät som inte ens passar in på spannet, ex, 192.168.0.0/24? Det blockeras.. Om vi vill tillåta alla nät då?

`ip prefix-list ALL permit 0.0.0.0/0 le 32`

Utan "le 32" kommer endast 0.0.0.0 0.0.0.0.0-routen matchas, dvs endast default-route tillåts. Ett sista exempel, lås oss säga att vi endast vill tillåta Class A-adresser med en /24 mask:

`ip prefix-list CLASSA permit 0.0.0.0/1`

Låt oss testa detta med vår befintliga topologi, säg att vi vill filtrera bort alla external routes som R2 skickar till R5:
```
ip prefix-list EXTERNAL seq 10 deny 20.0.0.0/22 ge 24
ip prefix-list EXTERNAL seq 20 permit 0.0.0.0/0 le 32
router ospf 1
distribute-list prefix EXTERNAL in
```
Observera att vi inte behöver använda oss av sequence-nummer, routern kommer själv lägga in detta (åtminstone i den 3750 jag labbar i). En sh ip route i R5 ger oss följande:
```
O IA 172.16.0.0/16 [110/40] via 192.168.0.1, 00:00:03, FastEthernet0/1
 10.0.0.0/25 is subnetted, 2 subnets
O IA 10.0.0.0 [110/30] via 192.168.0.1, 00:00:03, FastEthernet0/1
O IA 10.0.0.128 [110/20] via 192.168.0.1, 00:00:03, FastEthernet0/1
C 192.168.0.0/24 is directly connected, FastEthernet0/1
 30.0.0.0/24 is subnetted, 2 subnets
S 30.0.0.0 is directly connected, Null0
S 30.0.1.0 is directly connected, Null0
```
Stabilt! Vi kan precis som för access-listor kontrollera om vi fått några matches mot våra statements med följande kommando:
```
R5#sh ip prefix-list detail 
Prefix-list with the last deletion/insertion: hej
ip prefix-list EXTERNAL:
 count: 2, range entries: 2, sequences: 10 - 20, refcount: 2
 seq 10 deny 20.0.0.0/22 ge 24 (hit count: 8, refcount: 1)
 seq 20 permit 0.0.0.0/0 le 32 (hit count: 3, refcount: 1)
```

Filtrering via Route-map
------------------------

Route-maps kan liknas lite som ett scripting-språk då det använder sig av något som kan liknas vid IF- & Do-statements, cisco har dock valt att kalla det match och set. Låt oss testa göra samma sak som vi gjort tidigare, filterera bort de externa route'sen från R2 i R5. Route-maps använder sig dock av access-listor alternativt  prefix-list för att matcha ip-adresser/nät:
```
R5(config-route-map)#match ip address ?
 <1-199> IP access-list number
 <1300-2699> IP access-list number (expanded range)
 WORD IP access-list name
 prefix-list Match entries of prefix-lists
```
Vi har ju dock kvar prefix-listen från föregående exempel så låt oss använda den även här:
```
route-map RM-EXTERNAL deny 10
match ip address prefix-lists EXTERNAL
router ospf 1
distribute-list route-map RM-EXTERNAL in
```
En sh ip route på R5 visar följande:
```
Gateway of last resort is not set
C 192.168.0.0/24 is directly connected, FastEthernet0/1
 30.0.0.0/24 is subnetted, 2 subnets
S 30.0.0.0 is directly connected, Null0
S 30.0.1.0 is directly connected, Null0
```
Wtf? Lås oss gå tillbaka och se hur vår prefix-lista var uppbyggd!
```
R5#sh ip prefix-list 
ip prefix-list EXTERNAL: 2 entries
 seq 10 deny 20.0.0.0/22 ge 24
 seq 20 permit 0.0.0.0/0 le 32
```
Problemet är att vi nu tillåter alla route's i ACL:en (förutom de externa näten), men pga vårat deny-statement i route-mapen så nekas ju dessa istället. Låt oss ändra route-mapen till ett permit istället.
```
route-map RM-EXTERNAL permit 10
match ip address prefix-lists EXTERNAL
```
Kollar återigen på R5:
```
O IA 172.16.0.0/16 [110/40] via 192.168.0.1, 00:00:03, FastEthernet0/1
 10.0.0.0/25 is subnetted, 2 subnets
O IA 10.0.0.0 [110/30] via 192.168.0.1, 00:00:03, FastEthernet0/1
O IA 10.0.0.128 [110/20] via 192.168.0.1, 00:00:03, FastEthernet0/1
C 192.168.0.0/24 is directly connected, FastEthernet0/1
 30.0.0.0/24 is subnetted, 2 subnets
S 30.0.0.0 is directly connected, Null0
S 30.0.1.0 is directly connected, Null0
```
Sweet! Det gäller med andra ord att hålla ordning på begreppen när vi bygger lite mer avancerade ACL's senare. Den rekommendationen jag läst är att man bör alltid hålla sig till permit-statements i route-maps, och sen sköter man själva "deniandet" i prefix- & accesslistorna. Detta är bara en liten introduktion till de olika varianterna men inlägget börjar kännas lite väl långt så tar och avslutar här, för tro mig, det blir betydligt mer avancerat senare. :D