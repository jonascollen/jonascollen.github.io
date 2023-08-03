---
title: EIGRP Classic & Named mode - Key rotation
date: 2018-05-04 22:00
comments: true
categories: [EIGRP]
tags: [authentication, classic-mode, named-mode]
---
My earlier plan to finish reading the "Routing TCP/IP Vol.1" two weeks ago failed miserably. :D I'm just about done with the chapters on EIGRP & OSPFv2 currently, it's crazy how little you actually know after only reading the CCNP-levels of these protocols. This quote sums it up perfectly:

> The more you know, the more you know you don't know.

It sure requires going back quite a few times (at least for me) to be able to grasp all the details and I still have lots of work to do here I feel. Going to spend the weekend watching INE/CBT-videos and doing some labs to "relax" before continuing with OSPFv3 & ISIS. New goal is to be done with this book before I fly to Greece for a week the 25/5, maybe is a bit too much but on the other hand ISIS is more or less completely new for me so I think i'll need the extra days. I did this little lab today with a mix of both classic & named EIGRP and it shows the main differences between the modes pretty nicely. 

![](/assets/images/2018/05/full_topologydmvpn.png)

Requirements
------------

*   Configure classic mode on R1 - R4, AS100
*   Configure named-mode on R5, MULTI-AF AS100
*   Enable EIGRP on DMVPN
*   Set clock on R5 to current time and configure as NTP master
*   Configure R1 - R4 to use R5 as NTP-server
*   Authenticate adjacencies on DMVPN using the key-chain "KEY\_ROTATION"
    *   Key-id 10 password CISCO10
    *   Key-id 20 password CISCO20
    *   Key 10 should be valid between 00:00:00 Jan 1 1993 to 00:10:00 Jan 1 2030 (and accepted an extra 10min)
    *   Key 20 should be valid from 00:00:00 Jan 1 2030
*   Modify R5s clock and verify key-chain switches without losing adjacency

Let's start with setting up basic EIGRP & NTP first as it usually takes a while for routers to sync.

```
! R1 - R4
clock set 21:17:00 4 MAY 2018

conf t
ntp server 155.1.0.5

router eigrp 100
 network 150.1.0.0 0.0.255.255 
 network 155.1.0.0 0.0.255.255

! R5
clock set 21:17:00 4 MAY 2018

conf t
ntp master 1

router eigrp MULTI-AF
 address-family ipv4 autonomous-system 100
  network 150.1.0.0 0.0.255.255
  network 155.1.0.0 0.0.255.255
  af-interface Tu0
   no split-horizon
```
The syntax for named-eigrp looks pretty different from the classic one, the biggest change is that everything is now configured under the EIGRP-process, instead of being spread out like classic mode (authentication/eigrp split-horizon configured on interfaces etc). We should now have connectivity between all routers and NTP synced.

	R5#sh ip eigrp neighbors 
	EIGRP-IPv4 VR(MULTI-AF) Address-Family Neighbors for AS(100)
	H Address Interface Hold Uptime SRTT RTO Q Seq
	(sec) (ms) Cnt Num
	4 155.1.0.1 Tu0 11 00:46:00 50 1398 0 40
	2 155.1.0.4 Tu0 13 00:46:50 114 1398 0 31
	1 155.1.0.3 Tu0 14 00:46:55 48 1398 0 42
	0 155.1.0.2 Tu0 13 00:46:59 91 1398 0 26

	R1#sh ntp status
	**Clock is synchronized, stratum 2, reference is 155.1.0.5** 
	nominal freq is 250.0000 Hz, actual freq is 249.9998 Hz, precision is 2\*\*10
	ntp uptime is 357700 (1/100 of seconds), resolution is 4016
	reference time is DE974AC9.62D0E670 (21:10:33.386 UTC Fri May 4 2018)
	clock offset is 0.0000 msec, root delay is 3.00 msec
	root dispersion is 12.70 msec, peer dispersion is 4.56 msec
	loopfilter state is 'CTRL' (Normal Controlled Loop), drift is 0.000000557 s/s
	system poll interval is 128, last update was 423 sec ago.

	R1#sh ntp associations detail
	**155.1.0.5 configured, ipv4, our\_master, sane, valid, stratum 1**
	ref ID .LOCL., time DE974C5E.E7AE16F8 (21:17:18.905 UTC Fri May 4 2018)
	our mode client, peer mode server, our poll intvl 128, peer poll intvl 128
	root delay 0.00 msec, root disp 2.16, reach 377, sync dist 9.05
	delay 2.00 msec, offset 0.0000 msec, dispersion 4.56, jitter 0.97 msec
	precision 2\*\*10, version 4
	assoc id 39556, assoc name 155.1.0.5
	assoc in packets 64, assoc out packets 64, assoc error packets 22
	org time 00000000.00000000 (00:00:00.000 UTC Mon Jan 1 1900)
	rec time DE974C5F.628F5D38 (21:17:19.385 UTC Fri May 4 2018)
	xmt time DE974C5F.628F5D38 (21:17:19.385 UTC Fri May 4 2018)
	filtdelay = 3.00 3.00 3.00 3.00 3.00 2.00 3.00 3.00
	filtoffset = 0.50 -0.50 0.50 0.50 0.50 0.00 -0.50 0.50
	filterror = 1.95 1.98 3.97 4.00 5.98 6.01 8.04 8.07
	minpoll = 6, maxpoll = 10

Let's configure our key-chain now, it will look the same on all routers:

