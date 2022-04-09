---
layout: post
title: ARP and Thunderbolt-Ethernet issues with OS X Mavericks
date: 2014-08-17 10:45:20 UTC
updated: 2014-08-17 10:45:20 UTC
comments: true
categories: arp osx mavericks thunderbolt ethernet homepage
---

Recently I got a new MacBook Pro(late 2013 model) 13.3 Retina display and everything was working like charm while I was using wifi to connect to internet and that pretty much should be because you expect everything perfect after spending that much amount on a machine.
After moving to my university where wifi connectivity is not present at every place I switched to Thunderbolt to Gigabit Ethernet to connect to internet and this was the moment when things seemed to be changing and I started getting troubles connecting to internet.

Problem
-------
At the first time when I connected to internet it took about 15 min for internet to get stabilized and was continuously connecting and dissconnecting as if someone was continuously playing with the Ethernet cable, I ignored it and continued with my work assuming some problem with the VLAN used in university hostel and after some time the internet got dissconnected again automatically while working and came back after around 2 min.

I tried to ping google from terminal and observed quite significant packet loss like this one:

{% highlight bash %}
64 bytes from 74.125.236.128: icmp_seq=0 ttl=53 time=56.825 ms
64 bytes from 74.125.236.128: icmp_seq=1 ttl=53 time=56.839 ms
64 bytes from 74.125.236.128: icmp_seq=2 ttl=53 time=56.583 ms
64 bytes from 74.125.236.128: icmp_seq=3 ttl=53 time=56.770 ms
64 bytes from 74.125.236.128: icmp_seq=4 ttl=53 time=56.663 ms
64 bytes from 74.125.236.128: icmp_seq=5 ttl=53 time=56.894 ms
64 bytes from 74.125.236.128: icmp_seq=6 ttl=53 time=56.622 ms
64 bytes from 74.125.236.128: icmp_seq=7 ttl=53 time=58.007 ms
Request timeout for icmp_seq 8
Request timeout for icmp_seq 9
Request timeout for icmp_seq 10
Request timeout for icmp_seq 11
Request timeout for icmp_seq 12
64 bytes from 74.125.236.128: icmp_seq=13 ttl=53 time=57.267 ms
64 bytes from 74.125.236.128: icmp_seq=14 ttl=53 time=57.267 ms
64 bytes from 74.125.236.128: icmp_seq=15 ttl=53 time=57.267 ms
64 bytes from 74.125.236.128: icmp_seq=15 ttl=53 time=57.267 ms
64 bytes from 74.125.236.128: icmp_seq=16 ttl=53 time=56.713 ms
64 bytes from 74.125.236.128: icmp_seq=17 ttl=53 time=57.376 ms
64 bytes from 74.125.236.128: icmp_seq=18 ttl=53 time=57.811 ms
64 bytes from 74.125.236.128: icmp_seq=19 ttl=53 time=57.001 ms
64 bytes from 74.125.236.128: icmp_seq=20 ttl=53 time=56.869 ms
{% endhighlight %}

This was the first problem - Packet loss for ping requests. 

It could still be tolerated but the most annoying part was I was not able to connect to internet after a reboot or sleep and had to wait for around 15-30 min everytime for the connection to get stabilized.
I checked the netork preferences at the this time by going to 

System Preference > Network and observed that machine could not even fetch the Ip address from DHCP server and the Thuderbot to ethernet connection was continuously flickering Green and Red light.

I googled about the problem to find if its just my machine or other users are facing this problem and found some very useful sources of discussion:

1. [Discussion on Apple Support][apple-support]
2. [Reddit Discussion][reddit-discussion]
3. [MacStadum Blog providing updates on issue][macstadium-blog]
[apple-support]: https://discussions.apple.com/thread/5483424?start=0&tstart=0
[reddit-discussion]: https://www.reddit.com/r/sysadmin/comments/1yc6n1/packet_losses_with_new_os_x_mavericks_make_sure
[macstadium-blog]: https://www.macstadium.com/blog/osx-10-9-mavericks-bugs/

