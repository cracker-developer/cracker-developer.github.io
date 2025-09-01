---
title: 'TryHackMe: Compiled CTF Walkthrough'
author: 0xcracker
categories: [TryHackMe]
tags: [Binary, Reverse-Engineering, strings, ltrace, ghidra, strcmp, strcmp, C]
render_with_liquid: true
img_path: /images/TryHackMe/Compiled
image:
  path: /images/TryHackMe/Compiled/room_image.webp
---

🧰 Writeup Overview

This writeup explains the reverse-engineering of the provided ELF binary `Compiled.Compiled`, culminating in finding the correct input that triggers the message **Correct!**.

<a href="https://tryhackme.com/room/compiled"
target="_blank"
class="box-button" 
data-mobile-text="Compiled CTF Challenge | TryHackMe"
style="display: flex; align-items: center; background-color: #333; padding: 10px; border-radius: 5px; box-shadow: 2px 2px 10px rgba(0, 0, 0, 0.3); color: #a1a1a1ff; text-decoration: none;">
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Compiled CTF Challenge | TryHackMe
<img src="https://tryhackme.com/r/favicon.png" alt="icon" style="width: 48px; height: 48px; margin-right: 10px;">
</a>

## Initial Enumeration

### File Type Check

```sh
file Compiled.Compiled
```

* **ELF 64-bit LSB pie executable**, dynamically linked (using `ld‑linux‑x86_64.so.2`), **not stripped**: symbol names are available.

---

### Strings Inspection

```sh
strings Compiled.Compiled
```

Key findings:

* `Password: `
* `DoYouEven%sCTF`
* `__dso_handle`
* `_init`
* `Correct!` & `Try again!`

These strongly hint that input is processed via a custom `scanf` format string, with logic comparing two strings.

---

## Runtime Behavior

### Test Basic Execution

```sh
./Compiled.Compiled
# Try random password:
Password: test
 Try again!
```

### Monitor with `ltrace`

```sh
ltrace ./Compiled.Compiled
```

Input:

```
test
```

Output trace:

```c
write("Password: ", 1, 10, 0x7f7af50705c0)           = 10
__isoc99_scanf(0x55d159b6700f, 0x7ffe92dfe6b0, 0, 1024Password: test
) = 0
strcmp("", "__dso_handle")                            = -95
strcmp("", "_init")                                   = -95
printf("Try again!")                                  = 10
Try again!+++ exited (status 0) +++
```

---

## Reverse Engineering the Logic

Disassemble or Use `Ghidra` / IDA

![](/images/TryHackMe/Compiled/import.png)

![](/images/TryHackMe/Compiled/main.png)

Here’s a deep dive explanation of decompiled `pseudo‑C`  logic `main()` function, Let’s break it down line-by-line.

```c
undefined8 main(void)
{
  int iVar1;
  char local_28 [32];  // 32-byte buffer

  fwrite("Password: ", 1, 10, stdout);
  __isoc99_scanf("DoYouEven%sCTF", local_28);

  iVar1 = strcmp(local_28, "__dso_handle");
  if ((-1 < iVar1) && (iVar1 = strcmp(local_28,"__dso_handle"), iVar1 < 1)) {
    printf("Try again!");
    return 0;
  }

  iVar1 = strcmp(local_28, "_init");
  if (iVar1 == 0) {
    printf("Correct!");
  }
  else {
    printf("Try again!");
  }
  return 0;
}

```

🧠 Line-by-Line Breakdown

`char local_28[32];`
- A 32-byte buffer to store the user input.
- Used to store the result of the `scanf`.

---

`fwrite("Password: ", 1, 10, stdout);`
- Displays the prompt: `Password:`
- Equivalent to `printf("Password: ");` but done using <a href="https://www.tutorialspoint.com/c_standard_library/c_function_fwrite.htm" target="_blank">fwrite()</a> in this case.

---

`__isoc99_scanf("DoYouEven%sCTF", local_28);`,
📌 Key Point
- The format string `DoYouEven%sCTF` means:
- It expects the user to input a string that matches this exact pattern.
- For example, if user types DoYouEven`ABC`CTF, then:
The string `"ABC"` will be extracted and stored into `local_28`.
- Only the `%s` part is stored – the surrounding text (`DoYouEven` and `CTF`) is just expected for formatting.

🧪 Example:
```console
Input: DoYouEventestCTF
Then local_28 = "test"
```

Behavior:
- The format string "DoYouEven%sCTF" expects:

