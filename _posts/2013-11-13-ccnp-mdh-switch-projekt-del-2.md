---
title: CCNP – MDH Switchprojekt del 2
date: 2013-11-13 12:16
comments: true
categories: [MST, Route Manipulation]
tags: [hsrp]
---
Har varit lite dåligt med inlägg på senare tid, har pluggat multicast senaste ~2 veckorna men inte riktigt funnit motivationen att skriva inlägg om de mer teoretiska bitarna då det redan finns så oändligt med info om det redan. Tänkte istället visa några av lösningarna vi använt oss av i vårat senast switch-projekt.

Topologin ser ut enligt följande:
[![switchprojekt](/assets/images/2013/10/switchprojekt.png?w=630)](/assets/images/2013/10/switchprojekt.png) 

Vi kör EIGRP på alla L3-enheter, dvs R1-3, S1 & S3.

# MSTP

_7. Konfigurera MSTP på alla switchar. Lägg VLAN 10 och 20 till instans 1, och VLAN 30 och 99 till instans 2. Gör S1 till spanning-tree root for instans 1 och backup root för instans 2. S3 skall vara root for instans 2 och backuproot för instans 1

S1

```
spanning-tree mode mst
spanning-tree mst configuration
 name cisco
 revision 1
 instance 1 vlan 10, 20
 instance 2 vlan 30, 99
!
spanning-tree mst 1 priority 24576
spanning-tree mst 2 priority 28672
```
S2

```
spanning-tree mode mst
spanning-tree mst configuration
 name cisco
 revision 1
 instance 1 vlan 10, 20
 instance 2 vlan 30, 99
```

S3
```
spanning-tree mode mst
spanning-tree mst configuration
 name cisco
 revision 1
 instance 1 vlan 10, 20
 instance 2 vlan 30, 99
!
spanning-tree mst 1 priority 28672
spanning-tree mst 2 priority 24576
```

# HSRP

_8. S1 och S3 skall vara gateways för VLANen och ni skall använda HSRP för redundans. för access layer hosts i VLANs 10, 20, 30, and 99._ _9. S1 skall vara active HSRP router för VLAN 10 och 20 konfigurera S3 som backup. Konfigurera S3 som active router för VLAN 30 och 99 och S1 som backup för dessa VLAN._ 

S1
```
interface Vlan10
 description Client
 ip address 172.16.10.1 255.255.255.0
 ip helper-address 172.16.32.193
 standby 1 ip 172.16.10.254
 standby 1 priority 150
 standby 1 preempt
 no shutdown
!
interface Vlan20
 description Voice
 ip address 172.16.20.1 255.255.255.0
 standby 1 ip 172.16.20.254
 standby 1 priority 150
 standby 1 preempt
 no shutdown
!
interface Vlan30
 description Server
 ip address 172.16.30.1 255.255.255.0
 ip helper-address 172.16.32.193
 standby 2 ip 172.16.30.254
 standby 2 priority 80
 no shutdown
! 
interface Vlan99
 description Management
 ip address 172.16.99.1 255.255.255.0
 standby 2 ip 172.16.99.254
 standby 2 priority 80
 no shutdown
```

S3
```
interface Vlan10
 description Client
 ip address 172.16.10.3 255.255.255.0
 ip helper-address 172.16.32.193
 standby 1 ip 172.16.10.254
 standby 1 priority 80
 no shutdown
!
interface Vlan20
 description Voice
 ip address 172.16.20.3 255.255.255.0
 standby 1 ip 172.16.20.254
 standby 1 priority 80
 no shutdown
!
interface Vlan30
 description Server
 ip address 172.16.30.3 255.255.255.0
 ip helper-address 172.16.32.193
 standby 2 ip 172.16.30.254
 standby 2 priority 120
 standby 2 preempt
 no shutdown
!
interface Vlan99
 ip address 172.16.99.3 255.255.255.0
 standby 2 ip 172.16.99.254
 standby 2 priority 120
 standby 2 preempt
```
### EIGRP

