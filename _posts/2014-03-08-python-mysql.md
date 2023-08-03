---
title: Python & MySQL
date: 2014-03-08 16:56
comments: true
categories: [Mysql, Python]
---
Fortsättning på det tidigare inlägget om [Python & SNMP](http://roadtoccie.se/2014/03/01/python/ "Python & SNMP"), tänkte komplettera med att spara ner resultatet till en databas istället för att endast skriva ut till terminalen. Först installerar vi MySQL via "sudo apt-get install mysql" och python-plugin:et "python-mysqldb". Databasen vi vill sätta upp ser ut enligt följande: ![database](/assets/images/2014/03/database.png) När installationen är klar logga in med användaren du skapade:


`mysql -h localhost -u root -p`

Vi skapar sedan databasen my_network:

`CREATE DATABASE my_network;`

Och sedan tabellerna unit & interface:

```
USE my_network;


CREATE TABLE unit

(

id int unsigned NOT NULL auto_increment,

ip_address VARCHAR(16),

name VARCHAR(40),

model VARCHAR(40),

PRIMARY KEY (id),

UNIQUE (ip_address)

);

CREATE TABLE interface

(

id int unsigned NOT NULL auto_increment,

unit_id int unsigned NOT NULL,

name VARCHAR(40),

mac_address VARCHAR(40),

ip_address VARCHAR(16),

mask VARCHAR(16),

PRIMARY KEY (id)

);
```


Vilket ger följande resultat:

```
mysql> SHOW TABLES;

+----------------------+

| Tables_in_my_network |

+----------------------+

| interface            |

| unit                 |

+----------------------+

2 rows in set (0.00 sec)

mysql> DESCRIBE unit;

+------------+------------------+------+-----+---------+----------------+

| Field      | Type             | Null | Key | Default | Extra          |

+------------+------------------+------+-----+---------+----------------+

| id         | int(10) unsigned | NO   | PRI | NULL    | auto_increment |

| ip_address | varchar(16)      | YES  | UNI | NULL    |                |

| name       | varchar(40)      | YES  |     | NULL    |                |

| model      | varchar(40)      | YES  |     | NULL    |                |

+------------+------------------+------+-----+---------+----------------+

4 rows in set (0.00 sec)

mysql> DESCRIBE interface;

+-------------+------------------+------+-----+---------+----------------+

| Field       | Type             | Null | Key | Default | Extra          |

+-------------+------------------+------+-----+---------+----------------+

| id          | int(10) unsigned | NO   | PRI | NULL    | auto_increment |

| unit_id     | int(10) unsigned | NO   | MUL | NULL    |                |

| name        | varchar(40)      | YES  |     | NULL    |                |

| mac_address | varchar(40)      | YES  |     | NULL    |                |

| ip_address  | varchar(16)      | YES  |     | NULL    |                |

| mask        | varchar(16)      | YES  |     | NULL    |                |

+-------------+------------------+------+-----+---------+----------------+

6 rows in set (0.00 sec)
```

Vi kompletterar sedan koden med fetmarkerad text:  

```python
#!/usr/bin/env python
#Python
import netsnmp
import ipaddr
import sys
import subprocess
**import MySQLdb**
import prettytable
from prettytable import PrettyTable
 
checkarg = sys.argv[1:]
**#Database
conn = MySQLdb.connect("localhost", "root", "netadm01", "my_network")
cursor = conn.cursor()**
 
#Checks arguments and saves to variables
if len(checkarg) == 2:
    cCommunity = checkarg[0]
    cIP = checkarg[1]
    print cCommunity, cIP
    if isinstance(cCommunity, str):
        try:
            #Checks for valid IPv4-address
            ccIP = ipaddr.IPv4Address(cIP)
         
            try:
                #Modelname
                #Gets the Model-ID, returns numeric
                cModelInt = subprocess.Popen([r"snmpwalk","-v2c","-Oqv","-c",cCommunity,cIP,"sysObjectID"],stdout=subprocess.PIPE).communicate()[0]
                cModelSplit = cModelInt.split("::")
                cModelSplit = cModelSplit[1].rstrip()
 
                #Translate numeric Model-ID to string
                cModel = subprocess.Popen([r"snmptranslate","-m","CISCO-PRODUCTS-MIB","-IR",cModelSplit],stdout=subprocess.PIPE).communicate()[0]
                cModel = cModel.split("::")    
                       
                #Name
                oid = netsnmp.Varbind('sysName.0')
                cName = netsnmp.snmpget(oid, Version = 2, DestHost = cIP, Community = cCommunity)
 
                #Description
                oid = netsnmp.Varbind('sysDescr.0')
                cDesc = netsnmp.snmpget(oid, Version = 2, DestHost = cIP, Community = cCommunity)              
 
                #Output
                print ""
                print "Device info:"
                print ""               
                print "Host:", ccIP
                print "Model:", cModel[1].rstrip()
                print "Name:", cName[0]
                print cDesc[0]
                print ""
 
                **try:
                   
                    cursor.execute("INSERT INTO unit(ip_address,name,model) VALUES(%s,%s,%s)", (ccIP,cName[0],cModel[1].rstrip()))
                    conn.commit()
                    cID = cursor.lastrowid
                    print "Highest ID:",cID
 
                except:
                    cursor.execute("SELECT id FROM unit WHERE ip_address=%s", (ccIP))
                    dbexists = cursor.fetchone()[0]
                    if cursor.rowcount > 0:
                        print ccIP, " Already exists in the database, cleaning up. ID: ", dbexists
                        try:
                            cursor.execute("DELETE FROM unit WHERE unit.id=%s", (dbexists))
                            conn.commit()
                            cursor.execute("DELETE FROM interface WHERE interface.unit_id=%s", (dbexists))
                            conn.commit()
 
                            try:
                                cursor.execute("INSERT INTO unit(ip_address,name,model) VALUES(%s,%s,%s)", (ccIP,cName[0],cModel[1].rstrip()))
                                conn.commit()
                                cID = cursor.lastrowid
                                print "Highest ID:",cID
                            except:
                                print "Unable to insert data"
                        except:
                           
                            print "Delete failed"
                            pass**
                           
 
                x = PrettyTable(["Interface", "IP", "Subnet", "MAC"])
                x.align["Interface"] = "l"
                x.align["IP"] = "l"
                x.align["Subnet"] = "l"
               
                #Index of all the interfaces
                oid = netsnmp.Varbind("ifIndex")
                cIndex = netsnmp.snmpwalk(oid, Version = 2, DestHost = cIP, Community = cCommunity)
                               
                #Index of all the interfaces with IP
                oid = netsnmp.Varbind("ipAdEntIfIndex")
                cIndexIP = netsnmp.snmpwalk(oid, Version = 2, DestHost = cIP, Community = cCommunity)
                               
                #IP-Adress list
                oid = netsnmp.Varbind("ipAdEntAddr")    
                ipAdd = netsnmp.snmpwalk(oid, Version = 2, DestHost = cIP, Community = cCommunity)
               
                #MAC for all interfaces
                oid = netsnmp.Varbind("ifPhysAddress")
                cMAC = netsnmp.snmpwalk(oid, Version = 2, DestHost = cIP, Community = cCommunity)
               
                #Subnetmask for all IP interfaces
                oid = netsnmp.Varbind("ipAdEntNetMask")
                cNetmask = netsnmp.snmpwalk(oid, Version = 2, DestHost = cIP, Community = cCommunity)
               
                looped = 0
                               
                #Prints out interface-details
                for i in cIndex:
               
                    #Name of every interface
                    oid = netsnmp.Varbind("ifDescr."+i)
                    cIntName = netsnmp.snmpget(oid, Version = 2, DestHost = cIP, Community = cCommunity)
                   
                    try:
                        #Matches interface-list with interface+IP-list      
                        IPindex = cIndexIP.index(i)
                       
                        try:
                       
                            #Hex to readable
                            mac = cMAC[looped]
                            mac = ":".join( [ '%x'%(ord(c)) for c in mac ] )
                           
                            #Prints row
                            x.add_row([cIntName[0], ipAdd[IPindex], cNetmask[IPindex], mac])
                           ** cursor.execute("INSERT INTO interface(unit_id,name,ip_address,mask,mac_address) VALUES(%s,%s,%s,%s,%s)", (cID,cIntName[0],ipAdd[IPindex],cNetmask[IPindex],mac))
                            conn.commit()**
 
                        except:
                            pass
 
                        looped += 1
       
                    except:
 
                        try:
                            #Hex to readable
                            mac = cMAC[looped]
                            mac = ":".join( [ '%x'%(ord(c)) for c in mac ] )
                            #Prints row w/o IP
                            x.add_row([cIntName[0], '', '', mac])
                            **cursor.execute("INSERT INTO interface(unit_id,name,mac_address) VALUES(%s,%s,%s)", (cID,cIntName[0],mac))
                            conn.commit()**
                        except:
                                       
                            #Prints row w/o MAC & IP
                            x.add_row([cIntName[0], '', '', ''])
                            **cursor.execute("INSERT INTO interface(unit_id,name) VALUES(%s,%s)", (cID,cIntName[0]))
                            conn.commit()**
                        looped += 1
                print x
               ** #Close database
                cursor.close()
                conn.close()**
 
                try:
                    foutput = x.get_string()
                    f = open(cIP, "w")
                    f.write("Device info: " + cIP + "nCommunity: " + cCommunity + "n")
                    f.write("nHost: " + cIP)
                    f.write("nModel: " + cModel[1].rstrip())
                    f.write("nName:" + cName[0])
                    f.write("n" + cDesc[0])
                    f.write("n" + foutput + "n")
                   # f.writelines(x)
                except:
                    print "Failed to write file"       
                finally:
                    f.close()
            except:
                print "SNMP-polling failed"
                print x
        except:
            print "Invalid IP given:", cIP
    else:
        print "Invalid Community given (must be string)"
else:
    print "Error - you must set both Community and IP (ex ./3a.py public localhost):"
```

Exempel på output:

```
Device info: 192.168.152.9
Community: public

Host: 192.168.152.9
Model: catalyst355024
Name:3550-1
Cisco Internetwork Operating System Software 
IOS (tm) C3550 Software (C3550-I5Q3L2-M), Version 12.1(22)EA1a, RELEASE SOFTWARE (fc1)
Copyright (c) 1986-2004 by cisco Systems, Inc.
Compiled Fri 20-Aug-04 00:44 by yenanh
+--------------------+----------------+-----------------+-----------------+
| Interface          | IP             | Subnet          |       MAC       |
+--------------------+----------------+-----------------+-----------------+
| FastEthernet0/1    | 192.168.152.2  | 255.255.255.252 | 0:a:b7:9c:93:80 |
| FastEthernet0/2    |                |                 | 0:a:b7:9c:93:82 |
| FastEthernet0/3    |                |                 | 0:a:b7:9c:93:83 |
| FastEthernet0/4    |                |                 | 0:a:b7:9c:93:84 |
| FastEthernet0/5    |                |                 | 0:a:b7:9c:93:85 |
| FastEthernet0/6    |                |                 | 0:a:b7:9c:93:86 |
| FastEthernet0/7    |                |                 | 0:a:b7:9c:93:87 |
| FastEthernet0/8    |                |                 | 0:a:b7:9c:93:88 |
| FastEthernet0/9    |                |                 | 0:a:b7:9c:93:89 |
| FastEthernet0/10   |                |                 | 0:a:b7:9c:93:8a |
| FastEthernet0/11   |                |                 | 0:a:b7:9c:93:8b |
| FastEthernet0/12   |                |                 | 0:a:b7:9c:93:8c |
| FastEthernet0/13   |                |                 | 0:a:b7:9c:93:8d |
| FastEthernet0/14   |                |                 | 0:a:b7:9c:93:8e |
| FastEthernet0/15   |                |                 | 0:a:b7:9c:93:8f |
| FastEthernet0/16   |                |                 | 0:a:b7:9c:93:90 |
| FastEthernet0/17   |                |                 | 0:a:b7:9c:93:91 |
| FastEthernet0/18   |                |                 | 0:a:b7:9c:93:92 |
| FastEthernet0/19   |                |                 | 0:a:b7:9c:93:93 |
| FastEthernet0/20   |                |                 | 0:a:b7:9c:93:94 |
| FastEthernet0/21   |                |                 | 0:a:b7:9c:93:95 |
| FastEthernet0/22   |                |                 | 0:a:b7:9c:93:96 |
| FastEthernet0/23   |                |                 | 0:a:b7:9c:93:97 |
| FastEthernet0/24   | 192.168.152.5  | 255.255.255.252 | 0:a:b7:9c:93:80 |
| GigabitEthernet0/1 |                |                 | 0:a:b7:9c:93:99 |
| GigabitEthernet0/2 |                |                 | 0:a:b7:9c:93:9a |
| Null0              |                |                 |                 |
| Vlan1              |                |                 | 0:a:b7:9c:93:80 |
| Vlan2              | 192.168.152.9  | 255.255.255.248 | 0:a:b7:9c:93:80 |
| Vlan10             | 192.168.152.17 | 255.255.255.240 | 0:a:b7:9c:93:80 |
| Vlan12             | 192.168.152.97 | 255.255.255.224 | 0:a:b7:9c:93:80 |
| Vlan15             | 192.168.152.65 | 255.255.255.224 | 0:a:b7:9c:93:80 |
| Vlan20             | 192.168.152.49 | 255.255.255.240 | 0:a:b7:9c:93:80 |
| Vlan25             | 192.168.152.33 | 255.255.255.240 | 0:a:b7:9c:93:80 |
| Vlan153            | 192.168.153.1  | 255.255.255.0   | 0:a:b7:9c:93:80 |
+--------------------+----------------+-----------------+-----------------+
```
Ska försöka hitta tillbaka till Cisco snart men eventuellt kommer det några kortare inlägg om Cacti/Nagios & EEM här framöver först. Fullt upp i jobbsökandet dessutom, den 18-19:e mars ska jag till Oslo för lite intervjuer som verkar riktigt spännande! :)