---
title: BGP - Route Dampening
date: 2013-08-07 11:23
comments: true
categories: [BGP]
tags: [route-dampening]
---
Route Dampening är BGP's skydd mot flappande routes. Har vi 450,000+ routes i vår routing-tabell är det garanterat ett stort antal av dessa som kanske har problem med sin uppkoppling eller har ett interface som studsar. Utan att filtrera bort dessa uppdateringar skulle vår router troligtvis gå på knäna i princip direkt. Route dampening påverkar inte våra neighbors utan den filtrerar endast det specifika prefixet som har problem. Per default använder BGP en light-variant av dampening för sina utgående uppdateringar genom attt ha en wait-time satt till 30 sekunder för eBGP & 5 sekunder för iBGP innan den skickar ett BGP-Update paket. Om vi tar följande topologi så kan vi titta lite närmare på hur dampening fungerar i praktiken. 

[![confederations](/assets/images/2013/08/confederations.png)](/assets/images/2013/08/confederations.png)

Nätet 200.200.20.0/24 som annonseras via R5 till ISP-Telia/Tele2 har fått problem med sin fiberuppkoppling till R3 och tappar sin anslutning var 45:e sekund. R3 upptäcker att den inte längre har kontakt och informerar R2&R1 som i sin tur skickar ett BGP Update-paket till ISP-Telia/Tele2 att nätet inte längre är nåbart. ISP-routrarna sätter då per default ett "Penalty"-värde för 200.200.20.0/24 prefixet (1000). Penalty-värdet börjar dock genast räkna ner mot 0 igen tack vare funktionen "Decay Algorithm" där default-värdet är 15min. Efter 15 minuter så har penalty-värdet halverats till 500, 15min efter det har det halverats till 250 osv. 

Detta värde nollställs ej när routen 45 sekunder senare blir nåbar igen. När vi tappar länken en andra gång, så informerar vi återigen ISP-Telia/Tele2. Säg att Penalty-värdet nu möjligtvis har hunnit räkna till 995. Den lägger då till ytterligare 1000 till Penalty-värdet (totalt 1995). När penalty-värdet för prefixet 200.200.20.0/24 är >=2000 så har vi nått "Suppress-limit", både ISP-Telia&Tele2 börjar då ignorera uppdateringar för det prefixet (suppressed) och sätter nätet som unreachable. Reuse-limit är per default 750, när vårt nät stabiliserat och decay-algorithm fått värdet att gå under detta så börjar nätet användas igen. Med default-värden kommer med andra ord ett nät som blivit suppressed vara så i minst 30 minuter (>2000/2 15min, 1000/2 15min -> 500) . Max-tiden för dampening är dock 60 minuter oavsett hur instabil en länk är. Default-värden:

*   Penalty - 1000
*   Suppress Limit - 2000
*   Reuse Liimit - 750
*   Decay Algorithm (Half-life) - 15 min

BGP ger oss möjligheten att modifiera alla dessa förutom Penalty-värdet! Vi aktiverar Route Dampening och justerar ovanstående värden genom:

```
router bgp _n_
bgp dampening ....
```
Det är dock viktigt att vi verifierar att värdena vi konfigurerar faktiskt är giltiga genom följande formel: max-penalty = reuse-limit * 2^(max-suppress-time/half-life) Säg att vi konfigurerat följande värden:

*   Penalty 1000
*   Suppress Limit 10000
*   Reuse Limit 1500
*   Half-Life 30min
*   Maximum suppression 60min

max-penalty = 1500*2^(60/30) = **6000**. 

Vi kommer med andra ord aldrig nå vår suppress-limit! Det går att kombinera dampening med route-maps om vi exempelvis endast vill använda dampening för ett enskilt prefix.

```
ISP-Tele2(config)#route-map 200_dampening permit 10
ISP-Tele2(config-route-map)#match ip address 10
ISP-Tele2(config-route-map)#set dampening ?
<1-45> half-life time for the penalty
ISP-Tele2(config-route-map)#set dampening 5 ?
 <1-20000> penalty to start reusing a route
ISP-Tele2(config-route-map)#set dampening 5 1900 ?
 <1-20000> penalty to start suppressing a route
ISP-Tele2(config-route-map)#set dampening 5 1900 2000 ?
 <1-255> Maximum duration to suppress a stable route
ISP-Tele2(config-route-map)#set dampening 5 1900 2000 10
ISP-Tele2(config)#router bgp 2200
ISP-Tele2(config-router)#bgp dampening route-map 200_dampening
```
Vi testar stänga ner Lo0 på R5 en gång:
```
ISP-Tele2#sh ip bgp 200.200.20.0
BGP routing table entry for 200.200.20.0/24, version 21
Paths: (1 available, best #1, table Default-IP-Routing-Table)
Flag: 0x820
 Not advertised to any peer
 400
 172.16.100.1 from 172.16.100.1 (172.16.100.1)
 Origin IGP, localpref 100, valid, external, best
 **Dampinfo: penalty 820, flapped 1 times in 00:01:27**
```
Vi gör det några gånger till för att få penalty-värdet över 2000:
```
ISP-Tele2#sh ip bgp 200.200.20.0
BGP routing table entry for 200.200.20.0/24, version 24
Paths: (1 available, no best path)
Flag: 0x820
 **Not advertised to any peer**
 **400, (suppressed due to dampening)**
 172.16.100.1 from 172.16.100.1 (172.16.100.1)
 Origin IGP, localpref 100, valid, external
 **Dampinfo: penalty 2278, flapped 3 times in 00:04:36, reuse in 00:01:20**
```
[![dampening](/assets/images/2013/08/dampening.png)](/assets/images/2013/08/dampening.png)

Vi kan rensa bgp dampening för ett prefix via kommandot "clear ip bgp dampening 200.200.20.0 255.255.255.0".
Däremot behålls penalty-värdet! Så skulle länken flappa ytterligare en gång kommer det med all sannolikhet blockeras direkt igen.
```
ISP-Tele2#show ip bgp dampening flap-statistics
 BGP table version is 25, local router ID is 30.0.7.1
 Status codes: s suppressed, d damped, h history, * valid, > best, i - internal,
 r RIB-failure, S Stale
 Origin codes: i - IGP, e - EGP, ? - incomplete
Network From Flaps Duration Reuse Path
 *> 200.200.20.0 172.16.100.1 3 00:07:38 400
```
Det går även att nollställa penalty-värdet via "clear ip bgp 1.1.1.1 flap-statistics", detta rensar dock penalty-värdet för alla routes vi lärt oss från den specifika neighborn. Det avslutar det sista jag hade om BGP just nu, har tagit upp det viktigaste av det lilla som nämns i CCNP Route men även Ciscos cert 642-661 BGP från CCIP.
