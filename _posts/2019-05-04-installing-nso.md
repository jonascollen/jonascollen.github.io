---
title: Installing NSO
date: 2019-05-04 13:19 +0800
comments: true
categories: [NSO]
tags: [Automation, NSO]
toc: false
---
![NSO Logo](/assets/images/2019/05/26142009-1.png)

I've been wanting to test Cisco's NSO for a long time now and finally had some spare time to try and get it up and running on a local VM. It's free for non-production use now so go ahead and grab it over at  
[https://developer.cisco.com/site/nso/](https://developer.cisco.com/site/nso/) (available for both MacOS & Linux).

I installed Ubuntu 18.04 LTS on a fresh vmbox (I didn't manage to get it working on a raspberry), the only other pre-requirements NSO has is:

* Java JDK-7.0 or higher
* Ant
* python2 or python3
* python-paramiko

To make life simpler I also installed python3-pip

```text
$ sudo apt-get install python3-pip  
$ sudo apt-get install default-jdk  
$ sudo apt-get install ant  
$ pip3 install paramiko  
$ java -version  


 openjdk version "11.0.2" 2019-01-15  
 OpenJDK Runtime Environment (build 11.0.2+9-Ubuntu-3ubuntu118.04.3)  
 OpenJDK 64-Bit Server VM (build 11.0.2+9-Ubuntu-3ubuntu118.04.3, mixed mode, sharing)  
```

I then downloaded the file nso-4.7.linux.x86\_64.signed.bin from [Cisco](https://developer.cisco.com/site/nso/), installation was actually much more simpler than I had imagined.

```text
joco02 at labb-nso in ~/Downloads
$ ls
nso-4.7.linux.x86_64.signed.bin

$ sh nso-4.7.linux.x86_64.signed.bin --skip-verification
Unpacking...

joco02 at labb-nso in ~/Downloads
$ ls
cisco_x509_verify_release.py  nso-4.7.linux.x86_64.installer.bin  nso-4.7.linux.x86_64.installer.bin.signature  nso-4.7.linux.x86_64.signed.bin  README.signature  tailf.cer

joco02 at labb-nso in ~/Downloads
$ sh nso-4.7.linux.x86_64.installer.bin $HOME/ncs-4.7 --local-install
INFO  Using temporary directory /tmp/ncs_installer.41189 to stage NCS installation bundle
INFO  Unpacked ncs-4.7 in /home/joco02/ncs-4.7
INFO  Found and unpacked corresponding DOCUMENTATION_PACKAGE
INFO  Found and unpacked corresponding EXAMPLE_PACKAGE
INFO  Generating default SSH hostkey (this may take some time)
INFO  SSH hostkey generated
INFO  Environment set-up generated in /home/joco02/ncs-4.7/ncsrc
INFO  NCS installation script finished
INFO  Found and unpacked corresponding NETSIM_PACKAGE
INFO  NCS installation complete
```

When then have to source our new folder to enable built-in variables and then setup our environment.

```text
joco02 at labb-nso in ~/Downloads
$ source $HOME/ncs-4.7/ncsrc
$ ncs-setup --dest $HOME/ncs-run
```

That should be everything, we can now start NSO with `ncs`.

To check status we can use:

```text
    joco02 at labb-nso in ~/ncs-run
    $ ncs --status | grep status
    status: started
    
    joco02 at labb-nso in ~/ncs-run
    $ ncs --version
    4.7
```

We should now also be able to reach NSO's GUI/Webpage for administration at [http://serverip:8080/login.html](http://serverip:8080/login.html)

![Login page](/assets/images/2019/05/nso-login-1.png)

Login credentials are admin / admin

If your like me and allergic to GUIs we can also connect to the CLI instead.

```text
joco02 at labb-nso in ~/ncs-4.7
$ ncs_cli -u admin

admin connected from 192.168.15.188 using ssh on labb-nso
admin@ncs> ?
Possible completions:
  clear      - Clear parameter
  compare    - Compare running configuration to another configuration or a file
  configure  - Manipulate software configuration information
  describe   - Display transparent command information
  exit       - Exit the management session
  file       - Perform file operations
  help       - Provide help information
  id         - Show user id information
  monitor    - Real-time debugging
  ping       - Ping a host
  ping6      - Ping an ipv6 host
  quit       - Exit the management session
  request    - Make system-level requests
  script     - Script actions
  set        - Set CLI properties
  set-path   - Set relative show path
  show       - Show information about the system
  source     - File to source
  switch     - Change CLI style
  templatize - Find patterns in subtree.
  top        - Exit to top level and optionally run command
  traceroute - Trace the route to a remote host
  up         - Exit one level of configuration
```

We should now be able to do some labs on NSO! :) Cisco luckily provides a few "**[Network Elements Driver](https://www.cisco.com/c/en/us/products/collateral/cloud-systems-management/network-services-orchestrator/datasheet-c78-734669.html)**" for their netsim-devices for lab purposes, if you want to manage your own devices (real/virtual) you will have to buy a license for it as I understand it.

```text
joco02 at labb-nso in ~/ncs-4.7/packages/neds
$ ls
  a10-acos  cisco-ios  cisco-iosxr  cisco-nx  dell-ftos  juniper-junos
```
