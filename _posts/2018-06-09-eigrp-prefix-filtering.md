---
title: EIGRP - Prefix filtering
date: 2018-06-09 14:21
comments: true
categories: [EIGRP, Route Filtering, Route Manipulation]
tags: [acl, prefix-list, tags]
---
![](/assets/images/2018/05/full_topologydmvpn.png) 
Spent the day doing a bunch of different filtering techniques in EIGRP and stumbled upon something new regarding metric filtering that I haven't seen before so felt like it was worth a post. :)

Lab 1 - Filtering with prefix-lists
-----------------------------------

*   Configure a prefix-list on R4 that only blocks its loopback out on Gi1.45
*   Configure a prefix-list on R1 so it doesn't accept any updates from R4 on Gi1.146

Before we start our filtering our routing table should look something like this:

	R4#sh ip route eigrp | beg Gate
	Gateway of last resort is not set

	150.1.0.0/32 is subnetted, 10 subnets
	D 150.1.1.1 \[90/130816\] via 155.1.146.1, 00:49:27, GigabitEthernet1.146
	D 150.1.2.2 \[90/131328\] via 155.1.146.1, 00:01:19, GigabitEthernet1.146
	D 150.1.3.3 \[90/131072\] via 155.1.146.1, 00:28:38, GigabitEthernet1.146
	D 150.1.5.5 \[90/130816\] via 155.1.45.5, 00:49:30, GigabitEthernet1.45
	D 150.1.6.6 \[90/130816\] via 155.1.146.6, 00:49:27, GigabitEthernet1.146
	D 150.1.7.7 \[90/131072\] via 155.1.146.6, 00:49:11, GigabitEthernet1.146
	D 150.1.8.8 \[90/131072\] via 155.1.45.5, 00:49:30, GigabitEthernet1.45
	D 150.1.9.9 \[90/131328\] via 155.1.146.6, 00:49:11, GigabitEthernet1.146
	D 150.1.10.10 \[90/131328\] via 155.1.45.5, 00:49:30, GigabitEthernet1.45
	155.1.0.0/16 is variably subnetted, 18 subnets, 2 masks
	D 155.1.5.0/24 \[90/3072\] via 155.1.45.5, 00:49:30, GigabitEthernet1.45
	D 155.1.7.0/24 
	\[90/3328\] via 155.1.146.6, 00:49:11, GigabitEthernet1.146
	D 155.1.8.0/24 \[90/3328\] via 155.1.45.5, 00:49:30, GigabitEthernet1.45
	D 155.1.9.0/24 
	\[90/3584\] via 155.1.146.6, 00:49:11, GigabitEthernet1.146
	D 155.1.10.0/24 \[90/3584\] via 155.1.45.5, 00:49:30, GigabitEthernet1.45
	D 155.1.13.0/24 
	\[90/3072\] via 155.1.146.1, 00:49:27, GigabitEthernet1.146
	D 155.1.23.0/24 
	\[90/3328\] via 155.1.146.1, 00:28:38, GigabitEthernet1.146
	D 155.1.37.0/24 
	\[90/3328\] via 155.1.146.6, 00:28:38, GigabitEthernet1.146
	\[90/3328\] via 155.1.146.1, 00:28:38, GigabitEthernet1.146
	D 155.1.58.0/24 \[90/3072\] via 155.1.45.5, 00:49:30, GigabitEthernet1.45
	D 155.1.67.0/24 
	\[90/3072\] via 155.1.146.6, 00:49:27, GigabitEthernet1.146
	D 155.1.79.0/24 
	\[90/3328\] via 155.1.146.6, 00:49:11, GigabitEthernet1.146
	D 155.1.108.0/24 
	\[90/3328\] via 155.1.45.5, 00:49:30, GigabitEthernet1.45

Let's start by creating a prefix-list that blocks our loopback and permits everything else.

	ip prefix-list BLOCK\_R4\_LOOP deny 150.1.4.4/32
	ip prefix-list BLOCK\_R4\_LOOP permit 0.0.0.0/0 le 32

	router eigrp 100
	 distribute-list prefix BLOCK\_R4\_LOOP out Gi1.45