Which clearly indicate this issue with ARP caching by OS X Mavericks and problem with only some of the machines and not all.
So I was curious to check my arp output at this point of time and here is what it looked like:
{% highlight bash %}
Pankajs-MacBook-Pro:~ bitgeeky$ arp -a
Pankajs-MacBook-Pro:~ bitgeeky$ arp -a
? (10.1.34.1) at 0:7:e:64:f9:3f on en4 ifscope [ethernet]
? (10.1.34.255) at ff:ff:ff:ff:ff:ff on en4 ifscope [ethernet]
Pankajs-MacBook-Pro:~ bitgeeky$ arp -a
? (10.1.34.1) at 0:7:e:64:f9:3f on en4 ifscope [ethernet]
? (10.1.34.255) at ff:ff:ff:ff:ff:ff on en4 ifscope [ethernet]
Pankajs-MacBook-Pro:~ bitgeeky$ arp -a
? (10.1.34.1) at 0:7:e:64:f9:3f on en4 ifscope [ethernet]
? (10.1.34.99) at 28:92:4a:43:c2:36 on en4 ifscope [ethernet]
? (10.1.34.157) at b8:88:e3:e2:aa:c7 on en4 ifscope [ethernet]
? (10.1.34.255) at ff:ff:ff:ff:ff:ff on en4 ifscope [ethernet]
Pankajs-MacBook-Pro:~ bitgeeky$ arp -a
Pankajs-MacBook-Pro:~ bitgeeky$ arp -a
Pankajs-MacBook-Pro:~ bitgeeky$ arp -a
Pankajs-MacBook-Pro:~ bitgeeky$ arp -a
? (10.1.34.1) at 0:7:e:64:f9:3f on en4 ifscope [ethernet]
? (10.1.34.255) at ff:ff:ff:ff:ff:ff on en4 ifscope [ethernet]
Pankajs-MacBook-Pro:~ bitgeeky$ arp -a
? (10.1.34.1) at 0:7:e:64:f9:3f on en4 ifscope [ethernet]
? (10.1.34.70) at b4:b5:2f:2f:c4:64 on en4 ifscope [ethernet]
? (10.1.34.255) at ff:ff:ff:ff:ff:ff on en4 ifscope [ethernet]
Pankajs-MacBook-Pro:~ bitgeeky$ arp -a
Pankajs-MacBook-Pro:~ bitgeeky$ arp -a
{% endhighlight %}

This is at the moment when problem is lot worse.

I tried a bunch of proposed solution proposed on various internet sites but nothing worked for me and then I came across a script which explains the problem and gives a solution that worked for many other users [Disable Unicast ARP Cache Validation][script]

[script]: https://github.com/MacMiniVault/Mac-Scripts/blob/master/unicastarp/unicastarp-README.md

As mentioned it disables Unicast ARP cache validation. But... unfortunately this also did not work for me :-(

Solution
--------
The solution which is quite promising and worked for me is to:

1. Bind MAC address of networking interface(Thunderbolt to Gigabit ethernet in this case) to IP Address in DHCP configuration so that problem with ARP requests reduces quite significantly.
2. After this it is also useful to remove all network interfaces from network preferences by clicking the "-" button shown in image and reboot. It is because I observed Thunderbolt-Bridge interfaring with the Thunderbolt-Ethernet and making it Fail in Assist-Me menu for network configuration.

<img src="/public/images/network.png" alt="Network Preferences OS X Mavericks" style="float:left; margin-right:15px;" />

This solution works fine for me and I am able to connect to internet using Thunderbolt to Gigabit Ethernet but at times I still see some packet loss which is almost insignificant.

So there is no permanent solution as of now but I hope Apple to investigate more on this and fix this in their next release. Some people are really pushing them to fix the problem and I think most of updates about this issue can be found on [MacStadium Blog][macstadium-blog] who are using Mac Mini's in their data centeres, having the same problem with many of their machines and are in regular contact with the Apple Developers.

I thought this piece of interrogation and ispection worth sharing with everyone so that you don't waste time trying random hacks described on various internet websites if you have the same issue.
