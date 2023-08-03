---
title: EIGRP - Query-packet, Stubzone & SIA
date: 2013-06-10 20:04
comments: true
categories: [EIGRP]
tags: [sia, stubzone]
---
Query-packet
------------

När EIGRP tappar en successor-route och det ej finns någon feasible succesor ändras routen från "Passive" till "Active" i topology-table och routern börjar sedan skicka ut Query's på samtliga interface (exkl. successorn) där den frågar om någon annan neighbor känner till detta nät. Ta topologin nedan som exempel: 
[![query1](/assets/images/2013/06/query11.jpg)](/assets/images/2013/06/query11.jpg) 

R1 tappar sin länk och har inte längre en route till 10.0.0.0/24-nätet. Det finns ingen feasible successor då det endast är R1 som är anslutet till detta nät, 10.0.0.0/24 kommer därför ändras från Passive till Active och R1 kommer skicka ut Query's på övriga interface till R2 & R3 och fråga om de känner till detta nät. R2 & R3 saknar kännedom och kommer därför själva skicka en query till sina neighbors (R4, R5, R6, R7). 

[![query2](/assets/images/2013/06/query2.jpg)](/assets/images/2013/06/query2.jpg) 

R2 & R3 kommer inte svara R1 innan de själva fått svar från sina neighbors R4-7. 
[![query3](/assets/images/2013/06/query3.jpg)](/assets/images/2013/06/query3.jpg) 

Detta är ju ett relativt litet nät, men det är ganska lätt att inse hur långsamt och ineffektivt detta är när vi börjar prata om riktigt stora nät. Även om R1 skulle få tillbaka ett svar från en neighbor om en tillgänglig route till just 10.0.0.0/24 så kommer den EJ att använda denna innan den fått svar från samtliga neighbors den skickat query till. Vad händer då om R1 inte får svar på en Query? Låt os säga att länken mellan R3 & R7 är rejält överbelastad för stunden och har därför kraftig packetloss. Låt oss säga att Query-paketet som skickades från R3 försvinner på vägen och R7 känner därför inte till att både R1 & R3 sitter och väntar på att den skall svara. Detta fel kallas "**Stuck-in-active**".

Stuck-in-Active
---------------

