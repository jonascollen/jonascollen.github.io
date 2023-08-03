---
title: BGP - eBGP Multihop
date: 2013-07-22 20:48
comments: true
categories: [BGP]
tags: [ebgp multihop]
---
I tidigare inlägg har vi konstaterat att det är rekommenderat att använda sig av loopback-interface när vi sätter upp BGP-neighbors om vi har redundanta vägar. Något som inte är lika självklart är att det krävs ytterligare konfiguration för att sätta upp detta över eBGP tillskillnad mot iBGP.

iBGP
----

Vi testar först att sätta upp en enkel adjacency mellan loopback-adresserna inom ett AS (iBGP). 

![iBGP peer redundant](/assets/images/2013/07/ibgp-peer-redundant1.png) 

Konfig:
```
R1
interface Loopback0
 ip address 1.1.1.1 255.255.255.255
 !
 interface FastEthernet0/0
 ip address 172.16.0.1 255.255.255.252
 !
 interface FastEthernet0/1
 ip address 172.32.0.1 255.255.255.252
 !
 router bgp 100
 neighbor 13.37.0.1 remote-as 100
 neighbor 13.37.0.1 update-source Loopback0
 !
 ip route 13.37.0.1 255.255.255.255 172.16.0.2
 ip route 13.37.0.1 255.255.255.255 172.32.0.2
 ---
 ```
R2
```
 interface Loopback0
 ip address 13.37.0.1 255.255.255.255
 !
 interface FastEthernet0/0
 ip address 172.16.0.2 255.255.255.252
 !
 interface FastEthernet0/1
 ip address 172.32.0.2 255.255.255.252
 !
 router bgp 100
 neighbor 1.1.1.1 remote-as 100
 neighbor 1.1.1.1 update-source Loopback0
 !
 ip route 1.1.1.1 255.255.255.255 172.32.0.1
 ip route 1.1.1.1 255.255.255.255 172.16.0.1
```
Vilket ger följande när vi kör debug ip bgp all:
```
*Mar 1 00:18:01.267: BGP: 13.37.0.1 open active, local address 1.1.1.1
*Mar 1 00:18:01.295: BGP: 13.37.0.1 went from Active to OpenSent
*Mar 1 00:18:01.295: BGP: 13.37.0.1 sending OPEN, version 4, my as: 100, holdtime 180 seconds
*Mar 1 00:18:01.299: BGP: 13.37.0.1 send message type 1, length (incl. header) 45
*Mar 1 00:18:01.363: BGP: 13.37.0.1 rcv message type 1, length (excl. header) 26
*Mar 1 00:18:01.363: BGP: 13.37.0.1 rcv OPEN, version 4, holdtime 180 seconds
*Mar 1 00:18:01.363: BGP: 13.37.0.1 rcv OPEN w/ OPTION parameter len: 16
*Mar 1 00:18:01.367: BGP: 13.37.0.1 rcvd OPEN w/ optional parameter type 2 (Capability) len 6
*Mar 1 00:18:01.367: BGP: 13.37.0.1 OPEN has CAPABILITY code: 1, length 4
*Mar 1 00:18:01.367: BGP: 13.37.0.1 OPEN has MP_EXT CAP for afi/safi: 1/1
*Mar 1 00:18:01.367: BGP: 13.37.0.1 rcvd OPEN w/ optional parameter type 2 (Capability) len 2
*Mar 1 00:18:01.371: BGP: 13.37.0.1 OPEN has CAPABILITY code: 128, length 0
*Mar 1 00:18:01.371: BGP: 13.37.0.1 OPEN has ROUTE-REFRESH capability(old) for all address-families
*Mar 1 00:18:01.371: BGP: 13.37.0.1 rcvd OPEN w/ optional parameter type 2 (Capability) len 2
*Mar 1 00:18:01.371: BGP: 13.37.0.1 OPEN has CAPABILITY code: 2, length 0
*Mar 1 00:18:01.371: BGP: 13.37.0.1 OPEN has ROUTE-REFRESH capability(new) for all address-families 
BGP: 13.37.0.1 rcvd OPEN w/ remote AS 100
*Mar 1 00:18:01.375: BGP: 13.37.0.1 went from OpenSent to OpenConfirm
*Mar 1 00:18:01.375: BGP: 13.37.0.1 went from OpenConfirm to Established
*Mar 1 00:18:01.375: %BGP-5-ADJCHANGE: neighbor 13.37.0.1 Up
```
Lätt som en plätt.. OPEN-message från R2 till R1 ser ut enligt följande: 
![iBGP TTL](/assets/images/2013/07/ibgp-ttl.png)
```
R1#sh ip bgp neighbors 
BGP neighbor is 13.37.0.1, remote AS 100, **internal link**
 BGP version 4, remote router ID 13.37.0.1
 BGP state = Established, up for 00:05:52
```
Okej, inga problem där med andra ord. Kom dock ihåg att ändra update-source! BGP kräver att avsändar-adressen matchar med den neighbor-adress som är konfigurerad i den mottagande routern. Låt oss testa vad som händer när vi kör samma lösning men över eBGP.

eBGP
----

![eBGP peer redundant](/assets/images/2013/07/ebgp-peer-redundant.png) 

