---
title: BGP - Route Manipulation via Communities
date: 2013-07-31 22:51
comments: true
categories: [BGP,Route Manipulation]
---
Laddat upp med en dubbel espresso för nu är det dags att gå över Communities. Vi har i tidigare poster gått igenom hur vi kan påverka routing-beslut i BGP via:

*   [Weight](http://roadtoccie.se/2013/07/23/bgp-path-selection-part-ii-weight/) _- Endast lokalt_
*   [Local Preference](http://roadtoccie.se/2013/07/23/bgp-local-preference-rib-failures/) _- Endast inom ett AS/iBGP_
*   [AS_Path Prepending & MED](http://roadtoccie.se/2013/07/24/bgp-as_path-prepend-med/) _- Skickas till externa AS/eBGP_

Gemensamt för både AS_PATH Prepending & MED är dock att din peer/ISP måste acceptera att du skickar dessa attribut, något som är långt ifrån varken best practice eller vanligt förekommande. Oftast väljer istället ISP:n att helt enkelt ignorera dessa attribut från dina routing-uppdateringar om du försöker lägga till detta. 
[![dualhomed](/assets/images/2013/07/dualhomed.png)](/assets/images/2013/07/dualhomed.png) 

Tänk dig följande exempel - du som kund har köpt en dual-homed uppkoppling med en primär länk på 100Mbit & backup på 20Mbit. Hur stor tror du risken är att Telia skulle gå in och manuellt modifiera Weight eller Local Preference för dina länkar (alt. låta dig använda MED)? Med 10,000+ kunder så skulle det bli en del route-maps att hålla reda på för dem.. ;) Det är istället här användandet av Communities kommer in i bilden. BGP Communitys är ett 32bitars värde, där i det "nya formatet" (ip bgp-community new-format) delas upp i två 16bitars värde. Best practice är att använda sitt AS-nummer i det första 16bitars fältet, och sedan använda det sista fältet enligt eget önskemål för att skapa sina communitys. Så för en ISP med AS 300 hade dess community-värden exempelvis kunnat vara:

*   300:1
*   300:10
*   300:25000
*   300:60000

Om du istället skulle vilja använda det "gamla formatet" så skrivs hela värdet ut i en och samma 32bitars sträng. 300:10 hade då blivit: 0000 0001 0010 1100 0000 0000 1010 Vilket omskrivet i decimalt blir - 1,228,810. Så vi kan antingen skicka community-taggen 300:10, eller använda oss av det gamla formatet och skicka taggen 1228810 för att få samma effekt- ganska lätt att se vad som är enklast! :) Vi annonserar sedan community-taggen tillsammans med övriga BGP Attribut som Origin/Next hop/AS-Path etc, Community räknas förövrigt som du kanske kommer ihåg från posten "[BGP Key Attributes](http://roadtoccie.se/2013/07/29/bgp-key-bgp-attributes/)"  tidigare i veckan som ett "**Optional Attribute**". 

Alla routrar har med andra ord inte stöd för detta! Genom communitys kan vi som kund "tagga" våra routing-uppdateringar enligt de värden som vår ISP tagit fram. ISPn kan i sin tur sedan använda en default route-map för alla sina kunder där de matchar Community-värden och ifrån det sätter exempelvis ett förutbestämt Weight eller Local Preference värde. En vanlig lösning verkar vara att ISPn skickar ett excel-blad med alla sina förutbestämda communitys och dess "åtgärder", något i stil med detta: 
[![Community-excel](/assets/images/2013/07/community-excel.png)](/assets/images/2013/07/community-excel.png)

Om vi som kund sedan taggar vårat nät 200.0.0.0/24 med community 300:10 kommer då ISPn att ändra Local Preference till 10. Taggar vi med community 300:201 så kommer ISPn istället att använda AS_Path Prepend och lägga till ytterligare ett AS-hopp. Vi kan även kombinera flera communities om vi så önskar, taggar vi samma route med 300:50 OCH 300:202 så ändras både Local Preference & AS_Path Prepending! Det är ganska lätt att se hur enormt mycket mer skalbart och lättanvänt detta är för vår ISP, alternativet hade ju varit att skapa individuella route-maps för varje kund som önskar någon form av routing-modifiering..

Labbexempel
-----------

[![MED](/assets/images/2013/07/med.png)](/assets/images/2013/07/med.png)
Vi tar och testar detta i praktiken med ovanstående topologi.

*   Länken mellan R1 & R3 är på 50M
*   Länken mellan R1 & R4 är på 2M
*   Länken mellan R2 & R4 är på 100M

Vi vill således att vår ISP använder länken via R2-R4 primärt för trafik som kommer från "the Internet". Att modifiera Weight eller AS_Path-Prepending fungerar ej i detta fall, vår ISP måste istället använda Local Preference. Om du inte förstår varför så föreslår jag att du går tillbaka och läser posterna relaterat till detta länkat överst i dedå det är rätt basic vid det här laget. ;) Vår ISP har skickat följande lista med communities vi kan använda oss av: [
![Community-excel.2png](/assets/images/2013/07/community-excel-2png.png)](/assets/images/2013/07/community-excel-2png.png)

