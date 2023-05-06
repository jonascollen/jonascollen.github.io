---
title: Deployment-tools
date: 2018-06-24 19:54
author: Jonas Collén
comments: true
categories: [Flask, Automation, Project]
tags: [deployment-tools]
---
I've been busy these last few days working on migrating the website I created over at deployment-tools.se to Telias data center as it soon will be deployed for official use in the company, fun stuff! :) It's a web portal that generates configuration (initial, full config & verification) for deployment of aggregation nodes, new links, services etc based on Python & Flask.

I've written a few posts showing how to do things like automate config for DHCP, firewalls & IOS etc previously but this also provides a front end for the users instead of CLI-scripts. My weakest point is for sure design & CSS so i'm having a pretty rough time renewing the design (& HTML5) making it more dynamic but it's slowly coming together now. I'll host a demo-page the next few days when i'm closer to finishing it. 
![](/assets/images/2018/06/deploymenttools.jpg) 

Moving from my own Raspberry to a very secured and locked down release of RHEL7 was a challenge as well.. I've been trying to keep up with the reading in the mornings at least and i'm soon done with Internet Routing Architectures, it's an old book but a very good read, highly recommended as a good intro to BGP!