Konfig:
```
R1
interface Loopback0
 ip address 1.1.1.1 255.255.255.255
 !
 interface FastEthernet0/0
 ip address 172.16.0.1 255.255.255.252
 !
 interface FastEthernet0/1
 ip address 172.32.0.1 255.255.255.252
 !
 router bgp 100
 neighbor 13.37.0.1 remote-as 500
 neighbor 13.37.0.1 update-source Loopback0
 !
 ip route 13.37.0.1 255.255.255.255 172.16.0.2
 ip route 13.37.0.1 255.255.255.255 172.32.0.2
 ---
 ```
R2
```
 interface Loopback0
 ip address 13.37.0.1 255.255.255.255
 !
 interface FastEthernet0/0
 ip address 172.16.0.2 255.255.255.252
 !
 interface FastEthernet0/1
 ip address 172.32.0.2 255.255.255.252
 !
 router bgp 500
 neighbor 1.1.1.1 remote-as 100
 neighbor 1.1.1.1 update-source Loopback0
 !
 ip route 1.1.1.1 255.255.255.255 172.32.0.1
 ip route 1.1.1.1 255.255.255.255 172.16.0.1
```
En debug ip bgp all visar dock ingen aktivitet, ser inte heller något i wireshark märkligt nog (vilket jag själv blev lite förvånad över). Problemet är nämligen så att BGP vid ett eBGP-förhållande sätter TCP-paketens TTL till 1. Om du kollar det tidigare OPEN-meddelandet från exemplet från iBGP ovan kan vi se att TTL är satt till 255(!). Då TTL endast är satt till 1 sker följande:

1.  R1 skickar sitt OPEN-message till R2 med TTL=1
2.  R2 tar emot OPEN-meddelandet på Fa0/0 eller Fa0/1
3.  R2 ser att destinationen är dess Lo0, men innan den skickar vidare paketet minskar den TTL med -1
4.  R2 upptäcker att TTL=0 och kastar därför paketet innan det vidarebefodrar det till loopback

BGP har infört en lösning en för detta kallad ebgp-multihop, vi konfar det enligt följande:
```
R1(config-router)#neighbor 13.37.0.1 ebgp-multihop ?
 <1-255> maximum hop count
```
I detta fall behöver vi sätta det till 2 för både R1 & R2:

`neighbor 13.37.0.1 ebgp-multihop 2`

Genast börjar det hända saker! :)
```
*Mar 1 00:48:15.535: BGP: 1.1.1.1 went from Idle to Active
*Mar 1 00:48:15.543: BGP: 1.1.1.1 open active delayed 28864ms (35000ms max, 28% jitter)
*Mar 1 00:48:44.407: BGP: 1.1.1.1 open active, local address 13.37.0.1
*Mar 1 00:48:44.463: BGP: 1.1.1.1 went from Active to OpenSent
*Mar 1 00:48:44.463: BGP: 1.1.1.1 sending OPEN, version 4, my as: 500, holdtime 180 seconds
*Mar 1 00:48:44.471: BGP: 1.1.1.1 send message type 1, length (incl. header) 45
*Mar 1 00:48:44.555: BGP: 1.1.1.1 rcv message type 1, length (excl. header) 26
*Mar 1 00:48:44.555: BGP: 1.1.1.1 rcv OPEN, version 4, holdtime 180 seconds
*Mar 1 00:48:44.555: BGP: 1.1.1.1 rcv OPEN w/ OPTION parameter len: 16
*Mar 1 00:48:44.555: BGP: 1.1.1.1 rcvd OPEN w/ optional parameter type 2 (Capability) len 6
*Mar 1 00:48:44.555: BGP: 1.1.1.1 OPEN has CAPABILITY code: 1, length 4
*Mar 1 00:48:44.559: BGP: 1.1.1.1 OPEN has MP_EXT CAP for afi/safi: 1/1
*Mar 1 00:48:44.559: BGP: 1.1.1.1 rcvd OPEN w/ optional parameter type 2 (Capability) len 2
*Mar 1 00:48:44.559: BGP: 1.1.1.1 OPEN has CAPABILITY code: 128, length 0
*Mar 1 00:48:44.559: BGP: 1.1.1.1 OPEN has ROUTE-REFRESH capability(old) for all address-families
*Mar 1 00:48:44.559: BGP: 1.1.1.1 rcvd OPEN w/ optional parameter type 2 (Capability) len 2
*Mar 1 00:48:44.563: BGP: 1.1.1.1 OPEN has CAPABILITY code: 2, length 0
*Mar 1 00:48:44.563: BGP: 1.1.1.1 OPEN has ROUTE-REFRESH capability(new) for all address-families 
BGP: 1.1.1.1 rcvd OPEN w/ remote AS 100
*Mar 1 00:48:44.563: BGP: 1.1.1.1 went from OpenSent to OpenConfirm
*Mar 1 00:48:44.563: BGP: 1.1.1.1 went from OpenConfirm to Established
*Mar 1 00:48:44.567: %BGP-5-ADJCHANGE: neighbor 1.1.1.1 Up
```
Kollar vi i wireshark kan vi se att detta OPEN-message ser lite annorlunda ut. 

![eBGP TTL](/assets/images/2013/07/ebgp-ttl.png) 

Man kan kanske tycka att det är konstigt att detta bara är ett problem när vi använder eBGP men samtidigt måste vi tänka över vilket användningsområde det har. Ett iBGP-adjacency kan sträcka sig över flera kontinenter för multinationella företag, men eBGP kommer du alltid sätta upp i en "border-router" mot din ISP.