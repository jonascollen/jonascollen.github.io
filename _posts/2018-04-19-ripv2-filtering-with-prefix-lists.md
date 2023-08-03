---
title: RIPv2 Filtering with Prefix-lists
date: 2018-04-19 20:18
comments: true
categories: [RIP, Route Manipulation]
tags: [prefix-list, filtering]
---
![](/assets/images/2018/04/big.png)

A small lab on RIPv2 and the use of prefix-lists which had a pretty neat solution with filtering by advertising router that I hadn't seen before. **Requirements:**

*   Stop R5 from advertising the Loopback-prefixes of R6 & R7 to R8 with a prefix-list, everything else should be forwarded
*   In R5, filter out any RIP updates received from R4 over the DMVPN-cloud, other routes should be accepted over DMPVN

We enable RIPv2 on all routers with the very basic commands:

```
router rip
 version 2
 network 150.1.0.0
 network 155.1.0.0
 no auto-summary
```

Step1 should be fairly straightforward, we create a prefix-list denying the loopbacks of R6 & R7 and filter updates going out on Gi1.58 on R5.

```
ip prefix-list LO_FILTER deny 150.1.6.6/32
ip prefix-list LO_FILTER deny 150.1.7.7/32
ip prefix-list LO_FILTER permit 0.0.0.0/0 le 32
```

The "permit 0.0.0.0/0 le 32" works just like a "permit any any" in an access-list. Final step is to set which interface (Gi1.58) and in what direction it should be filtered (outgoing).

```
router rip
 distribute-list prefix LO_FILTER out GigabitEthernet1.58
```

After the invalid timer has expired the routes for R6 & R7s loopbacks should drop from R8s routing table while still getting the rest of the networks.

```
R8# sh ip route | beg Gate
Gateway of last resort is not set

150.1.0.0/32 is subnetted, 8 subnets
R 150.1.1.1 [120/2] via 155.1.58.5, 00:00:28, GigabitEthernet1.58
R 150.1.2.2 [120/2] via 155.1.58.5, 00:00:28, GigabitEthernet1.58
R 150.1.3.3 [120/2] via 155.1.58.5, 00:00:28, GigabitEthernet1.58
R 150.1.4.4 [120/2] via 155.1.58.5, 00:00:28, GigabitEthernet1.58
R 150.1.5.5 [120/1] via 155.1.58.5, 00:00:28, GigabitEthernet1.58
C 150.1.8.8 is directly connected, Loopback0
R 150.1.9.9 [120/4] via 155.1.58.5, 00:00:28, GigabitEthernet1.58
R 150.1.10.10 [120/1] via 155.1.108.10, 00:00:05, GigabitEthernet1.108
```

The next requirement is the tricky bit, here I was stuck for quite a while and I still haven't actually managed to find any current official documentation regarding it except for this deprecated [IOS 12.2 docs](https://www.cisco.com/c/en/us/td/docs/ios/12_2/iproute/command/reference/fiprrp_r/1rfrip.html). First step first, as we need to filter out routes from R4 over the DMVPN and accept the rest, let's create two (a bit strange I know but you'll soon see why) prefix-lists:

```
ip prefix-list ACCEPT_ALL permit 0.0.0.0/0 le 32

ip prefix-list BLOCK_R4 deny 155.1.0.4/32
ip prefix-list BLOCK_R4 permit 0.0.0.0/0 le 32
```

We then use an extension within the distribute-list command in RIP thats called "gateway", to first specify which networks we will accept (ACCEPT_ALL) filtered by gateway (BLOCK_R4). The actual command looks like this:

```
router rip
 distribute-list prefix ACCEPT_ALL **gateway** BLOCK_R4 in
```

We should now see every network except the ones advertised from R4 over the DMVPN-cloud (150.1.0.4):

```
R5#sh ip route | beg Gate
Gateway of last resort is not set

150.1.0.0/32 is subnetted, 10 subnets
R 150.1.1.1 [120/1] via 155.1.0.1, 00:00:07, Tunnel0
R 150.1.2.2 [120/1] via 155.1.0.2, 00:00:12, Tunnel0
R 150.1.3.3 [120/1] via 155.1.0.3, 00:00:06, Tunnel0
**R 150.1.4.4 [120/1] via 155.1.45.4, 00:00:09, GigabitEthernet1.45**
C 150.1.5.5 is directly connected, Loopback0
R 150.1.6.6 [120/2] via 155.1.0.1, 00:00:07, Tunnel0
R 150.1.7.7 [120/2] via 155.1.0.3, 00:00:06, Tunnel0
R 150.1.8.8 [120/1] via 155.1.58.8, 00:00:19, GigabitEthernet1.58
R 150.1.9.9 [120/3] via 155.1.0.3, 00:00:06, Tunnel0
R 150.1.10.10 [120/2] via 155.1.58.8, 00:00:19, GigabitEthernet1.58

R5#sh ip rip database 150.1.4.4 255.255.255.255
150.1.4.4/32
 [1] via 155.1.45.4, 00:00:20, GigabitEthernet1.45
```

Sweet! We're still receiving R4's loopback but over the physical link instead of the DMVPN.
