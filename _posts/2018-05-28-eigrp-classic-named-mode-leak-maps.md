---
title: EIGRP Classic & Named mode - Leak-maps
date: 2018-05-28 21:23
comments: true
categories: [EIGRP, Route Manipulation]
tags: [leak-map, redistribution]
---
Back from my vacation in Greece! Took a break from everything network-related to recharge but now i'm all set to keep crunching both labs & books. Currently fighting my way thru "IP Routing on IOS, IOS XE and IOS XR - An essential guide to understanding & implementing IP Routing Protocols" (2.1k pages - yikes... ). Trying to get back into the groove with an easier lab on EIGRP Leak-maps down below: 

![](/assets/images/2018/05/full_topologydmvpn.png)

Requirements:
-------------

*   Configure Classic mode on R4 & R5 with AS100 and enable over both ethernet + DMVPN
*   Configure Named mode on R1, R3, R6 & R7 with AS200 and enable over 155.1.0.0/16
*   Configure loopbacks on R4 and redistribute to EIGRP
    *   Lo40 -4.0.0.4/24
    *   Lo41 - 4.0.1.4/24
*   Configure loopbacks on R6 and redistribute to EIGRP
    *   Lo60 - 6.0.0.6/24
    *   Lo61 - 6.0.1.6/24
*   Configure default summary-routes on R4 & R6  to be advertised instead of Loopbacks
*   Configure a leak-map on R4 for traffic to Lo40 to be routed via DMVPN
*   Configure a leak-map on R6 for traffic to Lo60 to be routed via R3 from R1
*   If the DMVPN is down, traffic should still be rerouted on the backup-path

Let's start with the easy part and configure EIGRP, Loopbacks and redistribution:

```
!! General

! R4 & R5

router eigrp 100
 network 155.1.0.0 0.0.255.255
 network 150.0.0.0 0.0.0.255

! R1, R3, R6, R7

router eigrp MULTI-AF
 address-family ipv4 auto 200
  network 155.1.0.0 0.0.255.255

!! Loopbacks
! R4

int Lo40
 ip add 4.0.0.4 255.255.255.0
int Lo41
 ip add 4.0.1.4 255.255.255.0

router eigrp 100
 redistribute connected

! R6

int Lo60
 ip add 6.0.0.6 255.255.255.0
int Lo61
 ip add 6.0.1.6 255.255.255.0

router eigrp MULTI-AF
 address-family ipv4 auto 200
  topology base 
   redistribute connected
```
We should now have the baseline working.

	R5#sh ip route eigrp | beg Gate
	Gateway of last resort is not set

	4.0.0.0/24 is subnetted, 2 subnets
	**D EX 4.0.0.0 \[170/130816\] via 155.1.45.4, 00:18:52, GigabitEthernet1.45**
	**D EX 4.0.1.0 \[170/130816\] via 155.1.45.4, 00:18:52, GigabitEthernet1.45**
	150.1.0.0/32 is subnetted, 2 subnets
	D EX 150.1.4.4 \[170/130816\] via 155.1.45.4, 00:18:52, GigabitEthernet1.45

	R1#sh ip route eigrp | beg Gate
	Gateway of last resort is not set

	6.0.0.0/24 is subnetted, 2 subnets
	**D EX 6.0.0.0 \[170/10880\] via 155.1.146.6, 00:14:51, GigabitEthernet1.146**
	**D EX 6.0.1.0 \[170/10880\] via 155.1.146.6, 00:14:51, GigabitEthernet1.146**
	150.1.0.0/32 is subnetted, 2 subnets
	D EX 150.1.6.6 \[170/10880\] via 155.1.146.6, 00:14:51, GigabitEthernet1.146
	155.1.0.0/16 is variably subnetted, 10 subnets, 2 masks
	D 155.1.7.0/24 
	\[90/20480\] via 155.1.146.6, 00:15:53, GigabitEthernet1.146
	\[90/20480\] via 155.1.13.3, 00:15:53, GigabitEthernet1.13
	D 155.1.37.0/24 
	\[90/15360\] via 155.1.13.3, 00:15:53, GigabitEthernet1.13
	D 155.1.67.0/24 
	\[90/15360\] via 155.1.146.6, 00:15:53, GigabitEthernet1.146
	D 155.1.79.0/24 
	\[90/20480\] via 155.1.146.6, 00:15:53, GigabitEthernet1.146
	\[90/20480\] via 155.1.13.3, 00:15:53, GigabitEthernet1.13

Let's add our summary-routes to R4 & R6:

```
!! Summary default-route

! R4
int Gi1.45
 ip summary-address eigrp 100 0.0.0.0 0.0.0.0
int Tu0
 ip summary-address eigrp 100 0.0.0.0 0.0.0.0

! R6
router eigrp MULTI-AF
 address-family ipv4 auto 200
  af-interface Gi1.67
   summary-address 0.0.0.0 0.0.0.0
  af-interface Gi1.146
   summary-address 0.0.0.0 0.0.0.0
```

