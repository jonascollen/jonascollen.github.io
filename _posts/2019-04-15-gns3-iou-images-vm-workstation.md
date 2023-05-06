---
title: GNS3, IOU-images & VM Workstation
date: 2019-04-15 17:40
author: Jonas Collén
comments: true
categories: [GNS3]
tags: [vix,vmware]
---
I've been using GNS3 when playing around with smaller topologies instead of booting up my R610 which is more or less like working next to a jet plane, but had a few issues getting it up and running on a new Win10-installation so figured I should document my steps for future reference.

Start with downloading:

*   [GNS3](https://www.gns3.com/software)
*   [GNS3 VM](https://www.gns3.com/software/download-vm)
*   [VMware Workstation](https://www.vmware.com/se/products/workstation-player.html)
*   [VMware VIX API](https://my.vmware.com/web/vmware/login?bmctx=4C976C546DE4E8BA7BD58B8EEADF25A5B418821E70E4480C483939EC36F11A86&contextType=external&username=string&OverrideRetryLimit=1&action=%2F&password=sercure_string&challenge_url=https%3A%2F%2Fmy.vmware.com%2Fweb%2Fvmware%2Flogin&creds=username+password&request_id=-6717584917619567700&authn_try_count=0&locale=en_US&resource_url=https%253A%252F%252Fmy.vmware.com%252Fgroup%252Fvmware%252Fdetails%253FdownloadGroup%253DPLAYER-1400-VIX1170%2526productId%253D734)
*   [GNS3 Marketplace - IOU L2](https://www.gns3.com/marketplace/appliance/iou-l2)
*   [GNS3 Marketplace - IOU L3](https://www.gns3.com/marketplace/appliance/cisco-iou-l3)

You will also need the following bin-files:

*   i86bi-linux-l2-adventerprisek9-15.2d.bin
*   i86bi-linux-l3-adventerprisek9-15.4.1T.bin

Install everything as normal, follow the official [GNS3-guide](https://docs.gns3.com/1wdfvS-OlFfOf7HWZoSXMbG58C4pMSy7vKJFiKKVResc/index.html) if there's any questions. I started with VMware & VIX, imported the GNS3 VM, adjusted memory to at least 4GB RAM and added an extra bridge adapter so we can reach our topology from other devices on our network.

![](/assets/images/2019/04/gns3vm.png)

After installing GNS3 I had some strange issues that the program was unable to connect to the VM for some reason. The error-code I got was:

_Error while saving settings: GNS3VM: Error while executing VMware command: vmrun has returned an error: Unable to connect to host._  
  
_Error: The specified version was not found_

I found some old post recommending to downgrade the version of VMware station to 12.xx but a better fix I found was instead to edit the following file (depending on where you installed it):  
  
_C:\\Program Files (x86)\\VMware\\VMware VIX\\vixwrapper-config.txt_

Edit the following line:

    # latest un-versioned
    ws        19  vmdb  e.x.p Workstation-14.0.0
    player    19  vmdb  e.x.p Workstation-14.0.0
    
    To:
    # latest un-versioned
    ws        19  vmdb  e.x.p Workstation-14.0.0
    player    19  vmdb  15.0.4 Workstation-14.0.0
    

Restart WMware & GNS3 and you should be good to go. Import the IOU-appliances as normal, if you put the required .bin-files in the same folder it should find them automatically. Last step is to enter your license or GNS3 will refuse to start them, if you're unsure how to get hold of this contact Cisco or do some googling.