R5 should no longer see the 150.1.4.4/32 over the Gi1.45 link:

	R5#sh ip route 150.1.4.4
	Routing entry for 150.1.4.4/32
	Known via "eigrp 100", distance 90, metric 25984000, type internal
	Redistributing via eigrp 100
	**Last update from 155.1.0.4 on Tunnel0, 00:00:06 ago**
	Routing Descriptor Blocks:
	\* 155.1.0.4, from 155.1.0.4, 00:00:06 ago, via Tunnel0
	Route metric is 25984000, traffic share count is 1
	Total delay is 15000 microseconds, minimum bandwidth is 100 Kbit
	Reliability 255/255, minimum MTU 1400 bytes
	Loading 2/255, Hops 1

Lab 2 - Filtering with standard ACLs
------------------------------------

*   Configure a standard ACL on R9 to filter out all routes sourced from R7 that have an odd number in the third octet

We need an access-list that only matches odd numbers in the third octet, we control this by using our wildcard-mask. In other words the first bit must always be set to 0 for the network to be accepted (even numbers).

	access-list 1 permit 0.0.0.0 255.255.254.0  (_only check first bit in third octet, must be 0)_

	router eigrp 100
	 distribute-list 1 in GigabitEthernet 1.79

As we can see, the only odd-numbered network found in our routing table now is R9s own 150.1.9.9/32.

	R9#sh ip route | beg Gate
	Gateway of last resort is not set

	150.1.0.0/32 is subnetted, 6 subnets
	D 150.1.2.2 \[90/131328\] via 155.1.79.7, 00:12:23, GigabitEthernet1.79
	D 150.1.4.4 \[90/131328\] via 155.1.79.7, 01:00:16, GigabitEthernet1.79
	D 150.1.6.6 \[90/131072\] via 155.1.79.7, 01:00:16, GigabitEthernet1.79
	D 150.1.8.8 \[90/131840\] via 155.1.79.7, 01:00:16, GigabitEthernet1.79
	**C 150.1.9.9 is directly connected, Loopback0**
	D 150.1.10.10 \[90/132096\] via 155.1.79.7, 01:00:16, GigabitEthernet1.79

Lab 3 - Filtering with extended ACLs
------------------------------------

*   Shutdown R5s ethernetlink to R4
*   Use only extended ACLs to achieve the following:
    *   Reroute traffic destined to R4 & R6 Loopbacks to go via R2
    *   Reroute traffic destined to R1 & R2 Loopbacks to go via R3
    *   Reroute traffic destined to R7 & R9 Loopbacks to go via R1

When we use extended ACLs with distribute-lists remember that the source-field represents the **update-source** and destination-field the **network address.** By only closing R5's Gi1.45 link the routing table looks like this, most of the traffic is routed over the DMVPN-cloud:

	R5#sh ip route | beg Gate
	Gateway of last resort is not set

	150.1.0.0/32 is subnetted, 10 subnets
	D 150.1.1.1 \[90/25984000\] via 155.1.0.1, 00:00:09, Tunnel0
	D 150.1.2.2 \[90/25984000\] via 155.1.0.2, 00:00:09, Tunnel0
	D 150.1.3.3 \[90/25984000\] via 155.1.0.3, 00:00:09, Tunnel0
	D 150.1.4.4 \[90/25984000\] via 155.1.0.4, 00:00:09, Tunnel0
	C 150.1.5.5 is directly connected, Loopback0
	D 150.1.6.6 \[90/25984256\] via 155.1.0.4, 00:00:09, Tunnel0
	\[90/25984256\] via 155.1.0.1, 00:00:09, Tunnel0
	D 150.1.7.7 \[90/25984256\] via 155.1.0.3, 00:00:09, Tunnel0
	D 150.1.8.8 \[90/130816\] via 155.1.58.8, 00:22:56, GigabitEthernet1.58
	D 150.1.9.9 \[90/25984512\] via 155.1.0.3, 00:00:09, Tunnel0
	D 150.1.10.10 \[90/131072\] via 155.1.58.8, 00:22:56, GigabitEthernet1.58
	155.1.0.0/16 is variably subnetted, 18 subnets, 2 masks
	C 155.1.0.0/24 is directly connected, Tunnel0
	L 155.1.0.5/32 is directly connected, Tunnel0
	C 155.1.5.0/24 is directly connected, GigabitEthernet1.5
	L 155.1.5.5/32 is directly connected, GigabitEthernet1.5
	D 155.1.7.0/24 \[90/25856512\] via 155.1.0.3, 00:00:09, Tunnel0
	D 155.1.8.0/24 \[90/3072\] via 155.1.58.8, 00:22:15, GigabitEthernet1.58
	D 155.1.9.0/24 \[90/25856768\] via 155.1.0.3, 00:00:09, Tunnel0
	D 155.1.10.0/24 \[90/3328\] via 155.1.58.8, 00:22:15, GigabitEthernet1.58
	D 155.1.13.0/24 \[90/25856256\] via 155.1.0.3, 00:00:09, Tunnel0
	\[90/25856256\] via 155.1.0.1, 00:00:09, Tunnel0
	D 155.1.23.0/24 \[90/25856256\] via 155.1.0.3, 00:00:09, Tunnel0
	\[90/25856256\] via 155.1.0.2, 00:00:09, Tunnel0
	D 155.1.37.0/24 \[90/25856256\] via 155.1.0.3, 00:00:09, Tunnel0
	D 155.1.45.0/24 \[90/25856256\] via 155.1.0.4, 00:00:09, Tunnel0
	C 155.1.58.0/24 is directly connected, GigabitEthernet1.58
	L 155.1.58.5/32 is directly connected, GigabitEthernet1.58
	D 155.1.67.0/24 \[90/25856512\] via 155.1.0.4, 00:00:09, Tunnel0
	\[90/25856512\] via 155.1.0.3, 00:00:09, Tunnel0
	\[90/25856512\] via 155.1.0.1, 00:00:09, Tunnel0
	D 155.1.79.0/24 \[90/25856512\] via 155.1.0.3, 00:00:09, Tunnel0
	D 155.1.108.0/24 
	\[90/3072\] via 155.1.58.8, 00:22:15, GigabitEthernet1.58
	D 155.1.146.0/24 \[90/25856256\] via 155.1.0.4, 00:00:09, Tunnel0
	\[90/25856256\] via 155.1.0.1, 00:00:09, Tunnel0

