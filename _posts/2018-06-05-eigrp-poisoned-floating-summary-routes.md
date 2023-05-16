---
layout: post
title: EIGRP - Poisoned floating summary-routes
date: 2018-06-05 20:45
author: Jonas Collén
comments: true
categories: [EIGRP, Route Poisoning]
tags: [ summary route ]
---
![](/assets/images/2018/05/full_topologydmvpn.png)

Another cool little lab showcasing a new/redesigned functionality that apparently was introduced in IOS15.x  that was new to me. The lab has the following requirements:

*   Configure EIGRP Classic on R4, R5 & R8 with AS100, only enable it on the .45.0 & .58.0 segments
*   Configure loopbacks on R4 & R5 with subnet 160.1.x.x/24 (x - router id) and advertise them
*   Advertise a summary default-route from R4
*   Advertise a summary route of the two 16.0.1.x.x/24 routes without overlap on R5 to R8
*   Modify the summary route of R5 so a null0 route isn't installed

All basic steps until the last one so no explanation needed I feel. Keeping up with writing the full config in notepad only is starting to pay of as well, improving both speed and overall knowledge of syntax. Highly recommended! :)

	! R4

	interface Lo1
	 ip add 160.1.4.4 255.255.255.0

	router eigrp 100
	 network 155.1.45.0 0.0.0.255 
	 network 160.1.4.0 0.0.0.255

	interface Gi1.45
	 ip summary-address eigrp 100 0.0.0.0 0.0.0.0

	! R5

	interface Lo1
	 ip add 160.1.5.5 255.255.255.0

	router eigrp 100
	 network 155.1.45.0 0.0.0.255
	 network 155.1.58.0 0.0.0.255
	 network 160.1.5.0 0.0.0.255

	interface Gi1.58
	 ip summary-address eigrp 100 160.1.4.0 255.255.254.0

	! R8

	router eigrp 100
	 network 155.1.58.0 0.0.0.255

Checking the routes in R8 confirms that we "should" have connectivity to the two loopbacks.

	R8#sh ip route eigrp | beg Gate
	Gateway of last resort is 155.1.58.5 to network 0.0.0.0

	**D\* 0.0.0.0/0 \[90/3328\] via 155.1.58.5, 00:18:58, GigabitEthernet1.58**
	155.1.0.0/16 is variably subnetted, 7 subnets, 2 masks
	D 155.1.45.0/24 \[90/3072\] via 155.1.58.5, 00:18:58, GigabitEthernet1.58
	160.1.0.0/23 is subnetted, 1 subnets
	**D 160.1.4.0 \[90/130816\] via 155.1.58.5, 00:00:43, GigabitEthernet1.58**

But ping is still failing to R4:

	R8#ping 160.1.4.4
	Type escape sequence to abort.
	Sending 5, 100-byte ICMP Echos to 160.1.4.4, timeout is 2 seconds:
	U.U.U
	Success rate is 0 percent (0/5)

	R8#ping 160.1.5.5
	Type escape sequence to abort.
	Sending 5, 100-byte ICMP Echos to 160.1.5.5, timeout is 2 seconds:
	!!!!!
	Success rate is 100 percent (5/5), round-trip min/avg/max = 4/5/6 ms

How come? All summary-routes created in ex. EIGRP & OSPF will also add a Null0-route to avoid routing-loops! As we only have a default-route to R4 longest match will hit our null0-route instead for 160.1.4.4.

	R5#sh ip route eigrp | beg Gate
	Gateway of last resort is 155.1.45.4 to network 0.0.0.0

	D\* 0.0.0.0/0 \[90/3072\] via 155.1.45.4, 00:22:19, GigabitEthernet1.45
	160.1.0.0/16 is variably subnetted, 3 subnets, 3 masks
	**D 160.1.4.0/23 is a summary, 00:04:05, Null0**

	R5#sh ip cef 160.1.4.0 
	160.1.4.0/23
	attached to Null0

So how do we fix this? As mentioned a new feature was added to EIGRP in 15.x (I think in previous versions something similar was done on the interface instead). We can adjust the AD of our summary-route, and by setting it to 255 it will be deemed invalid (poisoned) and not installed in our routing table.

	! R5
	router eigrp 100
	 summary 160.1.4.0/23 distance 255

The summary-route is still active in a way as our router will keep suppressing the advertisements of  160.1.5.0/24 and just forward the default-route instead.

	R5#sh ip route eigrp | beg Gate
	Gateway of last resort is 155.1.45.4 to network 0.0.0.0

	D\* 0.0.0.0/0 \[90/3072\] via 155.1.45.4, 00:26:02, GigabitEthernet1.45

We can now reach both R4 & R5's loopback from R8.

	R8#ping 160.1.4.4
	Type escape sequence to abort.
	Sending 5, 100-byte ICMP Echos to 160.1.4.4, timeout is 2 seconds:
	!!!!!
	Success rate is 100 percent (5/5), round-trip min/avg/max = 5/6/7 ms

	R8#ping 160.1.5.5
	Type escape sequence to abort.
	Sending 5, 100-byte ICMP Echos to 160.1.5.5, timeout is 2 seconds:
	!!!!!
	Success rate is 100 percent (5/5), round-trip min/avg/max = 5/5/6 ms