---
title: Netmiko part 2
date: 2019-04-15 21:46
author: Jonas Collén
comments: true
categories: [Netmiko, Python, Scripting]
tags: [netmiko, python]
---
A follow-up on the latest post about [Netmiko](https://github.com/ktbyers/netmiko) where we logged in to our devices and did some show commands, let's continue and do a very common thing in production networks with many devices - pushing out new config from a template.

I'll be using our latest login\_simple.py from last post as a base and just modify it slightly. First we create our config-file that we want in all our devices., let's say we want to rollout a new route-map in our network.

**config\_template.txt**

    ip prefix-list DEFAULT-ROUTE seq 5 permit 0.0.0.0/0
    !
    route-map DEFAULT-INJECT permit 10
     match ip address prefix-list DEFAULT-ROUTE
     set extcommunity rt 300:301
    !
    route-map DEFAULT-INJECT permit 20
     set extcommunity rt 300:300

In netmiko we can then use "**send\_config\_from\_file**" when we've logged in to our device(s), super simple! Notice that we don't have to manage the context in our router (enable/conf t etc), netmiko does it for us.

![](/assets/images/2019/04/netmiko.png)

**conf\_from\_file.py**

    from netmiko import ConnectHandler
    from datetime import datetime
    from getpass import getpass
    
    DEVICES = ["192.168.15.10", "192.168.15.11", "192.168.15.12"]
    USERNAME = "joco02"
    PASSWORD = getpass()
    FILE = 'config_template.txt'
    
    def config_commands(devices, file):
        for device in devices:
            start_time = datetime.now()
            connection = ConnectHandler(
                device_type="cisco_ios", host=device,
                username=USERNAME, password=PASSWORD)
            hostname = connection.find_prompt()
            device_result = ["{0} {1} {0}".format("=" * 20, hostname)]
    
            command_result = connection.send_config_from_file(FILE)
            device_result.append(command_result)
    
            device_result_string = "\n\n".join(device_result)
            connection.disconnect()
            device_result_string += "\nElapsed time: " + str(datetime.now() - start_time)
            yield device_result_string
    
    def main():
    
        results = config_commands(DEVICES, FILE)
        for result in results:
            print(result)
    
    if __name__ == "__main__":
        main()

**Results:**

    $ python conf_from_file.py
    Password:
    ==================== R1# ====================
    
    config term
    Enter configuration commands, one per line.  End with CNTL/Z.
    R1(config)#ip prefix-list DEFAULT-ROUTE seq 5 permit 0.0.0.0/0
    R1(config)#!
    R1(config)#route-map DEFAULT-INJECT permit 10
    R1(config-route-map)# match ip address prefix-list DEFAULT-ROUTE
    R1(config-route-map)# set extcommunity rt 300:301
    R1(config-route-map)#!
    R1(config-route-map)#route-map DEFAULT-INJECT permit 20
    R1(config-route-map)# set extcommunity rt 300:300
    R1(config-route-map)#end
    R1#
    Elapsed time: 0:00:08.875642
    ==================== R2# ====================
    
    config term
    Enter configuration commands, one per line.  End with CNTL/Z.
    R2(config)#ip prefix-list DEFAULT-ROUTE seq 5 permit 0.0.0.0/0
    R2(config)#!
    R2(config)#route-map DEFAULT-INJECT permit 10
    R2(config-route-map)# match ip address prefix-list DEFAULT-ROUTE
    R2(config-route-map)# set extcommunity rt 300:301
    R2(config-route-map)#!
    R2(config-route-map)#route-map DEFAULT-INJECT permit 20
    R2(config-route-map)# set extcommunity rt 300:300
    R2(config-route-map)#end
    R2#
    Elapsed time: 0:00:08.799664
    ==================== R3# ====================
    
    config term
    Enter configuration commands, one per line.  End with CNTL/Z.
    R3(config)#ip prefix-list DEFAULT-ROUTE seq 5 permit 0.0.0.0/0
    R3(config)#!
    R3(config)#route-map DEFAULT-INJECT permit 10
    R3(config-route-map)# match ip address prefix-list DEFAULT-ROUTE
    R3(config-route-map)# set extcommunity rt 300:301
    R3(config-route-map)#!
    R3(config-route-map)#route-map DEFAULT-INJECT permit 20
    R3(config-route-map)# set extcommunity rt 300:300
    R3(config-route-map)#end
    R3#
    Elapsed time: 0:00:09.067742
    

We can now easily just change the config in our template and run the script again when it's time for another rollout.