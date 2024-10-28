---
author: "Jacob Kollasch"
title: "Arbitrary File Read on Checkpoint Security Gateway (CVE-2024-24919)"
date: 2024-10-05T12:00:08+00:00
description: "LetsDefend alert SOC287 writeup, and technical breakdown of CVE-2024-24919."
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
Going through this High Severity Web Attack alert on a Saturday Afternoon. "*Arbitrary File Read on Checkpoint Security Gateway*" sounds like someone is attempting to read files on our firewall. It's Saturday, so if I can get into some rabbit holes I will. Let's check it out!

## Analysis
### Alert Examination
CVE-2024-24919 is listed in the alert title. Let's look at this CVE real quick to try to get a better idea about why this alert was triggered.

![Jacob Kollasch](/arbitrary-file-read-checkpoint-gateway/cve-2024-24919-nist.png)
My understanding is checkpoint firewalls on the internet with remote access VPN enabled are vulnerable to arbitrary file read. Well if you are using your firewall as remote access VPN, i.e. employees WFH/off premises will connect to firewall to reach internal networks, then it will need to be on the public internet. So anyone using a checkpoint firewall to do remote access VPN could be vulnerable to this CVE assuming they are using a vulnerable version of the checkpoint firewall operation system.

Let's try to find some POC so we can understand how this attack actually might work.

Found a github repo with a python script POC. Run the script with the domain/IP of the target, it checks if target is vulnerable, and if it is allows you read /etc/passwd file by default. You can use the script to cd to other directories on the system.

I'm going to look at the python code and see how it works.

### CVE-2024-24191 POC Python Script Examination
#### check_vulnerablity Function Explanation
This POC script https://github.com/un9nplayer/CVE-2024-24919/blob/main/CVE-2024-24919.py contains 71 lines of code. The first function defined `check_vulnerablity` is sending 2 curl commands. The author made comments about the curl commands having various SSL options defined in the HTTP headers 1)Disable SSL verification 2)Specify SSL TLSv1.2. The commands are identical except for the second one specifys TLSv1.2 explicitly. They both use the `-k` flag to ignore ssl issues like invalid certificates and connect anyways. My hunch is the SSL stuff is related to ensuring a reliable connection for curls HTTP request, and isn't related to the firewalls vulnerability.

The more interesting portion of the curl requests is using the `--data-binary` http header to retrieve a file on the system starting with the `aCSHELL` directory and navigating up the file tree several levels using the linux `..` special character. I'm assuming they are ending at root and then specifies a variable `{file_name}` defined as `etc/passwd` earlier in the script.  

So the script is attempting to access a directory that is probably related to function of the firewalls VPN service using an HTTP POST request, and the firewall is not filtering this header to prevent directory traversal. Additionally, the file specified is being returned in the HTTP response coming from the firewall. I wonder how this was discovered? Perhaps some fuzzing on known directories for this checkpoint firewall VPN service.

Next the script loops through the HTTP response from the server looking for keywords commonly found in `/etc/passwd` file, such as `root` or `admin`. If these keywords are found it prints to the user that this host is vulnerable
![Jacob Kollasch](/arbitrary-file-read-checkpoint-gateway/poc-python-code.png)
#### main function explanation
This script uses the main function which I learned about in the first chapter of *Black Hat Python*. The main function is nice for scripts because you can define functions and classes in one part of the script, the main function of the script that uses those functions/classes runs under the `main` function. When you run the program from the command line like `python3 script.py` it executes the code in the main function first. Also if you import the script into other scripts/programs that script can use functions and classes without executing the main function. Seems like a good habit for python scripting.

The first thing the script does is read a file that the user fills out with the domain or IP of the target and will complain if not formatted correctly or found. Assuming the domain/IP is in the file formatted correctly the script extracts that data and assigns it to the `ip` variable used in the `check_vulnerablity` function defined earlier. The `check_vulnerablity` function does it's thing as I just explained above, and if it returns the output of the `etc/passwd` file the script asks the user if they want to display it to screen, and then it will ask if you want to get another dir. NEAT! 

This POC script is pretty simple, and I am wondering if it could easily be improved to include shodan api requests for this vuln and automatically grab files from those servers as they are found by shodan. That could be a fun project to show how a TA could automate the collection files from firewalls with this vuln. They could then use the usernames and servers IPs to attempt a brute force login to found usernames using common password lists. Could be fun to think about more.

