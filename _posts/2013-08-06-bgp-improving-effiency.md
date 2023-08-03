---
title: BGP - Improving effiency
date: 2013-08-06 23:08
comments: true
categories: [BGP]
tags: [path-mtu-discovery]
---
BGP ger oss möjligheten att justera en del parametrar för att effektivisera bgp processen och ge en bättre resurshantering. En router med 450,000+ routes att hantera uppskattar nog all hjälp den kan få. ;)

MTU-Discovery
-------------

BGP använder per default en MTU-size på endast 536 bytes (<Ios 12.2 visade det sig). I ett ethernet-segment har vi ju dock möjligheten att skicka 1500-bytes paket (1492 för PPPoE), och egentligen ännu större med hjälp av 
[Jumbo frames](http://en.wikipedia.org/wiki/Jumbo_frame) (om det är supportat). Genom att aktivera kommandot "ip tcp path-mtu-discovery" globalt på routern så aktiveras Path MTU Discovery vilket är en standardiserad teknik för att bestämma vilken MTU-size som stödjs mellan två hosts (i detta fall mellan routern och dess neighbor). Routern sätter då DF-biten (Don't Fragment) i IP-headern för alla utgående TCP-paket. Om någon enhet längs vägen till destinationen inte stödjer den MTU-size vi använt kan två saker hända:

1.  DF-biten är satt till 0 - paketet fragmenteras till en mindre MTU-size och skickas sedan vidare
2.  DF-biten är satt till 1 - Paketet droppas och returnerar en "ICMP - Fragmentation Needed (Type 3, Code 4)" innehållandes sin egen MTU-size

Då DF-biten i detta fall är satt till 1 så returneras ett Type 3-paket, vår router använder informationen till att justera ner sin egen MTU-size så den matchar. Det är med andra ord viktigt att vi tillåter ICMP Type 3-paket i vår brandvägg/ACL. Ett vanligt problem är annars att TCPs "3-way handshake" lyckas men sessionen sedan hänger sig när routern försöker skicka data, en så kallad "black hole-connection". 

![mtu-path-discovery](/assets/images/2013/08/mtu-path-discovery.png)

Denna process upprepas till MTU-sizen är tillräckligt liten och paketet nått hela vägen till destinationen. Cisco har en väldigt läsvärd whitepaper om IP Fragmentation & MTU-Discovery [här](http://www.cisco.com/image/gif/paws/25885/pmtud_ipfrag.pdf). Vi kan verifiera vad BGP valt att använda för värden via show ip bgp neighbors: 
[![mtu-path-discovery2](/assets/images/2013/08/mtu-path-discovery21.png)](/assets/images/2013/08/mtu-path-discovery21.png)
Haha.. Så går det när jag nöjer mig med att läsa teorin innan man testat det praktiskt. Här kan vi ju faktiskt ser att BGP valt en MTU-size 1460! Lite efterforskning visade att detta aktiveras per default för varje neighbor sedan [IOS v12.2](http://www.cisco.com/en/US/docs/ios/12_2sb/feature/guide/sbbgpmtu.html). Borde kanske ta och uppdatera mitt läsmaterial.. :D Vi ska nu för tiden kunna styra detta individuellt per neighbor om vi vill enligt den dokumentation jag hittat, men i den IOS-version (c3750 12.4(T15)) som jag använder är detta ej möjligt konstigt nog:

```
R2(config-router)#neighbor 10.10.12.1 transport ? 
 connection-mode Specify passive or active connection
```
Vi har dock fortfarande möjlighet att aktivera MTU Discovery globalt på routern om vi skulle vilja göra det för någon annan process så denna post inte blir totalt värdelös. :D
```
R2(config)#ip tcp path-mtu-discovery age-timer ?
 <10-30> Aging time
 infinite Disable pathmtu aging timer
```
Väljer vi infinte testar routern förbindelsen bara en gång och behåller sedan MTU-värdet, 30 sekunders intervaller är annars default.

Peer-groups
-----------

Med hjälp av Peer-groups kan vi minska systemresurserna som behövs vid skapandet av BGP Update-paket, det kan även förenkla vår konfiguration av neighbors, win-win! :) Peer-groups ger oss möjligheten att gruppera neighbor-syntaxer, som du säkert märkt vid det här laget så är det en hel del neighbor-syntaxer som upprepas gång på gång. Detta gör att routern sedan endast behöver generera ett BGP Update-paket som den sen kan replikera till samtliga grupp-medlemmar. Vi kan dock fortfarande ha individuell konfiguration för inkommande uppdateringar! Se följande konfiguration för en iBGP full mesh mellan fyra routrar till exempel:
```
router bgp 500
 no synchronization
 bgp log-neighbor-changes
 neighbor 2.2.2.2 remote-as 500
 neighbor 2.2.2.2 update-source Loopback0
 neighbor 2.2.2.2 next-hop-self
 neighbor 2.2.2.2 password cisco
 neighbor 3.3.3.3 remote-as 500
 neighbor 3.3.3.3 update-source Loopback0
 neighbor 3.3.3.3 next-hop-self
 neighbor 3.3.3.3 password cisco
 neighbor 4.4.4.4 remote-as 500
 neighbor 4.4.4.4 update-source Loopback0
 neighbor 4.4.4.4 next-hop-self
 neighbor 4.4.4.4 password cisco
 neighbor 172.16.51.5 remote-as 100
 no auto-summary
```
Vi ser att neighbor-syntaxerna är identiska för 2.2.2.2, 3.3.3.3 & 4.4.4.4, det är med andra ord ett perfekt läge att använda oss av en peer-group. Sedan IOS 12.0 så skapar routern detta själv "behind the scenes" även om du inte väljer att konfigurera detta själv. Vi kan verifiera detta med "show ip bgp update-group":
```
BGP version 4 update-group 1, external, Address Family: IPv4 Unicast
 BGP Update version : 0/0, messages 0
 Update messages formatted 0, replicated 0
 Number of NLRIs in the update sent: max 0, min 0
 Minimum time between advertisement runs is 30 seconds
 Has 1 member (* indicates the members currently being sent updates): 
 172.16.51.5
BGP version 4 update-group 2, internal, Address Family: IPv4 Unicast
 BGP Update version : 0/0, messages 0
 NEXT_HOP is always this router
 Update messages formatted 0, replicated 0
 Number of NLRIs in the update sent: max 0, min 0
 Minimum time between advertisement runs is 0 seconds
 Has 3 members (* indicates the members currently being sent updates): 
 2.2.2.2 3.3.3.3 4.4.4.4
```
Men rekommenderat är ändå att göra detta på egen hand och låta så lite som möjligt skötas "automatiskt" för att undvika fel (tänk no autonegotiate osv). Vi konfigurerar en peer-group enligt följande:
```
router bgp _n_
neighbor IBGP_ROUTERS peer-group
neighbor IBGP_ROUTERS remote-as 500
neighbor IBGP_ROUTERS update-source Loopback0
neighbor IBGP_ROUTERS next-hop-self
neighbor IBGP_ROUTERS password cisco
```
Våra neighbor-statements blir nu istället endast:
```
neighbor 2.2.2.2 peer-group IBGP_ROUTERS
neighbor 3.3.3.3 peer-group IBGP_ROUTERS
neighbor 4.4.4.4 peer-group IBGP_ROUTERS
neighbor 172.16.51.5 remote-as 100
```
Helt klar en mer "clean" konfig dessutom!
```
BGP peer-group is IBGP_ROUTERS, remote AS 500
 BGP version 4
 Default minimum time between advertisement runs is 0 seconds
For address family: IPv4 Unicast
 BGP neighbor is IBGP_ROUTERS, peer-group internal, members:
 2.2.2.2 3.3.3.3 4.4.4.4 
 Index 0, Offset 0, Mask 0x0
 NEXT_HOP is always this router
 Update messages formatted 0, replicated 0
 Number of NLRIs in the update sent: max 0, min 0
```
### Peer-Template

Det finns nu för tiden en form av "Peer-groups 2.0" - Peer template, det går dock ej att kombinera template med groups. Vi konfigurerar det via: router bgp _n_ template peer-policy _n_ [![peer-templates](/assets/images/2013/08/peer-templates.png)](/assets/images/2013/08/peer-templates.png) Alternativt session-relaterade kommandon:
```
router bgp _n_
template peer-session _n_
```
[![peer-sessions](/assets/images/2013/08/peer-sessions.png)](/assets/images/2013/08/peer-sessions.png) 
Vi aktiverar det sedan per neighbor genom:
```
neighbor 1.1.1.1 inherit peer-policy JONAS_POLICY
neighbor 1.1.1.1 inherit peer-session JONAS_SESSION
```
Input-Queues
------------

Ger oss möjligheten att justera hur många paket routern skall buffra tills processorn är tillgänglig innan den börjar använda "taildrops", dvs droppa de senast inkomna paketen. Default är 75, dvs. routern håller bara 75 paket i buffern innan den börjar droppa paket. Detta kan markant öka konvergenstiden när vi försöker sätta upp en BGP-session som använder "full routingtable"/450k+ routes då det är väldigt mycket data som måste processas.  Cisco rekommenderar att vi ökar detta till 1000, vilket görs på interface-nivå genom kommandot:

R2(config-if)#hold-queue 1000 in

Vi ska dock vara försiktiga med att sätta för stort värde då vi inte vill buffra TCP-Ack paket i onödan. Det är klart bättra att vi istället droppar dessa paket så avsändaren får möjlighet att justera sin window size, ett bra mellanvärde ska som sagt vara 1000.

BGP Scan-process
----------------

BGPs Scan-process var något jag nämnde lite kort i min introduktion till BGP som du hittar [här](http://roadtoccie.se/2013/07/21/bgp-the-basics/ "BGP – The basics"). BGP Scanner startar per default var 60:e sekund och går igenom hela BGP-table (sh ip bpg) och validerar att next-hop fortfarande är nåbart genom att kontrollera routerns [RIB](http://roadtoccie.se/2013/07/23/bgp-local-preference-rib-failures/).  Om en route upptäcks vara onåbar markeras den som "invalid", tas bort ur routing-table och informerar sina peers. Vi kan modifera detta intervall via kommandot "bgp scan-time" för att hitta invalid-routes snabbare och minska konvergenstiden , detta kan dock markant öka CPU-belastningen om BGP-table innehåller många routes.
```
R2(config-router)#bgp scan-time ?
 <5-60> Scanner interval (seconds)
```
BGP Advertisement-interval
--------------------------

Om routern behöver skicka ut en uppdatering till sina peers väntar den först ett förutbestämt antal sekunder innan den skickar uppdateringen, Detta ger routern möjlighet att samla ihop flera skiljda uppdateringar och skicka ut dessa samtidigt, exempelvis vid ett större nätfel eller dylikt. Exempelvis om vår BPG-Scanner upptäckt att vi inte längre kan nå next-hop adressen 200.0.21.1 för prefixet 191.16.0.0/16, väntar routern ett förutbestämt antal sekunder innan den skickar en uppdatering. Default-värdet är 30 sekunder för eBGP-peers & 5 sekunder för iBGP-peers. Vi kan ändra detta genom kommandot:
```
 R2(config-router)#neighbor 10.10.12.1 advertisement-interval ?
 <0-600> time in seconds
```
BGP Maximum-prefix
------------------

För att skydda oss mot exempelvis en felkonfigurering hos vår ISP har vi möjlighet att per neighbor bestämma det maximala antalet prefix vi accepterar att de skickar till oss. Vi kan som bekant kontrollera hur många prefix vi tar emot i dagsläget via kommandot sh ip bgp summary: 
[![prefixlimit](/assets/images/2013/08/prefixlimit.png)](/assets/images/2013/08/prefixlimit.png) 

Genom att konfigurera maximum-prefix behöver vi inte vara oroliga för att någon trött admin bakom exempelvis neighbor 10.10.12.1 skulle råka annonsera hela BGP-table på 450k routes till oss av misstag och kanske vår router.  :)
```
router bgp 64512
neighbor 10.10.12.1 maximum-prefix 20
```

Om vår neighbor överstiger det konfigurerade värdet stänger routern per default ner sessionen, och vi måste sedan manuellt göra en clear ip bgp för att få upp sessionen igen. Vi kan dock modifiera detta så att routern exempelvis först varnar vid en viss % av maxvärdet eller bara startar om sessionen.
```
R2(config-router)#neighbor 10.10.12.1 maximum-prefix 10 ?
 <1-100> Threshold value (%) at which to generate a warning msg
 restart Restart bgp connection after limit is exceeded
 warning-only Only give warning message when limit is exceeded
```
Next up - Route dampening. :)
