---
title: BGP - Advanced BGP Lab, del 3
date: 2013-12-01 16:01
comments: true
categories: [BGP]
---
Fortsättning från tidigare inlägg om GNS3Vaults Advanced BGP-lab som finns att läsa [BGP – Advanced BGP Lab, del 1](http://www.jonascollen.se/posts/bgp-gns3vault-com-advanced-bgp-lab-del-1/) & [BGP – Advanced BGP Lab, del 2](http://www.jonascollen.se/posts/bgp-advanced-bgp-lab-del-2/).

Path Modifications
------------------

![bgp-advanced-lab](/assets/images/2013/11/bgp-advanced-lab.png)

*   When R4 sends a ping to the loopback interface of R1 it should choose the path through R2. You are only allowed to make changes on R3.
*   Create another loopback interface on R1 with ip address 172.16.1.1 /24, advertise this in RIP.
*   When R4 sends a ping to the 172.16.1.1 address it should take the path through R3, you are only allowed to make changes on R4.
*   When R6 sends a ping towards the loopback interface on R11 it should go through AS300.
*   R7 should prefer the path through R11 for all external networks except for 172.16.1.1.
*   Configure AS300 so it is no longer a transit AS for AS200 to reach 172.16.1.1 in AS100. AS400 should not be influenced.

Vi har just nu full konvergens och kan pinga samtliga loopbacks i vår topologi. **1. When R4 sends a ping to the loopback interface of R1 it should choose the path through R2. You are only allowed to make changes on R3.** Tar vi en titt på R4 först kan vi se följande:

```
R4#sh ip bgp
BGP table version is 34, local router ID is 4.4.4.4
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal,
 r RIB-failure, S Stale
Origin codes: i - IGP, e - EGP, ? - incomplete
Network Next Hop Metric LocPrf Weight Path
* 1.1.1.0/24 2.2.2.2 0 100 i
*> 3.3.3.3 0 100 i
```

Den föredrar just nu vägen via R3 för att nå R1s loopback 1.1.1.1. Då vi endast fick ändra konfigurationen på R3 för att styra om trafiken behöver vi använda oss av något Path Attribute. Varken Weight eller Local Preference kan ju hjälpa oss i detta fall då R4 ligger i ett eget AS. MED eller AS-Path Prepending däremot!

```
R3(config)#ip prefix-list R1-Loopback permit 1.1.1.0/24
R3(config)#route-map R1-Loopback-MED permit 10
R3(config-route-map)#match ip add prefix-list R1-Loopback
R3(config-route-map)#set as-path prepend 100 100
R3(config-route-map)#route-map R1-Loopback-MED permit 20
R3(config-route-map)#exit
R3(config)#router bgp 100
R3(config-router)#neighbor 4.4.4.4 route-map R1-Loopback-MED out

R4#sh ip bgp 1.1.1.0/24
BGP routing table entry for 1.1.1.0/24, version 35
Paths: (2 available, best #2, table Default-IP-Routing-Table)
 Advertised to update-groups:
 1 2
 100 100 100
 3.3.3.3 from 3.3.3.3 (3.3.3.3)
 Origin IGP, localpref 100, valid, external
 **100**
 **2.2.2.2 from 2.2.2.2 (2.2.2.2)**
 **Origin IGP, localpref 100, valid, external, best**
```

**2**. **Create another loopback interface on R1 with ip address 172.16.1.1 /24, advertise this in RIP. When R4 sends a ping to the 172.16.1.1 address it should take the path through R3, you are only allowed to make changes on R4.**

```
R1(config)#int lo1
R1(config-if)#ip add 172.16.1.1 255.255.255.0
R1(config)#router rip
R1(config-router)#network 172.16.1.0

R4#sh ip bgp 172.16.1.0
BGP routing table entry for 172.16.1.0/24, version 38
Paths: (2 available, best #2, table Default-IP-Routing-Table)
Flag: 0x820
 Advertised to update-groups:
 1 2
 100
 3.3.3.3 from 3.3.3.3 (3.3.3.3)
 Origin IGP, localpref 100, valid, external
 **100**
 **2.2.2.2 from 2.2.2.2 (2.2.2.2)**
 **Origin IGP, localpref 100, valid, external, best**
```

I detta fall behöver vi endast ändra BGP lokalt, och bör således använda oss av "Weight".

```
R4(config)#ip prefix-list R1-Loopback2 permit 172.16.1.0/24
R4(config)#route-map R1-Loopback2-Weight permit 10
R4(config-route-map)#match ip add prefix-list R1-Loopback2
R4(config-route-map)#set weight 10
R4(config-route-map)#route-map R1-Loopback2-Weight permit 20
R4(config-route-map)#exit
R4(config)#router bgp 10
R4(config-router)#neighbor 3.3.3.3 route-map R1-Loopback2-Weight in

R4#sh ip bgp 172.16.1.0
BGP routing table entry for 172.16.1.0/24, version 39
Paths: (2 available, best #1, table Default-IP-Routing-Table)
Flag: 0x800
 Advertised to update-groups:
 1 2
 **100**
 **3.3.3.3 from 3.3.3.3 (3.3.3.3)**
 **Origin IGP, localpref 100, weight 10, valid, external, best**
 100
 2.2.2.2 from 2.2.2.2 (2.2.2.2)
 Origin IGP, localpref 100, valid, external
```

**3. When R6 sends a ping towards the loopback interface on R11 it should go through AS300.**

Just nu tar R6 vägen via R7->R11.

```
R6#traceroute 11.11.11.11
Type escape sequence to abort.
Tracing the route to 11.11.11.11
1 192.168.67.7 24 msec 28 msec 68 msec
 2 192.168.117.11 56 msec 72 msec *
```

Känns som vi kan använda oss av Weight här med då local preference även hade annonserats till R7.

```
R6(config)#ip prefix-list R11-Loopback permit 11.11.11.0/24
R6(config)#route-map R11-Loopback-Weight permit 10
R6(config-route-map)#match ip add prefix-list R11-Loopback
R6(config-route-map)#set weight 10
R6(config-route-map)#route-map R11-Loopback-Weight permit 20
R6(config-route-map)#exit
R6(config-router)#neighbor 5.5.5.5 route-map R11-Loopback-Weight in

R6#sh ip bgp 11.11.11.0/24
BGP routing table entry for 11.11.11.0/24, version 1007
Paths: (3 available, best #1, table Default-IP-Routing-Table)
Flag: 0x820
 Advertised to update-groups:
 1 2
 **300 400**
 **5.5.5.5 from 5.5.5.5 (5.5.5.5)**
 **Origin IGP, localpref 100, weight 10, valid, external, best**
 400
 7.7.7.7 (metric 156160) from 7.7.7.7 (7.7.7.7)
 Origin IGP, metric 0, localpref 100, valid, internal
 300 400
 4.4.4.4 from 4.4.4.4 (4.4.4.4)
 Origin IGP, localpref 100, valid, external
```

**4. R7 should prefer the path through R11 for all external networks except for 172.16.1.1.** 

Vi kan använda oss av Weight här med men själva tillvägagångssättet blir lite annorlunda nu när vi vill matcha allt utom just 172.16.1.0/24-nätet.

```
R7(config)#ip prefix-list R1-Loopback-R11 permit 172.16.1.0/24
R7(config)#route-map RM-Redirect-to-R11 permit 10
R7(config-route-map)#match ip address prefix-list R1-Loopback-R11
R7(config-route-map)#route-map RM-Redirect-to-R11 permit 20
R7(config-route-map)#set weight 20
R7(config-route-map)#exit
R7(config)#router bgp 200
R7(config-router)#neighbor 11.11.11.11 route-map RM-Redirect-to-R11 in
```

Vi sätter helt enkelt weight för alla nät som inte matchas i vår första sequence, dvs allt utom 172.16.1.0/24. Jämför vi 172.16.1.0/24 med 1.1.1.0/24 kan vi se följande:

```
R7#sh ip bgp 172.16.1.0
BGP routing table entry for 172.16.1.0/24, version 12
Paths: (2 available, best #2, table Default-IP-Routing-Table)
 Advertised to update-groups:
 1
 400 300 100
 11.11.11.11 from 11.11.11.11 (11.11.11.11)
 Origin IGP, localpref 100, valid, external
 **300 100**
 **6.6.6.6 (metric 156160) from 6.6.6.6 (6.6.6.6)**
 **Origin IGP, metric 0, localpref 100, valid, internal, best**

R7#sh ip bgp 1.1.1.0/24
BGP routing table entry for 1.1.1.0/24, version 43
Paths: (2 available, best #1, table Default-IP-Routing-Table)
Flag: 0x820
 Advertised to update-groups:
 2
 **400 300 100**
 **11.11.11.11 from 11.11.11.11 (11.11.11.11)**
 **Origin IGP, localpref 100, weight 20, valid, external, best**
 300 100
 6.6.6.6 (metric 156160) from 6.6.6.6 (6.6.6.6)
 Origin IGP, metric 0, localpref 100, valid, internal
```

**5. Configure AS300 so it is no longer a transit AS for AS200 to reach 172.16.1.1 in AS100. AS400 should not be influenced.**

Här behöver vi istället filtrera bort routing-uppdateringar för 172.16.1.0/24 i R4 & R5 mot R6. Då vi redan har en prefix-lista konfad på R4 som matchar 172.16.1.0/24-nätet återanvänder vi den istället.

```
R4(config)#do sh ip prefix-list
ip prefix-list R1-Loopback2: 1 entries
 seq 5 permit 172.16.1.0/24
R4(config)#route-map AS200 deny 10
R4(config-route-map)#match ip add prefix-list R1-Loopback2
R4(config-route-map)#route-map AS200 permit 20
R4(config-route-map)#exit
R4(config)#router bgp 10
R4(config-router)#neighbor 6.6.6.6 route-map AS200 out
```

Jag blir dock osäker på hur vi ska blockera AS200 från att lära sig om nätet via As400 istället (utan att lägga in specifik filtrering där också)?

```
R6#sh ip bgp 172.16.1.0/24
BGP routing table entry for 172.16.1.0/24, version 1015
Paths: (1 available, best #1, table Default-IP-Routing-Table)
Flag: 0x820
 Advertised to update-groups:
 2
 400 300 100
 7.7.7.7 (metric 156160) from 7.7.7.7 (7.7.7.7)
 Origin IGP, metric 0, localpref 100, valid, internal, best
```

Som synes tar den nu istället vägen via R7 -> AS400 R11  -> R10-> AS300 R9..  Jag skulle vilja säga att vi ej har någon möjlighet att påverka vad ett annat AS  i sin tur gör med näten vi annonserar till  dem. För att lösa labben hade jag lagt in samma filter på R11 mot R7 också. Då var vi tillslut klara med den avancerade labben, gick trots allt väldigt smärtfritt skönt nog. :) MDH har släppt slutprojektet i CCNP-kursen nu så eventuella inlägg närmaste tiden kommer troligtvis handla om det.
