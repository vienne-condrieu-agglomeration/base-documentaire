---
title: Déploiement et configuration d'un serveur PostgreSQL 15 - PostGIS 3.5 sous Rocky Linux 9.5
subtitle: PostgreSQL - Article N°01
authors:
    - Fabien ALLAMANCHE
categories:
    - article
comments: false
date: 2025-01-20
description: PostgreSQL - Déploiement et configuration d'un serveur PostgreSQL 15 - PostGIS 3.5 sous Rocky Linux 9.5
icon: simple/postgresql
license: Licence Ouverte / Open Licence - ETALAB
robots: index, follow
tags:
    - Potgres
    - Postgis
    - SQL
    - Rocky Linux
    - Linux
    - Série
    - Installation
---

# Déploiement et configuration d'un serveur PostgreSQL 15 - PostGIS 3.5 sous Rocky Linux 9.5

:fontawesome-solid-calendar: Date de publication initiale : 20 janvier 2025

## Ressources

### Schéma de partitionnement :

Disque /dev/sda : 25 GiB
Disque /dev/sdb : 160 GiB
Disque /dev/sdc : 32 GiB

```bash
NAME                               LABEL TYPE MOUNTPOINT    FSTYPE       SIZE
sda                                      disk                             32G
├─sda1                                   part /boot         ext2           1G
└─sda2                                   part               LVM2_member   31G
  ├─elephant-vg-root                     lvm  /             ext4          29G
  └─elephant-vg-swap                     lvm  [SWAP]        swap           2G
sdb                                      disk                            160G
├─sdb1
  ├─elephant-vg-data                     lvm  /var/lib/postgresql ext4    29G
sdc                                      disk                             32G
└─sdc1                                   part            LVM2_member      32G
  └─elephant-home-vg                     lvm  /home      ext4             32G
```


## Création de l'utilisateur `postgres`

### Vérification de l'existence du groupe utilisateur `postgres` :

Si on souhaite  créer un groupe avec un ID de groupe (GID) spécifique, utilisez l'option `--gid` ou `-g` :
```bash
[root@elephant ~]#
groupadd -g 26 postgres
```

### Ajout de l'utilisateur `postgres` :

On ajoute ensuite l'utilisateur `postgres` avec les options suivantes :
```bash
[root@elephant ~]#
useradd --uid 26 --gid 26 --shell /bin/bash --create-home /home/postgres --system --comment "PostgreSQL DBA default user" postgres
```

### Définition des variables d'environnement `postgresql` :

```bash
[root@elephant ~]#
vi /etc/profile.d/postgres.sh

export PGDATA=/var/lib/pgsql/15/data
export PATH=$PATH:/usr/pgsql-15/bin
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/pgsql-15/lib
```

## Installation de `PostgreSQL` :

### Installation du dépôt Extra Packages For Entreprise Linux (EPEL) :

```bash
[root@elephant ~]#
dnf install epel-release
```

### Activation du dépôt `CRB` (_Code Ready Builder_) :

```bash
[root@elephant ~]#
/usr/bin/crb enable
```

Ou : 
```bash
[root@elephant ~]#
dnf config-manager --set-enabled crb
```


### Installation du paquet `langpacks-fr` pour les paramètres régionaux :

```bash
[root@elephant ~]#
dnf install langpacks-fr glibc-langpack-fr
```

Si on est embêté par des erreurs SSL, on peut désactiver la vérification SSL :
```bash
[root@elephant ~]#
vi /etc/yum.repos.d/epel.repo

[epel]
name=Extra Packages for Enterprise Linux $releasever - $basearch
metalink=https://mirrors.fedoraproject.org/metalink?repo=epel-$releasever&arch=$basearch&infra=$infra&content=$contentdir
enabled=1
gpgcheck=1
countme=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-$releasever
sslverify=0
```

```bash
[root@elephant ~]#
vi /etc/yum.conf

main]
gpgcheck=1
installonly_limit=3
clean_requirements_on_remove=True
best=True
skip_if_unavailable=False
sslverify=false
```
### Configuration des paramètres régionaux du système :

```bash
[root@elephant ~]#
localectl set-locale fr_FR.UTF-8
```

### Téléchargement et installation du paquet d'installation du dépôt `postgresql` de la communauté :

```bash
[root@elephant ~]#
cd /home/
wget https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm
dnf install -y pgdg-redhat-repo-latest.noarch.rpm
```

ou 

```bash
[root@elephant ~]#
cd /home/
wget --no-check-certificate https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm
dnf install -y pgdg-redhat-repo-latest.noarch.rpm
```
### Désactivation du module `postgresql` intégré de Rocky :

```bash
[root@elephant ~]#
dnf -qy module disable postgresql
```

### Désactivation des différents dépôts de versions de `postgresql` :

```bash
[root@elephant ~]#
dnf config-manager --set-disabled pgdg12 pgdg13 pgdg14 pgdg15 pgdg16 pgdg17

dnf repolist
```

