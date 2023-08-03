---
title: Route Redistribution - Lab
date: 2013-07-01 23:29
comments: true
categories: [EIGRP, OSPF, Route Manipulation]
---
Gjorde en av Jeremys "Advanced route redistribution" från hans CCNP-nuggets serie idag. Haft rätt fullt upp förra veckan så har inte blivit så mycket pluggande, får försöka öka tempot igen nu. :) Labben var riktigt givande, satt fast rätt länge på vissa delar, framförallt det sista steget, där var jag tvungen att tjuvkika hur Jeremy löste det hela.. :P Topologin såg ut såhär: 
[![redist lab](/assets/images/2013/07/redist-lab.png)](/assets/images/2013/07/redist-lab.png) 

Och uppgiften var följande: 
[![redist lab 2](/assets/images/2013/07/redist-lab-2.png)](/assets/images/2013/07/redist-lab-2.png) 

Min slutkonfig för R2 & R3 blev i slutändan föjande:

```
access-list 1 permit 10.4.0.0 0.0.0.255
access-list 1 permit 10.4.1.0 0.0.0.255
access-list 2 permit 10.4.2.0 0.0.0.255
access-list 2 permit 10.4.3.0 0.0.0.255
access-list 3 permit 10.4.4.0 0.0.0.255

route-map EIGRP-INTO-OSPF deny 5
match tag 40
route-map EIGRP-INTO-OSPF permit 10
match ip address 1
set metric 100
set tag 10
route-map EIGRP-INTO-OSPF permit 20
match ip address 2
set metric 200
set tag 20
route-map EIGRP-INTO-OSPF deny 30
match ip address 3
route-map EIGRP-INTO-OSPF permit 40
set metric 300
set tag 30
router ospf 1
redistribute eigrp 100 subnets route-map EIGRP-INTO-OSPF
route-map OSPF-INTO-EIGRP deny 5
match tag 10 20 30
route-map OSPF-INTO-EIGRP permit 10
set tag 40
set metric 1500 1 255 1 1500
router eigrp 100
redistribute ospf 1 route-map OSPF-INTO-EIGRP
```

Och för R2 löste vi sista punkten med:
```
router eigrp 100
distance eigrp 90 105
```