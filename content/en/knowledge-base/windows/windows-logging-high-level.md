---
author: "Hugo Authors"
title: "Windows Logging High-Level"
date: 2024-06-23T12:00:06+09:00
description: "High level of Guide for Windows Logging in a domain environment with a SIEM."
draft: false
hideToc: false
enableToc: true
enableTocContent: true
author: Jacob Kollasch
authorEmoji: By
pinned: false
tags: 
- windows
- windows server
- logging
- SIEM
---
# Things To Configure
- **Increase Log Size**
- **Registry Auditing**
- **File Auditing**
	- I'm doing this with wazuh configuration agent via ossec.conf file
	- Need to learn more about *what* to log
- **Force Windows to use the 'Advanced Audit Policy Settings', and then configure the advanced audit policy**
	- Increases 'what' gets logged by windows by getting more granular with the system audit categories
	- done via GPO or Local Security Policy app
	- **Local Security Policy Config**
		- Force Advanced audit policy
			- Local Security Policy App > Security Settings > local polices > Security Options > set '**Audit: Force audit policy subcategory settings (Windows Vista or later) to override audit policy category settings**.' to '**enabled**'
		- Configure advanced policy
			- Local Security Policy App > Security Settings > Advanced Audit Policy Configuration > pick settings according to policy
			- Must check "Configure the following audit events" for no logging otherwise it will break the rest of the local policy/GPO
	- **GPO Config**
		- *Figure this out and update*
- **PowerShell/Command Line logging**
	- In advanced audit policy, you need to enable 'process creation' to enable 
	- You need to create a registry entry to enable `Event ID 4688` to show the actual command being run
	- Configure reg key locally using `REG.EXE` or configure via GPO
	- **REG.EXE Config**
		```
        reg add "hklm\software\microsoft\windows\currentversion\policies\system\audit" /v ProcessCreationIncludeCmdLine_Enabled /t REG_DWORD /d 1
        ```
	- **GPO Config**
		- *Figure this out and update*
- **Sysmon Configuration**
	- Logs more things in a more verbose way than native windows event logging
	- Sysmon forwards events to the event log
	- Good idea to start with a popular sysmon config file and edit from there. Florian Roth and SwiftOnSecurity github for starter xml files
- **SIEM Agent Configuration**
	- Forwards windows events to our SIEM
	- Creates a giant haystack of information even if we only audit "certain" logs, and have a "super dialed" in sysmon config
	- We are using wazuh, so we need to configure our wazuh agent to forward our logs we just enabled. 1. Windows event logs, 2. PowerShell Logs, 3. sysmon events
	- The default Wazuh agent config found in `ossec.conf` file  and it already logs windows events
	- We need to add the following localfile sections to our config file for logs to be forwarded
- **PowerShell**
    ```
    <localfile>
    <location>Microsoft-Windows PowerShell/Operational</location>
    <log_format>eventchannel</log_format>
    </localfile>
    ```
- **Sysmon**
    ```
    <localfile>
    <location>Microsoft-Windows-Sysmon/Operational</location>
    <log_format>eventchannel</log_format>
    </localfile>
    ```

- **SIEM Rules**
	- We create custom SIEM rules to generate 'meaningful' alerts
	- This is what allows us to find needles in the haystack. Without good rules how would you really know whats going on? We can capture all the data, including data of breach/attack, but no analyst can look through it all. The SIEM rules are what give defenders their super powers.
- **SIEM Alerts**
	- Custom alerts are created based off of rules
	- Can rank by severity
	- What analysts use to identify potentially suspicious activity
# Security Events of Note
`4624/4625` - successful/failed logon attempts
`4720` - new user creations
`4723/4724` - attempts to change/reset user password

# Setup Events
- Logs things like installing AD services via server manager, or promoting server to DC
- On a windows client it's a lot about windows updates. KBXXXXX is getting installed, or need reboot to install KB, etc..
# System Events
`7036/7040` - Service Started/Stopped
- Contains logs for NTP, storage, WMI