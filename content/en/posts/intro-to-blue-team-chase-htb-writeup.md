---
author: "Hugo Authors"
title: "Intro to Blue Team - Chase HTB Writeup"
date: 2023-11-11T12:00:06+09:00
description: "Quick Writeup for Hack the Boxes Chase room from the Intro to Blue Team Track"
draft: false
hideToc: false
enableToc: true
enableTocContent: true
author: Jacob Kollasch
authorEmoji: By
pinned: false
tags: 
- HTB
- Wireshark
- Blue Team
---
# Premise
One of our web servers triggered an AV alert, but none of the sysadmins say they were logged onto it. We've taken a network capture before shutting the server down to take a clone of the disk. Can you take a look at the PCAP and see if anything is up?

# First Look at Pcap
Starting at the top, our first packet reveals the IP address of the web server that must have triggered the AV alert [22.22.22.5]. [22.22.22.7], our client, is connecting to [22.22.22.5] on port 80, and initiating a TCP three-way handshake with a SYN message.
![Jacob Kollasch](/intro-to-blue-team-chase-htb-writeup-images/image-1.jpg)
The client and server will use this TCP connection to communicate using HTTP. Which means we should be able to see the contents of these HTTP messages because they are not encrypted using TLS.

I see a client setting up a TCP connection to a web server on port 80. Then they start HTTP messages over this connection. Let's use wiresharks 'Follow HTTP Stream' feature to easily see what's happeng. Okay it looks like the client, likely our attacker is using an upload.aspx function on this web server to upload netcat and setup a revers shell back to the attackers machine using port 4444.
![Jacob Kollasch](/intro-to-blue-team-chase-htb-writeup-images/image-2.png)
Look at the next packet after that HTTP stream. It's a TCP connection from our server to our attackers machine on port 4444. Confirming the revers shell is being used. Let's open follow the TCP stream for this connection. Right away we see the output of the reverse shell. The attacker uses the`whoami` and `ipconfig` which confirms they are connected to our web server when it returns the `iis apppool\defaultapppool` security context, and the ip address or our web sever.
![Jacob Kollasch](/intro-to-blue-team-chase-htb-writeup-images/image-3.png)
Next the attacker changes to the root directory and tries to run a command using powershell to download a file from the attackers machine. They probably setup a web server there and are using the `Invoke-WebReqest` powerwshell module to download a malicious file. But they get an error that the command is not found, and try again using the `certutil` command. However, `certutil` errors as well.
![Jacob Kollasch](intro-to-blue-team-chase-htb-writeup-images/image-4.png)
I won't lie this is where I got stuck and had to look at the walkthrough to find the flag, which is base32 encoded into the text file the attacker tried to download from their web server running on their attacking machine to our web server.
# Response
I think it would be fair to say this was an unsuccessful attack and we can dismiss the alert, but I wouldn't feel good about the fact it was able to happen. This web application should be looked at to makes sure the upload.aspx function cant' be abused to upload malicious files or any files by users if it's unintentional. I am also wondering why the attackers IP address is in the same subnet as our web server. Did they compromise an internal machine and are using it to attack our web server? Or is hack the box trying to make this easy to understand? If it is an internal machine like it seems. Then that hosts needs to be looked at further. If the attackers IP address is supposed to be external, then let's create a firewall rule to block outbound connections to that IP address, so the attacker has to at least change IP addresses to attack again.