---
title: Reliable static routing with Object tracking/IP SLA
date: 2018-04-09 09:14
comments: true
categories: [Route Manipulation]
tags: [ip-sla,object tracking, floating routes]
---
![](/assets/images/2018/04/tracking-topologi-1.png) 

Another rather basic lab regarding reliable static routing using object tracking with IP SLA, always good to refresh the fundamentals. :)

**Goals:**

1.  In R1 - Static route R4's loopback through the DMVPN-cloud
2.  In R5 - Static route R1's & R4's loopbacks through the DMVPN-cloud
3.  In R4 - Make a primary static route for R1s loopback thru Gi1.146
	*   Make sure the static route is only available when R1 is reachable thru Gi1.146, there's a switch between R1 & R4 so indirect link failures will not be noticed
	*   Reachability has to be checked every 5 seconds, R1 also has to respond within 2 seconds for the link to be deemed OK
4.  In R4 - Configure a backup static route for R1's loopback thru the DMVPN-cloud, only to be used when Gi1.146 is down

For step 3 the easiest (only?) solution for step 3 should just be to set up an IP SLA sending ICMP over the G1.146 link. I couldn't remember how the syntax looks for configuring the timers however. Trying to stay true to not cheating with checking "?" in CLI and keeping to notepad it was time to head over to the Cisco Doc. You can find IP SLA Configuration guide under: _Support > Product Support > Cisco IOS and NX-OS Software > Cisco IOS 15.4M&T > Configuration Guides > IP SLAs Configuration Guides_  _\> **Configuring IP SLAs UDP Echo Operations**_ The values we're looking for where frequency & timeout. Initial configuration should be something like this:

```
! 1-1 R1

ip route 150.1.4.4 255.255.255.255 Tu0 155.1.0.4

! 2-1 R5

ip route 150.1.1.1 255.255.255.255 Tu0 155.1.0.1
ip route 150.1.4.4 255.255.255.255 Tu0 155.1.0.4

! 3-1 R4

ip sla 1
 icmp-echo 155.1.146.1 source-ip 155.1.146.4
 frequency 5
 threshold 2000
 timeout 2000

ip sla schedule 1 life forever start-time now
track 1 ip sla 1

ip route 150.1.1.1 255.255.255.255 Gi1.146 155.1.146.1 track 1

! 4-1 R4
ip route 150.1.1.1 255.255.255.255 Tu0 150.1.0.1 2
```

Verification:

```
R4#ping 150.1.1.1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 150.1.1.1, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 5/5/5 ms

R4#sh ip sla statistics 1
IPSLAs Latest Operation Statistics

IPSLA operation id: 1
 Latest RTT: 4 milliseconds
Latest operation start time: 06:26:53 UTC Sat Apr 7 2018
**Latest operation return code: OK**
**Number of successes: 65**
Number of failures: 1
Operation time to live: Forever

R4#sh track
Track 1
 IP SLA 1 state
 **State is Up**
 2 changes, last change 00:04:08
 Latest operation return code: OK
 Latest RTT (millisecs) 5
 Tracked by:
 Static IP Routing 0

R4#sh ip route 150.1.1.1
Routing entry for 150.1.1.1/32
 Known via "static", distance 1, metric 0
 Routing Descriptor Blocks:
 **\* 155.1.146.1, via GigabitEthernet1.146**
 Route metric is 0, traffic share count is 1

R4#sh ip static route
Codes: M - Manual static, A - AAA download, N - IP NAT, D - DHCP,
 G - GPRS, V - Crypto VPN, C - CASA, P - Channel interface processor,
 B - BootP, S - Service selection gateway
 DN - Default Network, T - Tracking object
 L - TL1, E - OER, I - iEdge
 D1 - Dot1x Vlan Network, K - MWAM Route
 PP - PPP default route, MR - MRIPv6, SS - SSLVPN
 H - IPe Host, ID - IPe Domain Broadcast
 U - User GPRS, TE - MPLS Traffic-eng, LI - LIIN
 IR - ICMP Redirect
Codes in \[\]: A - active, N - non-active, B - BFD-tracked, D - Not Tracked, P - permanent

Static local RIB for default

**T 150.1.1.1/32 \[1/0\] via GigabitEthernet1.146 155.1.146.1 \[A\]**
**M \[2/0\] via Tunnel0 155.1.0.1 \[N\]**
```

Let's see what happens if we have an indirect link failure between R4 & R1:

```
R4#debug track state
_track state debugging enabled_
R4#debug ip routing
_IP routing debugging is on_

R1(config)#int gi1.146
R1(config-subif)#shut
```

Our IP SLA timeouts and our tracker changes state to down, which in turn leads to our main static route being pulled from the routing table. As we have a floating route with an administrative distance of 2 it should instead take over:

```
R4#
**track-sta (1) ip sla 1 state Up -> Down**
**RT: del 150.1.1.1 via 155.1.146.1, static metric \[1/0\]**
**RT: delete subnet route to 150.1.1.1/32**
RT: updating static 150.1.1.1/32 (0x0) :
 via 155.1.0.1 Tu0 0 1048578

**RT: add 150.1.1.1/32 via 155.1.0.1, static metric \[2/0\]**
RT: updating static 150.1.1.1/32 (0x0) :
 via 155.1.0.1 Tu0 0 1048578

R4#sh ip route 150.1.1.1
Routing entry for 150.1.1.1/32
 Known via "static", distance 2, metric 0
 Routing Descriptor Blocks:
 **\* 155.1.0.1, via Tunnel0**
 Route metric is 0, traffic share count is 1

R4#ping 150.1.1.1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 150.1.1.1, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 5/5/6 ms
```

Sweet! Going to focus more on reading again for a week or two until I finish Routing TCP/IP Volume 1, the holy bible of routing books. I'll try to stick with only doing labwork outside of my set "study hours", unfortunately it's just way more fun... :)