16. På S1 och S3 ska ni routa med EIGRP, det som skall synas i routingtabellerna på 2800-routrarna är en manuellt summerad adress av de adresser som finns på S1, S2 och S3. Givetvis skall adresser som används på R1, R2 och R3 summeras på lämpligt sätt om man tittar på det i någon av 3560-switcharna. Fixade en visio-bild så det blir lite tydligare vad det är som efterfrågas: 
![summary-projekt](/assets/images/2013/11/summary-projekt.png) 

Konfigen för S1 & S3 är väldigt simpel. S1

```
interface FastEthernet0/5
 description To R1
 no switchport
 ip address 172.16.11.1 255.255.255.0
 **ip authentication mode eigrp 1 md5**
 **ip authentication key-chain eigrp 1 GRPM**
 **ip summary-address eigrp 1 172.16.0.0 255.255.0.0**
 srr-queue bandwidth share 1 30 35 5
 priority-queue out 
 mls qos trust dscp
 auto qos trust 
 speed 100
 duplex full
```
S3
```
interface FastEthernet0/5
 description To R1
 no switchport
 ip address 172.16.33.3 255.255.255.0
 **ip authentication mode eigrp 1 md5**
 **ip authentication key-chain eigrp 1 GRPM**
 **ip summary-address eigrp 1 172.16.0.0 255.255.0.0**
 srr-queue bandwidth share 1 30 35 5
 priority-queue out 
 mls qos trust dscp
 spanning-tree portfast
 speed 100
 duplex full
```
För R1 vill vi endast annonsera 172.16.32.0/24 ner till S1, och 172.16.0.0/16 upp till R2.

```
interface FastEthernet0/1
 description to S1
 ip address 172.16.11.11 255.255.255.0
 **ip authentication mode eigrp 1 md5**
 **ip authentication key-chain eigrp 1 GRPM**
 **ip summary-address eigrp 1 172.16.32.0 255.255.255.0 5**
 duplex full
 speed 100
 auto qos voip trust 
 service-policy output AutoQoS-Policy-Trust
 no shutdown
```

För att filtrera bort alla "Connected"-nät och endast annonsera 172.16.0.0/16 till R2 behöver vi dock använda oss av en distribute-list.

```
interface Serial0/0/0
 description to R2
 bandwidth 256000
 ip address 172.16.32.1 255.255.255.192
 **ip authentication mode eigrp 1 md5**
 **ip authentication key-chain eigrp 1 GRPM**
 auto qos voip trust 
 clock rate 256000
 service-policy output AutoQoS-Policy-Trust
 no shutdown

ip prefix-list INTERNAL-R2 seq 5 permit 172.16.0.0/16

router eigrp 1
 passive-interface default
 no passive-interface FastEthernet0/1
 no passive-interface Serial0/0/0
 no passive-interface Serial0/0/1
 network 172.16.0.0 0.0.255.255
 **distribute-list prefix INTERNAL-R2 out Serial0/0/0**
 no auto-summary
```


Samma sak gäller för R3.

```

interface FastEthernet0/1
 description to S1
 ip address 172.16.33.13 255.255.255.0
 **ip authentication mode eigrp 1 md5**
 **ip authentication key-chain eigrp 1 GRPM**
 **ip summary-address eigrp 1 172.16.32.0 255.255.255.0 5**
 duplex full
 speed 100
 auto qos voip trust 
 service-policy output AutoQoS-Policy-Trust
 no shutdown

interface Serial0/0/0
 description to R1
 bandwidth 256000
 ip address 172.16.32.66 255.255.255.192
 **ip authentication mode eigrp 1 md5**
 **ip authentication key-chain eigrp 1 GRPM**
 auto qos voip trust 
 clock rate 256000
 service-policy output AutoQoS-Policy-Trust
 no shutdown

ip prefix-list INTERNAL-R2 seq 5 permit 172.16.0.0/16

router eigrp 1
 passive-interface default
 no passive-interface FastEthernet0/1
 no passive-interface Serial0/0/0
 no passive-interface Serial0/0/1 
 network 172.16.0.0 0.0.255.255
 **distribute-list prefix INTERNAL-R2 out Serial0/0/1**
 no auto-summary

```

Då jag nu precis börjat läsa CCDP denna vecka samtidigt som CCNP-kursen kan vi nog komma tillbaka till den här topologin och införa lite förbättringar senare. :)
