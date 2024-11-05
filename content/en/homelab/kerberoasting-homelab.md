---
author: "Jacob Kollasch"
title: "Keberoasting - Homelab"
date: 2024-07-31T12:00:08+00:00
description: "Explaining kerberoasting attacks by setting up and executing and attack in my homelab."
draft: false
hideToc: false
enableToc: true
enableTocContent: true
author: Jacob Kollasch
authorEmoji: Written by
pinned: false
tags: 
- homelab
- windows
- linux
- attack
- defend
- detect
- SOC
- kerberoasting
---
## Objectives
This guide aims to explain the following.
1. **Explanation** - What is Kerberoasting, why it works, what attackers need, and how to identify this vulnerability in Active Directory environments.
2. **Setup** - How to configure my homelab to be vulnerable to a Kerberoasting attack.
3. **Attack** How to execute a Kerberoasting attack, and discuss lateral movement after success.
4. **Detect** - Creating alerts for Kerberoasting attacks in my Wazuh SIEM running in my homelab.
5. **Remediation** - Steps a SOC analyst should take to stop this attack.

## Kerberoasting Attack Explained
Kerberoasting is an attack that targets service accounts in an Active Directory environment with the goal of cracking the password. Kerberoasting takes advantage of how the Kerberos authentication protocol that is used in Active Directory networks for users and services to prove their identity.

The obtained service account credentials can then be used by an attacker to escalate privileges, move laterally in the target network, exfiltrate data, and deploy malware. Exactly what the attacker will be able to do with the service account is largely dependent on the privileges and access of the service account, as well as the creativity of the attacker. 

## AS-REP Roasting
It's important to talk abut **AS-REP Roasting** in the same conversation as **Kerberoasting** because they are very similar attacks, but why they work is a little different

**AS-REP Roasting** targets a normal user account instead of a service account. That means it's a user account without an SPN configured. However, the user account must not have `Kerberos preauthentication` enabled. 

The **AS-REQ** message involved in the Kerberos authentication process contains the users password. It's not stored in plaintext, but rather is `encrypted with uesrs hash`. So the hash of the password stored in **KDC** database is used to 
### Why Kerberoasting Works
This is the most important thing to understand in order to defend and execute kerberoasting attacks. 

1. Services accounts contain a **SPN (Service Principal Name)**, which means the *service account password is contained in the service ticket*
2. Any AD user can do a query for which accounts have an **SPN**, and request a service ticket for for any service. **No special admin privileges required**
3. The service ticket that contains the encrypted password can be cracked with tools like `hashcat` by brute forcing against a password list

So the most difficult part of the executing the kerberoasting attack is finding a service account with a password that can be cracked with a good password list like `rock-you.txt` or with a custom password list tailored to the target.
## Attack Environment Setup
How does our Active Directory Environment need to be configured to pull off a kerberoasting attack?

Kerberoasting is taking advantage of poorly configured service accounts. Service accounts are similar to normal user accounts, but the difference that makes them vulnerable to Kerberoasting attack is they have a **SPN (Service Principal Name)** configured.
### Creating Vulnerable Service Account
1. Create a normal user account from Active Directory Users and Computers
2. Open Powershell as Administrator and use `setspn` tool generate SPN for our new service account
```
setspn -a WIN-SVR-TYRELL/tyrellsvc.tyrell.lab tyrell\tyrellsvc
```

## Executing Kerberos Attack
### What we need as an attacker to Execute Kerberos Attack
1. Vulnerable service account (User account configured with SPN)
2. Access to any AD user account, no admin or special privileges required.
3. Network access to reach Domain Controller in order to use tool like `rubeus` to request password hash. However, this does not have to be done from a domain joined machine.
4. Tool to request hash containing service account password. `rubeus` or `impacket` are popular options and will be covered from an attack and defense perspective here
5. Tool to crack or extract password from the received hash. `hashcat` will be used in this post.
6. Password list containing the password for the service account.

## Running a Kerberoasting Attack
```
Rubeus.exe kerberoast
```
![Jacob Kollasch](/kerberoasting-homelab/rubeus-roast.png)

## Cracking Password in Service Ticket
```
hashcat -a 0 -m 13100 hash.txt password-list.txt -o cracked-hash.txt
```
![Jacob Kollasch](/kerberoasting-homelab/hashcat-working.png)

Catting the contents of the output file we can see the password at the very end of the output after our hash
![Jacob Kollasch](/kerberoasting-homelab/cracked-hash.png)

## What Next? Lateral Movement
Keeping our attack hat on for a moment, let's think what can we do with our new service account. We already needed an AD user account to pull of this kerberoasting attack, so why did we get the credentials for this service account? 

Basically the hope is the service account has more privileges, and access to other systems that can be leveraged to do all sorts of bad guy activities like data exfiltration and deploying malware. 

For example, the service accounts are more likely to have.
1. **More privileges**
	- Service accounts can often RDP to into servers that our original account may not have been able to. They account might exist in order to run scheduled tasks, which could be a huge advantage to an attacker. There is a chance the account could even have domain admin rights allowing the attacker to essentially do whatever they want.
2. **Less likely to be disabled**
	- Service accounts should be tightly controlled by IT departments, but can easily be forgotten about and missed during termination processes if they belonged to an administrator. If the account was used for an application, then it could be left enabled for fear something will break if they are disabled
3. **Access to important systems**
	- The service account could have access to file servers or databases.

At this point it's crucial for the attacker to enumerate the privileges the service account has, and to determine what other systems the account may be able to access. 

## Detect, and Defense Against Kerberoasting Attacks Coming Soon
I plan to finish this article by covering the detection in Wazuah with a custom rule, and defense aspect explaining how to configure your infrastructure to prevent this attack from occurring.