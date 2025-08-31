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

<style>
.center img {
  display: block;
  margin-left: auto;
  margin-right: auto;
}

.wrap pre {
  white-space: pre-wrap;
}

.gif-container {
    text-align: center;
    margin: 30px 0;
}

.gif-responsive {
    width: 100%;
    max-width: 800px;
    height: 450px;
    border-radius: 12px;
    object-fit: cover;
    box-shadow: 0 10px 25px rgba(0, 0, 0, 0.4);
    transition: transform 0.3s ease, box-shadow 0.3s ease;
}

.gif-responsive:hover {
    transform: scale(1.02);
    box-shadow: 0 15px 35px rgba(0, 0, 0, 0.5);
}

/* Additional video styles */
.video-container {
    text-align: center;
    margin: 30px 0;
}

.video-responsive {
    width: 100%;
    max-width: 800px;
    height: 450px;
    border-radius: 12px;
    object-fit: cover;
    box-shadow: 0 10px 25px rgba(0, 0, 0, 0.4);
    transition: transform 0.3s ease, box-shadow 0.3s ease;
}

.video-responsive:hover {
    transform: scale(1.02);
    box-shadow: 0 15px 35px rgba(0, 0, 0, 0.5);
}

/* Mobile-only responsive styles */
@media (max-width: 768px) {
  .gif-responsive {
    width: 100% !important;
    max-width: 100% !important;
    height: auto !important;
  }
  
  .video-responsive {
    width: 100% !important;
    max-width: 100% !important;
    height: auto !important;
  }
  
  .box-button {
    max-width: 100% !important;
    width: 100% !important;
    padding: 12px 16px !important;
    justify-content: center !important;
    gap: 10px !important;
    position: relative;
  }
  
  /* Hide desktop text */
  .box-button {
    font-size: 0 !important;
  }
  
  /* Show mobile text from data attribute */
  .box-button::after {
    content: attr(data-mobile-text) !important;
    font-size: 14px !important;
    color: #a1a1a1 !important;
    text-align: center !important;
    white-space: nowrap !important;
  }
  
  .box-button img {
    width: 28px !important;
    height: 28px !important;
    margin-right: 0 !important;
  }
}
</style>
<script>
// Function to make only .redirect class links open in new tabs, but not work here actually i don'know why 
document.querySelectorAll('a.redirect').forEach(link => {
    link.setAttribute('target', '_blank');
    link.setAttribute('rel', 'noopener noreferrer');
});
</script>
