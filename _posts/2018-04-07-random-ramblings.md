---
title: Current progress & random ramblings
date: 2018-04-07 08:10
author: Jonas Collén
comments: true
categories: [General]
tags: [ccie recommended reading, doccd, vbs]
---
Thinking about switching over to English instead for this & future posts. When I started this blog in the middle of 2013 I couldn't find any Cisco/CCIE-focused blog that was written in Swedish, so my thinking was that there was a "gap to be filled". Over the years however I actually gotten more and more traffic outside of Sweden, and just checking this last month hits from within Sweden was only about 20% of total traffic.. 

So instead of just being a "dead page" for most people randomly googling something, and also the fact that I don't have to keep using horrible 'swenglish' in my posts which contains 99% technical words anyway - it's starting to feel like it's time for a change. In the middle of february this year I decided to set a real goal of at least being done with CCIE Written and be at least around ~60% ready for the lab before the year is over. 

Over the years I also realized this will never happen without having a set study schedule and setting aside several hours every week for both reading and lab work after. Since then i've been waking up at 04.45 on weekdays and studying 2-2½h before work, on weekends I sleep a little longer but dedicate at least ~4-5h each morning. This has been working great! In earlier attempts I always did the studying after work, and as months passed I just got too tired to retain info/kept falling asleep reading or other things got in the way. After work hours is more focused on doing labs now, it finally feels like I found a way that works for me in the long run. A small update regarding what i'm currently reading/have read this last 1½ months & what I have planned going forward (did a full restart):

*   Fundamentals:
    *   Interconnections - Bridges, Routers & Switches ✔
    *   TCP/IP Illustrated - Vol. 1 ✔
*   Switching:
    *   Cisco LAN Switching ✔
    *   Official Cert Guide CCNP Routing & Switching - Switch 300-115 (~50%, paused until I get some switching gear to lab on)
*   Routing:
    *   Routing TCP/IP - Volume 1 _(Currently reading)_
    *   IP Routing on IOS, IOS XE, IOS XR
    *   Internet Routing Architechtures
    *   Troubleshooting IP Routing Protocols
    *   Routing TCP/IP - Volume 2
*   IPv6
    *   IPv6 Theory, Protocol & Practice
*   MPLS
    *   MPLS Fundamentals
    *   MPLS Enabled Applications
*   Multicast
    *   CCIE Developing IP Multicast
*   QoS
    *   End to End QoS Network Design
*   Final prep for Written
    *   CCIE Routing & Switching - Official Certification Guide

A big problem i've always had is retaining information when moving on to another subject, ex going from reading details about of TCP/IP to Switching to Routing etc. Making my own flashcards has been a life saviour. I try to take at least a few minutes and go thru one set of cards on every break at work etc. A great (and free) site i've been using is cram.com that also have a good mobile app. I'm also trying to set a good foundation when it comes to lab work. From reading other CCIE-blogs like [lostintransit.se,](http://www.lostintransit.se)  [ipspace.net](https://blog.ipspace.net/), [rogerperkin](http://www.rogerperkin.co.uk/blog/) that passed the exam, being able to write out templates & general config straight in notepad is almost a requirement to reach the speed required to finishing the actual lab exam later. So I try to stay away from the CLI until I feel like i'm done with all the steps in each lab. I've been using the following format in notepad after reading [this](http://lostintransit.se/2016/03/11/ccie-ccie-spv4-review-by-nick-russo/) great post by Nick Russo regarding passing the CCIE SP4-lab, using one document for everything will make it easy to ctrl+f and find a certain config section if you have to go back etc:
```
! 3-1 R4 Object Tracking

ip sla 1
 icmp-echo 150.1.146.1 source-ip 150.1.146.4
 frequency 5
 threshold 2000
 timeout 2000
ip sla schedule 1 life forever start-time now
track 1 ip sla 1
ip route 150.1.1.1 255.255.255.255 Gi1.146 155.1.146.1 track 1

! 3-2 R4 Floating route

ip route 150.1.1.1 255.255.255.255 Tu0 150.1.0.1 2
```

