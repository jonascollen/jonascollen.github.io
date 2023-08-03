---
title: Python - Automatisera DHCP-scopes, IPTables- & interface-config
date: 2018-02-15 19:30
author: Jonas Collén
comments: true
categories: [Python]
---
Tänkte fixa ihop ett litet script för att automatiskt generera DHCP-scopes, IPTables- & router-config, går finfint att antingen bygga ihop med Flask och ta in näten via ett formulär från användaren eller hämta från valfri källa. Vi räknar med att näten som kommer in är i formatet "192.168.0.0/24" med radbrytning mellan varje nät.  Vi separerar även typen av nät genom att lägga till filnamnet vi hämtat det från. Exempelvis:

```
IPTV.txt
192.168.0.0/24
192.168.1.0/24
192.168.2.0/24
192.168.4.0/24

VOIP.txt
172.16.0.0/24
172.16.2.0/24
172.16.3.0/24
172.16.4.0/24

SURF.txt
10.10.0.0/24
10.10.1.0/24
10.10.2.0/24
10.10.3.0/24

OFFNET.txt
132.10.0.0/24
147.2.0.0/24
```
Börjar med att importa biblioteket netaddr & datetime samt att vi sparar ner ovanstående till variabeln "content"på valfritt vis, skapa sedan listor som vi kommer använda för att sortera ut näten.

```python
from netaddr import *
from datetime import datetime

iptv = []
voip = []
surf = []
offnet = []
```
Vi loopar sedan genom variabeln ord för ord och sätter en tag för respektive nättyp när vi får träff. Scriptet vet på vis att efterföljande ipnät tillhör ex. IPTV, och när den matchar mot VOIP sparas de i "voip" etc.

```python
 for line in content.split():
  # Tagging types of networks
  if "IPTV.txt" in line:
    net = "iptv"
  elif "VOIP.txt" in line:
    net = "voip"
  elif "SURF.txt" in line:
    net = "surf"
  elif "OFFNET.txt" in line:
    net = "offnet"
 
  # Saving networks to separate lists
  if net == "iptv":
    if "0/24" in line:
      iptv.append(line)
  elif net == "voip":
    if "0/24" in line:
      voip.append(line)
  elif net == "surf":
    if "0/24" in line:
      surf.append(line)
  elif net == "offnet":
    if "0/24" in line:
    offnet.append(line)
```

För validering kan vi även summera hur många nät vi hittat i respektive kategori.

```python
validation = '''Found the following networks:
 IPTV: {iptv}
 VoIP: {voip}
 SURF: {surf}
 OFFNET: {offnet}'''.format(iptv=len(iptv), voip=len(voip), surf=len(surf), offnet=len(offnet))
```

Nu har vi allt vi behöver för att börja skapa lite filer, IP-tables till att börja med. I detta fall vill jag endast skapa firewall-öppningar för näten IPTV, VOIP & Surf, vi måste därför separera dem från OFFNET-näten. Vi fixar även en tidsstämpel vi kan använda till att skapa unika filnamn.

```python
 # Creates a list of lists - all networks that needs firewall opening 
 fire_nets = [iptv, voip, surf]

 # Used for unique filename
 tfconfig_time = datetime.now().strftime("%Y%m%d-%H:%M:%S")
```
I IPTables vill vi exempelvis öppna för UDP-förfrågningar på port 67 (DHCP), vi loopar därför genom fire_nets och lägger in detta i en fördefinierad iptables-rad. Vi behöver även en funktion för att traversa listor i listor, utan detta blir outputen [192.168.0.0/24, 192.168.1.0/24...], [172.16.0.0/24, 172.16.1.0/24..], [10.10.0.0/24, 10.10.1.0/24...] vilket vi inte vill ha.

```python
def traverse(o, tree_types=(list, tuple)):
  if isinstance(o, tree_types):
    for value in o:
      for subvalue in traverse(value, tree_types):
        yield subvalue
      else:
       yield o

 # Appends a statement for each network in list that needs a firewall opening
 firewall = []
 for network in traverse(fire_nets):
   firewall.append('-A INPUT -p udp -s {NET} --dport 67 -j ACCEPT'.format(NET=network))
```

Vi spar sedan ner ovanstående nät i en fil.

```python
with open('/firewall/iptables-{time}.txt'.format(time=tfconfig_time), 'w') as outfile:
# Prints firewall-statements to file, uses print to get newline per statement - instead of insterting /n to text
 for fire in firewall:
   print(fire, file=outfile)
```
Så enkelt har vi skapat vår IP-tables configuration, nästa steg blir att skapa routerconfig. Vi kontrollerar så att respektive lista inte är tom (isf behöver vi ingen interface-config) och använder sedan modulen netaddr vi importerade tidigare för att hämta ut första giltiga ip-adress och subnätmask ur respektive nät.

