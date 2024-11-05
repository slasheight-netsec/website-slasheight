---
author: "Hugo Authors"
title: "Core Windows Processes Explained"
date: 2024-11-06T12:00:06+09:00
description: "Explaining Core Windows Processes and what's unusual."
draft: false
hideToc: false
enableToc: true
enableTocContent: true
author: Jacob Kollasch
authorEmoji: By
pinned: false
tags: 
- windows internals
- processes
---

Processes are containers for a set of resources used when executing an instance of a program. It is not the same as the program, which is a static sequence of instructions, or code.

So what are the most important windows processes to know as a SOC analyst? Let's start at the beginning, the first windows process called **System**.
## System
PID or "process ID" is assigned to a process at random, except for the **system** process, which is always "4".
### What does **system** process do?
Home of a very special thread that runs only in kernel mode. A kernel-mode system thread. **System** threads have all the attributes and contexts of regular "user-mode threads", but they only run in kernel-mode, executing code loaded in system space. An example, thread could run in Ntoskrnl.exe, or any other loaded device driver.

**System** threads don't have a user process address space, so **system** must allocate any dynamic storage from operating system memory heaps.
### Unusual things to look for with System Process
* Parent Process that is not 0
* Multiple instances
	* Should only be one instance of system process
* A different PID
	* It should always be 4
* Not running in session 0

## smss.exe (Session Manager Subsystem)
This is our first **user thread**, and it's also known as **Windows Session Manager**. It's responsible for creating new sessions.

**sms.exe** starts the kernel and user modes of the **Windows Subsystem**. The windows subsystem includes **win32k.sys (kernel mode)**,  **winsrv.dll (user mode)**, and **csrss.exe (user mode)**

**sms.exe** starts 2 process in session 0, which is the isolated session for the operating system, and 2 process in session 1, which is the user session. 

**csrss.exe (windows subsystems) & wininit.exe** in session 0

**csrss.exe & winlogon.exe** in session 1

Lastly, any other subsystem listed in the "Required" value of this reg key is launched
`HKLM\System\CurrentControlSet\Control\Session Manager\Subsystems`

**smss.exe** also creates **environment variables**, **virtual memory paging files**, and starts **winlogon.exe** (the windows logon manager)

### Normal things to look for
- **image path** ``%SystemRoot%\System32\smss.exe`
- **Parent Process** - System
- **Number of Instances** - One master, and a child instance per session after the sessions are created
- **User Account** - Local System
- **Start Time** - Within seconds of boot time for the master instance

### Unusual things to look for
- Different parent process other than systm(4)
- The image path is different from `C:\Windows\System32`
- More than one running process (children should self-terminate and exit after each new session)
- The running User is not the SYSTEM user
- Unexpected registry entries for subsystem

## csrss.exe
**Client Server Runtime Process** is the user-mode side of the windows subsystem. Always running, critical, and if terminated, the system will fail.

**csrss.exe** is responsible for the Win32 console window, process thread creation and deletion.

Each instance of **csrss.exe** loads csrsrv.dll, basesrv.dll, and winsrv.dll and others.

**csrss.exe** is responsible for:
- making the Windows API available to other processes
- mapping drive letters
- Handling the Windows shutdown process

## wininit.exe
Launches the following processes in **session 0**
- **services.exe (service control manager)** 
- **lsass.exe (Local Security Authority)**
- **lsaiso.exe** (associated with **Credential Guard and KeyGuard**), so will only be there if credential guard is enabled

### What is normal?
- **image path** ``%SystemRoot%\System32\wininit.exe`
- **Parent Process** - Created by an instance of smss.exe, but will show as `Non-existent process (384)` (smsss.exe calls this process then self-terminates)
- **Number of Instances** - One
- **User Account** - Local System
- **Start Time** - Within seconds of boot time
### What is unusual?
- Actual parent process that isn't smss.exe self-terminating
- Image file Path other than "`C:\Windows\System32`
- Subtle misspellings with the intention hide bad processes in plain view
- Multiple running instances
- Not running as SYSTEM

## services.exe
**services.exe** **(service control manager or SCM)** is responsible for
- handle system services
- loading services
- interacting with services
- ending services

**services.exe** maintains a database that can be queried using a Windows built-in-utility `sc.exe`

Information regarding services are stored in the registry at `HKLM\System\CurrentControlSet\Services`

**servcies.exe** also loads device drivers marked as "auto-start" into memory

When a user logs into a machine, **services.exe** sets the value of the "Last Known Good control set" (Last Known Good Configuration) in the registriy at `HKLM\System\Select\LastKnownGood`

**services.exe** it the parent to several key processes
- **svshost.exe**
- **spoolsv.exe**
- **msmpeng.exe**
- **dllhost.exe**

Terminating this process will cause "blue screen of death"

