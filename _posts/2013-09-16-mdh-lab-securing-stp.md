---
title: MDH Lab - Securing STP
date: 2013-09-16 22:21
comments: true
categories: [STP]
tags: [secure,bpduguard,udld]
---
Topologi
--------

![stp-secure](/assets/images/2013/09/stp-secure.png)

Objectives
----------

*   Secure the Layer 2 spanning-tree topology with BPDU guard.
*   Protect the primary and secondary root bridge with root guard.
*   Protect switch ports from unidirectional links with UDLD.

Background
----------

This lab is a continuation of Lab 6-1 and uses the network configuration set up in that lab. In this lab, you will secure the network against possible spanning-tree disruptions, such as rogue access point additions and the loss of stability to the root bridge by the addition of switches to the network. The improper addition of switches to the network can be either malicious or accidental. In either case, the network can be secured against such a disruption.

Genomförande
------------

Då det är precis samma topologi & HSRP-konfiguration som [förra labben](http://roadtoccie.se/2013/09/16/mdh-lab-hsrp-securing-l2/ "MDH Lab – HSRP & Securing L2") använder jag samma konfig nu.

## BPDU-Guard

[BPDU-guard](http://www.cisco.com/en/US/tech/tk389/tk621/technologies_tech_note09186a008009482f.shtml) konfigurerade jag redan i förra labben men vi kan väl ta det igen bara för att repetera.

```
S2(config)#int range fa0/6 - 20 
S2(config-if-range)#switchport mode access
S2(config-if-range)#switchport access vlan 100
S2(config-if-range)#switchport port-security
S2(config-if-range)#switchport port-security max 2
S2(config-if-range)#switchport port-security mac-address sticky
S2(config-if-range)#spanning-tree portfast
**S2(config-if-range)#spanning-tree bpduguard enable**
```
Observera att vi endast konfigurerar BPDU-guard på access-portar, och fungerar endast om vi även använder PortFast.

### Root-guard

Root-guard skyddar mot switchar som felaktigt skickar ut BPDU-paket med högre Bridge-priority för att försöka ta över rollen som root-bridge. I vår topologi har vi endast tre switchar, med S1 som primary för vl1 & vl200, och S2 som primary för vl100 kan vi enkelt säga att vi aldrig vill att access-switchen ska kunna bli root för något av vlanen. Vi ställer därför in root-guard på portarna ner mot S2 i både S1 & S3.
```
S1(config)#spanning-tree vlan 1,200 root primary
S1(config)#spanning-tree vlan 100 root secondary
S3(config)#spanning-tree vlan 100 root primary
S3(config)#spanning-tree vlan 1,200 root secondary
S1(config)#int range fa0/1 - 2 , po1
S1(config-if-range)#spanning-tree guard root
S1(config-if-range)#
*Mar 1 03:23:32.781: %SPANTREE-2-ROOTGUARD_CONFIG_CHANGE: Root guard enabled on port Port-channel1.
S3(config)#int range fa0/1 - 2 , po1
S3(config-if-range)#spanning-tree guard root
S3(config-if-range)#
*Mar 1 03:24:13.205: %SPANTREE-2-ROOTGUARD_CONFIG_CHANGE: Root guard enabled on port Port-channel1.
```
### UDLD

[UDLD](http://www.cisco.com/en/US/docs/switches/lan/catalyst3550/software/release/12.1_19_ea1/configuration/guide/swudld.html) använder vi för att upptäcka unidirectional-forwarding främst vid fiberuppkopplingar, men det finns även möjlighet att konfigurera detta på vanliga ethernet-interface också. ![UDLD](/assets/images/2013/09/udld.jpg) Så i mina ögon är aggresive-mode helt klart det bästa valet när vi konfigurerar upp UDLD, vilket vi gör enligt följande:
```
S1(config)#int range fa0/3 - 4 
S1(config-if-range)#udld port aggressive
S3(config)#int range fa0/3 - 4
S3(config-if-range)#udld port aggressive
```
Observera att vi endast lägger in det på de fysiska interfacen, det är ej möjligt att använda UDLD över en port-channel.
```
S1#sh udld fa0/3
Interface Fa0/3

Port enable administrative configuration setting: Enabled / in aggressive mode
Port enable operational state: Enabled / in aggressive mode
Current bidirectional state: Bidirectional
Current operational state: Advertisement - Single neighbor detected
Message interval: 7
Time out interval: 5
Entry 1
 ---
 Expiration time: 38
 Device ID: 1
 Current neighbor state: Bidirectional
 Device name: FDO1303Y3H8 
 Port ID: Fa0/3 
 Neighbor echo 1 device: FDO1303X3CX
 Neighbor echo 1 port: Fa0/3
Message interval: 15
 Time out interval: 5
 CDP Device name: S3
```

###  Storm-Control

Används för att förhindra att broadcast-trafik sänker våra förbindelser i s.k. broadcast-stormar (orsakat av en loop), mer info finns att läsa [här](http://www.cisco.com/en/US/docs/switches/lan/catalyst6500/ios/12.2SX/configuration/guide/storm.html). Själva konfigurationen är rätt enkel, vi lägger in detta på våra trunk-länkar mellan S1-S2-S3.

> Do not configure traffic storm control on ports that are members of an EtherChannel. Configuring traffic storm control on ports that are configured as members of an EtherChannel puts the ports into a suspended state.

Vi lägger därför in konfigen direkt på port-channeln istället.
```
S1(config)#int range po1 - 2
S1(config-if-range)#storm-control broadcast level ?
 <0.00 - 100.00> Enter rising threshold
 bps Enter suppression level in bits per second
 pps Enter suppression level in packets per second
S1(config-if-range)#storm-control broadcast level 60
S2(config)#inte range po1 - 2
S2(config-if-range)#storm-control bbroadcast level 60
S3(config)#int range po1 - 2
S3(config-if-range)#storm-control broadcast level 60
```
