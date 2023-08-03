---
title: BGP - Route Redistribution & Filtering
date: 2013-07-23 12:58
comments: true
categories: [BGP, Route Manipulation]
---
![bgp AS-path](/assets/images/2013/07/bgp-as-path.png)
Här kommer ett enkelt exempel på hur vi utför redistribution & filtrering in till BGP (som egentligen inte skiljer sig något från de övriga protokollen som vi redan labbat med). Låt oss säga att vi vill redistributa näten 200.0.2.0/24 & 200.0.3.0/24 från R5 in till BGP. Vi skapar först en access-lista (eller prefix-list):

```
access-list 1 permit 200.0.2.0
access-list 1 permit 200.0.3.0
```
Sedan en route-map:
```
route-map FILTER permit 10
match ip address 1
```
Vi går sedan in under BGP-processen och aktiverar redistribute men filtrerar med hjälp av vår route-map.
```
router bgp 100
redistribute connected route-map FILTER
```
Easy peasy.. Vi kan nu se förändringen i exempelvis R7:
```
R7#sh ip bgp
BGP table version is 15, local router ID is 192.168.0.1
Status codes: s suppressed, d damped, h history, \* valid, > best, i - internal,
 r RIB-failure, S Stale
Origin codes: i - IGP, e - EGP, ? - incomplete
Network Next Hop Metric LocPrf Weight Path
\* 5.5.5.0/24 172.16.76.6 0 200 500 100 i
\*> 172.16.47.4 0 500 100 i
\*> 6.6.6.0/24 172.16.76.6 0 0 200 i
\* 172.16.47.4 0 500 200 i
\*> 192.168.0.0 0.0.0.0 0 32768 i
\* 200.0.2.0 172.16.76.6 0 200 500 100 ?
\*> 172.16.47.4 0 500 100 ?
\* 200.0.3.0 172.16.76.6 0 200 500 100 ?
\*> 172.16.47.4 0 500 100 ?
R7#sh ip route | beg Gat
Gateway of last resort is not set
5.0.0.0/24 is subnetted, 1 subnets
B 5.5.5.0 \[20/0\] via 172.16.47.4, 00:20:19
 6.0.0.0/24 is subnetted, 1 subnets
B 6.6.6.0 \[20/0\] via 172.16.76.6, 00:19:48
B 200.0.2.0/24 \[20/0\] via 172.16.47.4, 00:06:16
 172.16.0.0/24 is subnetted, 2 subnets
C 172.16.47.0 is directly connected, FastEthernet0/0
C 172.16.76.0 is directly connected, FastEthernet0/1
B 200.0.3.0/24 \[20/0\] via 172.16.47.4, 00:06:16
C 192.168.0.0/24 is directly connected, Loopback0
```