### What is normal?
- **image path** ``%SystemRoot%\System32\services.exe`
- **Parent Process** - wininit.exe
- **Number of Instances** - One
- **User Account** - Local System
- **Start Time** - Within seconds of boot time
### What is unusual?
- Parent process different than wininit.exe
- Image file Path other than `C:\Windows\System32`
- Subtle misspellings with the intention hide bad processes in plain view
- Multiple running instances
- Not running as SYSTEM

## svchost.exe
**Service Host** (Host Process for Windows Services) manages Windows services. These services are implemented as DLL's.

DLLs to use for a service are stored in the registry under the `Parameters` subkey in `ServiceDLL` `HKLM\SYSTEM\CurrentControlSet\Services\SERVICE NAME\Parameters`

### -k parameter
`-k` is found in the binary path for **svshost.exe**. `-k` is for grouping similar services to share the same process. This was implemented to reduce resource consumption. Services that are bundled with svchost.exe will follow the `-k` flag 

example `C:\WINDOWS\system32\svchost.exe -k Dcomlaunch`

## Dangers of svchost.exe
Because there will always be multiple running processes of **svchost.exe** on any Windows machine, it is often the target of malicious use. For example, malware will pretend to be svshost.exe by naming itself slightly different. Another tactic is to install/call a malicious DLL.

### What is normal?
- **Image path** ``%SystemRoot%\System32\svchost.exe`
- **Parent Process** - services.exe
- **Number of Instances** - Many
- **User Account** - Varies (SYSTEM, Network Service, Local Service) depends on the instance. In Win10 some will run as the currently logged in user.
- **Start Time** - Typically within seconds of boot time, but can be started after boot.
### What is unusual?
- Parent process different than services.exe
- Image file Path other than `C:\Windows\System32`
- Subtle misspellings with the intention to hide bad processes in plain view
- Absence of the -k parameter in the command line entry for the process

## lsass.exe
Enforces security policy on the system
- Verifies users logging on
- Handles password changes
- Creates access token
- Writes to Windows Security Log
- Creates security tokens for SAM (security account manager), AD (Active Directory), and NETLOGON using authentication packages specified in `HKLM\System\CurrentControlSet\Control\Lsa`
	- These "Authentication Packages" are things like **kerberos** **NTLM**, and **KCD**
	- They are like protocols or software taking care of the lsass.exe duties
- Adversaries target this process commonly
	- **mimikatz** is used to dump creds
	- processes with slightly misspelled for the malware

### What is normal?
- **Image path** ``%SystemRoot%\System32\sass.exe`
- **Parent Process** - wininit.exe
- **Number of Instances** - one
- **User Account** - Local System
- **Start Time** - Within seconds of boot time
### What is unusual?
- Parent process different than services.exe
- Image file Path other than `C:\Windows\System32`
- Subtle misspellings with the intention to hide bad processes in plain view
- Multiple running instances
- Not running as SYSTEM

## winlogon.exe
- Responsible for **Secure Attention Sequence (SAS)** or the CTRL+ALT+DELETE, and loading the **user profile** by loading the users **NTUSER.DAT** into HKCU, and the **userinit.exe** process loads the user's shell
- Responsible for locking the screen and running the user's screensaver among other functions
- **smss.exe** launches this process along with a copy of **csrss.exe** in session 1 (user session)

### What is normal?
- **Image path** ``%SystemRoot%\System32\winlogon.exe`
- **Parent Process** - Created by instance of smss.exe that exits, so most tools won't provide the parent process
- **Number of Instances** - one or more
- **User Account** - Local System
- **Start Time** - Within seconds of boot time for the first instance (for Session 1), but additional instances occur as new sessions are created, typically through Remote Desktop or Fast User Switching logons
### What is unusual?
- An actual parent process
- Image file Path other than `C:\Windows\System32`
- Subtle misspellings with the intention to hide bad processes in plain view
- Not running as SYSTEM
- Shell value in the registry at `Computer\HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon` other than explorer.exe

## explorer.exe
- Also known as **Windows Explorer**
- Give the user access to their folders and files
- Provides functionality for other features, i.e. start menu, taskbar
- **userinit.exe** launches the value in `HKLM\Software\Microsoft\Windows NT\CurrentVersion\Winlogon\Shell` 
- **userinit.exe** then exits after spawning explorer.exe, so the parent process is non-existant
- There will many child process for explorer.exe

### What is normal?
- **Image path** ``%SystemRoot%\System32\explorer.exe`
- **Parent Process** - Created by **userinit.exe** that exits
- **Number of Instances** - one or more per interactively logged-in user
- **User Account** - Logged in users
- **Start Time** - First Instance when the first interactive user logon session begins
### What is unusual?
- An actual parent process (**userinit.exe** calls this process and exits)
- Image file Path other than `C:\Windows`
- Subtle misspellings with the intention to hide bad processes in plain view
- Running as an unknown user
- Outbound TCP/IP connections