- Literal "DoYouEven" at the start.

- A string (%s) in the middle.

- Literal "CTF" at the end.

- Only the %s part is stored in input.

> 🔑 Important: `%s` is wrapped by fixed strings, so **user must input format as string**
{: .prompt-info }

---

`strcmp(local_28, "__dso_handle")`
- Compares user input against `"__dso_handle"`.
- If equal, <a href="https://www.programiz.com/c-programming/library-function/string.h/strcmp" target="_blank">strcmp()</a> returns `0`.

Then we enter this condition:

```c
if ((-1 < iVar1) && (iVar1 = strcmp(local_28,"__dso_handle"), iVar1 < 1))
```
🧠 What's Going On?
- This is a tricky condition.
- It does a double comparison against `"__dso_handle"`, structured like:

```c
if (strcmp(...) >= 0 && strcmp(...) <= 0)
```
- So the only possible value that passes is when:
`strcmp(...) == 0`

If input is `"__dso_handle"` until if true:

- `printf("Try again!")` is shown.
-Function returns early.

🔐 This acts as a blacklist: if you input `"__dso_handle"` → it gets blocked.

---

`strcmp(local_28, "_init")`
- If the above check is bypassed, the code now compares input with "_init".

✅ If local_28 == "_init"
Message: `Correct!`

❌ Else
Message: `Try again!`

✅ Correct Summary
- If input is `"__dso_handle"` → "Try again!" Failure because Blacklisted.
- If input is `"_init"` → "Correct!"
- Anything else → "Try again!"

```
DoYouEven<password>
```

And only `<password>` is stored.

---

## Final Payload & Result

This triggers the "Correct!" message because:

- `scanf` extracts `_init` (between `DoYouEven` and the `end of input`).

- `_init` is not blacklisted and matches the expected string.

- If you `omit CTF`, scanf will still store _init (since `%s` stops at whitespace or end of input) ==> correct, but some binaries may reject due to extra input such this case.


- `CTF` (matches format, but ignored).

`%s` in scanf stops reading when it encounters several cases:
- Whitespace (space, tab, newline).
- End of input (if no more characters are available).

like that 

```bash
./Compiled.Compiled
```
**Enter:**

```
DoYouEven_initCTF
```
**Output:**

```
Try again!
```

---

```bash
./Compiled.Compiled
```

**Enter:**

```
DoYouEven_init
```

**Output:**

```
Correct!
```

```sh
ltrace ./Compiled.Compiled
```

**Enter:**

```
 DoYouEven_init
```

**Output:**
```c
fwrite("Password: ", 1, 10, 0x7f95a0c605c0)           = 10
__isoc99_scanf(0x5601aa8f200f, 0x7ffd41387e90, 0, 1024Password: DoYouEven_init
) = 1
strcmp("_init", "__dso_handle")                       = 10
strcmp("_init", "__dso_handle")                       = 10
strcmp("_init", "_init")                              = 0
printf("Correct!")                                    = 8
Correct!+++ exited (status 0) +++
```

👏 Clear indication that `_init` is accepted after bypassing the blacklist & making `strcmp("_init", "_init") = 0`.

---

<style>
img {
  transition: all 0.3s ease;
}

img:hover {
  transform: scale(1.05);
  filter: brightness(90%);
  box-shadow: 0 15px 35px rgba(0, 0, 0, 0);
}

img:center {
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
    box-shadow: 0 10px 25px rgba(0, 0, 0, 0);
    transition: transform 0.3s ease, box-shadow 0.3s ease;
}

.gif-responsive:hover {
    transform: scale(1.02);
    box-shadow: 0 15px 35px rgba(0, 0, 0, 0.5);
}

/* Additional video styles */
.video-container {
  text-align: center;
  margin: 30px auto;        /* centers inside post */
  max-width: 800px;         /* keeps container from being huge */
}

.video-responsive {
  width: 100%;              /* fill container */
  aspect-ratio: 16 / 9;    /* keeps correct proportions on desktop */
  border-radius: 12px;
  object-fit: cover;
  box-shadow: 0 10px 25px rgba(0.1, 0.1, 0, 0.1);
  transition: transform 0.3s ease, box-shadow 0.3s ease;
}

.video-responsive:hover {
  transform: scale(1.02);
  box-shadow: 0 15px 35px rgba(0, 0, 0, 0.4);
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
    aspect-ratio: auto;  /* let phone use natural aspect ratio */
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
