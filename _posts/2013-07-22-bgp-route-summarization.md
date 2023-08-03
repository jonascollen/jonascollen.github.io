---
title: BGP - Route Summarization
date: 2013-07-22 22:12
comments: true
categories: [BGP,Route summarization]
tags: [route-summarization]
---
Detta blir en kortare post om hur vi konfigurerar route summarization i BGP. Vi använder oss av en gammal topologi med tillägget att R5 nu annonserar följande nät förutom sin Loopback:

*   200.0.0.0/24
*   200.0.1.0/24
*   200.0.2.0/24
*   200.0.3.0/24

[![bgp routesummarization](/assets/images/2013/07/bgp-routesummarization.png)](/assets/images/2013/07/bgp-routesummarization.png)

R6 visar följande:

```
R6#sh ip bgp 
BGP table version is 7, local router ID is 6.6.6.6
Status codes: s suppressed, d damped, h history, \* valid, > best, i - internal,
 r RIB-failure, S Stale
Origin codes: i - IGP, e - EGP, ? - incomplete
Network Next Hop Metric LocPrf Weight Path
\*> 5.5.5.0/24 172.16.36.3 0 500 100 i
\*> 6.6.6.0/24 0.0.0.0 0 32768 i
\*> 200.0.0.0 172.16.36.3 0 500 100 i
\*> 200.0.1.0 172.16.36.3 0 500 100 i
\*> 200.0.2.0 172.16.36.3 0 500 100 i
\*> 200.0.3.0 172.16.36.3 0 500 100 i
R6#sh ip route | beg Gat
Gateway of last resort is not set
B 200.0.0.0/24 \[20/0\] via 172.16.36.3, 00:04:48
 5.0.0.0/24 is subnetted, 1 subnets
B 5.5.5.0 \[20/0\] via 172.16.36.3, 00:08:23
B 200.0.1.0/24 \[20/0\] via 172.16.36.3, 00:04:17
 6.0.0.0/24 is subnetted, 1 subnets
C 6.6.6.0 is directly connected, Loopback0
B 200.0.2.0/24 \[20/0\] via 172.16.36.3, 00:04:17
 172.16.0.0/24 is subnetted, 1 subnets
C 172.16.36.0 is directly connected, FastEthernet0/0
B 200.0.3.0/24 \[20/0\] via 172.16.36.3, 00:04:17
```
Om vi istället endast vill annonsera ett prefix för samtliga nät kan vi göra följande i R5:
```
router bgp 100 
aggregate-address 200.0.0.0 255.255.252.0 summary-only
```
Precis som för exempelvis OSPF skapar detta en summary-route som pekar till Null0, det är sedan denna route som annonseras ut på BGP.
```
R5#sh ip route | inc 200.0.0.0/22 
B 200.0.0.0/22 \[200/0\] via 0.0.0.0, 00:02:01, Null0
På R6 kan vi se förändringen direkt:
R6#sh ip bgp 
BGP table version is 12, local router ID is 6.6.6.6
Status codes: s suppressed, d damped, h history, \* valid, > best, i - internal,
 r RIB-failure, S Stale
Origin codes: i - IGP, e - EGP, ? - incomplete
Network Next Hop Metric LocPrf Weight Path
\*> 5.5.5.0/24 172.16.36.3 0 500 100 i
\*> 6.6.6.0/24 0.0.0.0 0 32768 i
\*> 200.0.0.0/22 172.16.36.3 0 500 100 i
R6#sh ip route | beg Gat
Gateway of last resort is not set
5.0.0.0/24 is subnetted, 1 subnets
B 5.5.5.0 \[20/0\] via 172.16.36.3, 00:11:29
 6.0.0.0/24 is subnetted, 1 subnets
C 6.6.6.0 is directly connected, Loopback0
 172.16.0.0/24 is subnetted, 1 subnets
C 172.16.36.0 is directly connected, FastEthernet0/0
B 200.0.0.0/22 \[20/0\] via 172.16.36.3, 00:00:10
```
Enkelt! :)