### Watchtower technical writeup for CVE-2024-24191
I read this https://labs.watchtowr.com/check-point-wrong-check-point-cve-2024-24919/#/ article to learn more about the technical details of this exploit. It was very interesting. The researcher who wrote this article heard about the vulnerability from checkpoint advising that it's being actively exploited and provided a patch at time of disclosure. The researcher thought they were being too vague, so they got their hands on a checkpoint firewall, the vulnerable software, and the patch. 

The researcher was able to a binary file for the new patch, put both versions of code into IDA to reverse engineer the code, then they did a diff on the two versions of FW to determine the changes that were made to fix the issue vaguely defined by the vendor. This lead them to notice new code that is logging the string "Suspected path traversal attack from". This had the researcher thinking this CVE is a path traversal bug that could lead to arbitrary file read on the device.

Without re-writing the researchers article that is very worth reading, basically the researcher finds some new functions called `send_path_traversal_alert_log` and `Sanatize_filename`. The second function calls the first, but the `sanatize_filename` function is called by a auto-generated function called `sub_80F09E0`. That auto generated function was referred to by HTTP path called `/clients/MyCRL`. At this point the researcher is convinced that this is a path traversal bug that affects the `/clients/MyCRL` endpoint. Pretty cool to learn how they were able to that.

After some research they found the purpose of this endpoint is to serve static files from a location on the file system. So if the developers aren't properly sanitizing requests made to this endpoint it could be exploited to server files you aren't supposed to be able to access. Well could they just navigate to a file by sending a http request to the potentially vulnerable endpoint with the new file location at the end like `/clients/MyCRL/target_file`? No that would be too easy, they went on to try directory traversal techniques like `..` and adding especial characters to escape any file sanitizing, but nothing was working.

Finally, they went back to the auto-generated function `sub_80F09E0` to look for more clears. It was there where they made a breakthrough. The `sub_80F09E0` function is checking a list of safe files to request, if it's on the list the file is served, and if not it's denied. The researcher found the `C` function `strstr` is used and it doesn't compare strings exactly, instead it searches one string for the contents of the other. So if the researcher includes a file on the allowed list, and then uses directory traversing techniques to access other files on the system it is allowed! This file was `aCSHELL`, so any POST request with this string in the POST BODY to our endpoint `/clients/MyCRL` will work. Sprinkle in some path traversal and the researcher was able to get the `/etc/passwd` file from they system. I should figure out why GET requests wouldn't work, but I need to keep moving. 

```
POST /clients/MyCRL HTTP/1.1
Host: <redacted>
Content-Length: 39

aCSHELL/../../../../../../../etc/shadow
```
Working Post Request

Wow, it sounds like witchcraft was used to find this bug, but it in reality was just reading the code and understanding what it's actually doing, then the researcher saw a way around it. Very cool!

There was much more detail explained in the article, so read it! However, I just wanted to show that I went through it to learn about this CVE and use that to improve my ability to analyze this alert. It was a very cool window into actual security researcher/bug hunting activity.
### Alert Examination Continued
We certainly jumped in a rabbit hole, but it was so worth it. I feel like already know what to look for now, so let's see if we get surprised and what else we can learn from this alert exercise.

| Category               | Data                                                                                                |
| ---------------------- | --------------------------------------------------------------------------------------------------- |
| Event Time             | Jun, 06, 2024, 03:12 PM                                                                             |
| Rule                   | SOC287 - Arbitrary File Read on Checkpoint Security Gateway [CVE-2024-24919]                        |
| Hostname               | CP-Spark-Gateway-01                                                                                 |
| Destination IP Address | 172.16.20.146                                                                                       |
| Source IP Address      | 203[.]160[.]68[.]12                                                                                 |
| HTTP Request Method    | POST                                                                                                |
| Requested URL          | 172.16.20.146/clients/MyCRL                                                                         |
| Request                | aCSHELL/../../../../../../../../../../etc/passwd                                                    |
| Alert Trigger Reason   | Characteristics exploit pattern Detected on Request, indicative exploitation of the CVE-2024-24919. |
| Device Action          | Allowed                                                                                             |

