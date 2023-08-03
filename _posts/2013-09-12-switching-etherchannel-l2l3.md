---
title: Switching - Etherchannel L2/L3
date: 2013-09-12 18:19
comments: true
categories: [Switch]
tags: [etherchannel]
---
Under dagens föreläsning om InterVLAN-routing kom det upp en intressant fråga gällande Etherchannel som ingen riktigt visste svaret på från följande bild: 

![etherchannel-l3-question](/assets/images/2013/09/etherchannel-l3-question.png) 
Frågan var - Räcker det inte att konfigurera "no switchport" på Port-channeln endast? Då jag testat det här en hel del tidigare tänkte jag det kunde vara lika bra att skriva ett inlägg om det. Det finns nämligen en hel del att tänka på när det kommer till i vilken ordning vi skapar vår port-channel. Om vi endast försöker konfigurera vår **port-channel**  till "no switchport" och sedan förlita oss på att den ska spegla detta till "parent"-interfacen lär vi se en hel del problem som jag tänkte ta och visa exempel på i det här inlägget. Om du istället är ute efter mer grundläggande information om Etherchannel se tidigare inlägg [här](http://roadtoccie.se/2013/09/07/switching-etherchannel/ "Switching – Etherchannel"). 

Något som du säkert redan känner till är att vid skapandet av Portchannel-interfacet så kollar switchen först vad "parent"-interfacen har för konfiguration och återspeglar sedan detta, likaså om vi i efterhand lägger till konfiguration på port-channeln så skickas detta (oftast) till våra parent-interface. Men inte helt självklart är att detta även gäller huruvida det ska skapas en **L2 eller L3-portchannel,** detta kan nämligen leda till en hel del problem... Vi tar och testar detta i praktiken:

```
S1(config)#inte range fa0/3 - 4
 S1(config-if-range)#shut
 S1(config-if-range)#channel-group 1 mode on
 Creating a port-channel interface Port-channel 1
 S1(config-if-range)#no shut
```
Om vi nu kollar status för port-channeln ser vi följande:
```
S1(config-if-range)#do sh etherchannel summary
 Flags: D - down P - bundled in port-channel
 I - stand-alone s - suspended
 H - Hot-standby (LACP only)
 R - Layer3 **S - Layer2**
 **U - in use** f - failed to allocate aggregator
M - not in use, minimum links not met
 u - unsuitable for bundling
 w - waiting to be aggregated
 d - default port
 Number of channel-groups in use: 1
 Number of aggregators: 1
Group Port-channel Protocol Ports
 ------+-------------+-----------+-----------------------------------------------
 1 Po1(**SU**) - Fa0/3(P) Fa0/4(P)
```
Vår port-channel är som synes en **Layer 2**-port och i status "In Use"/Bundled, allt ok så långt. Men frågan är vad som händer när vi försöker lägger in no switchport på Port-channel 1? Det bör ju även speglas till våra vanliga interface?
```
S1(config)#inte po1
S1(config-if)#no switchport 
**Command rejected: Not a convertible port.**
```
Det är nämligen så att om vi skapat en **Layer 2-portchannel** kan vi **EJ** lägga till **L3-interface** i vår channel-group. Observera att det inte är själva port-channeln som nekar "no switchport", utan det är när switchen försöker sätta parent-interfacen Fa0/3 & Fa0/4 till no switchport som det tar stopp. Vi kan verifiera detta med att även försöka lägga in det direkt på interfacen istället.
```
S1(config-if)#int range fa0/3 - 4
S1(config-if-range)#no switchport 
S1(config-if-range)#
```
Nu kanske du tänker - men vafan det fungerade! Vi fick ju inget felmeddelande? Men låt oss ta en titt på vår port-channel igen...
```
S1(config-if-range)#do sh etherchannel summary
Flags: **D - down** P - bundled in port-channel
 I - stand-alone s - suspended
 H - Hot-standby (LACP only)
 R - Layer3 **S - Layer2**
 U - in use f - failed to allocate aggregator
M - not in use, minimum links not met
 u - unsuitable for bundling
 w - waiting to be aggregated
 d - default port

Number of channel-groups in use: 1
Number of aggregators: 1
Group Port-channel Protocol Ports
------+-------------+-----------+-----------------------------------------------
1 Po1(**SD**) -
```
Observera att det inte längre finns några portar i vår channel-group! Let's repeat, vi kan **EJ** ha **L3-interface** i en **L2-portchannel**! Etherchanneln anser därför att interfacen fa0/3 & fa0/4 är invalid och plockar bort dem från gruppen, switchen är till och med så snäll att den tar bort vår channel-group konfig:
```
Building configuration...
Current configuration : 63 bytes
!
interface FastEthernet0/3
 no switchport
 no ip address
end
interface FastEthernet0/4
 no switchport
 no ip address
end
```
För att krångla till det lite ytterligare, när vi inte längre har några medlemmar i en channel-group kan vi faktiskt ändra vår L2-portchannel till L3. :)
```
S1(config-if-range)#int po1
% Command exited out of interface range and its sub-modes.
 Not executing the command for second and later interfaces
S1(config-if)#no switchport
*Mar 1 01:00:57.852: %LINK-3-UPDOWN: Interface Port-channel1, changed state to down
S1(config-if)#do sh etherchannel summary
Flags: **D - down** P - bundled in port-channel
 I - stand-alone s - suspended
 H - Hot-standby (LACP only)
 **R - Layer3** S - Layer2
 U - in use f - failed to allocate aggregator
M - not in use, minimum links not met
 u - unsuitable for bundling
 w - waiting to be aggregated
 d - default port

Number of channel-groups in use: 1
Number of aggregators: 1
Group Port-channel Protocol Ports
------+-------------+-----------+-----------------------------------------------
1 Po1(**RD**) -
```
Vi har nu en Layer 3-portchannel och kan därför lägga till våra L3-interface till gruppen igen.
```
S1(config-if)#int range fa0/3 - 4     
S1(config-if-range)#channel-group 1 mode on
*Mar 1 01:02:27.719: %LINK-3-UPDOWN: Interface Port-channel1, changed state to up
*Mar 1 01:02:28.726: %LINEPROTO-5-UPDOWN: Line protocol on Interface Port-channel1, changed state to up
```
För att summera - det är EXTREMT viktigt att tänka efter i vilken ordning vi skapar vår etherchannel.  Att konfigurera interfacen i följande ordning hade orsakat samma problem:
```
int range fa0/3 - 4
channel-group 1 mode on
no switchport

int port-channel 1
no switchport
```
Vill vi ha en L3-portchannel måste vi ALLTID specificera no switchport på interfacet INNAN vi ansluter den till en channel-group (som sedan i sin tur skapar port-channeln). Alternativt så skapar vi port-channeln själva först och konfigurerar parent-interface i efterhand:
```
interface port-channel 5
no switchport
ip add 10.0.0.1 255.255.255.0
int range fa0/3 - 4
no switchport 
channel-group 5 mode on
```
Här måste vi dock istället passa oss så vi inte försöker lägga till ett L2-interface till en L3-portchannel och återigen får problem. ;)
```
S1(config)#int port-channel 5
S1(config-if)#no switchport 
S1(config-if)#ip add 10.0.0.1 255.255.255.0
S1(config-if)#int range fa0/3 - 4
S1(config-if-range)#channel-group 5 mode on
**Command rejected (Port-channel5, Fa0/3): Either port is L2 and port-channel is L3, or vice-versa
% Range command terminated because it failed on FastEthernet0/3**
```
En hel del att tänka på med andra ord som kanske inte är så självklart alla gånger. :)