Another important thing (that i'm still terrible at using myself currently), when you're unsure of a specific command or how to solve it, try to stay away from google and use Cisco's official documentation instead. There's a huge benefit in being confident knowing where to find specific material when you're at the actual exam, remember - there is no search-field (except for ctrl+f on your current page). Unfortunately most videos going over how to use the doccd efficiently is not updated and it actually looks like Cisco recently did another revamp of the entire section (so INEs v5.1 doccd-video is also ood). The good news however is that it seems to be much faster now! :) Here's some good links i've found (which hopefully is reachable during the exam?):

*   [Master command list](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/mcl/allreleasemcl/all-book.html) (_Support > Product Support > Cisco IOS and NX-OS Software > Cisco IOS 15.4M&T > Master Index_[)](https://www.cisco.com/c/en/us/support/ios-nx-os-software/ios-15-4m-t/products-product-indices-list.html)
*   [Command references & Configuration Guides](https://www.cisco.com/c/en/us/support/ios-nx-os-software/ios-software-release-15-4-3-m/model.html) (_Support > Product Support > Cisco IOS and NX-OS Software > Cisco IOS 15.4M&T )_

Not really sure about which IOS they are actually running nowadays, but the difference between any 15.x version should be very small. Speaking of lab-prep i've actually gone all out and ordered myself an US-layout keyboard as well just to get used to the layout and type faster... :) A few more tips regarding lab work, to speed things up going between different labs, on each router I created a "blank.cfg" config with only the bare minimums, ex:

```
hostname R1
no ip domain lookup
alias exec con conf t
alias exec sib show ip int brief
alias exec srb show run | b
alias exec sri show run int
line con 0
 exec-timeout 0 0
 logging synchronous

copy run flash:blank.cfg
```

I then made a small VBS-script that logs in to each router (going by open tabs in SecureCRT) and replacing current config with the blank.cfg, no more reloading!
```
\# $language = "VBScript"
# $interface = "1.0"
dim nIndex, objCurrentTab

For nIndex = 1 to crt.GetTabCount
 Set objCurrentTab = crt.GetTab(nIndex)
 objCurrentTab.Activate
 if objCurrentTab.Session.Connected = True then
 crt.Sleep 500
 objCurrentTab.Screen.Send "end" & vbcr
 crt.Sleep 500
 objCurrentTab.Screen.Send "config replace flash:blank.cfg" & vbcr
 crt.Sleep 500
 objCurrentTab.Screen.Send "y" & vbcr
 crt.Sleep 500
 end if
 Next
```
The only caveat is that subinterfaces you've created earlier will still be visible as "deleted", but I haven't found any other downside.

```
CSR-R10# sh int desc
Interface Status Protocol Description
Gi1 up up 
Gi1.10 deleted down 
Gi1.108 deleted down 
Gi2 admin down down 
Gi3 admin down down
```
For loading initial configs from ex. INE's workbook I found a great little script, you only have to select which folder you want to load from and it does the rest. Unfortunately I can't manage to find the actual author again to give credit.. I'll update the page when I find it. The only gotcha is that each tab in SecureCRT will have to have the same name as the config files, ex R1 for R1.txt etc.  Script looks like this:

```
\# $language = "VBScript"
# $interface = "1.0"

Option Explicit

Dim strPath

strPath = SelectFolder( "" )
If strPath = vbNull Then
 msgbox "Cancelled"
Else
' msgbox "Selected Folder: """ & strPath & """"
End If

dim objFSO, objStartFolder, objFolder, colFiles, objFile, Xcount, nIndex, objCurrentTab 
Const ForReading = 1
Const ForWriting = 2

For nIndex = 1 to crt.GetTabCount
 Set objCurrentTab = crt.GetTab(nIndex)
 objCurrentTab.Activate
 if objCurrentTab.Session.Connected = True then
 crt.Sleep 500
 objCurrentTab.Screen.Send "!" & vbcr
 crt.Sleep 500
 objCurrentTab.Screen.Send "enable" & vbcr
 crt.Sleep 500
 objCurrentTab.Screen.Send "config t" & vbcr
 crt.Sleep 500
 objCurrentTab.Screen.Send "! " & objCurrentTab.Caption & vbcr 
 crt.Sleep 1000

Dim fso, file, str, filename, char1, char2
 Set fso = CreateObject("Scripting.FileSystemObject")
 filename = strPath & "\\" & objCurrentTab.Caption & ".txt"

' Test if File is Unicode or ASCII

Set file = fso.OpenTextFile(filename)
char1 =file.read(1)
char2 =file.read(1)
file.close

if asc(char1) = 255 and asc(char2) = 254 then
 set file = fso.opentextfile(filename,,,true)
else
 set file = fso.opentextfile(filename)
end if

objCurrentTab.Screen.Send "Opening file: " & filename

'objCurrentTab.Screen.Synchronous = True

Do While file.AtEndOfStream <> True

str = file.Readline
 objCurrentTab.Screen.Send str & Chr(13)
 crt.Sleep 10
 
 Loop

'objCurrentTab.Screen.Synchronous = False

end if
 file.close
 set fso = Nothing
 Next

Function SelectFolder( myStartFolder )

Dim objFolder, objItem, objShell

On Error Resume Next
 SelectFolder = vbNull

Set objShell = CreateObject( "Shell.Application" )
 Set objFolder = objShell.BrowseForFolder( 0, "Select Folder", 0, myStartFolder )

If IsObject( objfolder ) Then SelectFolder = objFolder.Self.Path

Set objFolder = Nothing
 Set objshell = Nothing
 On Error Goto 0
End Function
```
> "Motivation is crap, be driven"

I recommend everyone to watch [this podcast](https://www.youtube.com/watch?v=5tSTk1083VY) with David Goggins & Joe Rogan whenever you're starting to lose momentum, goosebumps! :)