Well at first glance I would say a checkpoint firewall was attacked using the path traversal/arbitrary file read attack defined in CVE-2024-24919. They used a POST request to the vulnerable endpoint, and they request in the POST body targets the vulnerable directory `aCSHELL` and used path traversal to request the `/etc/passwd` file. 

Destination IP address is a local IP address in the `172.16.0.0/12` range. And the source IP address is a public IP address. You would think the attack was done from the internet, so how would they have targeted firewall on a private IP address? Not going to get stuck on that thought right now.
### Data Collection/Analysis
So let's look into this alert and gather useful data using the data from our alert.
#### Source IP - 203[.]160[.]68[.]12
Source IP address is public, so let's check it's reputation.
![Jacob Kollasch](/arbitrary-file-read-checkpoint-gateway/abuseipdb.png)

![Jacob Kollasch](/arbitrary-file-read-checkpoint-gateway/virus-total.png)

![Jacob Kollasch](/arbitrary-file-read-checkpoint-gateway/ip-rep-check-3.png)
I can only assume the fictional company we are defending is not located in China, so why is this public IP address hosted by a Chinese telecom connecting to our firewall configured as a remote access VPN? It is probably not for good reasons, and we assume it to be malicious. Besides that fact, we have 2 reports on IPDB, one of which is specifically referencing the CVE that triggered our alert, and VirusTotal has 3 vendors flagging this IP. Safe to say this IP has a bad reputation, and we don't want it connecting to our systems at all.

Almost forgot to check LetsDefend `Threat Intel` field. It's saying it's related our CVE.
![Jacob Kollasch](/arbitrary-file-read-checkpoint-gateway/letsdefend-threat-feed.png)

#### Destination IP address - 172.16.20.146
Got my head scratching again thinking about why the heck is our Checkpoint Firewall configured as a remote access VPN getting hit on a private IP address? Sure the firewall could have a private IP address configured to a management interface, or an internal interface, but a threat actor from the internet shouldn't be attacking those interfaces... again from the internet. Maybe the public IP used to access the VPN is proxied to this firewall (sounds hand-wavy) so the alert shows our the firewalls private IP. There might be some advanced edge network config at this fake company I am not aware of. I am going to chalk it up to letdefend not wanting their purposefully vulnerable lab equipment hanging out on the internet just so this alert looks more real. Hopefully I'm not overlooking something obvious, so if you are reading this somehow and know the answer I would love to hear it. 

