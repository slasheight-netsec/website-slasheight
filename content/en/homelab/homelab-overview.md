---
author: "Jacob Kollasch"
title: "Homelab Overview"
date: 2024-11-04T12:00:08+00:00
description: "The why and how behind my cybersecurity homelab. A simulated corporate Windows and Linux Domain environment with a Wazuh SIEM for attacking, detecting, and defending."
draft: false
hideToc: false
enableToc: true
enableTocContent: true
author: Jacob Kollasch
authorEmoji: Written by
pinned: true
tags: 
- homelab
- windows
- linux
- attack
- defend
- detect
- SOC
---
![Jacob Kollasch](/homelab/homelab-10-04-24.png)

## Why build a homelab?
The goal of my homelab is to create a learning environment where I can put into practice attacking and defending a corporate windows domain environment running a SIEM. This will highlight my efforts to learn the skills necessary to become a SOC analyst. Projects completed in my homelab will be showcased here on my personal website.

Specifically, my cybersecurity homelab will enable me to practice and learn the following skills.
- SOC Engineering and Analysis - Administrating Wazuh SIEM, creating customer alerts, and writing rules to detect my own attacks.
- Offensive Security - Executing real threat actor techniques to attack the network.
- Infrastructure Security - Attempting to harden lab infrastructure against my own attacks.
- Windows/Linux Administration - Configuring lab setups will allow me to practice common server administration tasks on a regular basis. I.e. creating users, managing permissions, process management, package management, networking, etc) 

Most 'projects' completed in my homelab will consist of an attack or threat actor technique that I will go through from three perspectives. Attack, Detect, and Defend. Inside the limitations of my homelab and depending on the actual attack, I will go into each perspective and execute them in my lab. 

For example, kerberoasting. For the "attack" portion, I will actually get the kerberoasting attack working in my lab. First, documenting whatever configurations are needed to make the lab environment susceptible to a successful kerberoasting attack. Then, I will run the attack against my lab infrastructure proving it works. 

Afterwards, in the "detect" portion, I will show how to configure windows logging and Wazuh to detect kerberoasting, and generate a meaningful alert. I will actually re-run the attack with detentions in place. 

Finally, in the 'defend' section, I will show how to configure my homelab to prevent kerberoasting attacks as much as possible. 

This will leave me with experience about the given attack type from a red team, blue team, and infrastructure security perspective. Hopefully, achieving a well rounded understanding of the attack.

Beyond specific projects, like going through common attacks, this homelab will allow me to test out and play with security tools new and old in a simulated corporate domain environment with minimal setup and fear of breaking things. If I see an interesting TA technique on a DFIR report I can try it for myself quickly, maybe I want to get into the nitty gritty of configuring fail2ban on a linux server, or maybe I want play with an open-source EDR like Lima Charlie.

There are a lot of really good pre-setup labs out there that I have used a lot in the past, and will continue to use in addition to my homelab going forward, like hackthebox.com and letsdefend.io. However, there is a lot of really useful knowledge in setting up those labs. The infrastructure configuration needed to get some lab scenarios working can teach you a lot about the in's and out's of active directory, and working in a domain environment. Another thing I like about a personal homelab is you can go way deeper, and control every aspect of the lab when it's running in your own lab. No configuration setting or log file will be hidden from your view for better or worse.

This homelab is **not** for malware analysis and should not be copied if you plan on doing so. A malware analysis lab would need to take more precautions against malware spreading to the rest of the lab or even my home network. Tools like any.run and tria.ge in my opinion are much better tools for malware analysis unless you are very very confident in your own lab environment.

## Infrastructure
Instead of running separate physical servers and host computers I am going to virtualize each host on a proxmox hypervsor that is running directly on an old Dell Optiplex 9020 tower I bought on ebay used. The only changes I made to the dell were replacing spinning rust with an ssd and filled the ram slots up with cheap ddr3. On paper, my server is low on resources, but in practice I'm able to run all my cybersecuity VMs at once. I usually power them down when not in use to avoid the fans spinning up high every now and then. It has made no noticalbe change to my powerbill and it's always on as I run a pihole and plex server for my home network on linux containers on the same hypervisor. 