As we're doing summarization our loopbacks advertisements will be suppressed and replaced with an internal 0.0.0.0/0 route:

	R1#sh ip route eigrp | beg Gate
	Gateway of last resort is 155.1.146.6 to network 0.0.0.0

	**D\* 0.0.0.0/0 \[90/10880\] via 155.1.146.6, 00:01:30, GigabitEthernet1.146**
	**155.1.0.0/16 is variably subnetted, 10 subnets, 2 masks**
	D 155.1.7.0/24 \[90/20480\] via 155.1.13.3, 00:01:30, GigabitEthernet1.13
	D 155.1.37.0/24 
	\[90/15360\] via 155.1.13.3, 00:01:30, GigabitEthernet1.13
	D 155.1.67.0/24 
	\[90/20480\] via 155.1.13.3, 00:01:30, GigabitEthernet1.13
	D 155.1.79.0/24 
	\[90/20480\] via 155.1.13.3, 00:01:30, GigabitEthernet1.13

	R5#sh ip route eigrp | beg Gate
	Gateway of last resort is 155.1.45.4 to network 0.0.0.0

	**D\* 0.0.0.0/0 \[90/3072\] via 155.1.45.4, 00:00:20, GigabitEthernet1.45**

Next step is to use a leak-map so traffic going to R4s loopback is routed via DMVPN-cloud instead of the ethernet-segment. This will be easily solved by advertising that specific route out on our Tu0-interface together with our default-route. Longest-match makes routers prefer our specific-route instead of the default to get to 4.0.0.4. To implement this we use a "leak-map", I found it in the official DOC [here](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_eigrp/command/ire-cr-book/ire-i1.html#wp2135400909) & [here](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_eigrp/configuration/15-mt/ire-15-mt-book/ire-enhanced-igrp.html#GUID-87869953-0896-4333-8EE5-B747855C7108).

```
!! Leak-map

! R4
ip prefix-list LOOP permit 4.0.0.4/24

route-map LEAK permit 10
 match ip add prefix-list LOOP

int Tu0
 ip summary-address eigrp 100 0.0.0.0 0.0.0.0 leak-map LEAK
```

Neighbors will do a graceful-restart and then the results should be visible in R5:

	R5#sh ip route eigrp | beg Gate
	Gateway of last resort is 155.1.45.4 to network 0.0.0.0

	D\* 0.0.0.0/0 \[90/3072\] via 155.1.45.4, 00:07:01, GigabitEthernet1.45
	4.0.0.0/24 is subnetted, 1 subnets
	**D EX 4.0.0.0 \[170/25984000\] via 155.1.0.4, 00:00:05, Tunnel0**

	R5#ping 4.0.0.4
	Type escape sequence to abort.
	Sending 5, 100-byte ICMP Echos to 4.0.0.4, timeout is 2 seconds:
	!!!!!
	Success rate is 100 percent (5/5), round-trip min/avg/max = 4/4/6 ms

If we close our Tunnel-interface traffic will still be routed over the default-route to Gi1.45.

	! R4
	int Tu0
	shut

	R5#sh ip route eigrp | beg Gate
	Gateway of last resort is 155.1.45.4 to network 0.0.0.0

	**D\* 0.0.0.0/0 \[90/3072\] via 155.1.45.4, 00:09:36, GigabitEthernet1.45**

	R5#ping 4.0.0.4 
	Type escape sequence to abort.
	Sending 5, 100-byte ICMP Echos to 4.0.0.4, timeout is 2 seconds:
	!!!!!
	Success rate is 100 percent (5/5), round-trip min/avg/max = 4/4/6 ms

Sweet! Now we just have to do the same thing in R6, by leaking our route on the Gi1.67-interface R1 will prefer the route to R3 over going directly to R6 for reaching 6.0.0.0/24.

```
!! Leak-map

! R6
ip prefix-list LOOP permit 6.0.0.6/24

route-map LEAK permit 10
 match ip add prefix-list LOOP

router eigrp MULTI-AF
 address-family ipv4 auto 200
  af-interface gi1.67
   summary-address 0.0.0.0 0.0.0.0 leak-map LEAK
```

Let's check R1 again:

	R1#sh ip route eigrp | beg Gate
	Gateway of last resort is 155.1.146.6 to network 0.0.0.0

	**D\* 0.0.0.0/0 \[90/10880\] via 155.1.146.6, 00:11:33, GigabitEthernet1.146**
	6.0.0.0/24 is subnetted, 1 subnets
	**D EX 6.0.0.0 \[170/21120\] via 155.1.13.3, 00:00:20, GigabitEthernet1.13**
	155.1.0.0/16 is variably subnetted, 10 subnets, 2 masks
	D 155.1.7.0/24 \[90/20480\] via 155.1.13.3, 00:11:33, GigabitEthernet1.13
	D 155.1.37.0/24 
	\[90/15360\] via 155.1.13.3, 00:11:33, GigabitEthernet1.13
	D 155.1.67.0/24 
	\[90/20480\] via 155.1.13.3, 00:11:33, GigabitEthernet1.13
	D 155.1.79.0/24 
	\[90/20480\] via 155.1.13.3, 00:11:33, GigabitEthernet1.13

All is good! I'm really starting to like EIGRP's named mode the more I use it, the classic feels so clunky now. Until next time... :)