Vi börjar med att konfigurera upp vår egen utrustning så kollar vi senare på hur ISPn hanterar detta. Med ren grundkonfig kan vi se att ISP just nu föredrar den sämre länken via R1 på 50Mbit för att nå nätet 10.0.12.0/24. 
[![bgp-community-default](/assets/images/2013/07/bgp-community-default.png)](/assets/images/2013/07/bgp-community-default.png)

Vi börjar med att konfigurera R1 - men vad är det vi vill åstadkomma? Kom ihåg att för Local Preference så är högst värde bäst (default 100), vi vill med andra ord att ISP ändrar Local Preference till <100 för länken mellan R1 - R3, och sedan sätter ett ännu lägre värde för länken mellan R1 - R4.  Enligt excel-listan behöver vi då sätta följande communities:

*   Länken mellan R1 - R3 - community 65000:50
*   Länken mellan R1 - R4 - community 65000:10

Vi börjar dock med en access-lista:

`access-list 1 permit 10.0.12.0 0.0.0.255`

Sedan en route-map:
```
route-map PeerToR3 permit 10
match ip address 1
 set community 65000:50
route-map PeerToR4 permit 10
match ip address 1
 set community 65000:10
```
Hur får vi då routern att skicka communities till sina peers? Vi måste först aktivera "bgp-community new format" då vi använt oss av det nya formatet att skriva communities:

`ip bgp-community new-format`

Vi behöver sedan bara konfigurera följande mot våra peers:
```
router bgp 500
neighbor 191.0.0.2 route-map PeerToR3 **out**
neighbor 191.0.0.2 send-community
neighbor 191.0.0.6 route-map PeerToR4 **out**
neighbor 191.0.0.6 send-community
```
Vi gör i princip samma sak för R2, men här behöver vi egentligen inte ändra LP då vi sänkt de andra länkarna till <100, men det är ju bra träning om inte annat.. :)
```
access-list 1 permit 10.0.12.0 0.0.0.255
route-map PeerToR4 permit 10
match ip address 1
 set community 65000:110
ip bgp-community new-format
router bgp 500
neighbor 191.0.0.10 route-map PeerToR4 **out**
neighbor 191.0.0.10 send-community
```
Svårare än så är det inte! Hur har då vår ISP konfigurerat sitt nät? Vi skapar först något som kallas "community-list", här specificerar vi våra communities så vi sedan kan använda oss av dessa i route-maps:
```
ip community-list 1 permit 65000:10
ip community-list 2 permit 65000:50
ip community-list 3 permit 65000:110
```
Dessa fungerar precis som en access-lista och det finns möjlighet att göra både standard & extended-versioner:
```
R3(config)#ip community-list ?
 <1-99> Community list number (standard)
 <100-500> Community list number (expanded)
 expanded Add an expanded community-list entry
 standard Add a standard community-list entry
```
Vi skapar sedan en default route-map vi använder till alla våra kunder:
```
route-map CUSTOMER_DEFAULT permit 10
match community 1
 set local-preference 10
route-map CUSTOMER_DEFAULT permit 20
match community 2
 set local-preference 50
route-map CUSTOMER_DEFAULT permit  30
match community 3
 set local-preference 110
route-map CUSTOMER_DEFAULT permit 40
```
Inget konstigt direkt, istället för match ip address där vi sedan använt oss av en access-lista använder vi här istället match community, vilket hänvisar till den ip community-list vi tidigare skapat.
```
ip bgp-community new-format
router bgp 65000
neighbor x.x.x.x route-map CUSTOMER_DEFAULT **in**
```
Detta ger följande resultat: 

R3
[![bgp-community-finalr3](/assets/images/2013/07/bgp-community-finalr3.png)](/assets/images/2013/07/bgp-community-finalr3.png)

R4
[![bgp-community-finalr4](/assets/images/2013/07/bgp-community-finalr4.png)](/assets/images/2013/07/bgp-community-finalr4.png)

R5
[![bgp-community-finalr5](/assets/images/2013/07/bgp-community-finalr5.png)](/assets/images/2013/07/bgp-community-finalr5.png) 

Vackert! Vi behöver givetvis inte begränsa oss till endast ändra Weight/Local Preference, möjligheterna är rätt rejäla ;)

```
R5(config-route-map)#set ?
 as-path Prepend string for a BGP AS-path attribute
 automatic-tag Automatically compute TAG value
 clns OSI summary address
 comm-list set BGP community list (for deletion)
 community BGP community attribute
 dampening Set BGP route flap dampening parameters
 default Set default information
 extcommunity BGP extended community attribute
 interface Output interface
 ip IP specific information
 ipv6 IPv6 specific information
 level Where to import route
 local-preference BGP local preference path attribute
 metric Metric value for destination routing protocol
 metric-type Type of metric for destination routing protocol
 mpls-label Set MPLS label for prefix
 nlri BGP NLRI type
 origin BGP origin code
 tag Tag value for destination routing protocol
 traffic-index BGP traffic classification number for accounting
 vrf Define VRF name
 weight BGP weight for routing table
```
Det får avsluta communities för den här gången, nästa post blir troligtvis om Route-Reflectors eller Confederations.
