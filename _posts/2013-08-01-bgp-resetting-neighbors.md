---
title: BGP - Resetting Neighbors
date: 2013-08-01 10:10
comments: true
categories: [BGP]
tags: [soft-reset]
---
Detta blir en kortare post om de olika tillvägagångssätten vi kan använda oss av för att starta om en BGP-session. Varje gång vi lägger till exempelvis filtrering eller taggning av routinginformation så behöver vi som bekant starta om vår neighbor-relation för att förändringarna ska börja gälla. I våra labbar har jag oftast använt mig av "clear ip bgp *", detta river ju dock ner alla BGP-sessioner routern har vilket kanske inte är ett problem i en labbmiljö, men skulle du göra samma sak hos en ISP kan du nog börja leta efter ett nytt jobb.. :) En "hard reset" innebär att den lokala routern river ner neighbor-relationship, stänger TCP-sessionen och rensar sitt BGP-table från all information den lärt sig från neighborn, samma sak händer ju även hos vår neighbor givetvis. Vad kan vi använda oss av för att göra en "hard reset"?

*   neighbor _x.x.x.x_ shutdown
*   clear ip bgp *
*   clear ip bgp _n (as-nummer)_
*   clear ip bgp _x.x.x.x_ (ip-adress)
*   Starta om routern

Alternativet till detta är att istället göra en "**soft reset**" vilket behåller både tcp-sessionen & neighbor-adjacency, istället skickas endast nya BGP Update-paket! Dessa Update-paket justeras efter eventuella inbound/outbound-filters vi lagt till. Det finns flera olika tillvägagångssätt att göra en soft reset på:

*   clear ip bgp * soft
*   clear ip bgp _x.x.x.x_ soft

Men vi kan även specificera inbound/outbound om vi bara har utför ändringar i en riktning:

*   clear ip bgp _x.x.x.x_ out
*   clear ip bgp _x.x.x.x_ soft out
*   clear ip bgp _x.x.x.x_ in
*   clear ip bgp _x.x.x.x_ soft in

När vi använder oss av en softreset outbound behöver routern endast se över sitt BGP-table och generera nya BGP Update-paket enligt de filter vi konfigurerat. Men det blir inte riktigt lika enkelt när vi istället använder oss av inbound, för kom ihåg att BGP till skillnad mot RIP ej skickar routing-uppdateringar om den tror att vi redan har den senaste informationen. "clear ip bgp x.x.x.x soft in" är det äldre av de två kommandona vi kan använda oss av, och kräver att den lokala routern redan är konfigurerad med "neighbor x.x.x.x soft-reconfiguration inbound" mot den specifika neighborn vi vill göra soft reset mot. Detta kommando tvingar routern att  spara BGP Update-information från den specifika neighborn (vilket givetvis tar upp extra resurser) i sitt minne, routern använder sedan den cache'ade informationen när vi gör en soft reset för att bygga sitt BGP-table enligt de filter vi lagt in. "clear ip bgp x.x.x.x in" är en förbättrad version av samma kommando och tar bort kravet på att routern måste spara ner informationen. Istället används en ny funktion, "route refresh", vilket ber vår neighbor att skicka en full BGP Update.  Vi kan ta reda på om vår neighbor har stöd för "route refresh" via kommandot s_how ip bgp neighbor x.x.x.x_:

```
 BGP neighbor is 10.0.12.2, remote AS 500, internal link
 BGP version 4, remote router ID 191.0.0.9
 BGP state = Established, up for 00:39:13
 Last read 00:00:13, last write 00:00:13, hold time is 180, keepalive interval is 60 seconds
 Neighbor capabilities:
 **Route refresh: advertised and received(old & new)**
 Address family IPv4 Unicast: advertised and received
```
Såhär ser flödet ut om vi kollar i wireshark mellan R1 & R2: 
[![route-refresh-topology](/assets/images/2013/08/route-refresh-topology.png)](/assets/images/2013/08/route-refresh-topology.png) 

Vi kör en "clear ip bgp 10.0.12.2 in" i R1, detta genererar ett "ROUTE-REFRESH" paket: 
[![route-refresh](/assets/images/2013/08/route-refresh.png)](/assets/images/2013/08/route-refresh.png) 

R2 svarar sedan med ett BGP Update-paket: 
[![bgp-update](/assets/images/2013/08/bgp-update.png)](/assets/images/2013/08/bgp-update.png)