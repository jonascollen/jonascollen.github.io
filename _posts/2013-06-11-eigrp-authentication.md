---
title: EIGRP - Authentication
date: 2013-06-11 14:27
comments: true
categories: [EIGRP]
tags: [authentication]
---
Något jag glömt nämna men som är väldigt basic är att aktivera autentisering för EIGRP.

Konfigen är enligt följande:
```
key chain LAB
 key 1
   key-string CISCO
interface x/x
ip authentication key-chain eigrp 1 LAB
ip authentication mode eigrp 1 md5
```
Debuggar vi hello-paket kan vi nu se följande:
```
*Mar 1 01:05:31.375: EIGRP: received packet with MD5 authentication, key id = 1
*Mar 1 01:05:31.379: EIGRP: Received HELLO on Serial0/1 nbr 192.168.45.5
*Mar 1 01:05:31.379: AS 1, Flags 0x0, Seq 0/0 idbQ 0/0 iidbQ un/rely 0/0 peerQ un/rely 0/0
```
Viktigt är att nyckeln har samma nummer på båda routrar annars kommer neighbor adjacency att misslyckas. Vi testar detta genom att sätta Key 2 på den ena neighborn istället:
```
*Mar 1 01:09:50.555: EIGRP: Sending HELLO on Serial0/1
*Mar 1 01:09:50.559: AS 1, Flags 0x0, Seq 0/0 idbQ 0/0 iidbQ un/rely 0/0
*Mar 1 01:09:53.083: EIGRP: pkt authentication key id = 2, key not defined or not live
```
Det finns även möjlighet att göra använda sig av dynamiska/tidsstyrda nycklar, detta kräver dock att båda routrarna synkar mot en NTP-server så att de har samma tid.
```
key chain LAB
 key 1
 key-string CISCO
 accept-lifetime 00:00:00 Jan 1 2013 00:00:00 Jan 1 2014
 send-lifetime 00:00:00 Jan 1 2013 00:00:00 Jan 1 2014
 key 2
 key-string CISCO2
 accept-lifetime 00:00:00 Jan 1 2014 00:00:00 Jan 1 2015
 send-lifetime 00:00:00 Jan 1 2014 00:00:00 Jan 1 2015
```
Efter att ha aktiverat detta tappar vi dock neigbor-adjacency, varför? En debug visar föjande:
```
*Mar 1 01:14:34.439: EIGRP: interface Serial0/1, No live authentication keys
*Mar 1 01:14:35.503: EIGRP: interface Serial0/1, No live authentication keys
*Mar 1 01:14:35.507: EIGRP: Sending HELLO on Serial0/1
```
En show clock visar problemet, nyckeln kommer inte börja användas förrän om 11 år.. :)
```
Dobbs#sh clock
*01:15:45.683 UTC Fri Mar 1 2002
```
Efter att vi ställt in klockan så får vi direkt upp adjacency, detta visar varför det är så viktigt med tidssynkronisering!
```
Dobbs#clock set 16:22:00 11 June 2013 
*Jun 11 16:22:00.000: %SYS-6-CLOCKUPDATE: System clock has been updated from 01:17:01 UTC Fri Mar 1 2002 to 16:22:00 UTC Tue Jun 11 2013, configured from console by console.
Dobbs#
Jun 11 16:22:01.871: %DUAL-5-NBRCHANGE: IP-EIGRP(0) 1: Neighbor 192.168.45.4 (Serial0/1) is up: new adjacency
```
