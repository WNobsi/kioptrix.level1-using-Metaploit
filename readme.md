
# Kioptrix.level1 using Metasploit

Kioptrix Level 1 is a beginner-friendly vulnerable Linux machine originally created for learning penetration testing and exploitation techniques. The goal is to identify exposed services, enumerate potential vulnerabilities, gain initial access to the target system, and ultimately obtain root privileges.

This machine is widely used by aspiring penetration testers to practice a complete attack lifecycle, including reconnaissance, service enumeration, vulnerability assessment, exploitation, and privilege escalation. In this writeup, the compromise is achieved using the Metasploit Framework (in this write-up) to exploit a vulnerable network service and gain administrative access to the target host.

Link: https://www.vulnhub.com/entry/kioptrix-level-1-1,22/


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
## WorkFlow~

### Step 1: Host Discovery

The first step was to identify the IP address of the target machine on the local network. This can be accomplished using tools such as **Netdiscover** or **Nmap**.

#### Option 1: Using Netdiscover

```bash
sudo netdiscover -r 192.168.126.0/24
```

#### Option 2: Using Nmap Ping Sweep

```bash
nmap -sn 192.168.126.0/24
```

#### Results

After scanning the subnet, the target machine was identified with the following IP address:

```text
192.168.126.129
```

This IP address will be used throughout the remainder of the assessment for enumeration and exploitation.

### Why This Step?

Before any enumeration or exploitation can begin, the attacker must identify the target's network address. Host discovery helps locate active systems on the network and confirms that the vulnerable Kioptrix machine is reachable.

