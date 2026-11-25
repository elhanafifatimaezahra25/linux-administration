# Partie 3 : gestion des fichiers et permissions

Fait partie du projet [Linux Administration](../../tree/master). Cette branche définit les propriétaires et les permissions de l'arborescence de l'application, `/opt/mon-projet`, sur le serveur `srv-fz` (Ubuntu Server 26.04.1 LTS). Elle s'appuie sur les comptes et groupes créés à la [Partie 2](../../tree/partie-2-utilisateurs) : `admin`, `developer`, `operator`, `appuser`, `developers` et `operators`.

## Arborescence

```
/opt/mon-projet/
├── app/       code de l'application
├── config/    configuration et secrets
├── data/      données produites par l'application
├── logs/      journaux de l'application
├── scripts/   scripts d'administration
└── backups/   sauvegardes
```

L'arborescence a été créée en une commande :

```bash
sudo mkdir -p /opt/mon-projet/{app,config,data,logs,scripts,backups}
```

## Politique de permissions

| Dossier | Propriétaire | Groupe | Droits | Raison |
|---|---|---|---|---|
| `/opt/mon-projet` | `root` | `root` | `755` | Tout le monde peut traverser le dossier, seul `root` le modifie |
| `app/` | `root` | `developers` | `2775` | Les développeurs écrivent le code, l'application le lit |
| `config/` | `root` | `appuser` | `750` | Contient les secrets : seule l'application les lit |
| `data/` | `appuser` | `appuser` | `750` | Données de l'application, personne d'autre n'y accède |
| `logs/` | `appuser` | `operators` | `2750` | L'application écrit, les opérateurs lisent |
| `scripts/` | `root` | `operators` | `750` | Scripts d'administration, exécutables par les opérateurs |
| `backups/` | `root` | `root` | `700` | Sauvegardes sensibles, réservées à l'administrateur |

### Lire les droits

Chaque chiffre correspond à une catégorie : **propriétaire**, **groupe**, **autres**. Il additionne trois droits : lecture `r` = 4, écriture `w` = 2, exécution `x` = 1. Pour un dossier, `x` signifie « pouvoir y entrer ».

| Valeur | Symbolique | Signification |
|---|---|---|
| `7` | `rwx` | lire, écrire, entrer |
| `5` | `r-x` | lire et entrer, sans écrire |
| `0` | `---` | aucun accès |

Ainsi, `750` donne tous les droits au propriétaire, la lecture au groupe et rien aux autres.

### Permission spéciale : setgid

Le `2` placé devant `2775` et `2750` est le bit **setgid**. Sur un dossier, il fait **hériter le groupe du dossier** à tout fichier créé dedans, au lieu du groupe principal de son créateur :

- un fichier créé par `developer` dans `app/` appartient au groupe `developers`, et non au groupe `developer` ;
- un journal écrit par `appuser` dans `logs/` appartient au groupe `operators`, qui peut donc le lire.

Dans `ls -l`, le setgid apparaît sous la forme d'un `s` à la place du `x` du groupe.

## 1. Appliquer la politique

```bash
cd /opt/mon-projet

# Racine du projet
sudo chown root:root /opt/mon-projet
sudo chmod 755 /opt/mon-projet

# app : chown pour le propriétaire, chgrp pour le groupe, setgid
sudo chown root app
sudo chgrp developers app
sudo chmod 2775 app

# config : secrets lisibles par l'application seule
sudo chown root:appuser config
sudo chmod 750 config

# data : données de l'application
sudo chown appuser:appuser data
sudo chmod 750 data

# logs : l'application écrit, les opérateurs lisent, setgid
sudo chown appuser:operators logs
sudo chmod 2750 logs

# scripts : administration
sudo chown root:operators scripts
sudo chmod 750 scripts

# backups : réservé à root
sudo chown root:root backups
sudo chmod 700 backups
```

## 2. Vérification des permissions

```bash
ls -l /opt/mon-projet
stat -c '%A  %a  %U:%G  %n' /opt/mon-projet /opt/mon-projet/*
```