There might be a much easier way to do this, but the way I solved it was by blocking all possible "alternative" routes learned in the DMVPN so only the targeted router was accepted as a valid update-source. So if we start with R4 & R6 that we're supposed to be routed towards R2, lets first create access-list that deny R1, R3 & R5 as update-sources for 150.1.4.4/32. ! R5 access-list 100 deny ip host 155.1.0.1 host 150.1.4.4 access-list 100 deny ip host 155.1.0.3 host 150.1.4.4 access-list 100 deny ip host 155.1.0.4 host 150.1.4.4 access-list 100 permit ip any any router eigrp 100 distribute-list 100 in Tunnel0 The only way to reach 150.1.4.4 over the DMVPN should now be 155.1.0.2.

	R5#sh ip route 150.1.4.4
	Routing entry for 150.1.4.4/32
	Known via "eigrp 100", distance 90, metric 25984768, type internal
	Redistributing via eigrp 100
	**Last update from 155.1.0.2 on Tunnel0, 00:00:14 ago**

To adjust the rest of the networks we just have do modify our ACL with more entries.

	! R5
	no access-list 100
	! Filter for R4
	access-list 100 deny ip host 155.1.0.1 host 150.1.4.4
	access-list 100 deny ip host 155.1.0.3 host 150.1.4.4
	access-list 100 deny ip host 155.1.0.4 host 150.1.4.4
	! Filter for R6
	access-list 100 deny ip host 155.1.0.1 host 150.1.6.6
	access-list 100 deny ip host 155.1.0.3 host 150.1.6.6
	access-list 100 deny ip host 155.1.0.4 host 150.1.6.6
	! Filter for R1
	access-list 100 deny ip host 155.1.0.1 host 150.1.1.1
	access-list 100 deny ip host 155.1.0.2 host 150.1.1.1
	access-list 100 deny ip host 155.1.0.4 host 150.1.1.1
	! Filter for R2
	access-list 100 deny ip host 155.1.0.1 host 150.1.2.2
	access-list 100 deny ip host 155.1.0.2 host 150.1.2.2
	access-list 100 deny ip host 155.1.0.4 host 150.1.2.2
	! Filter for R7
	access-list 100 deny ip host 155.1.0.2 host 150.1.7.7
	access-list 100 deny ip host 155.1.0.3 host 150.1.7.7
	access-list 100 deny ip host 155.1.0.4 host 150.1.7.7
	! Filter for R9
	access-list 100 deny ip host 155.1.0.2 host 150.1.9.9
	access-list 100 deny ip host 155.1.0.3 host 150.1.9.9
	access-list 100 deny ip host 155.1.0.4 host 150.1.9.9
	! Permit rest
	access-list 100 permit ip any any