```python
with open('/loopbacks/interfaces-{time}.txt'.format(time=tfconfig_time), 'w') as outfile:

  if iptv:
    print("ninterface Lo500", file=outfile)
    for network in iptv:
      loopaddr = IPNetwork(network) # Saves IP-info (uses netaddr)
      print(' ipv4 address ' + str(loopaddr[1]) + ' ' + str(loopaddr.netmask) + ' secondary', file=outfile) # Prints 1st available hostadress + netmask from variable

  if voip:
   print("ninterface Lo600", file=outfile)
   for network in voip:
     loopaddr = IPNetwork(network)
     print(' ipv4 address ' + str(loopaddr[1]) + ' ' + str(loopaddr.netmask) + ' secondary', file=outfile)

  if surf:
   print("ninterface Lo700", file=outfile)
   for network in surf:
     loopaddr = IPNetwork(network)
     print(' ipv4 address ' + str(loopaddr[1]) + ' ' + str(loopaddr.netmask) + ' secondary', file=outfile)

  if offnet:
   print("ninterface Lo800", file=outfile)
   for network in offnet:
     loopaddr = IPNetwork(network)
     print(' ipv4 address ' + str(loopaddr[1]) + ' ' + str(loopaddr.netmask) + ' secondary', file=outfile)
```

Sista steget blir att skapa DHCP-scope filer, detta blir något krångligare men förhoppningsvis är det inte allt för rörigt. Vi skapar först en textfil för respektive nät vilket kommer användas som mall för våra scopes. Exempelvis:

```
/templates/voip-scope.txt

subnet $NET netmask $MASK {
 pool {
  allow members of "voip";
  deny members of "notallowed";
  range $FIRST $LAST;
  option routers $OPTROUTER;
  option broadcast-address $BROADCAST;
  option subnet-mask $MASK;
  }
}
```

Skapa liknande filer för alla nättyper, vi använder sedan netaddr igen för att få ut all info vi behöver till scope-filen från respektive nät genom att loopa varje lista.  Vi öppnar även vår scope-mall och ersätter ex. $FIRST $LAST med infon vi redan hämtat ut genom användandet av dictionarys. Supersmidigt! Glöm inte att appenda infon så vi inte skriver över filen för varje loop-körning.

```python
 # Generate scopes for VoIP
 for network in voip:
   ip = IPNetwork(network)
   net = str(ip[0])
   mask = str(ip.netmask)
   gwip = str(ip[1])
   bcip = str(ip.broadcast)
   first_ip = str(ip[2])
   last_ip = str(ip.broadcast - 1)
   # Testing
   #print('Net: {net}, Mask: {mask}, GW: {gwip}, BC: {bcip}, host {first_ip}, last useable: {last_ip}'.format(net=net, mask=mask, gwip=gwip, bcip=bcip, first_ip=first_ip, last_ip=last_ip))

  replacements = {'$NET':net, '$MASK':mask, '$FIRST':first_ip, '$LAST':last_ip, '$OPTROUTER':gwip, '$BROADCAST':bcip}

  with open('/templates/voip-scope.txt') as infile, open('/dhcp-scopes/voip-{time}.txt'.format(time=tfconfig_time), 'a') as outfile:
    for line in infile:
      for src, target in replacements.items():
        line = line.replace(src, target)
        outfile.write(line)
```

Vi gör sedan precis likadant för respektive scope-fil. Done!

```
#### Surf DHCP-Scope Networks ####
 
  subnet 10.10.0.0 netmask 255.255.255.0 {
    pool {
         allow members of "surf";
         deny members of "notallowed";
         range 10.10.0.2 10.10.0.254;
         option routers 10.10.0.1;
         option broadcast-address 10.10.0.255;
         option subnet-mask 255.255.255.0;
         }
  }
  subnet 10.10.1.0 netmask 255.255.255.0 {
    pool {
         allow members of "surf";
         deny members of "notallowed";
         range 10.10.1.2 10.10.1.254;
         option routers 10.10.1.1;
         option broadcast-address 10.10.1.255;
         option subnet-mask 255.255.255.0;
         }
  }
  subnet 10.10.2.0 netmask 255.255.255.0 {
    pool {
         allow members of "surf";
         deny members of "notallowed";
         range 10.10.2.2 10.10.2.254;
         option routers 10.10.2.1;
         option broadcast-address 10.10.2.255;
         option subnet-mask 255.255.255.0;
         }
  }
  subnet 10.10.3.0 netmask 255.255.255.0 {
    pool {
         allow members of "surf";
         deny members of "notallowed";
         range 10.10.3.2 10.10.3.254;
         option routers 10.10.3.1;
         option broadcast-address 10.10.3.255;
         option subnet-mask 255.255.255.0;
         }
  }

#### Interface config ####

interface Lo700
 ipv4 address 10.10.0.1 255.255.255.0 secondary
 ipv4 address 10.10.1.1 255.255.255.0 secondary
 ipv4 address 10.10.2.1 255.255.255.0 secondary
 ipv4 address 10.10.3.1 255.255.255.0 secondary

#### Firewall config ####

-A INPUT -p udp -s 10.10.0.0/24 --dport 67 -j ACCEPT
-A INPUT -p udp -s 10.10.1.0/24 --dport 67 -j ACCEPT
-A INPUT -p udp -s 10.10.2.0/24 --dport 67 -j ACCEPT
-A INPUT -p udp -s 10.10.3.0/24 --dport 67 -j ACCEPT
```

Har implementerat något liknande på jobbet om än med betydligt fler funktioner då vi hanterar en rätt stor mängd nät mer eller mindre dagligen och det har helt klart förenklat arbetet. Kanske kan leda till lite inspiration för någon annan.. :)