```
drwxr-xr-x  755  root:root  /opt/mon-projet
drwxrwsr-x  2775  root:developers  /opt/mon-projet/app
drwx------  700  root:root  /opt/mon-projet/backups
drwxr-x---  750  root:appuser  /opt/mon-projet/config
drwxr-x---  750  appuser:appuser  /opt/mon-projet/data
drwxr-s---  2750  appuser:operators  /opt/mon-projet/logs
drwxr-x---  750  root:operators  /opt/mon-projet/scripts
```

## 3. Tests par rôle

`sudo -u <compte>` exécute une commande à la place d'un autre compte, ce qui permet de tester chaque rôle depuis le compte d'administration.

### Le développeur

```bash
sudo -u developer touch app/test.py
ls -l app/test.py
sudo -u developer ls config
sudo -u developer ls backups
```

```
-rw-r--r-- 1 developer developers 0 Nov 25 10:48 app/test.py
ls: cannot open directory 'config': Permission denied
ls: cannot open directory 'backups': Permission denied
```

Le développeur écrit dans `app/`, et son fichier appartient au groupe `developers` grâce au setgid. Il n'a accès ni aux secrets ni aux sauvegardes.

### L'application

```bash
sudo -u appuser touch logs/app.log
sudo ls -l logs/app.log
sudo -u appuser touch data/test.db
sudo -u appuser touch app/pirate.py
```

```
-rw-r--r-- 1 appuser operators 0 Nov 25 10:48 logs/app.log
touch: cannot touch 'app/pirate.py': Permission denied
```

L'application écrit ses journaux et ses données, mais ne peut pas modifier son propre code. Si elle était compromise, l'attaquant ne pourrait donc pas réécrire le programme.

### L'opérateur

```bash
sudo -u operator cat logs/app.log
sudo -u operator touch logs/pirate.log
```

```
touch: cannot touch 'logs/pirate.log': Permission denied
```

L'opérateur lit les journaux (`cat` réussit, le fichier est vide), mais ne peut ni les modifier ni en créer.

### Nettoyage

```bash
sudo rm -f app/test.py logs/app.log data/test.db
```

## Bilan

| Test | Attendu | Obtenu |
|---|---|---|
| `developer` écrit dans `app/` | Autorisé, groupe `developers` | Conforme |
| `developer` lit `config/` | Refusé | Conforme |
| `developer` lit `backups/` | Refusé | Conforme |
| `appuser` écrit dans `logs/` | Autorisé, groupe `operators` | Conforme |
| `appuser` écrit dans `data/` | Autorisé | Conforme |
| `appuser` écrit dans `app/` | Refusé | Conforme |
| `operator` lit `logs/` | Autorisé | Conforme |
| `operator` écrit dans `logs/` | Refusé | Conforme |

## Remarques

- **umask.** Les fichiers créés reçoivent les droits `rw-r--r--` (`644`), car le umask par défaut d'Ubuntu (`022`) retire l'écriture au groupe. Le groupe `developers` peut donc lire les fichiers des autres développeurs, mais pas les modifier. Pour un vrai travail à plusieurs dans `app/`, il faudrait un umask `002` pour les développeurs, ou des ACL par défaut (`setfacl -d -m g:developers:rwx app`).
- **Le compte d'administration n'entre pas partout sans `sudo`.** `elhanafi` n'appartient pas au groupe `operators` : `ls -l logs/app.log` échoue sans `sudo`. C'est cohérent avec le moindre privilège, l'administrateur passe par `sudo` quand il en a besoin.
- **Sticky bit.** La troisième permission spéciale, le sticky bit, n'est pas utilisée ici. On la voit sur `/tmp` (`drwxrwxrwt`) : chacun peut y créer des fichiers, mais seul leur propriétaire peut les supprimer.

## Environnement

| Élément | Valeur |
|---|---|
| Serveur | `srv-fz`, Ubuntu Server 26.04.1 LTS, noyau 7.0.0-44, arm64 |
| Virtualisation | UTM (QEMU) sur Mac Apple Silicon |
| Adresse | 192.168.64.50 (statique) |

## Auteur

**El Hanafi Fatima Ezahra**, ENSAM Meknès