Då EIGRP använder sig av RTP förväntar den sig alltid respons på Query-paket inom **3 minuter** (går att justera med kommandot "_**timers active**_" under router eigrp x. Får den inget svar under den tiden kommer den döda sin neighbor-adjacency med den specifika routern, detta leder ju i sin tur till att den tappar alla routes den lärt sig från den specifika router och kommer på så vis behöva skicka ytterligare Querys ut på nätverket, inte helt optimalt! Om vi tar nedanstående topologi som exempel så kan vi följa vad som händer: 
[![sia](/assets/images/2013/06/sia.jpg)](/assets/images/2013/06/sia.jpg)

1.  R1 tappar sin länk och saknar feasible successor
2.  R1 skickar en Query ang den specifika routen till R2
3.  R2 har ingen information om routern och skickar därför en Query vidare till R3
4.  R3 har ingen info och har inte heller någon annan neighbor, den skickar därför ett svar tillbaka till R2 med att den ej känner till nätet
5.  R2 skickar sitt Reply-paket till R1, men pga exempelvis dålig radiolänk tappas detta paket
6.  Efter 3 minuter kommer R1 döda sin neighbor adjacency med R2 inkluderat alla routes den lärt sig från just R2

I senare versioner av Cisco's IOS (>12.1) så infördes två nya paket, **SIA query** & **SIA reply**. Om R1 i detta fall inte fått något svar på sitt Query-paket efter 1½ minut kommer den skicka en SIA Query till R2 och fråga vad som händer, R2 svarar med en SIA reply och neighbor adjacencys behålls uppe.

Stubzone & Summary-routes
-------------------------

Hur kommer vi då till rätta med det här problemet med att Query-paket fullständigt svämmar över nätverket när en route går ner? Det finns två alternativ, och det absolut enklaste är via summary-routes. När du skapar en summary-route installerades denna i routing-table och pekar till Null0. Låt oss bygga upp följande topologi och kolla hur det ser ut i respektive router (skippade dock R5 + R6). 
[![query1](/assets/images/2013/06/query11.jpg)](/assets/images/2013/06/query11.jpg) Såhär ser routing-table ut för R7:

```
_Gateway of last resort is not set_
_172.16.0.0/30 is subnetted, 4 subnets_
_D 172.16.0.4 [90/358400] via 172.16.1.5, 00:07:49, FastEthernet0/1_
_C 172.16.1.4 is directly connected, FastEthernet0/1_
_D 172.16.0.0 [90/332800] via 172.16.1.5, 00:07:49, FastEthernet0/1_
_D 172.16.1.0 [90/307200] via 172.16.1.5, 00:07:49, FastEthernet0/1_
 _10.0.0.0/24 is subnetted, 1 subnets_
_D 10.0.0.0 [90/309760] via 172.16.1.5, 00:01:02, FastEthernet0/1_
```
Om vi nu stänger ner länken till 10.0.0.0/24 kommer vi se följande om vi kör _debug eigrp packets query update reply på R7_:
```
_*Mar 1 00:10:44.107: EIGRP: Received QUERY on FastEthernet0/1 nbr 172.16.1.5_
_*Mar 1 00:10:44.111: AS 10, Flags 0x0, Seq 16/0 idbQ 0/0 iidbQ un/rely 0/0 peerQ un/rely 0/0_
_*Mar 1 00:10:44.123: EIGRP: Enqueueing REPLY on FastEthernet0/1 nbr 172.16.1.5 iidbQ un/rely 0/1 peerQ un/rely 0/0 serno 6-6_
_*Mar 1 00:10:44.131: EIGRP: Sending REPLY on FastEthernet0/1 nbr 172.16.1.5_
_*Mar 1 00:10:44.135: AS 10, Flags 0x0, Seq 6/16 idbQ 0/0 iidbQ un/rely 0/0 peerQ un/rely 0/1 serno 6-6_
```
  Kollar vi i Wireshark ser Query-paketet ur såhär: 
[![r7 query](/assets/images/2013/06/r7-query.png)](/assets/images/2013/06/r7-query.png) 
**R3** skickar ett multicast till 224.0.0.10 för att annonsera att den inte längre kan nå nätet. 

**R7** svarar först med ett Ack och sedan med ett Reply-paket som ser mer eller mindre identiskt ut: 
[![r7 reply](/assets/images/2013/06/r7-reply.png)](/assets/images/2013/06/r7-reply.png) 

Skillnaden är egentligen att **R7** skickar sitt svar till **R3** (obs! ej direkt till **R1**) som ett unicast men anger också att nätet är unreachable. Kollar vi sedan på R1 ser vi att vi får svar från både R2 & R3.
```
_*Mar 1 00:51:09.835: EIGRP: Received REPLY on FastEthernet0/1 nbr 172.16.1.2_
_*Mar 1 00:51:09.839: AS 10, Flags 0x0, Seq 27/41 idbQ 0/0 iidbQ un/rely 0/0 peerQ un/rely 0/0_
_*Mar 1 00:51:09.843: EIGRP: Received REPLY on FastEthernet0/0 nbr 172.16.0.2_
_*Mar 1 00:51:09.847: AS 10, Flags 0x0, Seq 44/42 idbQ 0/0 iidbQ un/rely 0/0 peerQ un/rely 0/0_
```
### Summary-route

Genom att använda oss av en summary-route för 10.0.0.0/24 så kommer R1's neighbors fortfarande ha kännedom om nätet trots att R1 inte längre når det, detta är extremt användbart i riktigt stora nät då vi inte längre behöver skicka query's mellan varenda router i nätverket, R1's neighbors kan istället svara direkt med ett "Nej" på frågan om de har någon route till det önskade nätet och behöver inte skicka vidare till sina neighbors, som skickar vidare till sina neighbors osv... De vet ju om att R1 har detta nät! Vi konfigurerar följande summary-route på R1's interface mot R2 & R3: _ip summary-address eigrp 10 10.0.0.0 255.255.252.0 5_ Kollar vi routing table på R1 & R7 ser det nu ut såhär: **R1**
```
172.16.0.0/30 is subnetted, 4 subnets
D 172.16.0.4 [90/307200] via 172.16.0.2, 00:15:00, FastEthernet0/0
D 172.16.1.4 [90/307200] via 172.16.1.2, 00:15:00, FastEthernet0/1
C 172.16.0.0 is directly connected, FastEthernet0/0
C 172.16.1.0 is directly connected, FastEthernet0/1
 10.0.0.0/8 is variably subnetted, 3 subnets, 2 masks
D 10.0.0.0/24 [90/156160] via 192.168.0.2, 00:00:04, FastEthernet1/0
D 10.0.0.0/22 is a summary, 00:04:14, Null0
D 10.0.1.0/24 [90/156160] via 192.168.0.2, 00:03:09, FastEthernet1/0
```
**R7**
```
Gateway of last resort is not set
172.16.0.0/30 is subnetted, 4 subnets
D 172.16.0.4 [90/358400] via 172.16.1.5, 00:31:45, FastEthernet0/1
C 172.16.1.4 is directly connected, FastEthernet0/1
D 172.16.0.0 [90/332800] via 172.16.1.5, 00:31:45, FastEthernet0/1
D 172.16.1.0 [90/307200] via 172.16.1.5, 00:31:45, FastEthernet0/1
 10.0.0.0/22 is subnetted, 1 subnets
D 10.0.0.0 [90/437760] via 172.16.1.5, 00:01:59, FastEthernet0/1
```
Om vi nu återigen stänger ner länken till 10.0.0.0/24, vad händer? R3 får Query-paketet precis som vanligt:
```
*Mar 1 01:17:49.879: EIGRP: Received QUERY on FastEthernet0/0 nbr 172.16.1.1
*Mar 1 01:17:49.883: AS 10, Flags 0x0, Seq 95/0 idbQ 0/0 iidbQ un/rely 0/0 peerQ un/rely 0/0
*Mar 1 01:17:49.895: EIGRP: Enqueueing REPLY on FastEthernet0/0 nbr 172.16.1.1 iidbQ un/rely 0/1 peerQ un/rely 0/0 serno 40-40
*Mar 1 01:17:49.903: EIGRP: Sending REPLY on FastEthernet0/0 nbr 172.16.1.1
*Mar 1 01:17:49.903: AS 10, Flags 0x0, Seq 63/95 idbQ 0/0 iidbQ un/rely 0/0 peerQ un/rely 0/1 serno 40-40
```
Men det stannar där! R7 får aldrig något query-paket av R3. Om vi haft ytterligare 20-30 routrar efter R3 så har vi nu sparat in oerhört många onödiga query/reply-sändningar och minskar även risken för "**Stuck in Active**". Kollar vi routing-tabellen för R7 igen så har det inte skett någon förändring:
```
172.16.0.0/30 is subnetted, 4 subnets
D 172.16.0.4 [90/358400] via 172.16.1.5, 00:55:49, FastEthernet0/1
C 172.16.1.4 is directly connected, FastEthernet0/1
D 172.16.0.0 [90/332800] via 172.16.1.5, 00:55:49, FastEthernet0/1
D 172.16.1.0 [90/307200] via 172.16.1.5, 00:55:49, FastEthernet0/1
 10.0.0.0/22 is subnetted, 1 subnets
D 10.0.0.0 [90/437760] via 172.16.1.5, 00:18:52, FastEthernet0/1
```
Men vad händer om vi pingar en adress i 10.0.0.0/24-nätet nu då? Om vi kör en debug ip packet som pekar på en ACL jag konfat i R1 och pingar från R7:
```
R7#ping 10.0.0.1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.0.0.1, timeout is 2 seconds:
U.U.U
Success rate is 0 percent (0/5)
```
Observera att vi får **Unreachable**, inte timeout! Kollar vi i R1 kan vi se följande:
```
R1#
*Mar 1 01:31:00.271: IP: tableid=0, s=172.16.1.6 (FastEthernet0/1), d=10.0.0.1 (Null0), routed via RIB
*Mar 1 01:31:02.331: IP: tableid=0, s=172.16.1.6 (FastEthernet0/1), d=10.0.0.1 (Null0), routed via RIB
*Mar 1 01:31:04.367: IP: tableid=0, s=172.16.1.6 (FastEthernet0/1), d=10.0.0.1 (Null0), routed via RIB
```
Paketen skickas ut på Null0-interfacet (blackhole), paketet skulle ju annars skickas vidare till default-gateway och med all sannolikhet orsaka en routingloop.

### Stubzone

Ibland är det inte alltid möjligt att använda sig av summary-routes, vi skulle då istället kunna använda oss av en så kallad **Stubzone**. Detta anger vi på routrar som inte har någon annan väg "ut" från sina lokala nät, exempelvis branchoffice-routrar. Om vi går tillbaka till den topologi vi precis använt skulle vi här kunna ange R4, R5, R6 & R7 som stubzone. Stubzone-routrar får INGA Query-paket! [![query1](/assets/images/2013/06/query11.jpg)](/assets/images/2013/06/query11.jpg) Stubzone har flera inställningsmöjligheter som även går att kombinera (förutom Recieve-only som endast går att köra "solo"):

*   **Recieve-only**: Stubroutern kommer inte att annonsera något nät över huvudtaget (men kommer fortfarande lyssna på updates från andra routrar, ungefär som passive-interface för RIP)
*   **Connected**: Stubroutern kommer endast annonsera Directly Connected-nät
*   **Static**: Stubroutern kommer endast annonsera statiska route (kräver att du redistributar dessa in till eigrp först)
*   **Summary**: Stubroutern kommer endast annonsera summary-routes
*   **Redistribute**: Stubroutern kommer endast annonsera redistribute-routes

Per default är det Connected och Summary som tillåts annonseras. Vi tar och testar detta! Jag har tagit bort summary-routen från topologin, så när 10.0.0.0/24-nätet går ner får vi återigen query-paket till R7 från R3 . Jag har även lagt till 4st Loopback-nät inklusive en summary-route (/16) för att ha något att laborera med i **R7**, såhär ser routing table ut:
```
1.0.0.0/24 is subnetted, 4 subnets
C 1.1.0.0 is directly connected, Loopback3
C 1.1.1.0 is directly connected, Loopback0
C 1.1.2.0 is directly connected, Loopback1
C 1.1.3.0 is directly connected, Loopback2
 172.16.0.0/30 is subnetted, 4 subnets
D 172.16.0.4 [90/358400] via 172.16.1.5, 01:15:20, FastEthernet0/1
C 172.16.1.4 is directly connected, FastEthernet0/1
D 172.16.0.0 [90/332800] via 172.16.1.5, 01:15:20, FastEthernet0/1
D 172.16.1.0 [90/307200] via 172.16.1.5, 01:15:21, FastEthernet0/1
 10.0.0.0/24 is subnetted, 2 subnets
D 10.0.0.0 [90/437760] via 172.16.1.5, 00:00:08, FastEthernet0/1
D 10.0.1.0 [90/437760] via 172.16.1.5, 00:00:32, FastEthernet0/1
```
Och dessa routes kan **R3** se utan problem:
```
1.0.0.0/16 is subnetted, 1 subnets
D 1.1.0.0 [90/409600] via 172.16.1.6, 00:00:08, FastEthernet0/1
 172.16.0.0/30 is subnetted, 4 subnets
D 172.16.0.4 [90/332800] via 172.16.1.1, 01:24:14, FastEthernet0/0
C 172.16.1.4 is directly connected, FastEthernet0/1
D 172.16.0.0 [90/307200] via 172.16.1.1, 01:52:11, FastEthernet0/0
C 172.16.1.0 is directly connected, FastEthernet0/0
 10.0.0.0/24 is subnetted, 2 subnets
D 10.0.0.0 [90/412160] via 172.16.1.1, 00:07:03, FastEthernet0/0
D 10.0.1.0 [90/412160] via 172.16.1.1, 00:07:28, FastEthernet0/0
```
Om vi nu konfigurerar R7 till stub connected, vad händer? Summary-routen försvinner från R3 (R7 får endast annonsera sina connected-nät):
```
1.0.0.0/24 is subnetted, 4 subnets
D 1.1.0.0 [90/409600] via 172.16.1.6, 00:00:08, FastEthernet0/1
D 1.1.1.0 [90/409600] via 172.16.1.6, 00:00:08, FastEthernet0/1
D 1.1.2.0 [90/409600] via 172.16.1.6, 00:00:08, FastEthernet0/1
D 1.1.3.0 [90/409600] via 172.16.1.6, 00:00:08, FastEthernet0/1
 172.16.0.0/30 is subnetted, 4 subnets
D 172.16.0.4 [90/332800] via 172.16.1.1, 01:27:11, FastEthernet0/0
C 172.16.1.4 is directly connected, FastEthernet0/1
D 172.16.0.0 [90/307200] via 172.16.1.1, 01:55:09, FastEthernet0/0
C 172.16.1.0 is directly connected, FastEthernet0/0
 10.0.0.0/24 is subnetted, 2 subnets
D 10.0.0.0 [90/412160] via 172.16.1.1, 00:10:02, FastEthernet0/0
D 10.0.1.0 [90/412160] via 172.16.1.1, 00:10:26, FastEthernet0/0
```
Om vi byter till stub recieve-only?
```
172.16.0.0/30 is subnetted, 4 subnets
D 172.16.0.4 [90/332800] via 172.16.1.1, 01:29:44, FastEthernet0/0
C 172.16.1.4 is directly connected, FastEthernet0/1
D 172.16.0.0 [90/307200] via 172.16.1.1, 01:57:41, FastEthernet0/0
C 172.16.1.0 is directly connected, FastEthernet0/0
 10.0.0.0/24 is subnetted, 2 subnets
D 10.0.0.0 [90/412160] via 172.16.1.1, 00:12:33, FastEthernet0/0
D 10.0.1.0 [90/412160] via 172.16.1.1, 00:12:57, FastEthernet0/0
```
R3 kan nu inte längre se 1.1.0.0-1.1.3.0 näten. Vi återgår till stub connected summary och tar istället och testar vad som händer när 10.0.0.0/24-nätet går ner. Från R3 (fa0/0 går till R1 och fa0/1 går till R7):
```
*Mar 1 02:10:48.147: EIGRP: Received QUERY on FastEthernet0/0 nbr 172.16.1.1
*Mar 1 02:10:48.147: AS 10, Flags 0x0, Seq 189/0 idbQ 0/0 iidbQ un/rely 0/0 peerQ un/rely 0/0
*Mar 1 02:10:48.163: EIGRP: Enqueueing REPLY on FastEthernet0/0 nbr 172.16.1.1 iidbQ un/rely 0/1 peerQ un/rely 0/0 serno 95-95
*Mar 1 02:10:48.167: EIGRP: Enqueueing UPDATE on FastEthernet0/1 iidbQ un/rely 0/1 serno 96-96
*Mar 1 02:10:48.167: EIGRP: Enqueueing UPDATE on FastEthernet0/1 nbr 172.16.1.6 iidbQ un/rely 0/0 peerQ un/rely 0/0 serno 96-96
***Mar 1 02:10:48.171: EIGRP: Sending UPDATE on FastEthernet0/1**
*Mar 1 02:10:48.171: AS 10, Flags 0x0, Seq 138/0 idbQ 0/0 iidbQ un/rely 0/0 serno 96-96
*Mar 1 02:10:48.175: EIGRP: Sending REPLY on FastEthernet0/0 nbr 172.16.1.1
*Mar 1 02:10:48.175: AS 10, Flags 0x0, Seq 137/189 idbQ 0/0 iidbQ un/rely 0/0 peerQ un/rely 0/1 serno 95-95
***Mar 1 02:10:48.319: EIGRP: Received QUERY on FastEthernet0/1 nbr 172.16.1.6**
*Mar 1 02:10:48.319: AS 10, Flags 0x0, Seq 60/0 idbQ 0/0 iidbQ un/rely 0/0 peerQ un/rely 0/0
*Mar 1 02:10:48.331: EIGRP: Enqueueing UPDATE on FastEthernet0/0 iidbQ un/rely 0/1 serno 96-97
***Mar 1 02:10:48.331: EIGRP: Enqueueing REPLY on FastEthernet0/1 nbr 172.16.1.6 iidbQ un/rely 0/1 peerQ un/rely 0/0 serno 97-97**
*Mar 1 02:10:48.335: EIGRP: Enqueueing UPDATE on FastEthernet0/0 nbr 172.16.1.1 iidbQ un/rely 0/0 peerQ un/rely 0/0 serno 96-97
*Mar 1 02:10:48.339: EIGRP: Sending UPDATE on FastEthernet0/0
*Mar 1 02:10:48.339: AS 10, Flags 0x0, Seq 139/0 idbQ 0/0 iidbQ un/rely 0/0 serno 96-97
***Mar 1 02:10:48.347: EIGRP: Sending REPLY on FastEthernet0/1 nbr 172.16.1.6**
*Mar 1 02:10:48.347: AS 10, Flags 0x0, Seq 140/60 idbQ 0/0 iidbQ un/rely 0/0 peerQ un/rely 0/1 serno 97-97
```
Detta hade jag inte alls förväntat mig! Observera dock att R3 **inte** längre skickar någon **Query** till R7, däremot en **Update**  att 10.0.0.0/24 inte längre är nåbart, se följande wireshark-bild: 
[![stubreply](/assets/images/2013/06/stubreply.png)](/assets/images/2013/06/stubreply.png) 

Inget konstigt där egentligen, men det märkliga är det som händer efteråt! **R7** skickar en **Query** till R3 och frågar efter en route till 10.0.0.0/24 (!?). 
[![stubquery](/assets/images/2013/06/stubquery.png)](/assets/images/2013/06/stubquery.png)   

Detta tyckte jag var väldigt märkligt, så egentligen har vi ju inte minskat antal Query-paket, skillnaden är att det nu är R7 som skickar det till R3 istället för tvärtom (och att R3 måste först skicka ett Update-paket till R7). Detta känns ju som att det bara bidrar till mer overhead? Ska ta och forska vidare lite men detta inlägg har blivit alldeles för långt så tar och avslutar det här med denna "cliffhanger" hehe. På återseende!
