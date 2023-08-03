---
title: BGP - The basics
date: 2013-07-21 18:32
comments: true
categories: [BGP]
---
BGP (**B**order **G**ateway **P**rotocol) är tillskillnad från RIP/OSPF/EIGRP/ISIS ett External/Exterior Gateway Protocol och är just nu uppe i version 4 som släpptes redan 1994. Det främsta tillägget vid v4 var möjligheten att använda CIDR & Route aggregation (CIDR fanns inte ens som en RFC innan det släpptes i BGPv4). När behöver vi då använda oss av BGP? Här är ett exempel på några bra grundregler:

> Use BGP if at least one of the following conditions exists:
> 
> *   The AS allows packets to transit through it to reach other autonomous systems
> *   The AS has multiple connections to other autonomous systems
> *   The flow of traffic entering and leaving the AS must be manipulated
> 
> When NOT to use BGP:
> 
> *   A single connection to the Internet or another AS
> *   Lack of memory or processor power on routers to handle constant BGP updates
> *   You have limited understanding of route filtering and the BGP path-selection process
> *   Low bandwith between autonomous systems

Här är en väldigt sevärd video från en av skaparna av protokollet, Dr. Yakov Rekhter, där han berättar lite om historiken bakom BGP och annat matnyttigt från Googles Tech Talks: [BGP - Protocol Design](https://www.youtube.com/watch?v=_Mn4kKVBdaM)

Basics
------

*   Använder sig av TCP (port 179)
*   Är ett "Path Vector Protocol"
*   Alla paket skickas som unicast
*   Till skillnad från IGPs använder sig BGP inte (endast) av metric för att räkna ut den bästa vägen
*   Neighbors måste konfigureras manuellt, behöver dock ej vara på samma subnät
*   Endast en TCP-session per neighbor (trots redundanta länkar)
*   Använder keepalives och hold timers (default 60/180 sec)
*   När neighbor-relation formas mellan två AS används External BGP (eBGP)
*   När neighbor-relation formas inom samma AS används Internal BGP (iBGP)
*   Användandet av Loopback-interface för neighbor-statements är rekommenderat när det finns redundanta länkar

Här finns förresten en bra förklaring av vad ett [Autonomous System](https://en.wikipedia.org/wiki/Autonomous_System_(Internet)) är. En full BGP routing table består i dagsläget av närmare 400,000 routes(!). 

[![BGP_Table_growth.svg](/assets/images/2013/07/bgp_table_growth-svg.png)](/assets/images/2013/07/bgp_table_growth-svg.png)

Processer
---------

BGP drivs av fyra processer:

*   BGP Open - Startar peering mot våra neighbors
*   BGP I/O - Förbereder / Behandlar BGP Updates & Keepalives
*   BGP Scanner - Kontrollerar next-hop addresser och bestämmer över vilka routes som skall annonnseras
*   BGP Router - Räknar ut "Best path", hanterar routingförändringar

BGP Scanner startar periodvis för att verifiera att vi fortfarande kan nå next-hop addresser för routes i vår routing-table.  Detta kan leda till att andra processer som körs samtidigt (ex icmp) påverkas, trafik som färdas GENOM routern ska dock ej påverkas då det (bör) hanteras av CEF (Cisco Express Forwarding).

Tables
------

*   BGP Table
*   Neighbor Table
*   Routing Table

Precis som för ex. OSPF så sparas alla BGP prefix & Path Attributes i BGP Table. Routern utför sedan egna beräkningar över vilken route som är bäst och som i sin tur installeras i routing table. OBS - En stor skillnad mot just OSPF är dock att BGP endast annonserar den bästa routen till sina neighbors för en viss destination och bortser från alla övriga möjliga vägar den eventuellt känner till.

BGP Message types
-----------------

De message-types som BGP använder sig av för att upprätta neighbor relations och utbyta routing-information är följande:

*   Open - Skickas efter vi manuellt konfigurerat en neighbor och en TCP-session har etablerats och kan liknas det Hello-paket OSPF/EIGRP skickar.
*   Update - Används för att utbyta routing information
*   Keepalive - Används för att upprätthålla neighbor-sessionen och resettar holdtimern
*   Notification - Används när ett fel inträffat och resettar neighbor-sessionen

Lås oss använda oss av följande topologi och titta närmare på dessa paket:
![bgp basics](/assets/images/2013/07/bgp-basics.png)

### Open-message

BGP's Open-message innehåller vissa parametrar (där vissa måste matchas med den angivna neighborn för att en adjacency skall bildas), dessa är:

*   Version (8-bit field) - BGP Version nummer (2,3 eller 4). Försöker du peera med en neighbor som använder sig av en tidigare version nekas Open-paketet, routern kommer istället försöka skicka med en äldre version (3), paketet accepteras inte förrän versionerna stämmer överens. BGP är därmed bakåtkompitabelt med äldre versioner, men oddsen att du skall träffa på detta är väl mer eller mindre obefintligt nu för tiden då v4 släpptes redan 1994(!)
*   My autonomous system (16-bit field) - Avsändarens Autonomous System-nummer
*   Hold Time (16-bit field) - Om Hold Timer skiljer sig används den lägst angivna tiden av båda, default hos cisco är 180 sek. Kan sättas till 0, keepalive kommer då ej att skickas.
*   BGP Identifier (32-bit field) - Avsändarens Router-ID
*   Optional Parameters- För att annonsera support för "optional capabilites", ex. authentisering, multiprotocol support, route refresh m.m)

Avsändarens ip-adress & AS-nummer måste matchas med det neighbor-statement den mottagande routern har samt ev. lösenord om autentisering aktiverats, router-id får inte heller vara samma på de båda routrarna. Nedan visar ett Open-paket skickat från R5 till R1: 
![open packet](/assets/images/2013/07/open-packet.png)

### Router-ID

BGP väljer sitt Router-ID precis som OSPF enligt följande princip:

1.  Om vi använt kommandot bgp router-id _rid_
2.  Den högsta Loopback-adressen vars interface är Up/Up
3.  Det högsta adressen av routerns alla övriga interface och som är Up/Up

### Update-message

Innehåller information om en specifik AS-path och all dess tillhörande routing-information med exempelvis vilka nät som finns nåbara inom detta, finns flera paths skickas det ett specifikt Update-paket för varje path. Kan innehålla följande fält:

*   Withdrawn Routes - En lista med adresser som skall tas bort från BGPs routing process (ej längre nåbara etc)
*   Path attributes - AS-Path, Origin, Local Preference
*   Network Layer Reachability Information - Innehåller en lista med ip-adress prefix som kan nås via denna AS-path

Om vi utökar vår topologi och annonserar nätet 5.5.5.0/24 från R5 och 6.6.6.0/24 från R6 in till BGP kan vi se följande Update-paket skickat från R1 till R5: 
![update packet](/assets/images/2013/07/update-packet.png) ![update packet wireshark](/assets/images/2013/07/update-packet-wireshark.png)

### Neighbor-states

BGP har även fem olika states den genomgår innan vi har full adjacency (Established) mellan två neighbors, detta kan liknas vid OSPF-processen där vi går från 2Way-state till Exstart-state osv.

*   Idle - Vi har konfigurerat en neighbor men BGP har inte initierats ännu
*   Connect - BGP väntar på att en TCP-session ska upprättas
*   Active - TCP-sessionen är upprättad men ingen BGP-information har skickats ännu
*   OpenSent - Open-paket skickat, TCP-sessionen är etablerad men inget Open-paket har mottagits
*   OpenConfirm - Open-paket är både skickat & mottaget, TCP-sessionen är etablerad. Routern väntar nu på ett BGP Keepalive-paket för att bekräfta att alla neighbor-relaterade parametrar matchades, eller ett Notification-paket som informerar om något fel inträffat (parameter mismatch t.ex.)
*   Established - Alla neighbor-parametrar accepterades och vi har nu full neighbor adjacency. Routern börjar nu skicka BGP Update-paket.

Vi kan verifiera detta genom kommandot sh ip bgp summary eller sh ip bgp neighbors.
```
R5#sh ip bgp summary
BGP router identifier 5.5.5.5, local AS number 100
BGP table version is 5, main routing table version 5
2 network entries using 240 bytes of memory
2 path entries using 104 bytes of memory
3/2 BGP path/bestpath attribute entries using 372 bytes of memory
1 BGP AS-PATH entries using 24 bytes of memory
0 BGP route-map cache entries using 0 bytes of memory
0 BGP filter-list cache entries using 0 bytes of memory
Bitfield cache entries: current 1 (at peak 2) using 32 bytes of memory
BGP using 772 total bytes of memory
BGP activity 3/1 prefixes, 3/1 paths, scan interval 60 secs
Neighbor V AS MsgRcvd MsgSent TblVer InQ OutQ Up/Down State/PfxRcd
172.16.51.1 4 500 20 20 5 0 0 **00:16:00** 1

R5#sh ip bgp neighbors 
BGP neighbor is 172.16.51.1, remote AS 500, external link
 BGP version 4, remote router ID 1.1.1.1
 **BGP state = Established, up for 00:16:33**
 Last read 00:00:32, last write 00:00:32, hold time is 180, keepalive interval is 60 seconds
 Neighbor capabilities:
 Route refresh: advertised and received(old & new)
 Address family IPv4 Unicast: advertised and received
 Message statistics:
 InQ depth is 0
 OutQ depth is 0
 Sent Rcvd
 Opens: 1 1
 Notifications: 0 0
 Updates: 1 1
 Keepalives: 19 19
 Route Refresh: 0 0
 Total: 21 21
 Default minimum time between advertisement runs is 30 seconds
For address family: IPv4 Unicast
 BGP table version 5, neighbor version 5/0
 Output queue size: 0
 Index 1, Offset 0, Mask 0x2
 1 update-group member
 Sent Rcvd
 Prefix activity: ---- ----
 Prefixes Current: 1 1 (Consumes 52 bytes)
 Prefixes Total: 1 1
 Implicit Withdraw: 0 0
 Explicit Withdraw: 0 0
 Used as bestpath: n/a 1
 Used as multipath: n/a 0
Outbound Inbound
 Local Policy Denied Prefixes: -------- -------
 Bestpath from this peer: 1 n/a
 Total: 1 0
 Number of NLRIs in the update sent: max 1, min 1
**Connections established 1; dropped 0**
 Last reset never
**Connection state is ESTAB,** I/O status: 1, unread input bytes: 0 
Connection is ECN Disabled, Mininum incoming TTL 0, Outgoing TTL 1
Local host: 172.16.51.5, Local port: 179
Foreign host: 172.16.51.1, Foreign port: 27320
Connection tableid (VRF): 0
Enqueued packets for retransmit: 0, input: 0 mis-ordered: 0 (0 bytes)
Event Timers (current time is 0x3F7670):
Timer Starts Wakeups Next
Retrans 21 0 0x0
TimeWait 0 0 0x0
AckHold 20 17 0x0
SendWnd 0 0 0x0
KeepAlive 0 0 0x0
GiveUp 0 0 0x0
PmtuAger 0 0 0x0
DeadWait 0 0 0x0
Linger 0 0 0x0
ProcessQ 0 0 0x0
iss: 580665860 snduna: 580666319 sndnxt: 580666319 sndwnd: 15926
irs: 414342032 rcvnxt: 414342486 rcvwnd: 15931 delrcvwnd: 453

SRTT: 282 ms, RTTO: 429 ms, RTV: 147 ms, KRTT: 0 ms
minRTT: 12 ms, maxRTT: 300 ms, ACK hold: 200 ms
Status Flags: passive open, gen tcbs
Option Flags: nagle
IP Precedence value : 6
Datagrams (max data segment is 1460 bytes):
Rcvd: 24 (out of order: 0), with data: 21, total data bytes: 453
Sent: 39 (retransmit: 0, fastretransmit: 0, partialack: 0, Second Congestion: 0), with data: 21, total data bytes: 458
 Packets received in fast path: 0, fast processed: 0, slow path: 0
 Packets send in fast path: 0
 fast lock acquisition failures: 0, slow path: 0
```
Vi kan även kontrollera att TCP-sessionen är igång genom kommandot:
```
R5#sh tcp brief
TCB Local Address Foreign Address (state)
66B535F4 172.16.51.5.179 172.16.51.1.27320 ESTAB
```
I den tidigare posten [BGP - Internal BGP & Transit-Areas](http://www.jonascollen.se/posts/bgp-internal-bgp-transitarea/)  kan du se flera exempel på hur vi konfigurerar neighbor-relationships.

Cisco's Enterprise Definitioner
-------------------------------

I det fall vi behöver använda oss av BGP benämner Cisco nätmodellerna enligt följande för "Enterprise Customers" (vilket CCNP Route inriktar sig på): Single Homed - 1 länk per ISP, 1 ISP Dual Homed - 2+ länkar per ISP, 1 ISP Single Multihomed - 1 länk per ISP, 2+ ISPs Dual Multihomed - 2+ länkar per ISP, 2+ ISPs

### Single Homed

[![singlehomed](/assets/images/2013/07/singlehomed.png)](/assets/images/2013/07/singlehomed.png)
När vi endast har en länk mellan vårat företag och ISP. Här finns det endast en möjlig nexthop-adress för alla externa adresser och det finns med andra ord ingen egentlig anledning att använda oss av BGP. De vanligaste lösningarna innebär att vi antingen:

*   Vi använder en statisk default route i vår CPE som pekar på ISP-A,  ISP-A konfigurerar i sin tur en statiskt route för vårat publika nät
*   Vi använder oss av BGP, men ISP-A annonserar endast en default-route till oss, och vi annonserar vårat publika nät till ISP-A

Båda dessa lösningar har dock en nackdel att interna paket vars destination är någonstans internt på nätverket men som inte hittar en matchande route, kommer forwardas ut på "internet" via vår default-route innan vi når en unreachable hos ISP-A.  Cisco rekommenderar därför att vi skapar en "discard route" för våra interna nät, genom att sätta en statisk route för exempelvis 10.0.0.0/8-nätet som pekar på Null0.

### Dual Homed

[![dualhomed](/assets/images/2013/07/dualhomed.png)](/assets/images/2013/07/dualhomed.png)
[![dualhomed2](/assets/images/2013/07/dualhomed2.png)](/assets/images/2013/07/dualhomed2.png)

När vi har två upplänkar till en och samma ISP kallas detta Dual homed. Här har vi nu möjlighet att influera vilken väg vi vill att paket skall ta, men om vi endast vill utföra något av följande:

*   Föredra en uppkoppling för samtliga destinationer, men om denna länk går ner så skall trafiken slå över till den andra länken
*   Lastbalansera 50/50 mellan de båda länkarna, men om en länk går ner så skall trafiken slå över till den andra länken

I båda dessa fallen duger det gott och väl med användandet av statiska routes! Om vi istället vill specificera vilken länk vi ska använda för specifika destinationer kommer BGP in i bilden. Låt oss bygga vidare lite på vår topologi för att enklare förklara: 
[![dualhomed google](/assets/images/2013/07/dualhomed-google.png)](/assets/images/2013/07/dualhomed-google.png) 

Vi kan här se att den snabbaste vägen för att ta oss till Google är via ISP-A2, istället för att gå omvägen via ISP-A1 och sedan genom "molnet" för att tillslut hamna hos Google. För att lyckas med detta måste vi modifiera BGPs "Path Attributes" för att internt få vägen via CustomerABC-2 mer attraktiv. Detta kommer dock bli en en egen post då det är ett väldigt brett ämne.. Något som är värt att notera är att om vi har ett core-nät bakom våra ABC-routrar är det mycket möjligt att vi även behöver använda oss av iBGP där också för att undvika eventuella routing-loopar som annars kan uppstå.

### Single Multihomed

[![singlemultihomed2](/assets/images/2013/07/singlemultihomed2.png)](/assets/images/2013/07/singlemultihomed2.png) 
[![singlemultihomed](/assets/images/2013/07/singlemultihomed.png)](/assets/images/2013/07/singlemultihomed.png)

### Dual Multihomed

[![dualmultihomed](/assets/images/2013/07/dualmultihomed.png)](/assets/images/2013/07/dualmultihomed.png)

### Partial & Full Updates

En ISP ger i de flesta fall tre olika möjligheter för att utbyta BGP-information. Som vi tidigare nämnt börjar en full BGP table nu närma sig 450,000 routes vilket ställer enorma krav på hårdvaran. Ska du dessutom peera med två ISPs med en och samma router kommer den ha en bgp table per neighbor, det blir dvs 900,000 routes totalt som routern ska orka hantera. :) De alternativ som finns är:

*   Default route only - Inte så svår att lista ut, ISPn annonserar endast en default-route via BGP
*   Full updates - ISPn annonserar hela BGP tablen
*   Partial Updates - ISPn annonserar endast specifikt valda prefix samt en default-route via BPG

Partial Updates ger fördelen att vi har möjligheten att välja en "bättre" route för ett specifikt nät vi är intresserad av utan att behöva spara alla övriga routes i vår egen utrustning.