Loopback prefixes for R4 & R6 should now be routed to R2, R1 & R2 to R3, R7 & R9 to R1.

	R5#sh ip route | inc 150.1 
	150.1.0.0/32 is subnetted, 10 subnets
	**D 150.1.1.1 \[90/25984256\] via 155.1.0.3, 00:00:16, Tunnel0**
	**D 150.1.2.2 \[90/25984256\] via 155.1.0.3, 00:00:16, Tunnel0**
	D 150.1.3.3 \[90/25984000\] via 155.1.0.3, 00:02:42, Tunnel0
	**D 150.1.4.4 \[90/25984768\] via 155.1.0.2, 00:02:40, Tunnel0**
	C 150.1.5.5 is directly connected, Loopback0
	**D 150.1.6.6 \[90/25984768\] via 155.1.0.2, 00:00:15, Tunnel0**
	**D 150.1.7.7 \[90/25984512\] via 155.1.0.1, 00:00:16, Tunnel0**
	D 150.1.8.8 \[90/130816\] via 155.1.58.8, 00:08:37, GigabitEthernet1.58
	**D 150.1.9.9 \[90/25984768\] via 155.1.0.1, 00:00:16, Tunnel0**
	D 150.1.10.10 \[90/131072\] via 155.1.58.8, 00:08:37, GigabitEthernet1.58

Lab 4 - Filtering with Offset Lists
-----------------------------------

*   Configure an offset-list on R7 so traffic to 150.1.3.3/32 is routed to R6
*   If the link to R6 is down, traffic should be rerouted directly to R3

Before we do any changes, here is how R7 sees the 150.1.3.3 network currently.

	R7#sh ip eigrp topology 150.1.3.3/32
	EIGRP-IPv4 Topology Entry for AS(100)/ID(150.1.7.7) for 150.1.3.3/32
	State is Passive, Query origin flag is 1, 1 Successor(s), FD is 130816
	Descriptor Blocks:
	1**55.1.37.3 (GigabitEthernet1.37), from 155.1.37.3, Send flag is 0x0**
	**Composite metric is (130816/128256), route is Internal**
	Vector metric:
	Minimum bandwidth is 1000000 Kbit
	Total delay is 5010 microseconds
	Reliability is 255/255
	Load is 1/255
	Minimum MTU is 1500
	Hop count is 1
	Originating router is 150.1.3.3

As R7 has a direct link to R3 with a decently low feasible distance, there's no alternative route that matches the feasibility condition (adv. distance lower than 130816). But by making the current route less attractive we should be able to make R7 prefer the route to R6 instead as long as it's up. First we create a simple ACL that matches our network.

	access-list 1 permit host 150.1.3.3

We then offset the metric of our current route to make it less desirable.

	router eigrp 100
	 offset-list 1 in 10000 Gi1.37

Our routing table in R7 looks like this after the changes:

	%DUAL-5-NBRCHANGE: EIGRP-IPv4 100: Neighbor 155.1.37.3 (GigabitEthernet1.37) is resync: intf route configuration changed
	R7#sh ip eigrp topology 150.1.3.3/32
	EIGRP-IPv4 Topology Entry for AS(100)/ID(150.1.7.7) for 150.1.3.3/32
	State is Passive, Query origin flag is 1, 1 Successor(s), FD is 131328
	Descriptor Blocks:
	**155.1.67.6 (GigabitEthernet1.67), from 155.1.67.6, Send flag is 0x0**
	**Composite metric is (131328/131072), route is Internal**
	Vector metric:
	Minimum bandwidth is 1000000 Kbit
	Total delay is 5030 microseconds
	Reliability is 255/255
	Load is 1/255
	Minimum MTU is 1500
	Hop count is 3
	Originating router is 150.1.3.3
	**155.1.37.3 (GigabitEthernet1.37), from 155.1.37.3, Send flag is 0x0**
	**Composite metric is (140816/138256), route is Internal**
	Vector metric:
	Minimum bandwidth is 1000000 Kbit
	Total delay is 5400 microseconds
	Reliability is 255/255
	Load is 1/255
	Minimum MTU is 1500
	Hop count is 1
	Originating router is 150.1.3.3

	R7# sh ip route 150.1.3.3
	Routing entry for 150.1.3.3/32
	Known via "eigrp 100", distance 90, metric 131328, type internal
	Redistributing via eigrp 100
	Last update from 155.1.67.6 on GigabitEthernet1.67, 00:01:19 ago
	Routing Descriptor Blocks:
	\* 155.1.67.6, from 155.1.67.6, 00:01:19 ago, via GigabitEthernet1.67
	Route metric is 131328, traffic share count is 1
	Total delay is 5030 microseconds, minimum bandwidth is 1000000 Kbit
	Reliability 255/255, minimum MTU 1500 bytes
	Loading 1/255, Hops 3

