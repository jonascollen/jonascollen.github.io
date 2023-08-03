---
title: MDH Lab - HSRP & Securing L2
date: 2013-09-16 17:41
comments: true
categories: [HSRP]
---
Topologi
--------

![6-1](/assets/images/2013/09/6-1.png)

Objectives
----------

*   Secure the Layer 2 network against MAC flood attacks.
*   Prevent DHCP spoofing attacks.
*   Prevent unauthorized access to the network using AAA and 802.1X.

Background
----------

A fellow network engineer that you have known and trusted for many years has invited you to lunch this week. At lunch, he brings up the subject of network security and how two of his former co-workers had been arrested for using different Layer 2 attack techniques to gather data from other users in the office for their own personal gain in their careers and finances. The story shocks you because you have always known your friend to be very cautious with security on his network. His story makes you realize that your business network has been cautious with external threats, Layer 3–7 security, firewalls at the borders, and so on, but insufficient at Layer 2 security and protection inside the local network. When you get back to the office, you meet with your boss to discuss your concerns. After reviewing the company’s security policies, you begin to work on a Layer 2 security policy. First, you establish which network threats you are concerned about and then put together an action plan to mitigate these threats. While researching these threats, you learn about other potential threats to Layer 2 switches that might not be malicious but could threaten network stability. You decide to include these threats in the policies as well. Other security measures need to be put in place to further secure the network, but you begin with configuring the switches against a few specific types of attacks, including **MAC flood attacks, DHCP spoofing attacks, and unauthorized access** to the local network. You plan to test the configurations in a lab environment before placing them into production.

Genomförande
------------

