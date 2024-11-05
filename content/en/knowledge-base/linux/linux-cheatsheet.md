---
author: "Hugo Authors"
title: "Linux Cheat Sheet"
date: 2024-11-04T12:00:00+00:00
description: "My Linux quick reference guide to commands, syntax, and basics of system administration."
draft: false
hideToc: false
enableToc: true
enableTocContent: true
author: Jacob Kollasch
authorEmoji: Written By
pinned: false
tags: 
- tmux
- linux
- productivity
---
## Special Characters
`.`     - current directory
`..`   - up one directory
`*`     - wildcard
`#`     - comment character ignored by shell
`~`     - tilde expansion represents users /home directory
`>`     - redirects output to a file (overwrites)
`>>`   - redirects output to a file (appends)
`|`     - pipe redirects output of one command as input to another
`&&`   - runs a second command if the first one succeeds
`!!`   - last command
`||`   - runs a second command if the first one fails
`2>`   - redirects standard error to a file (overwrites)
`2>>` - redirects standard error to a file (appends)
`&>`   - redirects standard output and error to a file (overwrites)
`&>>`   - redirects standard output and error to a file (appends)

## Shell Keyboard Shortcuts
The goal with these shortcuts is to avoid taking your hands off the home row.
`Ctrl+r` - search through command history
`Ctrl+l` - clears the screen
`Ctrl+a` - moves cursor to the beginning of the line
`Ctrl+e` - moves cursor to the end of the line
`Ctrl+f` - moves cursor one character to the right
`Ctrl+b` - moves cursor one character to the left
`Ctrl+u` - deletes everything before the cursor
`Ctrl+w` - deletes the word before the cursor
`Ctrl+d` - deletes the word after the cursor
`Ctrl+z` - suspends the current process
`fg` to resume process

## Navigating the File System
### pwd
`pwd` displays the current directory 
```
╭─jacob@pop-os ~  
╰─➤ pwd                                                       
/home/jacob
```
### ls
`ls` lists the contents of the current directory
```
╭─jacob@pop-os ~/scratch  
╰─➤  ls
example-folder  example.txt
```

`ls -lath` useful flags to do a long list of all files sorted by time with human readable file sizes
```
╭─jacob@pop-os ~/scratch  
╰─➤  ls -lath
total 12K
drwxr-x---+ 51 jacob jacob 4.0K Aug 20 18:55 ..
drwxrwxr-x   3 jacob jacob 4.0K Aug 20 18:53 .
drwxrwxr-x   2 jacob jacob 4.0K Aug 20 18:53 example-folder
-rw-rw-r--   1 jacob jacob    0 Aug 20 18:52 example.txt
```
### cd
`cd /where/you/want/to/go` command changes the directory
```
╭─jacob@pop-os ~  
╰─➤  cd /
╭─jacob@pop-os /  
╰─➤  pwd
/
```

`cd ~` takes you to right to your home directory
`cd ..` moves you up a directory

## Viewing File System Contents 
### cat
`cat` displays the contents of a file
```
╭─jacob@pop-os ~/scratch  
╰─➤  cat example.txt
This is the text inside the file example.txt
```

### less
`less` is `more`. That is a linux joke you might here old people say. `less` is inspired by the command `more` which let's you look at the contents of files using your keyboard to navigate, but `less` has more features than `more`.
```
less /etc/services
```

`q` quits less
`j` moves down one line
`k` moves up one line

`h` opens the less help menu

Arrow keys work
Page up/down work

### head
`head` prints the first 10 lines of a file by default
```
╭─jacob@pop-os ~/scratch  
╰─➤  head /etc/services 
# Network services, Internet style
#
# Updated from https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml .
#
# New ports will be added on request if they have been officially assigned
# by IANA and used in the real-world or are needed by a debian package.
# If you need a huge list of used numbers please install the nmap package.

tcpmux		1/tcp				# TCP port service multiplexer
echo		7/tcp
```

`head -n [number]` prints the specified number of lines

### tail
`tail` prints the last 10 lines of a file be default
```
╭─jacob@pop-os ~/scratch  
╰─➤  tail /etc/services
sgi-cad		17004/tcp			# Cluster Admin daemon
binkp		24554/tcp			# binkp fidonet protocol
asp		27374/tcp			# Address Search Protocol
asp		27374/udp
csync2		30865/tcp			# cluster synchronization tool
dircproxy	57000/tcp			# Detachable IRC Proxy
tfido		60177/tcp			# fidonet EMSI over telnet
fido		60179/tcp			# fidonet EMSI over TCP

# Local services
```

`tail -n [number]` prints the specified number of lines
`tail -f` displays data appended to file as it grows

### file
`file` determines type of file and provides useful information about the file. `file` doesn't just look at the file extension, but inspects the contents and can reveal some useful information.
```
╭─jacob@pop-os ~/scratch  
╰─➤  file /etc/services 
/etc/services: ASCII text

╭─jacob@pop-os ~/scratch  
╰─➤  file /dev/sda     
/dev/sda: block special (8/0)

╭─jacob@pop-os ~/scratch  
╰─➤  file /usr/bin/pip
/usr/bin/pip: Python script, ASCII text executable

╭─jacob@pop-os ~/scratch  
╰─➤  file ~/Pictures/10-08-23-hiking-mt-biersdat/DSCF0503.JPG 
/home/jacob/Pictures/10-08-23-hiking-mt-biersdat/DSCF0503.JPG: JPEG image data, Exif standard: [TIFF image data, little-endian, direntries=13, manufacturer=FUJIFILM, model=X-T30 II, orientation=upper-left, xresolution=190, yresolution=198, resolutionunit=2, software=Digital Camera X-T30 II Ver1.20, datetime=2023:10:07 08:39:09], baseline, precision 8, 4416x2944, components 3
```

### du

### df

## Directory and File Management
### mkdir

### rmdir

### rm

### touch

### cp

### mv
`mv [file] [destination]` command

## Permissions and Ownership
### chmod

### chown

### chgrp


## Process Management
### ps

## Networking
### ip addr

### tcpdump

### traceroute

### ping

### nslookup

### dig

## Searching and Finding
### find

### locate

### grep

### awk

### sed


## System Information

### Package Management

### Important Log Files

### UFW - Firewall Management

### iptables - Firewall Management

## Intrusion Detection
### fail2ban

## SSH Config

## Vim Bascis

## Running Scripts

## Crontab and Cronjobs