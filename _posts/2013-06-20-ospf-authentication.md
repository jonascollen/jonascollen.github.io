---
title: OSPF - Authentication
date: 2013-06-20 22:53
comments: true
categories: [OSPF]
tags: [authentication]
---
OSPF kan precis som övriga routingprotokoll använda sig av autentisering. Till skillnad från EIGRP finns dock förutom MD5-kryptering även möjligheten att använda clear text, fast det är väl inte direkt någon fördel.. ;) Varje OSPF-paket kommer autentiseras när vi aktiverar detta över en länk, dvs inte bara när adjacency bildas. Om vi kollar en debug för R2 av OSPF-paketen i följande topologi kan vi se detta "in action". 

[![ospf authentication](/assets/images/2013/06/ospf-authentication.png)](/assets/images/2013/06/ospf-authentication.png)

*Mar 1 00:11:10.471: OSPF: rcv. v:2 t:1 l:48 rid:10.0.0.1
 aid:0.0.0.0 chk:C494 **aut:0** auk: from FastEthernet0/0

Aut-fältet visar vilken autentisering som används.

*   Aut 0 - Ingen autentisering
*   Aut 1 - Clear text
*   Aut 2 - MD5-krypterat

### Clear-text

Detta är givetvis inte rekommenderat att använda, men det kan ändå vara bra att känna till hur vi konfigurerar det.

```
R1
int fa0/0
ip ospf authentication
ip ospf authentication-key pa$$word (8 texten är max)

R2
Int fa0/0
ip ospf authentication
ip ospf authentication-key pa$$word (8 texten är max)
```
Istället för att skriva raden "ip ospf authentication" för varje interface  går det även att göra det globalt via:
```
router ospf 1
area 0 authentication
```
Det krävs dock fortfarande att vi anger en nyckel för respektive interface så det är väl individuellt vad man tycker är enklast. En debug visar att aut nu ändrats till 1.
```
*Mar 1 00:18:00.931: OSPF: rcv. v:2 t:1 l:48 rid:10.0.0.1
 aid:0.0.0.0 chk:C493 aut:1 auk: from FastEthernet0/0
```
I wireshark kan vi se följande: 

[![auth cleartext](/assets/images/2013/06/auth-cleartext.png)](/assets/images/2013/06/auth-cleartext.png)

 Ganska enkelt att se varför vi inte vill använda oss av clear text.. :)

### MD5-kryptering

Låt oss byta till MD5 istället.
```
R1
int fa0/0
ip ospf message-digest 1 md5 pa$$word
ip ospf authentication message-digest
R2
int fa0/0
ip ospf message-digest 1 md5 pa$$word
ip ospf authentication message-digest
```
En debug visar följande:
```
*Mar 1 00:25:10.559: OSPF: rcv. v:2 t:1 l:48 rid:10.0.0.1
 aid:0.0.0.0 chk:0 aut:2 keyid:1 seq:0x3C7ECA46 from FastEthernet0/0
```
Som synes har vi nu även key-id angivet vilket **måste** matcha mellan routrarna. Ändrar vi R1 till key 2 (ip ospf message-digest **2** md5 pa$$word) så tappar vi adjacency och ser följande i en debug:
```
R1(config-if)#do debug ip ospf adj
OSPF adjacency events debugging is on
*Mar 1 00:28:06.111: OSPF: Rcv pkt from 10.0.0.2, FastEthernet0/0 : Mismatch Authentication Key - No message digest key 1 on interface
```
Om vi jämför MD5 mot clear text i wireshark ser vi fördelen med kryptering.

[![auth md5](/assets/images/2013/06/auth-md5.png)](/assets/images/2013/06/auth-md5.png) 

Ingen direkt djupdykning idag men just autentisering är rätt basic.
