---
author: "Hugo Authors"
title: "Windows Authorization Process Explained"
date: 2024-06-23T12:00:06+09:00
description: "Explaining how the Windows Authorization Process Work."
draft: false
hideToc: false
enableToc: true
enableTocContent: true
author: Jacob Kollasch
authorEmoji: By
pinned: false
tags: 
- hugo
- github
- netlify
---
This article is about windows **authorization**, or the process of how the Windows OS allows access to resources, like opening a file, directory, or registry key.

# Basic Access Check
## Comparing Windows Auth to Linux Auth 
- Window auth process is much more complex than linux 
- There are many pieces and acronyms to windows auth. A handy glossary - [https://learn.microsoft.com/en-us/windows/win32/secgloss/a-gly?redirectedfrom=MSDN](https://learn.microsoft.com/en-us/windows/win32/secgloss/a-gly?redirectedfrom=MSDN) 
- Imagine you are logged into your Windows system and want to open a file, or **read access** to the file. How is that access controlled and what is the process to decide? 
- On linux/unix systems, each user has a **user ID** and **group ID**, and the **access target** (file, in this example) 
- The **access target** (file) has a list of access permissions for example 
	- -rwxr-xr-- 1 root root 2048 Jun 13 10:00 file.txt 
- The linux/unix OS checks the file permissions, and determines that everyone has access to **read** this file.  
	- The first group of letters after the dash in the access permissions list is for every user, and because there is a 'rwx' it means every user has **read, write,** and **execute** permissions for our example file 
- The user is then allowed to **read** this file  
- This is how most linux/unix authorization decisions can be broken down
## Access Token
When you first login to a Windows computer a **Access Token** is created for the user. Doesn't matter if local user account, or domain joined user account you get an **access token**.

This **access token** is created for the user's logon thread. Logon threads are the new thread within the LSASS process spawned to handle this users logon session. 

*All child process lauched by the user will inherit this **access token***.

This **access token** will contain some useful information to identify the user
- User's Security Identifier (Owner SID)
	- Uniquely identifies the user
- Group Security Identifier (Group SID)
	- Identifies all groups a user is a member of
- Privileges
	- system level permissions i.e. ability to reboot system, change certain settings etc.
- Access Control Entries
	- Based off ACLs to dictate what resources the token can access and with what permissions
- Token type
	- Primary token or impersonation token

Okay, so now you are logged in and have an access token assigned, so let's say you go to open a file. This will generate an **access request** that will specify what the user wants to do with that file. The access request will align with an access right also known as 'permissions'. There are different sets of access rights that contain individual access rights inside them.
## Access Rights
- **Generic Access Rights** - Most important
	- Simplified perms that map both standard and object -specific rights i.e. **GENERIC_READ** **GENERIC_WRITE**
- **Standard Access Rights** - Most important
	- General perms that apply to all types of projects i.e. 'READ_CONTROL'
- **Object-Specific Rights**
	- Perms unique to specific types of objects i.e. file object would have "Read", "write", "Execute"
- **Special Access Rights**
	- Manually configured to meet specific needs that aren't pre-defined by the system

Access rights boil down to numbers that represent different access rights. The numbers are represented in bitmasks that are typically represented in hex and can get quite complicated. Most are defined in the Windows API for example.

For example, the `FILE_ALL_ACCESS` constant, which specifies all possible file permissions, is defined as `0x001F01FF` in the Windows API.

## Access Request
Back to Access Requests, so you opened a file and this generated an **Access Request**. Windows handles this access request with the **Security Reference Monitor (SRM)**. **SRM** will compare the requested access right in the actual access request, the users Access token, and the access permissions for the requested file. The key function for granting or denying access within SRM is the **SEAccessCheck**.

The access permissions for the requested file are stored in a structure called the **Security Descriptor**. Each **Securable Object**, files, directories, reg keys, has a security Descriptor that defines who can do what with that object.

Key takeaways so far:
- Each user **logon thread has an access token**  
    _(Read this as: Each user got assigned an access token)_
- A required action, e.g. to open a file, is expressed in an **Access Right**, e.g. GENERIC_READ
- Each [Securable Object](https://msdn.microsoft.com/en-us/library/windows/desktop/aa379557(v=vs.85).aspx) has a **Security Descriptor**, which defines who can do what with that object.
- The **Security Reference Monitor (SRM)** is the software component in charge to evaluate if a certain process should be granted access to the requested object for the requested access right.

These are the basics of Windows Authorization, so let's get deeper into some components.

# Security Descriptor, DACL & SACL
Each **Securable Object** has a **Security Descriptor**. The Descriptor defines an object structure to keep record of whom the object belongs to (Owner and Group) and to whom access is granted or denied for which actions.

The owner and group field are expressed by a Secure Identifier (SID). In windows a SID is a unique sting that identifies a security principal, user, group, etc. SIDs can be looked up https://learn.microsoft.com/en-US/windows-server/identity/ad-ds/manage/understand-security-identifiers

The SID structure has three blocks

- First block, always starts with an 'S', and describes the group, i.e. Domain Everyone (S-1-1-0), Anonymous (S-1-5-7), Authenticated Users (S-1-5-11) which can be looked up in the register for Will-known SIDs
- Second block, is a unique domain ID (also present in each default WORKGROUP) (S-1-5-21) i.e. (S-1-5-21-**2884053423034315659790-78458365**-500)
- Final block, **relative ID (RID)**, describes a user group inside a security group, i.e. group of built-in Administrators group (S-1-5-32-544) or the built-in Guests group (S-1-5-32-546) or the famous default Admin group, sometimes referred to as RID 500 Admin (S-1-5-21-2884053423034315659790-78458365-**500**)

## DACLs
Discretionary Access Control Lists (DACL) is a a component of the Security Descriptor that every securable object has. The DACL is a list controlled by the owner of the Security Descriptor. DACLs specify the access particular users or groups can have on the securable object
- Contains 3 **Access Control Entries (ACE)** 
- Each **ACE** has important fields set
	- SID: User or Group to which ACE applies
	- Mask: numeric value that ressembles addition of multiple access rights
	- AceType: Specifies whether access should be allowed or denied for the given **SID** and based on the **mask**

"The SCM will review the Security Descriptor of the requested object and iterate over each **ACE** in the Security Descriptor’s **DACL** and check whether the requesting user - identified by it’s **SID** and requested access right (e.g. _GENERIC_READ_) match with an **Access Mask** inside of an **ACE**. If a match is found the **AceType** of that ACE will determine if access is granted or denied." - from cskander article

If a **security descriptor** has no DACL, or DACL is set to null, then **full access for any user**

If a **security descriptor** has a DACL, but no **ACE's** than **no access is granted to object**

## Recap So Far
So far we have discussed **access tokens** that are generated for a user when they log in. 

When a **securable object** (file, etc.) is accessed the **access token** for the user trying to access the object is compared against the **security descriptor** for the the object. 

The **security descriptor** contains an **SID** for the owner and another for the group of the of object. 

**Security descriptors** contain an **SACL** and **DACL** list. 
	**SACL** holds **MIL** which hasn't been covered yet. 
	**DACL** holds **ACEs** that specify 1)what groups/users apply to the **ACE**, 2) what access rights the ACE allows or denies access to, and 3) whether it allows or denies

