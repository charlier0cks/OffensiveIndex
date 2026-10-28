---
platform: hackthebox
name: touch
os: windows
difficulty: easy
date: 2026-10-25
tags:
  - ctf
  - kiosk
  - kiosk_breakout
  - mysql
  - mysql_udf
  - dll
  - default_password
---
# Touch

![[assets/Pasted image 20261025145434.png]]

> [!info] Box Info
> - **Platform: HackTheBox**
> - **OS: Windows**
> - **Difficulty: Easy**

## Machine Information
---
In the Layover machine, you uncovered a name and a booking confirmation code. Maybe they'll come in handy here? Jenny Crawford/KS7X2M
## Recon
---
### Nmap

``` 
PORT     STATE SERVICE       VERSION
135/tcp  open  msrpc         Microsoft Windows RPC
3389/tcp open  ms-wbt-server Microsoft Terminal Service
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
8443/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Did not follow redirect to /login
|_http-cors: GET POST PUT OPTIONS
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```
## Webapp
#### Login Portal

![[assets/Pasted image 20261025150930.png]]
#### Directory Fuzzing

```
login            [Status: 200, Size: 3572, Words: 122, Lines: 3, Duration: 19ms]
api              [Status: 403, Size: 35, Words: 2, Lines: 1, Duration: 18ms]
	status       [Status: 200, Size: 115, Words: 3, Lines: 1, Duration: 17ms]
	scan         [Status: 405, Size: 30, Words: 3, Lines: 1, Duration: 19ms]

```
## Initial Access
---
In the login portal's source, there is a hint for the password.

![[assets/Pasted image 20261025151040.png]]

From our previous enumeration, the serial number was found in the */api/status* endpoint.

![[assets/Pasted image 20261025153140.png]]

Using the serial number resulted in a successful login. The dashboard reveals a new set of credentials. *KioskUser:K!0sk2026#*

![[assets/Pasted image 20261025153403.png]]

These credentials ended up working via RDP.

![[assets/Pasted image 20261025153907.png]]

![[assets/Pasted image 20261025171224.png]]

Entered correct information and came across a consistent error for the passport scan that fails to match the passport. Back in the dashboard I powered the scanner off and tried again and saw a different result.

![[assets/Pasted image 20261025175729.png]]

Clicking on the URL opened up the Edge browser. I was able to display the contents of a file.

![[assets/Pasted image 20261025180446.png]]

[ikarus23 - Kiosk Mode Breakout](https://github.com/ikarus23/kiosk-mode-breakout)

```
The first step could be to e.g. find a link somewhere in the main application that is clickable. By pressing on it a web browser window might open. The second step is to find a way from that popup/window to critical applications like the cmd.exe. In the example of previously opened edge browser it could be: press the three dots (menu) -> chose "Downloads" -> press the folder symbol (open downloads folder) -> navigate in the file browser to C:\windows\system32 and double click on osk.exe -> then double click on cmd.exe. Now you have command line interface with a full keyboard on a device that might only have a touchscreen.
```

However, I look for powershell at *c:/windows/syswow64/windowspowershell/v1.0/powershell.exe*

![[assets/Pasted image 20261027201831.png]]

## Privilege Escalation
---
Found mysql credentials -> root:"HTB@irw4ys_DB!2026"

![[assets/Pasted image 20261027202111.png]]

![[assets/Pasted image 20261027202408.png]]

Since we are the root user we can execute system commands by installing a malicious user-defined function (UDF) library.

[Privilege Escalation via Library](https://hacktricks.wiki/en/network-services-pentesting/pentesting-mysql.html#privilege-escalation-via-library)

During this process, one issue was that the secure-file-priv was set to NULL, meaning file import and export operations are disabled globally.

![[assets/Pasted image 20261027212011.png]]

I did notice we can write to the my.ini file and set secure_file_priv to *empty*. However, I still would need to restart the MySQL service and I do not have the permissions to do that.

Fortunately, *Authenticated Users* have modify (M) permissions on the plugins folder. I can copy the malicious library into the folder.

![[assets/Pasted image 20261027212301.png]]

With this step satisfied, I can proceed with the last few steps that let us eventually add the *KioskUser* to the administrators local group.

![[assets/Pasted image 20261027212447.png]]