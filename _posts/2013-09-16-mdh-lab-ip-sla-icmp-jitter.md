---
title: MDH Lab - IP SLA ICMP & Jitter
date: 2013-09-16 16:23
comments: true
categories: [IP-SLA]
tags: [jitter]
---
Topologi
--------

[![lab-ipsla](/assets/images/2013/09/lab-ipsla.png?w=553)](/assets/images/2013/09/lab-ipsla.png)

Objectives
----------

*   Configure trunking, VTP, and SVIs.
*   Implement IP SLAs to monitor various network performance characteristics.

Background
----------

Cisco IOS IP service level agreements (SLAs) allow users to monitor network performance between Cisco devices (switches or routers) or from a Cisco device to a remote IP device. Cisco IOS IP SLAs can be applied to VoIP and video applications as well as monitoring end-to-end IP network performance.In this lab, you configure trunking, VTP, and SVIs. You configure IP SLA monitors to test ICMP echo network  and jitter performance between S1 and each switch.

Genomförande
------------

Hoppar över grundkonfigen för Etherchannels/Vlan/VTP/Trunkar så all L2-konfig är redan klart. Se [tidigare inlägg](http://roadtoccie.se/2013/09/16/mdh-lab-hsrp/ "MDH Lab – HSRP") om detta skulle vara intressant.. Vi börjar med att konfa upp L3 för samtliga switchar. S1

```
S1(config)#int vlan 1
 S1(config-if)#ip add 172.16.1.1 255.255.255.0
 S1(config-if)#no shut
 S1(config-if)#int vlan 100
 S1(config-if)#ip add 172.16.100.1 255.255.255.0
 S1(config-if)#int vlan 200
 S1(config-if)#ip add 172.16.200.1 255.255.255.0
 S1(config-if)#exit
 S1(config)#ip routing
```
S2
```
S2(config)#inte vlan 1
 S2(config-if)#ip add 172.16.1.102 255.255.255.0
 S2(config-if)#no shut
```
S3
```
S3(config)#inte vlan 1
 S3(config-if)#ip add 172.16.1.103 255.255.255.0
 S3(config-if)#no shut
```
Då återstår det endast att sätta upp IP SLA. Enligt labben ska vi konfigurera upp ICMP-Echo & Jitter på S1 mot både S2 & S3. För att kunna testa just jitter behöver vi konfigurera IP SLA-responders på S2 & S3, detta behövs dock inte om vi endast vill använda ICMP. S2 & S3
```
S2(config)#ip sla responder
 S2(config)#ip sla responder udp-echo ipaddress 172.16.1.1 port 2525
S3(config)#ip sla responder
 S3(config)#ip sla responder udp-echo ipaddress 172.16.1.1 port 2525
```
Vi konfar sedan upp SLA-mätningen på S1. Vi behöver en SLA-instans per mätnings-funktion, då vi vill testa både ICMP & Jitter behövs därför två mot S2 och två mot S3. S1
```
S1(config)#ip sla 20
 S1(config-ip-sla-echo)#icmp-echo 172.16.1.102
 S1(config-ip-sla-echo)#frequency 5
 S1(config-ip-sla-echo)#timeout 1000
 S1(config-ip-sla-echo)#exit
 S1(config)#
S1(config)#ip sla 25
 S1(config-ip-sla)#udp-jitter 172.16.1.102 2525
 S1(config-ip-sla-jitter)#frequency 20
 S1(config-ip-sla-jitter)#precision milliseconds
 S1(config-ip-sla-jitter)#timeout 1000
 S1(config-ip-sla-jitter)#exit
 S1(config)#
S1(config)#ip sla 30
 S1(config-ip-sla)#icmp-echo 172.16.1.103
 S1(config-ip-sla-echo)#frequency 5
 S1(config-ip-sla-echo)#timeout 1000
 S1(config-ip-sla-echo)#exit
 S1(config)#
S1(config)#ip sla 35
 S1(config-ip-sla)#udp-jitter 172.16.1.103 2525
 S1(config-ip-sla-jitter)#frequency 20
 S1(config-ip-sla-jitter)#precision milliseconds
 S1(config-ip-sla-jitter)#timeout 1000
 S1(config-ip-sla-jitter)#exit
 S1(config)#end
```
Vi behöver sedan bara schemalägga och starta våra IP SLA-mätningar.
```
S1(config)#ip sla schedule 20 life forever start-time now
S1(config)#ip sla schedule 25 life forever start-time now
S1(config)#ip sla schedule 30 life forever start-time now
S1(config)#ip sla schedule 35 life forever start-time now
```
Vi kan verifiera vår IP-SLA konfig med "sh ip sla configuration".
```
S1#sh ip sla configuration 25
IP SLAs, Infrastructure Engine-II.
Entry number: 25
Owner: 
Tag: 
Type of operation to perform: udp-jitter
Target address/Source address: 172.16.1.102/0.0.0.0
Target port/Source port: 2525/0
Type Of Service parameter: 0x0
Request size (ARR data portion): 32
Operation timeout (milliseconds): 1000
Packet Interval (milliseconds)/Number of packets: 20/10
Verify data: No
Vrf Name: 
Control Packets: enabled
Schedule:
 Operation frequency (seconds): 20
 Next Scheduled Start Time: Start Time already passed
 Group Scheduled : FALSE
 Randomly Scheduled : FALSE
 Life (seconds): Forever
 Entry Ageout (seconds): never
 Recurring (Starting Everyday): FALSE
 Status of entry (SNMP RowStatus): Active
Threshold (milliseconds): 5000
Distribution Statistics:
 Number of statistic hours kept: 2
 Number of statistic distribution buckets kept: 1
 Statistic distribution interval (milliseconds): 20
Enhanced History:
```
Det går även att verifiera på S2 & S3 så att de verkligen tar emot förfrågningar från S1.
```
S2#sh ip sla responder
IP SLAs Responder is: Enabled
Number of control message received: 8 Number of errors: 0 
Recent sources:
 172.16.1.1 [01:13:22.643 UTC Mon Mar 1 1993]
 172.16.1.1 [01:13:02.644 UTC Mon Mar 1 1993]
 172.16.1.1 [01:12:42.638 UTC Mon Mar 1 1993]
 172.16.1.1 [01:12:22.639 UTC Mon Mar 1 1993]
 172.16.1.1 [01:12:02.641 UTC Mon Mar 1 1993]
Recent error sources:
udpEcho Responder: 
 IP Address Port
 172.16.1.1 2525
```
För att se resultatet av våra mätningar, använd "show ip sla statistics".
```
S1#sh ip sla statistics
Round Trip Time (RTT) for Index 20
 Latest RTT: 8 ms
Latest operation start time: *01:14:11.767 UTC Mon Mar 1 1993
Latest operation return code: OK
Number of successes: 40
Number of failures: 1
Operation time to live: Forever

Round Trip Time (RTT) for Index 25
 Latest RTT: 2 ms
Latest operation start time: *01:13:55.904 UTC Mon Mar 1 1993
Latest operation return code: OK
RTT Values
 Number Of RTT: 10
 RTT Min/Avg/Max: 2/2/4 ms
Latency one-way time milliseconds
 Number of Latency one-way Samples: 0
 Source to Destination Latency one way Min/Avg/Max: 0/0/0 ms
 Destination to Source Latency one way Min/Avg/Max: 0/0/0 ms
Jitter time milliseconds
 Number of SD Jitter Samples: 9
 Number of DS Jitter Samples: 9
 Source to Destination Jitter Min/Avg/Max: 0/1/2 ms
 Destination to Source Jitter Min/Avg/Max: 0/1/1 ms
Packet Loss Values
 Loss Source to Destination: 0 Loss Destination to Source: 0
 Out Of Sequence: 0 Tail Drop: 0 Packet Late Arrival: 0
Voice Score Values
 Calculated Planning Impairment Factor (ICPIF): 0
 Mean Opinion Score (MOS): 0
Number of successes: 11
Number of failures: 0
Operation time to live: Forever

Round Trip Time (RTT) for Index 30
 Latest RTT: 1 ms
Latest operation start time: *01:14:15.038 UTC Mon Mar 1 1993
Latest operation return code: OK
Number of successes: 40
Number of failures: 0
Operation time to live: Forever

Round Trip Time (RTT) for Index 35
 Latest RTT: 2 ms
Latest operation start time: *01:14:04.217 UTC Mon Mar 1 1993
Latest operation return code: OK
RTT Values
 Number Of RTT: 10
 RTT Min/Avg/Max: 1/2/3 ms
Latency one-way time milliseconds
 Number of Latency one-way Samples: 0
 Source to Destination Latency one way Min/Avg/Max: 0/0/0 ms
 Destination to Source Latency one way Min/Avg/Max: 0/0/0 ms
Jitter time milliseconds
 Number of SD Jitter Samples: 9
 Number of DS Jitter Samples: 9
 Source to Destination Jitter Min/Avg/Max: 0/1/1 ms
 Destination to Source Jitter Min/Avg/Max: 0/1/2 ms
Packet Loss Values
 Loss Source to Destination: 0 Loss Destination to Source: 0
 Out Of Sequence: 0 Tail Drop: 0 Packet Late Arrival: 0
Voice Score Values
 Calculated Planning Impairment Factor (ICPIF): 0
 Mean Opinion Score (MOS): 0
Number of successes: 10
Number of failures: 0
Operation time to live: Forever
```
IP SLA går dock att använda till så mycket mer än att bara köra lite jitter/udp-test sen, exempelvis PBR/HSRP-tracking etc, vilket jag kommer ta upp i senare inlägg. :)