Okay and now let’s put in the full picture for the **SeAccessCheck**:  
After the Integrity Level-Check (IL-Check), which will be described in the following section, the Security Reference Monitor (SRM) queries the Security Descriptor of the requested object for the following information:

- Who is the owner of the requested object?  
    → If the requesting user is the owner of the requested object: Grant Access
- If the requesting user is not the owner, iterate over all **ACEs** in the **DACL** of the Security Descriptor and determine:  
    – Does the ACE apply for the requesting user by comparing the ACE SID to the user SID and his/her group SIDs ?  
    – Does the access mask contain the requested access right, e.g. does the access mask contain the GENERIC_READ access right ?  
    – Does the access type is set to allow access or is it set to deny access?
- If any ACCESS_DENIED_ACE_TYPE matches the user and requested access right: Deny Access
- If no ACCESS_DENIED_ACE_TYPE but a ACCESS_ALLOWED_ACE_TYPE matches the user and requested access right: Grant Access

# Integrity Level Check / Mandatory Integrity Control (MIC)
**MIC** check occurs before **DACL** check, and determines access to a securable object. 

That sounds redundant, and the center of authorization checks and most access is determined by DACL, but there was a need for **MIC** checks and they were introduced in Windows Vista along with **User Access Control (UAC)**. In simple terms, **MIC** and **UAC** allow for nuanced control and was designed to prevent compromised processes from accessing sensitive resources.

Every **Securable Object** gets a **Mandatory Integrity label** set in **SACL**. **MIL's** have one of four **Integrity Levels**. *Default for every object is **Medium*** 
- Low
- Medium
- High
- System

**Integrity Levels** prevent prevent lower IL processes from accessing higher IL objects.
Example. Internet explorer has a low **IL** process when started. If the **IE** process compromised by exploiting a vulnerability, they would not be able to access a user's documents because they were created with the **default medium IL**.
## What happens during IL check? IL Policy
SRM compares the IL of the requesting process with IL of request object and decides to go to the IL DACL check (becuase they share the same IL level), or denies access based on **IL Policy** defined for that object. 