Hoppar återigen över grundkonfigen för att få upp Port-channels/Trunking/VLAns, se [tidigare inlägg](http://roadtoccie.se/2013/09/16/mdh-lab-hsrp/ "MDH Lab – HSRP") för detta. Vi börjar med att sätta upp HSRP mellan S1 & S3. S1

```
S1(config)#int vlan 1 
S1(config-if)#ip add 172.16.1.10 255.255.255.0
S1(config-if)#standby 1 ip 172.16.1.1
S1(config-if)#standby 1 priority 100
S1(config-if)#standby 1 preempt
S1(config-if)#
S1(config-if)#int vlan 100
S1(config-if)#ip add 172.16.100.10 255.255.255.0
S1(config-if)#standby 1 ip 172.16.100.1
S1(config-if)#standby 1 priority 150
S1(config-if)#standby 1 preempt
S1(config-if)#
S1(config-if)#int vlan 200
S1(config-if)#ip add 172.16.200.10 255.255.255.0
S1(config-if)#standby 1 ip 172.16.200.1
S1(config-if)#standby 1 priority 100
S1(config-if)#standby 1 preempt
S1(config-if)#end
```
S3
```
S3(config)#int vlan 1 
S3(config-if)#ip add 172.16.1.30 255.255.255.0
S3(config-if)#standby 1 ip 172.16.1.1
S3(config-if)#standby 1 priority 150
S3(config-if)#standby 1 preempt
S3(config-if)#
S3(config-if)#int vlan 100
S3(config-if)#ip add 172.16.100.30 255.255.255.0
S3(config-if)#standby 1 ip 172.16.100.1
S3(config-if)#standby 1 priority 100
S3(config-if)#standby 1 preempt
S3(config-if)#
S3(config-if)#int vlan 200
S3(config-if)#ip add 172.16.200.30 255.255.255.0
S3(config-if)#standby 1 ip 172.16.200.1
S3(config-if)#standby 1 priority 150
S3(config-if)#standby 1 preempt
S3(config-if)#
S1#sh standby brief
 P indicates configured to preempt.
 |
Interface Grp Pri P State Active Standby Virtual IP
Vl1 1 100 P Standby 172.16.1.30 local 172.16.1.1
Vl100 1 150 P Active local 172.16.100.30 172.16.100.1
Vl200 1 100 P Standby 172.16.200.30 local 172.16.200.1
```
Nästa uppgift var att säkra vårat nät mot DHCP-spoofing attacker. För att lyckas med detta kan vi använda oss av "[dhcp snooping](http://www.cisco.com/en/US/docs/switches/lan/catalyst6500/ios/12.2SX/configuration/guide/snoodhcp.html)". Då endast S2 är access-layer switch med användare inkopplade behöver vi bara konfa snooping där. S2
```
S2(config)#!Aktiverar DHCP-snooping funktionen
S2(config)#ip dhcp snooping 
S2(config)#!Startar DHCP-snooping för vlan 100 & 200
S2(config)#ip dhcp snooping vlan 100,200
```
Alla portar räknas som per default untrusted, dvs får EJ skicka DHCPOFFERs & DHCPACKs. Vi måste därför konfigurera trusted där vi har vår DHCP-server. I detta fall har vi ingen lokalt på lanet, därför sätter vi trunk-portarna till trusted.
```
S2(config)#inte range fa0/1 - 4 , po1 - 2
S2(config-if-range)#ip dhcp snooping trust
```
Vi kan verifiera vår konfig med "sh ip dhcp snooping".
```
S2#sh ip dhcp snooping
Switch DHCP snooping is enabled
DHCP snooping is configured on following VLANs:
100,200
DHCP snooping is operational on following VLANs:
100,200
DHCP snooping is configured on the following L3 Interfaces:
Insertion of option 82 is enabled
 circuit-id format: vlan-mod-port
 remote-id format: MAC
Option 82 on untrusted port is not allowed
Verification of hwaddr field is enabled
Verification of giaddr field is enabled
DHCP snooping trust/rate is configured on the following Interfaces:
Int
*Mar 1 02:14:22.416: %SYS-5-CONFIG_I: Configured from console by consoleerface Trusted Rate limit (pps)
------------------------ ------- ----------------
FastEthernet0/1 yes unlimited
FastEthernet0/2 yes unlimited
FastEthernet0/3 yes unlimited
FastEthernet0/4 yes unlimited
Port-channel1 yes unlimited
Port-channel2 yes unlimited
```
MAC-flood attacker blir bara lite repetition från CCNA, genom att begränsa antal mac-adresser en port kan lära sig behöver vi inte vara rädda för att råka ut för cam-table overflow. Även här behöver vi endast konfa S2 då det är den enda switchen i access-layer.
```
S2(config)#int range fa0/6 - 20 
S2(config-if-range)#switchport mode access
S2(config-if-range)#switchport access vlan 100
S2(config-if-range)#switchport port-security
S2(config-if-range)#switchport port-security max 2
S2(config-if-range)#switchport port-security mac-address sticky
S2(config-if-range)#spanning-tree portfast
S2(config-if-range)#spanning-tree bpduguard enable
```
Detta gör att switchen endast kan lära sig max 2st mac-adresser per interface (2x15 =  30 mac-adresser). För att begränsa access till nätet är ett alternativ att använda oss av 802.1x, vilket kräver att användarna autentiserar sig innan porten slår över till forwarding. Mer info om dot1x finns att läsa [här.](http://www.cisco.com/en/US/docs/switches/lan/catalyst6500/ios/12.2SX/configuration/guide/dot1x.html)
```
S2(config)#aaa new-model
S2(config)#!Autentiserar anvandare mot anvandardatabasen som ligger lokalt pa switchen
S2(config)#aaa authentication dot1x default local
S2(config)#!aktiverar dot1x
S2(config)#dot1x system-auth-control
S2(config)#!skapar anvndare
S2(config)#username admin1 password cisco
S2(config)#username user1 password cisco
S2(config)#username user2 password cisco
S2(config)#inte range fa0/6 - 20
S2(config-if-range)#!aktiverar dot1x-autentisering pa interfacen
S2(config-if-range)#dot1x port-control auto
S2(config-if-range)#
*Mar 1 02:28:23.114: %LINEPROTO-5-UPDOWN: Line protocol on Interface FastEthernet0/11, changed state to down
*Mar 1 02:28:23.131: %LINEPROTO-5-UPDOWN: Line protocol on Interface FastEthernet0/18, changed state to down
```
Observera att både Fa0/11 & Fa0/18 genast gick ner efter vi la in konfigen, detta pga användarna ej är autentiserade ännu.
```
S2#sh dot1x interface fa0/11
Dot1x Info for FastEthernet0/11
-----------------------------------
PAE = AUTHENTICATOR
PortControl = AUTO
ControlDirection = Both 
HostMode = SINGLE_HOST
Violation Mode = PROTECT
ReAuthentication = Disabled
QuietPeriod = 60
ServerTimeout = 30
SuppTimeout = 30
ReAuthPeriod = 3600 (Locally configured)
ReAuthMax = 2
MaxReq = 2
TxPeriod = 30
RateLimitPeriod = 0
```
Rekommenderat är dock att koppla autentiseringen mot en extern RADIUS-server istället.