```
! R1 - R4

key chain KEY\_ROTATION
 key 10
  accept-lifetime 00:00:00 1 Jan 1993 00:15:00 Jan 1 2030
  send-lifetime 00:00:00 1 Jan 1993 00:05:00 Jan 1 2030
  key-string CISCO10
 key 20
  accept-lifetime 00:00:00 1 Jan 2030 infinite
  send-lifetime 00:00:00 1 Jan 2030 infinite
  key-string CISCO20
```

By setting the accept-lifetime 10 minutes longer we should avoid losing connectivity when it's time to roll over to the next key. On classic-mode we enable authentication on the interface and in named-mode under our af-interface.

```
! R1 - R4

int tu0
 ip authentication key-chain eigrp 100 KEY\_ROTATION
 ip authentication mode eigrp 100 md5

! R5

router eigrp MULTI-AF
 address-family ipv4 autonomous-system 100
 af-interface Tu0
  authentication key-chain KEY\_ROTATION
  authentication mode md5
```

Let's verify again:

```
R2#debug eigrp packets hello
 (HELLO)
EIGRP Packet debugging is on
R2#
**EIGRP: received packet with MD5 authentication, key id = 10**
EIGRP: Received HELLO on Tu0 - paklen 60 nbr 155.1.0.5
 AS 100, Flags 0x0:(NULL), Seq 0/0 interfaceQ 0/0 iidbQ un/rely 0/0 peerQ un/rely 0/0

R2#show key chain 
Key-chain KEY\_ROTATION:
 key 10 -- text "CISCO10"
 accept lifetime (00:00:00 UTC Jan 1 1993) - (00:15:00 UTC Jan 1 2030) **\[valid now\]**
 send lifetime (00:00:00 UTC Jan 1 1993) - (00:05:00 UTC Jan 1 2030) **\[valid now\]**
 key 20 -- text "CISCO20"
 accept lifetime (00:00:00 UTC Jan 1 2030) - (infinite)
 send lifetime (00:00:00 UTC Jan 1 2030) - (infinite)
```

Let's change our clock and see what happens:

```
! R5

set clock 23:59:00 Dec 31 2029

R5#sh key chain 
Key-chain KEY\_ROTATION:
 key 10 -- text "CISCO10"
 accept lifetime (00:00:00 UTC Jan 1 1993) - (00:15:00 UTC Jan 1 2030) **\[valid now\]**
 send lifetime (00:00:00 UTC Jan 1 1993) - (00:05:00 UTC Jan 1 2030) **\[valid now\]**
 key 20 -- text "CISCO20"
 accept lifetime (00:00:00 UTC Jan 1 2030) - (infinite)
 send lifetime (00:00:00 UTC Jan 1 2030) - (infinite)

R5#sh clock 
**00:00:04.531 UTC Tue Jan 1 2030**
R5#sh key chain 
Key-chain KEY\_ROTATION:
 key 10 -- text "CISCO10"
 accept lifetime (00:00:00 UTC Jan 1 1993) - (00:15:00 UTC Jan 1 2030) **\[valid now\]**
 send lifetime (00:00:00 UTC Jan 1 1993) - (00:05:00 UTC Jan 1 2030) **\[valid now\]**
 key 20 -- text "CISCO20"
 accept lifetime (00:00:00 UTC Jan 1 2030) - (infinite) **\[valid now\]**
 send lifetime (00:00:00 UTC Jan 1 2030) - (infinite) **\[valid now\]**
```

As NTP is horribly slow to converge, or if they just think master is insane with the big change in time, I manually set the time in R1 - R4 to speed things up. Checking debug in R1 we're still receiving key-10 as it's still valid with our current config:

```
**EIGRP: received packet with MD5 authentication, key id = 10**
EIGRP: Received HELLO on Tu0 - paklen 60 nbr 155.1.0.5
 AS 100, Flags 0x0:(NULL), Seq 0/0 interfaceQ 0/0 iidbQ un/rely 0/0 peerQ un/rely 0/0
EIGRP: Sending HELLO on Gi1.146 - paklen 20
 AS 100, Flags 0x0:(NULL), Seq 0/0 interfaceQ 0/0 iidbQ un/rely 0/0
```
5 minutes later we should see the switch to key-20:
```
R5#sh clock
00:05:12.716 UTC Tue Jan 1 2030
R5#sh key
R5#sh key chain
Key-chain KEY\_ROTATION:
 key 10 -- text "CISCO10"
 accept lifetime (00:00:00 UTC Jan 1 1993) - (00:15:00 UTC Jan 1 2030) **\[valid now\]**
 send lifetime (00:00:00 UTC Jan 1 1993) - (00:05:00 UTC Jan 1 2030)
 key 20 -- text "CISCO20"
 accept lifetime (00:00:00 UTC Jan 1 2030) - (infinite) **\[valid now\]**
 send lifetime (00:00:00 UTC Jan 1 2030) - (infinite) \[**valid now\]**
```
And in R1 we can see R5 is now using key20 instead without losing adjacency:
```
EIGRP: received packet with MD5 authentication, key id = 20
EIGRP: Received HELLO on Tu0 - paklen 60 nbr 155.1.0.5

R1#sh ip eigrp neighbors 
EIGRP-IPv4 Neighbors for AS(100)
H Address Interface Hold Uptime SRTT RTO Q Seq
 (sec) (ms) Cnt Num
1 155.1.0.5 Tu0 11 **01:11:08** 46 1398 0 39
```
