---
title: Wifinetic
published: true
layout: post
---

<br />

---------------
Wifinetic is an easy difficulty Linux machine which presents an intriguing network challenge, focusing on wireless security and network monitoring. An exposed FTP service has anonymous authentication enabled which allows us to download available files. One of the file being an OpenWRT backup which contains Wireless Network configuration that discloses an Access Point password. The contents of shadow or passwd files further disclose usernames on the server. With this information, a password reuse attack can be carried out on the SSH service, allowing us to gain a foothold as the netadmin user. Using standard tools and with the provided wireless interface in monitoring mode, we can brute force the WPS PIN for the Access Point to obtain the pre-shared key ( PSK ). The pass phrase can be reused on SSH service to obtain root access on the server.

<br />

<iframe style="aspect-ratio: 16 / 9; width: 65%; display: block; margin: auto;" src="https://www.youtube.com/embed/h30u5a1IoPY?si=AX78N08r1KvtcMpg" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

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
sudo nmap -sS -sU 10.10.10.247 -p- -T4 --min-rate 5000 -oN all_ports.txt --open -n -Pn -v
```

![All_Ports](/assets/Wifinetic/1.png)
<br />
<br />
<br />

There are **3 open ports**:
+ **21/tcp**
+ **22/tcp**
+ **53/tcp**

<br />
Let's check which **services** are running in these ports:

```bash
sudo nmap -sS 10.10.10.247 -p 21,22,53 -T4 --min-rate 5000 -oX open_ports.xml -oN open_ports.txt --version-all -n -Pn -A -v
```

![Open_Ports](/assets/Wifinetic/2.png)
<br />
<br />
<br />

We can see that the services correspond to:
+ **21/tcp vsftpd 3.0.3**
+ **22/tcp OpenSSH 8.2p1**
+ **53/tcp tcpwrapped**

<br />

Now we will seek for **vulnerabilities**:

```bash
sudo nmap -sS 10.10.10.247 -p 21,22,53 -T4 --min-rate 5000 --script="vuln and safe or intrusive and safe or discovery" -oN vulns.txt -oX vulns.xml -n -Pn -v
```

![Vulns](/assets/Wifinetic/3.png)
<br />
<br />
<br />

Which did not return anything useful.

<br />

------

<br />
### Foothold

I personally loved this machine as I love **wireless pentesting**, although I have a few problems figuring out what to do due to a lack of methodology for discovering Wi-Fi APs on the CLI, it is still a really enjoyable machine.

<br />

The first thing I did is connecting to the **FTP service** and downloading every file, this is possible as **anonymous login** is enabled:

```bash
ftp 10.10.11.247
Name (10.10.11.247:tanades): anonymous
ftp> mget *
```

![Files](/assets/Wifinetic/4.png)
<br />
<br />
<br />

After investigating every file, we discovered a mention to *Reaver*, two emails and the important stuff, after decompressing the **tar file**, we can see a **copy of some files of the */etc/* folder**, among them ***passwd*** and another directory called ***config/***. In the *passwd* file we can discover an user called ***netadmin***:

![passwd](/assets/Wifinetic/5.png)
<br />
<br />
<br />

Meanwhile, in the config folder, there is a file called ***wireless***, which contains a **password**:
![wireless_config](/assets/Wifinetic/6.png)
<br />
<br />
<br />

As this is the only information we have, we can check if the newly discovered user **reuses password**:
```bash
ssh netadmin@10.10.11.247
```

![ssh](/assets/Wifinetic/7.png)
<br />
<br />
<br />

It actually reuses passwords, so we just have to **grab the flag** and continue to privilege escalation:
<br />
![userown](/assets/Wifinetic/8.png)
<br />
<br />
<br />


------

<br />
### Privilege Escalation

This machine privilege escalation is where the **Wi-Fi hacking** thing kicks in.

<br />

While enumerating Linux, we discovered **multiple wireless network interfaces**, one among them is in **monitor mode**:

```bash
iwconfig
```

![iwconfig](/assets/Wifinetic/9.png)
<br />
<br />
<br />

One of this interfaces is also working as an **AP**, providing service with the **SSID *OpenWrt***:

```bash
iw dev
```

![iw_dev](/assets/Wifinetic/10.png)
<br />
<br />
<br />

Continuing our enumeration we discover an **executable used for wireless pentesting that have network capabilities**, Reaver:

```bash
getcap / -r 2>/dev/null
```

![capabilities](/assets/Wifinetic/11.png)
<br />
<br />
<br />

With this data, we can craft our attack, we are going to use reaver to attempt to **crack the WPS pin of the wireless network OpenWrt**, we need to indicate the **interface to use**, which have to be in monitor mode, so we need to use ***mon0***, and the **BSSID of the AP**, which is the **MAC address** of the ***wlan0*** interface:

```bash
reaver -i mon0 -b 02:00:00:00:00:00 
```

![reaver_crack](/assets/Wifinetic/12.png)
<br />
<br />
<br />

We have already got the user flag by attempting to try **password reusing**, let's try it once again:

```bash
su root
```

![root](/assets/Wifinetic/13.png)
<br />
<br />
<br />

Great, let's grab the flag and get out:
<br />
![rootown](/assets/Wifinetic/14.png)
<br />
<br />
<br />
