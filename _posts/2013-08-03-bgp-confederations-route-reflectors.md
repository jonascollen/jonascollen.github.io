---
title: BGP - Confederations & Route Reflectors
date: 2013-08-03 22:38
comments: true
categories: [BGP]
tags: [confederations,route reflector]
---
Confederations
--------------

![confederations](/assets/images/2013/08/confederations.png)
På grund av hur Split-Horizon fungerar (se tidigare inlägg om detta [här](http://roadtoccie.se/2013/07/07/bgp-internal-bgp-transitarea/) & [här](http://roadtoccie.se/2013/08/03/bgp-confederations-route-reflectors/)) så måste vi sätta upp våra iBGP-neighbors i ett full-mesh förhållande.

Detta blir redan vid bara fyra routrar som i ovanstående topologi en väldans massa neighbors att konfigurera (12). Men genom att använda oss "**BGP** **Confederations"** får vi möjligheten att dela upp vårat AS(400) i flera mindre interna AS! I detta exempel delades kontoren upp efter ort, men det går givetvis att bestämma helt och hållet själv efter egna preferenser. ISP-Telia & Tele2 kommer fortfarande endast peera med AS400 precis som tidigare. Confederations används nämligen endast inom vårat eget AS och confederation-taggen tas bort innan informationen skickas vidare till våra eBGP-neighbors (externa AS). Genom att dela upp kontoren i Västerås & Stockholm till varsitt AS minskar vi betydligt på antalet iBGP-neighbors vi behöver konfigurera, och gör nätet i allmänhet lättare att administrera. Nu är ju detta ett extremt litet nät, men om du istället tänker dig en ISP-mijö där varje router här kanske i verkligheten motsvarar 200-300 routrar så ser man ganska snabbt hur stökigt det kan bli om vi ej haft någon möjlighet att segmentera näten. Vilka AS-nummer ska vi då använda oss av för våra Confederation-grupper? 

Vi kan ju inte gärna bara ta några per random då vi med all sannolikhet kommer ta ett nummer som redan ägs av ett annat företag. Även om vi inte annonserar ut detta på internet så kommer vi själva inte kunna nå prefixen som tillhör det AS vi nu angett. Kom ihåg att BGP undviker routing-loopar genom att inspektera AS_Path - finner den sitt eget AS-nummer i en uppdatering så ignoreras den. [RFC 6996](https://tools.ietf.org/html/rfc6996) anger **AS 64512 - 65534 som Privata AS,** detta fungerar precis på samma sätt som konceptet med privata ip-adresser!  Låt oss därför sätta AS 64512 för Västerås-kontoret och AS 64513 för Stockholm. Genom att lägga R1-R3 i ett eget AS så innebär det att mellan R3 - R5 kommer upprättas en eBGP-koppling istället för iBGP! Detta innebär även att du nu måste konfigurera ebgp-multihop om du peerar mot loopback-adresser. BGP attribut, som exempelvis"Next Hop" eller "Local Preference", fungerar dock fortfarande som att de skulle vara **iBGP-peers**!

Konfigurering
-------------

Låt oss börja med R1.
```
router bgp 64512
bgp confederation identifier 400
 bgp confederation peers 64513 
 neighbor 10.10.12.2 remote-as 64512
 neighbor 172.16.100.2 remote-as 2200
```
bgp confederation identifier _n_ är det AS vi kommer annonsera till våra eBGP-peers, dvs ISP-Telia & Tele2. bgp confederation peers _n_ listar de confederation as-nummer du vill peera med, i detta fall har vi endast AS64513. Våra neighbor-statements ersätts nu med vårat confederation-as 64512 istället för AS 400. Svårare än så är det inte rent konfigmässigt! R2
```
router bgp 64512
bgp confederation identifier 400
 bgp confederation peers 64513 
 neighbor 10.10.12.1 remote-as 64512
 neighbor 10.10.23.3 remote-as 64512
 neighbor 172.32.200.2 remote-as 1500
```
R3
```
router bgp 64512
bgp confederation identifier 64512
 bgp confederation peers 64513 
 network 192.168.0.0
 neighbor 10.10.23.2 remote-as 64512
 neighbor 10.10.35.5 remote-as 64513
```
R5
```
router bgp 64513
bgp confederation identifier 400
 bgp confederation peers 64512 
 neighbor 10.10.35.3 remote-as 64512
```
Detta ger oss följande output i R5: 
![r5](/assets/images/2013/08/r5.png)

Intressant att notera här är AS_Path - routern känner igen att 64512 är ett "confederation"-AS och skrivs därav inom parentes. Vi saknar dock en hel del routes, kan du se varför utifrån konfigen ovan? iBGP Split-Horizon! R2 kommer aldrig skickade vidare den routing-information den fått från R1 via iBGP. Vi har här tre möjliga lösningar:

*   Vi skapar ett full-mesh neighbor-relationship inom vår confederation
*   Vi flyttar R3 till AS64513, R2 kommer då vidarebefordra uppdateringarna då länken betraktas som en eBGP-peer
*   Route-Reflectors

Att flytta R3 till AS64513 är givetvis fullt möjligt men samtidigt så har vi ju då inte längre någon indelning efter ort, och tillkommer det fler routrar senare till vår topologi kan det nog vara väldigt lätt att det blir fel förr eller senare. Att istället skapa en full mesh-lösning känns kanske enklast för oss just nu, främst då det i denna topologi endast innebär ytterligare ett neighbor-relationship mellan R1 & R3. Vi skulle dock istället kunna använda oss av en funktion kallad "Route-Reflectors".

Route-Reflectors
----------------

Genom att aktivera route-reflection stänger vi indirekt av iBGP's Split-Horizon, vilket tillåter oss vidarebefordra routing-uppdateringar till andra iBGP-peers. 
![routereflector](/assets/images/2013/08/routereflector.png)

Tar vi en närmare titt på vår topologi i Västerås ser vi att R2 är väl den som bäst skulle kunna fylla denna roll. Konfigurationen är oerhört enkel som mycket annat när det kommer till BGP, det är snarare själva planeringen som är det svåra. Då vi stänger av Split-horizon kan detta leda till enorma problem med routing-loopar om vi inte är väldigt försiktiga! Det gäller därför att verkligen tänka efter var vi implementerar detta. I detta fall finns det ingen risk att R3 lär sig routes från R1 från någon annan router än just R2, likaså för R1 gällande R3's routes. Vi aktiverar detta på R2 via:
```
router bgp 65435
 neighbor 10.10.12.1 route-reflector-client
 neighbor 10.10.23.3 route-reflector-client
```
Löjligt enkelt va? :) R1 kan nu se R3/R5's routes och vice versa: 
![r1-routereflector](/assets/images/2013/08/r1-routereflector.png) 

R5
![r5-routereflector](/assets/images/2013/08/r5-routereflector.png)

Det finns dock betydligt mer att gå över gällande Route-Reflectors, men det får bli i en senare post - troligtvis imorgon!