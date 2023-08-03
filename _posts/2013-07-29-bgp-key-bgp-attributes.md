---
title: BGP - Key BGP Attributes
date: 2013-07-29 20:13
comments: true
categories: [BGP]
---
Detta blir ett kort inlägg som sträcker sig lite utanför CCNP-nivån. BGP Attributes är information/metrics om en specifik route som ingår i "[BGP Update"-paketen](http://roadtoccie.se/2013/07/21/bgp-the-basics/) som skickas mellan peers. Dessa attribut används främst av BGPs "Best path selection"-process som vi gått igenom många gånger tidigare och är kategoriserade som antingen "Well-known" eller "Optional attributes".

*   "Well-known" (standard) - vilket måste stödjas av alla leverantörers BGP-implementering (Cisco, Juniper etc.)
*   Optional (proprietary) - stödjs endast av en/flera tillverkare

Well-known Attributes
---------------------

Well-known attribut är i sin tur uppdelade i två kategorier, **Mandatory** = Måste alltid finnas med, samt **Discretionary** vilket är "optional" och behöver ej ingå i update-paketen (men måste fortfarande stödjas av **alla leverantörer**!).

*   Origin * (igp/egp/?)
*   AS-Path *
*   Next-Hop *
*   Local Preference
*   Atomic Aggregate

* Mandatory - Måste alltid ingå i BGPs update-paket för att det skall vara giltigt Detta stämmer ju överens med den bild vi fått av [Local Preference](http://roadtoccie.se/2013/07/23/bgp-local-preference-rib-failures/) då vi vet att det endast gäller inom ett AS (iBGP) och skickas inte med i uppdateringar över eBGP.  Atomic aggregate används som du kanske redan listat ut för [route-summering](http://roadtoccie.se/2013/07/22/bgp-route-summarization/) och är med andra ord inget som alltid används, därav klassas även den som "Discretionary".

Optional Attributes
-------------------

*   Multi-Exit Descriminator (MED)
*   Aggregator
*   Community

Här är några exempel på några av de de vanligast implementerade"optional"-attributen, men som till skillnad från Well-known inte behöver stödjas, känner routern inte till attributet så ignoreras det bara. Detta ger möjligheten för tillverkare etc. att ta fram egna attribut, och detta är även den största orsaken att vi fortfarande använder BGPv4 som inte behövts uppgraderas till version 5 på över 10 år. Nya tillägg läggs hela tiden till, men pga att de räknas som optional attribut behöver "koden" inte skrivas om för "alla andra". Detta (och mycket annat) tas upp i videon från Google Tech Talks som jag länkat innan, väldigt sevärd! [https://www.youtube.com/watch?v=_Mn4kKVBdaM](https://www.youtube.com/watch?v=_Mn4kKVBdaM) Hur MED används och konfigureras har vi redan tittat på i ett tidigare inlägg, [se här](http://roadtoccie.se/2013/07/24/bgp-as_path-prepend-med/). Aggregator informerar om VILKEN router det är som gjort route-summeringen. Community ger oss möjligheten att dela upp ett AS i flera "communitys". Mer om detta i en senare post!