### IL Policy
The IL Policy of an object is based on three defined states contained in the Security Descriptor. Defined by [SYSTEM_MANDATORY_LABEL_ACE Structure](https://msdn.microsoft.com/en-us/library/windows/desktop/aa965848(v=vs.85).aspx)
- SYSTEM_MANDATORY_LABEL_NO_WRITE_UP ( _0x1_ )
- SYSTEM_MANDATORY_LABEL_NO_READ_UP ( _0x2_ )
- SYSTEM_MANDATORY_LABEL_NO_EXECUTE_UP ( _0x4_ 

For example if the SACL ACE **SYSTEM_MANDATORY_LABEL_ACE_TYPE** is set to 0x0000003 it means IL policy denies READ and Write attempts from process with lower IL level. Its the addition of WRITE 0x1, and READ 0x2 to equal 0x3.  
### Real Reason for Integrity Level Check
Imagine Internet Explorer is compromised without MIC. The IE process would be launched by the logged in user and owned by that user. So the DACL check would grant IE process access to the user's resources if compromised. 

# Privileges Compared to Access Rights
So far we have discussed access rights. Access rights control access to securable objects. **Privileges** control rights for accounts and groups just like access rights, but for system-related operations. For example, the ability to shutdown the system, load drivers, or change system time.

**Privileges** are assigned by system administrators to user and group accounts, but the **system** grants or denies access to **securable objects** based on access rights granted in **ACEs** in the **securable object** **DACL**

Every system has an account database that stores privileges for each user and group account. When a user logs in, an **access token** is created that holds a list of the users privileges, including to the user or to the groups user logging in belongs. These privileges only apply to the local computer.

**Domain accounts can have different privileges on different computers**

## Privileges Enumeration
`whoami /all` will list what privileges the user logged in has. If command prompt, or powershell instance used to run this command is **Run as Administrator** *more **privileges** will be listed*.

**Privileges** can be looked up here https://msdn.microsoft.com/en-us/library/windows/desktop/bb530716(v=vs.85).aspx

# UAC (User Access Control)
When a user logs in to a windows computer the logon thread is assigned a user access token and all child processes launched by that user **will inherit this token**. 

**UAC** was needed to prevent admin users, who would receive a high integrity level token with all admin privileges, from having all child processes inherit that high level token.

When a non local RID-500er admin (that is a special use case where no UAC is used unless enabled with reg key) logs into a system with UAC enabled **two access tokens are created**.
- **Full Admin Token** with high integrity level and all admin privileges
- **Filtered Admin Token** with medium integrity level and reduced set of privileges

These two tokens are linked in the kernel allowing threads to access both tokens.

So if an application like powershell is launched that process will be spawned using the filtered token. This powershell instance will have reduced privileges, so you won't be able to do as many things like editing reg keys, starting/stopping services, installing software via powershell, etc.

However if you open powershell with the "Run as Administrator" option the powershell process will be spawned with the full admin token. After answering the UAC prompt.

# Two Types of Tokens
User Access Tokens hold useful information about the user and groups. The Security Resource Monitor Uses these tokens to determine who is asking for access.

Two important kinds of tokens
- Primary tokens - assigned to processes
- Impersonation tokens - assigned to threads **Like our user logon thread**

Threads can have an impersonation token as well as a primary token, but **impersonation token will always be prioritized**.

# Impersonation
An important field of impersonation tokens is the **impersonation level**, and If it's set to 'impersonation' than that means teh token can be impersonated.

Why would a token need to be impersonated? Think of a user wants to access a file on a remote share. The server hosting the file needs to decide if the user is allowed to access this file. Well the users access token isn't stored on the remote server. The token is stored in the memory of the local system. Microsoft came up with a technique, **impersonation**, where the server will pretend to be the user and do the action as the user by duplicating the users impersonation token from their logon thread. Remember, threads use impersonation tokens always.

# Conclusion,  Sources, and Where to Learn More
Source https://csandker.io/2018/06/14/AWindowsAuthorizationGuide.html - The post you just read is essentially just my notes from reading this amazing guide multiple times, so all real credit goes to him. I tried to put the windows authorization process in my own words, and align it with my own thinking process so it's easier for me to reference. I feel like I could go over this again and again and still learn more about Windows authorization. 

I've also been reading Chapter 7: Security from Windows Internals Part 1, and find a lot of overlap to what was covered here, but it goes into way more detail, so if you are looking to really deep dive this stuff that's an excellent place to go.