Homelabs DO NOT HAVE TO BE EXPENSIVE. Would I like to buy a fully kitted out Intel nuc and run more VM's with more resources at once on a fanless PC? Yes, of course I want several, and to cluster them just for sake of seeing big numbers on my proxmox dashboard. However, now while money is tight I can do everything I need with this old dell tower. This optiplex has been running my previous homelabs since 2020 when I was using esxi, and eve-ng to lab for cisco certs in addition to plex and pi-hole.

The lab will consist of the following hosts.

| Hostname             | OS                  | Description                                                                                                      | Static IP Address | Domain Joined | Snapshot State                                                                |
| -------------------- | ------------------- | ---------------------------------------------------------------------------------------------------------------- | ----------------- | ------------- | ----------------------------------------------------------------------------- |
| LAB-DC-01            | Windows Server 2022 | Windows Domain Controller for TYRELL domain.                                                                     | 10.10.60.32       | Y             | - Joined to Domain. <br>- wazuh agent installed<br>                           |
| LAB-SVR-SIEM-01      | Debian 11           | Debian Server running Wazauh SIEM. GUI is installed.                                                             |                   | Y             | - Wazuh SIEM installed<br>-                                                   |
| LAB-CLIENT-WIN10-01  | Windows 10          | Windows 10/11 client. Whenever a windows host or account takeover is apart of attack this host will be targeted. |                   | Y             | - Joined to Domain.<br>- Static IP reservation set<br>- wazuh agent installed |
| LAB-CLIENT-HACK-01   | Debian 11           | Debian client used for hacking in lab environment.                                                               |                   | N             |                                                                               |
| LAB-CLIENT-WINSOC-01 | Windows 10          | Windows 10/11 client used for windows analysis and blue team activity.                                           |                   | Y             | - wazuh agent installed                                                       |

## Networking
I am going to segment this network from my home network by creating a new VLAN for the lab on my home router (Meraki MX67W), and I can configure network access control between the lab and my home network on the router as needed. 

I could get crazy with networking for the lab by creating a ton of specific VLAN's, getting super granular with firewall rules between the VLAN's, create jump box and VPN just to access my lab, or put the whole lab behind a PFSENSE firewall that I meticulously pour over the configuration. However, unless I want to focus on those specific activities/skills they seem like a huge waste of time, and are going to get in the way labbing and learning other cybersecurity skills. I'm trying to avoid things that could take a lot of time to administrator, and may not be needed for the lab activity at hand.

As I said earlier, this is specifically **not** a malware analysis lab, so I don't have to worry about my "malicious" activities resulting with something  a crypto miner or info stealer accidentally installed on my personal devices.

## Resetting Lab
I need a quick and easy way to reset my lab to a base state, so I don't have to manually reverse configurations and worry about breaking things too much. Resetting my lab back to a base state is something I've always overlooked when doing homelabs in the past. I can tell you from personal experience that nothing makes you want to give up on studying all together than messing up your lab beyond repair, and realizing you have to build the whole thing up from scratch.

My process to reset my lab will consist of defining a base state configuration that I want each essential VM to have when starting a new lab exercise. Next, I will take snapshots in prox-mox of each VM in their base state. Before, starting a lab exercise I will restore each VM to their base state snapshot.

There are certainly fancier and likely better ways to reset my lab, but another issue I've ran into with previous homelab's is over complicating things ultimately distracting too much from the act of home labbing. So I'm keeping it simple with snapshots. I'm comfortable with the risk of not being able to restore a snapshot and having to re-build, but I don't want to have to re-build every time.

The base state of my for each VM is briefly defined the infrastructure table from earlier. The goal for each VM's base state snapshot is to have a corporate network stood up and ready to attack or defend.