### Activation du dépôt de la version de `postgresql` souhaitée :

```bash
[root@elephant ~]#
dnf -qy module enable pgdg15
dnf config-manager --set-enabled pgdg15
```

### Mise à jour du système :

```bash
[root@elephant ~]#
dnf -y update
```

### Installation des paquets `postgresql` :

```bash
[root@elephant ~]#
dnf -y install \
	postgresql15 \
	postgresql15-contrib \
	postgresql15-devel \
	postgresql15-docs \
	postgresql15-libs \
	postgresql15-plpython3 \
	postgresql15-server \
	postgis35_15 \
	postgis35_15-client \
	postgis35_15-devel \
	postgis35_15-docs \
	postgis35_15-utils \
	pgrouting_15
```

### Initialisation du moteur de base de données :

```bash
[root@elephant ~]#
/usr/pgsql-15/bin/postgresql-15-setup initdb
```

### On s'assure que la BDD a été initialisé :

```bash
[root@elephant ~]#
stat /var/lib/pgsql/15/data/postgresql.conf
```
### Activation et démarrage du service `postgresql` :

```bash
[root@elephant ~]#
systemctl enable --now postgresql-15.service
```

On vérifie que le service a bien démarré :
```bash
[root@elephant ~]#
systemctl status postgresql-15.service
```

### Autorisation dans le pare-feu `firewalld` du protocole `postgresql` :

```bash
[root@elephant ~]#
firewall-cmd --permanent --zone=public --add-service=postgresql
firewall-cmd --reload
```


## Configuration des accès à la BDD :

### Configuration globale du service `posgtresql` :

```bash
[root@elephant ~]#
vi /var/lib/pgsql/15/data/postgresql.conf
```


```bash
#listen_addresses = 'localhost'

listen_addresses = '*'
```
  
### Configuration de l'authentification basée sur l'hôte :

```bash
[root@elephant ~]#
vi /var/lib/pgsql/15/data/pg_hba.conf
```

```bash
# DO NOT DISABLE!
# If you change this first entry you will need to make sure that the
# database superuser can access the database using some other method.
# Noninteractive access to all databases is required during automatic
# maintenance (custom daily cronjobs, replication, and similar tasks).
#
# Database administrative login by Unix domain socket
local   all             postgres                                peer

# TYPE  DATABASE        USER            ADDRESS                 METHOD

# "local" is for Unix domain socket connections only
local   all             all                                     peer
# IPv4 local connections:
host    all             all             127.0.0.1/32                    scram-sha-256
host    all             dyndb           10.10.22.66 255.255.255.192     scram-sha-256
host    all             dyndb           10.10.22.65 255.255.255.192     scram-sha-256
host    all             dyndb           10.10.22.103 255.255.255.192    scram-sha-256
host    all             dyndb           10.10.26.108 255.255.255.192    scram-sha-256
host    vca             aepdb           10.10.22.66 255.255.255.192     scram-sha-256
#host   vca             aepdb           10.10.26.46 255.255.255.0       scram-sha-256
host    vca             aepdb           all                             scram-sha-256
host    vca             dyndb           195.42.149.140/32               scram-sha-256
host    all             all             10.10.22.64/26                  scram-sha-256
# IPv4 VCA VPN IPs
host    all             all             10.10.33.0/24                   scram-sha-256
#DOCKER CONTAINER
host    all             dyndb           172.19.0.0/16                   scram-sha-256
host    vca             dyndb           172.20.0.2/16                   scram-sha-256
host    dolibarr        dyndb           172.23.0.2/16                   scram-sha-256

#GEORCHESTRA
#host     georchestra all         10.10.39.229   255.255.255.224        md5
#host    georchestra dyndb       192.168.254.3   255.255.255.248       md5
#host    georchestra geonetwork  192.168.254.3   255.255.255.248       md5
#host    geonetwork  dyndb       192.168.254.3   255.255.255.248       md5
#host    cadastrapp  dyndb       192.168.254.3   255.255.255.248       md5
#host    viennagglo  dyndb       192.168.254.3   255.255.255.248       md5

#GEOSERVER MONITORING
#host    geoserver_monitoring    dyndb   10.10.22.102    255.255.255.224 md5

#INTRAGEO - PEGASE
host    viennagglo  dyndb       192.168.20.112  255.255.240.0         scram-sha-256

# IPv6 local connections:
host    all             all             ::1/128                 md5

```

## Création du/des rôles  `postgresql` :

```bash
[root@elephant home]# 
su - postgres
Dernière connexion : vendredi 17 janvier 2025 à 16:26:12 UTC sur lxc/tty1
[postgres@elephant ~]$ psql
psql (15.10)
Saisissez « help » pour l'aide.

postgres=# CREATE ROLE dyndb WITH ENCRYPTED PASSWORD 'mypassword';
```