As we're still receiving the route from R3 we have a backup if our link to R6 goes down.

Lab 5 - Filtering with administrative distance
----------------------------------------------

*   Configure AD filtering on R3 so traffic towards 150.1.7.7 is sent to R1

Should be pretty easy, as you maybe know we can set AD on either route or by neighbor + route, in our case we need to do the latter.

	access-list 1 permit host 150.1.7.7

	router eigrp 100
	 distance 255 155.1.37.7 0.0.0.0 1

	R3#sh ip route 150.1.7.7 
	Routing entry for 150.1.7.7/32
	Known via "eigrp 100", distance 90, metric 131328, type internal
	Redistributing via eigrp 100
	**Last update from 155.1.13.1 on GigabitEthernet1.13, 00:00:37 ago**
	Routing Descriptor Blocks:
	**\* 155.1.13.1, from 155.1.13.1, 00:00:37 ago, via GigabitEthernet1.13**
	Route metric is 131328, traffic share count is 1
	Total delay is 5030 microseconds, minimum bandwidth is 1000000 Kbit
	Reliability 255/255, minimum MTU 1500 bytes
	Loading 1/255, Hops 3

Lab 6 - Filtering with Route maps
---------------------------------

This lab had something new that I haven't seen before. The requirements we're as follows:

*   Configure on R4:
    *   Loopback1 - 160.1.4.4/32, redistribute to EIGRP with tag 4
    *   Loopback2 - 170.1.4.4/32, redistribute to EIGRP
*   Configure a route-filter on R2 that denies routes with tag 4
*   Configure a route-filter on R3 that denies routes with a metric between 120000 - 140000
*   Filters should not affect any other routes advertised

Let's start with the R4, shouldn't be anything tricky. We need two prefix-lists to match the different Loopbacks and a route-map to tag the desired route.

	!! R4
	interface Lo1
	 ip add 160.1.4.4 255.255.255.255

	interface Lo2
	 ip add 170.1.4.4 255.255.255.255

	ip prefix-list 160 permit 160.1.4.4/32
	ip prefix-list 170 permit 170.1.4.4/32

	route-map TAG permit 10
	 match ip address prefix-list 160
	 set tag 4
	route-map TAG permit 20
	 match ip address prefix-list 170

	router eigrp 100
	 redistribute connected route-map TAG

If we check the routes learned on R2 now 160.1.4.4/32 should be tagged.

	**Routing entry for 160.1.4.4/32**
	Known via "eigrp 100", distance 170, metric 131328
	**Tag 4, type external**
	Redistributing via eigrp 100
	Last update from 155.1.23.3 on GigabitEthernet1.23, 00:02:55 ago
	Routing Descriptor Blocks:
	\* 155.1.23.3, from 155.1.23.3, 00:02:55 ago, via GigabitEthernet1.23
	Route metric is 131328, traffic share count is 1
	Total delay is 5030 microseconds, minimum bandwidth is 1000000 Kbit
	Reliability 255/255, minimum MTU 1500 bytes
	Loading 1/255, Hops 3
	**Route tag 4**

	R2#sh ip route 170.1.4.4
	**Routing entry for 170.1.4.4/32**
	Known via "eigrp 100", distance 170, metric 131328, type external
	Redistributing via eigrp 100
	Last update from 155.1.23.3 on GigabitEthernet1.23, 00:02:59 ago
	Routing Descriptor Blocks:
	\* 155.1.23.3, from 155.1.23.3, 00:02:59 ago, via GigabitEthernet1.23
	Route metric is 131328, traffic share count is 1
	Total delay is 5030 microseconds, minimum bandwidth is 1000000 Kbit
	Reliability 255/255, minimum MTU 1500 bytes
	Loading 1/255, Hops 3

