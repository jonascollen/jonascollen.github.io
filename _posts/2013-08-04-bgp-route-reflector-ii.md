---
title: BGP - Route Reflector II
date: 2013-08-04 19:09
comments: true
categories: [BGP]
tags: [route reflector]
---
Detta blir en fortsättning på gårdagens inlägg om [Confederations & Route Reflectors](http://roadtoccie.se/2013/08/03/bgp-confederations-route-reflectors/). Genom användandet av Route Reflectors får vi möjligheten att undvika de problem som annars uppstår på grund av iBGP's Split-Horizon. När vår RR mottager en routing-uppdatering hanterar den informationen enligt följande:

*   Härstammar paketet från en eBGP-peer vidarebefordras paketet till samtliga iBGP & eBGP-peers
*   Härstammar paketet från en iBGP-peer ("Non-Client") vidarebefordras paketet till samtliga eBGP-peers samt "Clients"
*   Härstammar paketet från en iBGP-peer ("Client") vidarebefordras paketet till samtliga peers (förutom avsändaren)

Jämför detta med en standard iBGP-peer som per default **EJ** skickar vidare någon routing-information! Vi konfigurerar om en neighbor är att betrakta som "Client" eller ej under BGP-processen för vår RR, men mer om detta senare. 
![routereflector-topology](/assets/images/2013/08/routereflector-topology.png) 

Redan vid en mindre topologi som den ovan blir det nästan löjligt många iBGP-peers vi behöver konfigurera upp för att vi ska få ett full-mesh förhållande. Men genom att istället använda oss av Route Reflectors räcker det att vi peerar mot en (eller flera för redundans) av de centrala routrarna som sedan vidarebefordrar routing-informationen till resterande routrar inom vårat AS. 

![routereflector-redundant](/assets/images/2013/08/routereflector-redundant.png) 
När vi får in en routing-uppdatering via eBGP vidarebefordrar "klienten" detta till samtliga RR, som i sin tur sprider denna information vidare till sina neighbors. Detta innebär att klienterna får dubbla uppdateringar, vilket dock inte är något problem om vi tänker efter. Våra routingprotokoll är ju vana vid att få flera alternativa vägar att välja på och ta beslut om vilken väg den anser är bäst. Något som dock kan bli problematiskt är om vi har 3-4+ RR i ett redundant nät. 

![routereflector-topology2](/assets/images/2013/08/routereflector-topology21.png)
I det här fallet så vidarebefordrar en "klient" sin uppdatering till sin RR, vilket skickar vidare informationen till alla peers (inkluderat övriga RR). Resterande RR's gör ju dock samma sak när de mottager paketet vilket vips leder till att vi har en loop i vårat nät. Vi kan lösa detta via ett iBGP-Attribut som kallas "**Cluster-ID**", en RR taggar då alla routing-uppdateringar den vidarebefordrar med just detta cluster-id (samt sitt router-id som originator).

Genom att gruppera våra RRs till ett och samma kluster ger det övriga RR's möjlighet att inspektera BGP Update-paket den mottager, skulle den se sitt eget cluster-id/router-id så ignoreras paketet!   Detta innebär dock som sagt att vi måste konfigurera neighbor-relations till samtliga RRs vi har i vårat nät. 

![routereflector-topology3](/assets/images/2013/08/routereflector-topology3.png)
Det går även bra att segmentera vårat AS och ha flera mindre klusters på ungefär samma sätt som vi använder Confederations, vi peerar sedan mellan RR/klustren för att sprida routing-informationen. Själva konfigurationen är oerhört enkel, det är snarare konceptet hur vi ska implementera detta på bästa sätt som är det svåra. För att konfigurera en router till RR  lägger vi till följande i våra neighbor-statements för våra "klienter"/övriga RRs:

```
router bgp _n_
 neighbor _x.x.x.x_ route-reflector-client
 bgp cluster-id _n_
```
Själva klienterna är helt omedvetna om att de faktiskt är en RR-klient och kräver ingen extra konfiguration för att detta skall fungera.