Searching the IP on `Endpoint Security` page shows us this is indeed a checkpoint firewall in the `LetsDefend` domain. Last login was `Jun, 05, 2024, 09:05 AM`, and are alert was triggered on `Jun, 06, 2024, 03:12 PM`, so it's safe to say **this attack didn't result in the threat actor logging in to the firewall**. Hostname is `CP-Spark-Gateway-01`, owned by `admin` (nice... I'll be sure to ping `admin` on teams about this). 
![Jacob Kollasch](/arbitrary-file-read-checkpoint-gateway/ld-host-info.png)

OS version is `Check Point R80.20 Gaia`. Let's see if that's the vulnerable version. `Ctrl+f` then searching `R80.20` on the NIST CVE bulletin shows me it is a vulnerable version. If only `admin` would have patched this internet facing firewall before getting hit... shoot.
![Jacob Kollasch](/arbitrary-file-read-checkpoint-gateway/nist-patch-info.png)

If this wasn't a LetsDefend lab, and I only knew that this firewall was administered using an account labeled `admin` I would try to determine who is using this private IP address range at our company and reach out to that team. The hostname might give me so more clues about who owns this firewall. In reality, if I couldn't determine it myself, I would ask my team, then boss, and anyone who might know in order to get this system patched.

Looking at our firewall on the `Endpoint Security` again I see it's running a `vpn` service. Another indicator we are setup to be vulnerable to this CVE.
![Jacob Kollasch](/arbitrary-file-read-checkpoint-gateway/vpn-service-endpoint.png)

Switching over to the `Network Action` tab for the firewall I see our malicious source IP `203[.]160[.]68[.]12` from our alert in the **destination IP address** logs... dated the day after the alert. The logs don't go before Jun 6 20204, so perhaps the attacker is coming back and reading different files from the firewall? I also see the firewall connecting to several IP address very similar to the one that triggered our alert. Same first three octets, and seeing `.11`, `.13`, `.15`, `.16`, `.17`, and `.20`. I'm assuming they are also malicious and could be related. Okay, so maybe the threat actor came back and tried to read more files manually after an automated script found this firewall to be vulnerable? Just a hunch.
![Jacob Kollasch](/arbitrary-file-read-checkpoint-gateway/endpoint-network-activity.png)

### Log Analysis
Jumping over to the `Log Analysis` section to see what we can find. Starting with a search for our malicious source address we get 2 firewall entries. Both alerts are targeting the IP of our firewall, and are POST requests. The first alert is hitting the root `/` endpoint with no data in the request body. Perhaps they were just testing connectivity. The second alert targets are vulnerable `/clients/MyCRL` endpoint and the request body contains the path to the directory vulnerable to path traversal, and the `..` path traversal method is used to reach `/etc/passwd`.
![Jacob Kollasch](/arbitrary-file-read-checkpoint-gateway/endpoint-log.png)

## Findings
At this point I am sure that this alert is a **true positive**, and it the attack appears **successful**. In order to really know if the attack was successful I think we would need to be logging HTTP responses, but I believe that's very uncommon because of how many logs it would generate and the privacy implications. 

I `contained` the firewall in the LetsDefend lab environment and escalated the alert to tier 2. 

## IOCs
The only IOC I came up with was the threat actors IP address.

203.160.68[.]12  - IP address

## Recommendations
My recommendation would be to patch the firewall because it has been released, and then test if firewall is still vulnerable to this attack when running the patched OS.

## Prevention
Keep systems patched regularly, but this exploit was out in the wild and being exploited before vendor even knew and built patch to prevent. Unless you are writing your own firewall FW (which would be worse) there isn't a ton you can do. Perhaps buy firewalls from a vendor who has never had a bug or has perfect software. Sadly, they all have issues like this eventually. 

Make sure teams are paying attention to CVEs and vuln reports related to vendors and gear your shop actually uses so at least you can get a jump on pulling vulnerable systems/patching.

## Checking Community Walkthroughs
Believe me or not everything done so far was done with my own brain and tools available. Honestly, looking up the CVE before getting into the alert seemed like cheating. I think I was just able to understand what's happening form reading the python POC code, and learning about how it was discovered from watchtowr labs.

So if I was on the job as a SOC analyst the above is the work it the work my employer would have got, but I want to LEARN. Now I am going to peak at the community walkthroughs to see if I can learn something for the future.

I skimmed through this very nice writeup https://jasonrowe.substack.com/p/soc287 and was able to answer my biggest question. Was the attack successful, and how to determine that? I was pretty sure it was successful just because everything was perfect for this exploit to work, but to KNOW for sure, it was so obvious and I should have thought about this. I should have searched HTTP logs on the firewall, and if a 200 response code was sent to our malicious IP address in response to the malicious POST request that was sent, then we can say the attack was indeed successful. I didn't know how to easily do this in LetsDefend, but its not bad at all. You search the firewalls IP in `Log Management` and it will reveal an `access.log` in this case so we can check those HTTP logs.
![Jacob Kollasch](/arbitrary-file-read-checkpoint-gateway/endpoint-detailed-log.png)

Then we find the log with the timestamp corresponding to the bad POST request, and boom that's a 200. So the attack was successful, and we checked the HTTP logs on the firewall to determine that by seeing response was sent.
![Jacob Kollasch](/arbitrary-file-read-checkpoint-gateway/http-response.png)
# Conclusion
Well there you have it that was my thought process when solving this alert walkthrough. I plan on documenting as many as I can in the next couple of months in order to prove to employers I have the skills to be hired as a SOC analyst. I went deep on the CVE itself and I don't regret at all. It was a great learning experience.

# Lessons Learned
- **Personal Reflection**: I feel really good about this alert. It wasn't very difficult for me to understand what was going on, and I was able to learn some really interesting things about exploit research. My favorite part was learning about the technical details of the CVE. Doing so gave me a huge advantage for analyzing this alert.
- **Challenges and Solutions**: I failed to truly determine if the attack was successful by finding the HTTP response, and I didn't figure it out without looking at community walkthrough. This is a symptom of not knowing how to use my "tools" correctly. In this case the tools are inside the LetsDefend platform. The more time I spend working on these alerts the less mistakes like this will happen.