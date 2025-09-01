---
title: 'TryHackMe: TakeOver CTF Walkthrough'
author: 0xcracker
categories: [TryHackMe]
tags: [ffuf, Subdomain-Enumeration, DNS, DNS-Enumeration, Subdirectory-Enumeration]
render_with_liquid: true
img_path: /images/TryHackMe/Takeover
image:
  path: /images/TryHackMe/TakeOver/room_image.webp
---

🧰 Writeup Overview

This challenge revolves around `subdomain enumeration`.

<a href="https://tryhackme.com/room/takeover"
target="_blank"
class="box-button" 
data-mobile-text="TakeOver CTF Challenge | TryHackMe"
style="display: flex; align-items: center; background-color: #333; padding: 10px; border-radius: 5px; box-shadow: 2px 2px 10px rgba(0, 0, 0, 0.3); color: #a1a1a1ff; text-decoration: none;">
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;TakeOver CTF Challenge | TryHackMe
<img src="https://tryhackme.com/r/favicon.png" alt="icon" style="width: 48px; height: 48px; margin-right: 10px;">
</a>


## Discovery DNS && Subdomain

Our website is located at `https://futurevera.thm`

> Hint: Don't forget to add the `your-tun0` in `/etc/hosts` for `futurevera.thm`
{: .prompt-tip }

<a href="https://www.computerhope.com/jargon/s/subdirec.htm" target="_blank">Subdirectories</a> discovery
```sh
ffuf -u https://futurevera.thm/FUZZ -w /usr/share/wordlists/dirb/common.txt -fc 403,402,404 -c
```

`-c` for colors

![](/images/TryHackMe/TakeOver/Subdirectory-discovery.png)

> You can use these files located inside `/usr/share/wordlists/seclists/Discovery/DNS/`
{: .prompt-info }

<a href="https://en.wikipedia.org/wiki/Subdomain" target="_blank">Subdomains</a> discovery
```sh
ffuf -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt  \
-u https://10.10.77.46 \ # Make sure to put the IP not the domain here in this case, So as not to cause problems
-H "Host: FUZZ.futurevera.thm" \
-fs 4605 -c
```

![](/images/TryHackMe/TakeOver/Subdomain-disvovery.png)

> When you discover a subdomain, put it in your hosting file `/etc/hosts`, so you can access it on the web.
{: .prompt-warning }

---

## Discover results on the web

![](/images/TryHackMe/TakeOver/1.png)

---

![](/images/TryHackMe/TakeOver/2.png)

---

![](/images/TryHackMe/TakeOver/3.png)

---

![](/images/TryHackMe/TakeOver/4.png)

---

![](/images/TryHackMe/TakeOver/5.png)

---

Here we go, goodbye

<div class="gif-container">
<img src="/gifs/labubu.gif" alt="GIF" class="gif-responsive">
</div>

---