Looks good! Now just have to do a route-map and filter our incoming routes on R2.

	route-map BLOCK\_TAG4 deny 10
	match tag 4
	route-map BLOCK\_TAG4 permit 20

	router eigrp 100
	distribute-list route-map BLOCK\_TAG4 in

We should now only see the 170.1.4.4 in R2.

	R2#sh ip route eigrp | inc EX
	D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area 
	D EX 170.1.4.4 \[170/131328\] via 155.1.23.3, 00:05:21, GigabitEthernet1.23

	R2#sh ip route 160.1.4.4 
	% Network not in table

	R2#sh ip eigrp topology 160.1.4.4/32
	EIGRP-IPv4 Topology Entry for AS(100)/ID(150.1.2.2)
	%Entry 160.1.4.4/32 not in topology table

And the final step, filter out routes based on metric. Before configuring anything our routing table looks like this on R2:

	150.1.0.0/32 is subnetted, 10 subnets
	D 150.1.1.1 \[90/130816\] via 155.1.13.1, 00:09:51, GigabitEthernet1.13
	D 150.1.2.2 \[90/130816\] via 155.1.23.2, 00:09:51, GigabitEthernet1.23
	C 150.1.3.3 is directly connected, Loopback0
	D 150.1.4.4 \[90/131072\] via 155.1.13.1, 00:09:51, GigabitEthernet1.13
	D 150.1.5.5 \[90/131328\] via 155.1.13.1, 00:09:51, GigabitEthernet1.13
	D 150.1.6.6 \[90/131072\] via 155.1.37.7, 00:09:51, GigabitEthernet1.37
	\[90/131072\] via 155.1.13.1, 00:09:51, GigabitEthernet1.13
	D 150.1.7.7 \[90/130816\] via 155.1.37.7, 00:09:51, GigabitEthernet1.37
	D 150.1.8.8 \[90/131584\] via 155.1.13.1, 00:09:51, GigabitEthernet1.13
	D 150.1.9.9 \[90/131072\] via 155.1.37.7, 00:09:51, GigabitEthernet1.37
	D 150.1.10.10 \[90/131840\] via 155.1.13.1, 00:09:51, GigabitEthernet1.13

Matching by metric was new to me but it was pretty easy to find the command as it had to be done with a route-map in my mind. We use the match statement with metric, which has the following options:

	R3(config-route-map)#match metric ?
	<1-4294967295> Metric value
	external match route using external protocol metric
	R3(config-route-map)#match metric 120000 ?
	+- deviation option to match metric in a range
	<1-4294967295> Metric value

For us to match metrics between 120,000 - 140,000 we set the "average" and use deviation-option.

	!! R3
	route-map BLOCK\_METRIC deny 10
	match metric 130000 +- 10000
	route-map BLOCK\_METRIC permit 20

	router eigrp 100
	distribute-list route-map BLOCK\_METRIC in

The 150.1.x.x we're earlier routed via Gi1.13 & Gi1.37 but should now be filtered out and now take a worse route with metrics above 140k.

	R3#sh ip route | inc 150.1
	150.1.0.0/32 is subnetted, 10 subnets
	D 150.1.1.1 \[90/27264000\] via 155.1.0.5, 00:04:17, Tunnel0
	D 150.1.2.2 \[90/27264000\] via 155.1.0.5, 00:04:17, Tunnel0
	C 150.1.3.3 is directly connected, Loopback0
	D 150.1.4.4 \[90/27264000\] via 155.1.0.5, 00:04:17, Tunnel0
	D 150.1.5.5 \[90/27008000\] via 155.1.0.5, 00:16:14, Tunnel0
	D 150.1.6.6 \[90/27264256\] via 155.1.0.5, 00:04:17, Tunnel0
	D 150.1.7.7 \[90/27264512\] via 155.1.0.5, 00:04:17, Tunnel0
	D 150.1.8.8 \[90/27008256\] via 155.1.0.5, 00:16:14, Tunnel0
	D 150.1.9.9 \[90/27264768\] via 155.1.0.5, 00:04:17, Tunnel0
	D 150.1.10.10 \[90/27008512\] via 155.1.0.5, 00:16:14, Tunnel0

Sweet! Reading is going pretty well currently, I sort of finished "Routing on IOS, IOS XE, IOS XR", jumping over the chapters I haven't read anything about yet (multicast/bgp/route manipulation etc to backtrack later). Just started "Internet Routing Architectures" that seems to be focused solely on BGP, fun times! :)