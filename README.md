# Projet : Docker Windows

## Introduction

Aujourd'hui on note que plus de **80% des ordinateurs** détiennent un système d'exploitation type Windows. Cette part de marché est bien évidemment confrontée aux cas d'entreprises. C'est donc pourquoi nous avons choisi de traiter un cas Docker sur un server Windows. 
Au cours de cette documentation seront expliqués : 
*  Comment fonctionne Docker au sein d'un environnement Windows et comment **dockerd** se comporte.
*  Les spécificités de **l'image IIS** fournie par Windows ainsi que la configuration de mise en place avec Docker.
*  La création d'un containeur ayant le rôle de base de données.


# 1. Docker linux et Windows
## 1.1. Généralités
![](https://i.imgur.com/np4UPWQ.png)

**SandBox(Bac à sable) :** Une fois un conteneur démarré, toutes les actions d’écriture, telles que les modifications du système de fichiers, les modifications du Registre ou les installations de logiciels, sont capturées dans cette couche.

*plus de détails sur le fonctionnement des containers [ici](https://azure.microsoft.com/en-us/blog/containers-docker-windows-and-trends/)*


##  1.1.1 Generation Docker

Lorsque l'on lance le **processus de generation docker** (docker build),chaque instruction au sein du dockerfile nécessitant une action est exécutée, l’une après l’autre, dans son propre conteneur temporaire. On obtient donc une nouvelle couche d'images pour chaque instruction nécéssitant une action.
```
# escape=`
FROM windowsservercore
RUN dism /online /enable-feature /all /featurename:iis-webserver /NoRestart
RUN echo "Hello World - Dockerfile" > c:\inetpub\wwwroot\index.html
CMD [ "cmd" ]
```
Par exemple, sur un dockerfile comme ci-dessus,
on utilise l'OS de base windowsservercore, IIS est installé, puis un site web simple est créé.
On aura donc plusieurs couches et chacune dépandant de la précédante. On peut utiliser **docker history** pour voir le détail. L’image se compose de **quatre couches**: la base, et une pour chaque instruction.
```
PS V:\IIS> docker history iis

IMAGE               CREATED              CREATED BY                                      SIZE                COMMENT
f4caf476e909        16 seconds ago       cmd /S /C REM (nop) CMD ["cmd"]                 41.84 kB
f0e017e5b088        21 seconds ago       cmd /S /C echo "Hello World - Dockerfile" > c   6.816 MB
88438e174b7c        About a minute ago   cmd /S /C dism /online /enable-feature /all /   162.7 MB
6801d964fda5        4 months ago                                                         0 B
```
Pour optimiser cela on pourrait tout simplement regrouper les commandes d'instruction pour qu'une couche corresponde à plusieurs instructions.

## 1.2. Conteneurs
Sur Windows et sur Linux, Docker ne se compote pas tout à fait de la même manière. En effet, sur Windows on retrouve différents **types de conteneurs**.
* **Les conteneurs Windows Server** : ces conteneurs assurent l'isolation entre les applications grâce aux namespaces et à une technologie d'isolation de ressources processeur. Le conteneur Windows partage le même noyau que son hôte ce qui veut dire qu'il faut faire attention au code que l'on récupère. Du fait que le conteneur partage le même noyau que son hôte, il est donc nécéssaire que ces deux derniers aient la même version de noyau.
* **L'isolation Hyper-V** fonctionnalité Windows qui permet de lancer une machine virtuelle la plus minimale possible afin d'y lancer le conteneur. Dans ce cas, le noyau de l'hôte n'est pas partagé avec d'autres conteneurs exécutés sur l'hôte.

Ce sont deux fonctionnalitées que l'on peut changer au moment de l'exécution. Si notre conteneur est un conteneur Windows Server, on pourra toujours l'éxécuter avec l'isolation Hyper-V et vice-versa.

https://docs.microsoft.com/fr-fr/virtualization/windowscontainers/about/index

## 1.3. Fonctionnement de Docker Linux et Windows
### 1.3.1. Installation

**Windows** : Lors de l'installation Docker crée une VM Linux qui s'appelle **MobyLinuxVM** qui est basée sur l'OS **Alpine**. Ensuite L'application Docker se connecte à cette machine afin de pouvoir créer nos conteneurs avec le matériel nécéssaire à son bon fonctionnement. Durant l'installation est aussi configuré un **sous-réseau** pour cette VM afin que nos conteneurs puissent communiquer avec les réseaux NAT et local.

**Linux** : En ce qui concerne l'installation de docker sous Linux, on doit seulement Docker Engine (le moteur) et management tools. On n'a pas besoin de créer une machine virtuelle ou un réseau virtuel car les conteners s'occuperont automatiquement de la configuration.

### 1.3.2. Commandes Docker

Sur linux et Windows, on retrouve exactement les mêmes commandes. La seule différence entre les deux OS à ce niveau la se passe lors de l'éxecution.

**Windows** : Impossible de lancer les commandes Docker autrement qu'avec Powershell.

**Linux** : On peut utiliser n'importe quel émulateur de terminal pour lancer les commandes

### 1.3.3. Dockerfile
**Windows** : le fichier de configuration se trouve dans **C:\ProgramData\Docker\config\daemon.json**
* Toutes les options de configuration de Docker disponibles ne s’appliquent pas forcément à Docker sur Windows.
* Ce fichier n'existe pas forcément et il peut être créé.
* La valeur par défaut de stockage des images est **c:\programdata\docker** elle correspond à la variable data-root

**Linux** : le chemin du fichier de conf est **/etc/docker/daemon.json** 

Dans les deux cas, on peut modifier ces fichiers avec la commande **sc config**. C'est donc seulement l'emplacement du fichier de configuration qui altère.
### 1.3.4 Stockage des conteneurs

**Windows et Linux**

Pour partager des donées persistantes, Docker a mis en place le **DVC** (Data Volume Container) pour partager des données entre container. Ce mécanisme consiste à créer un conteneur nommé puis à lui **attacher des volumes**. Il sera ensuite possible de consommer ces volumes depuis d’autres conteneurs.
***
**Windows**
Dans une installation par défaut, les niveaux sont stockés dans **C:\ProgramData\docker** et répartis dans les répertoires «image» et «windowsfilter». grâce à la configuration data-root (daemon.json), on peut changer ces emplacements.

Remarque, **ReFS** (système de fichier propriétaire Micosoft) n'est pas pris en charge, seul **NTFS** est pris en charge pour le stockage de niveaux.
***
**Windows et Linux**

Les **montages liés** permettent à un conteneur de partager un répertoire avec l’hôte et partager ce volume avec plusieurs conteneurs. Un **volume nommé** ou un montage **SMB** (Server Message Block) permettent au conteneur de s'exécuter sur plusieurs machines avec un accès aux même fichiers.
### 1.3.5. Securité

**Windows** : Sur Windows on a une sécurité en plus qui est l'isolation Hyper-V. C'est à dire que même si un attaquand est en mesure de compromettre le noyau, il ne pourra pas sortir du conteneur et en toucher d'autres ou encore atteindre l'hôte. le compte prédéfini est **LocalSystem**. On peut en plus définir des droits en fonction des containers qui permettrons la lecture et l'écriture en fonction du niveau de fiabilité.

Les conteneurs Windows Server qui ont recours à l'isolement des processus sont différents. Ils utilisent **l'identité des processus** au sein du conteneur pour accéder aux données. Il y a donc une **ACL** mise en place. L'identité de processus en cours d'execution dans le conteneur sera utilisée pour accéder aux fichiers et répertoires du volume monté au lieu de LocalSystem. l'accès doit donc être accordé pour l'exploitataion des données.

### 1.3.6 A savoir

Si l'on travaille sur Hyper-V, par sécurité, on ne peut pas ajouter une autre machine virtuelle à l'interieur. Il faudra donc lancer cette procédure afin de ne plus avoir de problèmes :
```
PS C:\>Set-VMProcessor -VMName WS2016 -ComputerName Nano -  ExposeVirtualizationExtensions $true
```
puis autoriser sur notre machine hôte le **spoofing d'adresses MAC** de sorte à ce que notre machine virtuelle utilise une autre adresse MAC pour ses VM ou conteneurs.
Cette petite incompatibilité peut être corrigée dans les **paramètres de notre VM**.
### 1.3.7 Liens symboliques
**Linux et Windows** (à vérifier)
Les **liens symboliques** sont résolus dans le conteneur. Si l'on doit lier un chemin d’accès hôte vers un conteneur et que ce chemin est un lien symbolique ou contient des liens symboliques, le conteneur ne sera pas en mesure d’y accéder.
***
 
### 1.3.7. Conclusion

Quel que soit l'endroit où Docker s'execute, l'experience utilisateur de Docker est toujours la même (bien qu'il y ait quelques différences de configurations et processus arrière plan). Il est peut-être plus agréable d'utiliser Docker sur linux sachant que l'on peut utiliser tous les shells et qu'il soit un peu moins long lors de l'installation. Cependant les deux OS sont sensiblements pareils d'un point de vue utilisateur.

# 2. IIS et Docker

IIS (internet information services) est un server web proposé par Windows existant aussi en tant qu'image Docker. Il est flexible fiable.
Des images sont déjà proposées pour utiliser IIS

https://blog.alexellis.io/run-iis-asp-net-on-windows-10-with-docker/

On trouvera plus d'information sur cette partie en suivant, lors de la mise en pratique et l'installation de IIS.

# 3. Oracle database et Docker

A compléter

# 4. Mise en pratique

## 4.1. Installation de Docker sur Windows 

*En ce qui concerne les prérequis il est nécéssaire pour suivre cette documentation d'avoir au minimum un Windows server 2016.*

Pour travailler dans un environnement propre, nous allons donc rajouter un disque sur notre serveur spécialement dédié a la contenerisation, il sera appelé **V:\\**

Tout d'abord il faut installer la fonctionnalité containers, pour cela on utilisera la commande powershell suivante :

```
V:\> install-windowsfeature containers
```
Puis on redémarre notre serveur.

Afin de partir sur une base stable, il faut penser à télécharger la bonne version de docker. Sur Windows elle peut être trouvée **[ici](https://store.docker.com/editions/community/docker-ce-desktop-windows).**
Ce lien propose aussi une brève présentation des fonctionnalités sur Windows ainsi que sa configuration.

Après avoir suivi le wizard d'installation de docker on ajoute le rôle Hyper-V, en effet celui ci est nécessaire pour la création de containers.


## 4.2. Installation et configuration IIS 

On va donc créer sur notre nouveau disque **V:\\** un dossier correspondant au server **IIS**, ensuite nous allons créer un fichier qui s'appelle **Dockerfile**.
On aura donc **V:\IIS\Dockerfile** et c'est au sein de ce fichier que nous allons ecrire les lignes suivantes correspondant à l'implémentation de notre image IIS : 
```
# escape=`
FROM microsoft/windowsservercore:ltsc2016

RUN powershell -Command `
    Add-WindowsFeature Web-Server; `
    Invoke-WebRequest -UseBasicParsing -Uri "https://dotnetbinaries.blob.core.windows.net/servicemonitor/2.0.1.6/ServiceMonitor.exe" -OutFile "C:\ServiceMonitor.exe"

EXPOSE 80

ENTRYPOINT ["C:\\ServiceMonitor.exe", "w3svc"]
```
Lorsque l'on va faire un **docker build .** on aura une image Windows server core 2016 qui va être importée avec le rôle **IIS** (WebServer) implémenté. 
**Attention**, la version de noyau de l'image doit correspondre à la version de noyau de son hôte sinon on aura un incompatibilité et cela ne va pas fonctionner.

```
PS V:\IIS> docker build -t iis-site .
PS V:\IIS> docker run -d -p 8080:80 --name my-running-site iis-site
```
On peut dès lors vérifier sur notre navigateur si on peut utiliser http://localhost pour confirmer le bon fonctionnement de notre  conteneur.
Il est aussi possible de se connecter grâce à l'IP de notre container.
```
docker inspect -f "{{ .NetworkSettings.Networks.nat.IPAddress }}" my-running-site
```

## 4.3. Installation et configuration BDD
Des images docker de base de données sont déjà disponibles sur Windows. Sur le Docker Store on peut retrouver différentes images correspondantes, nous allons utiliser une image qui contiendra la version **Oracle Database Server 12.2.0.1 Entreprise Edition** sur **Oracle Linux 7**.

### 4.3.1 Prérequis

Nous allons ici utiliser le référentiel **GIT** pour plusieurs raisons :
* Dans un premier temps, il **n'inclut pas les fichiers binaires d'installation**. Cela faciliterait donc la gestion des conteneurs si il faut adapter certains scripts à d'autres versions que oracle 12c.
* Il n'inclut pas de script pour l'installation de Windows on va donc exécuter les étapes de création du conteneur manuellement. Grâce à ce type d'installation, on verra donc ce qui est nécéssaire pour que cette solution fonctionne sur windows.

On va tout d'abord installer le bon référentiel qui est [Docker-images](https://github.com/oracle/docker-images) que l'on peut télécharger directement (ZIP) ou que l'on peut récuperer avec la commande suivante : 
```
V:\>git clone https://github.com/oracle/docker-image.git 
```

Il faudra ensuite utiliser les fichiers d'installation Oracle que l'on peut télécharger [ici](http://www.oracle.com/technetwork/database/enterprise-edition/downloads/index.html)

Les fichiers téléchargés doivent rester dans le répertoire **V:\GitHub\docker-images\OracleDatabase\dockerfiles\12.1.0.2**

```
V:\GitHub\docker-images\OracleDatabase\dockerfiles\12.1.0.2>dir
```

### 4.3.2 Création du conteneur

Lors de la création du conteneur il faut veiller à utiliser **les scripts d'installation de la version choisie**. Dans ce cas, Oracle 12.1.0.2 Entreprise Edition. Elle implique directement les fichiers de configuration et les scripts du répertoire **docker-images** préalablement téléchargé.

```
V:\GitHub\docker-images\OracleDatabase\dockerfiles\12.1.0.2>docker build -t 
oracle/database:12.1.0.2-ee -f Dockerfile.ee .
```
Durant le build, différents fichiers binaires sont éxécutés comme l'installation de l'image **Linux 7-slim** ainsi que les **mises à jour des paquets Linux**.
On peut voir directement le conteneur qui a été créé : 
```
V:\GitHub\docker-images\OracleDatabase\dockerfiles\12.1.0.2>docker image ls
    REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
    oracle/database     12.1.0.2-ee         c13aefb41772        4 minutes ago       10.6GB
    oraclelinux         7-slim              987003ebb1d5        2 months ago        118MB
```

### 4.3.3 Création de la base de données.
Jusqu'à présent, aucune base de données n'a été créée.

**Attention :** le contenu d'un conteneur est immuable, c'est à dire qu'à chaque redémarrage les données modifiées sont suprimées. Pour les conserver, il faut ajouter un paramètre lors de l'execution du conteneur qui va mapper le répertoire de données interne à celui qui existe dans notre hôte qui est **-v**.

Élément à prendre en compte dans Windows : cette correspondance de répertoire a une **syntaxe différente** de celle utilisée sous Linux et il est **uniquement autorisé à utiliser des répertoires situés sur le même disque** que celui où Docker a été installé (ici V:\).

La commande suivante va lancer l'image **ora121** et on pourra utiliser les ports **1521**, **5500** et **8080** pour comuniquer avec. De plus il faut veiller à ce que la base de données utilise **au moins un même port** que notre serveur IIS pour que l'on puisse établir une communication. On défini ensuite les variables d'environnement propres à oracle et comme dit précédemment -v pour montrer le répertoire de travail dans le conteneur.
```
docker run --name ora121  -p 1521:1521 -p 5500:5500 -p 8080:80 -e ORACLE_SID=orcl -e ORACLE_PDB=pdb1 -e 
ORACLE_PWD=Oracle_123  -e ORACLE_CHARACTERSET=AL32UTF8 -v  
//v/users/calero/.docker/persistentdisk/ora121://opt/oracle/oradata  oracle/database:12.1.0.2-ee
```
Si on rencontre des erreurs, il peut être utile de vérifier comment le mappage a été configuré dans notre conteneur grâce à la commande suivante : 
```
docker inspect oracle/database:12.1.0.2-ee
```
Elle va nous permettre de voir la configuration de notre conteneur, l'entrée "Volumes" doit affichier deux chemins si ce n'est pas le cas, le mappage n'a pas été fait.

### 4.3.4. Connexion à la BDD
Pour ce faire on va vérifier la configuration réseau de notre image pour pouvoir s'y connecter.
```
V:\Users\toto>docker-machine ip
192.168.99.100
```
On va donc se connecter via la bonne ip et le bon port
```
V:\Users\toto>sqlplus system/Oracle_123@\"192.168.99.100:1521/orcl\"
```
Sinon on peut se connecter d'une autre manière à savoir, utiliser le binaire sqlplus à l'interieur de Docker directement.
```
V:\Users\toto>docker ps
    CONTAINER ID        IMAGE                         COMMAND                  CREATED 
    2c1afcad6a50        oracle/database:12.1.0.2-ee   "/bin/sh -c 'exec $O…"   11 hours ago        

    STATUS                     PORTS                                                                    NAMES
    Up 11 hours (healthy)      0.0.0.0:1521->1521/tcp, 0.0.0.0:5500->5500/tcp, 0.0.0.0:8080->80/tcp    ora121

    V:\Users\toto>docker exec -ti 2c1afcad6a50 sqlplus pdbadmin/Oracle_123@pdb1

    SQL*Plus: Release 12.1.0.2.0 Production on Sun Feb 4 16:23:47 2018

    Copyright (c) 1982, 2014, Oracle.  All rights reserved.


    Connected to:
    Oracle Database 12c Enterprise Edition Release 12.1.0.2.0 - 64bit Production
    With the Partitioning, OLAP, Advanced Analytics and Real Application Testing options
    
SQL>
```
Dès lors, on pourra passer nos requêtes SQL.


## 5. Améliorations possibles

* Docker compose
* Lier les deux conteneur
* Créer un conteneur qui incluera le serveur et la database
* Ajouter un framework web style flask pour monter une archi avec interface graphique dans un autre conteneur
* Plus de sécu (ports, règles de sécu, users,firewall)
* ajouter annuaire LDAP connecté a la BDD 




A placer :+1: 
*identité du processus en cours d'exécution dans le conteneur «ContainerAdministrator» sur WindowsServerCore et «ContainerUser» dans les conteneurs Nano Server*
