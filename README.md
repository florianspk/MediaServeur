# Debian 10 - Installation d’une seebox anonyme et sécurisée avec Deluge, Jackett, Sonarr, Radarr, Lidarr, Docker, Traefik et Jellyfin

Date: 22/05/2022
Date Created: 22 mai 2022 15:01

<aside>
☝ Je vais vous parlez ici de la procédure pour installer une seedbox automatisée, anonyme et sécurisée sur Debian 10 avec l’utilisation de Docker et OpenVPN.

</aside>

# Les outils

> A quoi sert ce tutoriel et cette liste d’outils ?
>

> Ces derniers permet d’automatiser le téléchargement des films, séries TV et albums de musique dès leur disponibilité pour ensuite les streamer sur votre ordinateur, smartphone ou encore votre smartTV, alors cet article est fait pour vous ! Côté sécurité, vous serez totalement anonyme et protégé !
>

### Docker

Votre seedbox sera déployée en quelques minutes ! Docker permet en quelques commandes d’installer et d’exécuter tous les outils dont vous aurez besoin. Chaque outil est isolé et possède son propre container Docker. Chaque container contient toutes les dépendances (librairies, démons, configurations, etc.) nécessaires à son exécution sans interférer avec les autres outils ou d’autres services installés sur votre serveur. Enfin les mises à jour des outils sont simplifiées et peuvent être totalement automatisées. Les images Docker sont versionnées et permettent de redéployer une version précédente très facilement.

### Deluge

Cette outils vas vous permettre de télécharger des films, séries TV et fichiers audio directement depuis des fichiers .torrent. **Deluge** est donc un client BitTorrent multiplateforme libre basé sur libtorrent. Il est réputé pour sa stabilité, sa vitesse et son côté poids plume et dispose d’une interface claire et intuitive. Enfin il s’avère extrêmement modulable grâce à la possibilité de lui ajouter de nombreux plugins.

![Untitled](https://github.com/florianspk/MediaServeur/blob/5641ac801828db4be254f5adbb8e7b14b8ec32a7/media/img-readme/Untitled.png)

### Jackett

Cette outil permet de combler ce manque et prend en charge plus d’une centaine de trackers. De nombreux trackers Français (YGGtorrent) sont supportés ainsi que des trackers privés et semi-privés nécessitant un compte. **Jackett** fonctionne comme un serveur proxy, lorsque vous effectuez une recherche via Sonarr ou Radarr, celui-ci transforme et transmet la requête au tracker, analyse la réponse puis renvoie les résultats à l’application émettrice. Jackett prend aussi en charge les flux RSS.

![Untitled](https://github.com/florianspk/MediaServeur/blob/5641ac801828db4be254f5adbb8e7b14b8ec32a7/media/img-readme/Untitled%201.png)

### Sonarr

Ce dernier permet de rechercher vos fichiers *.torrent* et d’automatiser le téléchargement de vos séries préférées. Vous ajoutez une série dans l’interface en précisant la qualité et la langue souhaitées et Sonarr recherchera celle-ci via les indexers configurés. Enfin Sonarr ajoutera automatiquement dans Deluge la série en téléchargement. Parmi les nombreuses fonctionnalités de Sonarr, celui-ci dispose de tâches automatiques et quotidiennes ajoutant automatiquement un épisode fraichement sorti. L’interface dispose aussi d’un calendrier répertoriant les sorties des prochains épisodes.

![https://howto.wared.fr/wp-content/uploads/2020/04/sonarr_screenshot.png](https://howto.wared.fr/wp-content/uploads/2020/04/sonarr_screenshot.png)

**Radarr** et **Lidarr** ont un fonctionnement similaire à Sonarr mais respectivement pour les films et les fichiers audio.

### Treafik

Traefik est un routeur Edge open-source qui fait de la publication de vos services une expérience amusante et facile. Il reçoit les demandes au nom de votre système et trouve les composants qui sont responsables de leur traitement.

![Untitled](https://github.com/florianspk/MediaServeur/blob/5641ac801828db4be254f5adbb8e7b14b8ec32a7/media/img-readme/Untitled%202.png)

### Jellyfin

Ce dernier est un serveurmultimedia. Il s'agit d'un fork de Emby (anciennement Media Browser)
devenu officiellement propriétaire en 2018. Il permet de mettre sa médiathèque à disposition sur le web, qu'il s'agisse de contenu vidéo (films et séries, télévision), audio, ou d'images.

![Untitled](https://github.com/florianspk/MediaServeur/blob/5641ac801828db4be254f5adbb8e7b14b8ec32a7/media/img-readme/Untitled%203.png)

<aside>
☝ Nous allons donc mettre tous ces outils en place pour avoir notre propre serveur de film, série et musique

</aside>

---

# Prérequis

- Votre utilisateur doit avoir accès à *sudo*.
- Les paquets *`curl`* et *`software-properties-common*` doivent être installés sur votre système. Dans le doute, tapez la commande suivante :

```bash
sudo apt-get install -y curl software-properties-common
```

- Vos dépôts APT doivent être à jour. Dans le doute, tapez la commande suivante :

```bash
sudo apt-get update
```

---

# I**nstallation de Docker & Docker Compose**

Installez *Docker* :

```bash
sudo apt-get install -y docker.io
```

- Vérifiez que *Docker* est correctement installé avec la commande :

```bash
❯ docker -v
Docker version 20.10.16, build aa7e414
```

La commande doit retourner la version installée de *Docker*.

Téléchargez *Docker Compose* avec la commande suivante en modifiant la version si besoin avec la dernière release du **[repository officiel de Docker](https://github.com/docker/compose/releases/latest)** :

```bash
sudo curl -L https://github.com/docker/compose/releases/download/1.27.4/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
```

Ajoutez les droits d'exécution sur le binaire de Docker Compose :

```bash
sudo chmod +x /usr/local/bin/docker-compose
```

Vérifiez l'installation de *Docker Compose* avec la commande :

```bash
docker-compose -v
docker-compose version 1.27.4, build 8a1c60f6
```

Si l'installation s'est correctement effectuée, cette commande doit vous renvoyer la version de

*Docker Compose*

# La mise en place de traefik

Traefik est un reverse-proxy et un load-balancer moderne conçu (par un Français) pour faciliter le déploiement des microservices (Docker, Kubernetes, AWS, etc.). Traefik est extrêmement simple à configurer et gère automatiquement vos certificats délivrés par Let’s Encrypt. De plus, il est capable de charger vos containers dynamiquement sans interruption de service et dispose d’un dashboard affichant l’ensemble de vos routes configurées.

> Un reverse proxy est le récepteur frontal de toutes les requêtes HTTP venant de l’extérieur. Celui-ci les redirigera vers les bonnes instances des conteneurs Docker. Dans notre cas, nous disposons d’une multitude de containers Docker et nous ne pouvons pas toutes les faire écouter sur le port 80 ou 443, d’où l’utilisation d’un reverse proxy. Ajoutez à cela un joli nom de domaine à votre seedbox en sécurisant le tout avec le protocole SSL/TLS et un certificat Let’s Encrypt.
>

![https://howto.wared.fr/wp-content/uploads/2020/05/traefik_docker.png](https://howto.wared.fr/wp-content/uploads/2020/05/traefik_docker.png)

### Création de l’utilisateur *traefik*

Si vous avez plusieurs *docker-compose.yml* ou si vous avez pour projet d’en créer par la suite, il est préférable d’isoler et de dédier un *docker-compose.yml* à Traefik. Il est recommandé de ne pas le lancer sous votre super-utilisateur et de créer un utilisateur dédié.

- Ajoutez l’utilisateur traefik :

```bash
sudo adduser traefik
```

- Ajoutez-le au groupe *docker* :

```bash
sudo adduser traefik docker
```

- Attribuer le dossier treafik a l’utilisateur traefik  :

```bash
sudo chown -r treafik:traefik /home/traefik
```

### Le réseau Traefik

Dans la première partie de ce tutoriel, chaque container Docker partage le réseau de votre machine pour être accessible en local avec la propriété `network_mode: "host"`. Avec un reverse proxy, cette configuration n’a plus lieu d’être et nous allons en profiter pour ajouter un peu plus de sécurité en isolant vos containers dans un réseau uniquement accessible par Traefik et vos  containers.

Connectez-vous sous l’utilisateur *traefik* :

```bash
su traefik
```

- Déplacez-vous dans le répertoire personnel de cet utilisateur :

```bash
cd
```

- Créez le réseau dédié à Traefik et à l’ensemble de vos containers :

```bash
docker network create traefik_network
```

### Explication de L’image Traefik

```yaml
version: '3.7'
services:
  traefik:
    image: "traefik:v2.2"
    container_name: "traefik"
    restart: unless-stopped
    networks:
      - traefik_network
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /home/traefik/traefik.toml:/traefik.toml:ro
      - /home/traefik/traefik_dynamic.toml:/traefik_dynamic.toml
      - /home/traefik/acme.json:/acme.json
      - /home/traefik/log/access.log:/var/log/access.log

networks:
  traefik_network:
    external: true
```

- **/var/run/docker.sock:/var/run/docker.sock:ro** : précise le socket Unix de votre docker. Laissez cette valeur par défaut.
- **/home/traefik/traefik.toml:/traefik.toml:ro** : précise le chemin du fichier de configuration de Traefik. Nous le créerons à l’étape suivante.
- **/home/traefik/acme.json:/acme.json** : précise le chemin du fichier contenant les informations relatives à vos certificats.
- **traefik_network** : ici nous spécifions le réseau créé
  précédemment et dédié à Traefik. Nous ajouterons les différents
  containers dans ce réseau par la suite.

### Configuration Traefik

Traefik peut être configuré de différentes manières, nous allons
détailler ici, la configuration recommandée par Traefik, à savoir par un
fichier statique au format TOML.

1. Traefik stocke les informations liées au certificat dans un fichier **acme.json**. Vous devez au préalable lui attribuer un minimum de permissions :

```bash
chmod 600 /home/traefik/acme.json
```

- Créez le fichier de configuration Traefik ***/home/traefik/traefik.toml*** en ajoutant les lignes suivantes. L’unique propriété à modifier est l’**email** :

    ```
    debug = true
    logLevel = "DEBUG"
    
    [providers]
        [providers.file]
            # Dynamic files
            filename = "traefik_dynamic.toml"
            watch = true
        [providers.docker]
            endpoint = "unix:///var/run/docker.sock"
            exposedByDefault = false
            watch = true
    
    [entryPoints]
        [entryPoints.web]
            address = ":80"
    
        [entryPoints.web.http]
            [entryPoints.web.http.redirections]
                [entryPoints.web.http.redirections.entryPoint]
                    to = "websecure"
                    scheme = "https"
    
        [entryPoints.websecure]
            address = ":443"
    
    [certificatesResolvers.leresolver.acme]
      email = "email@gmail.com"
      storage = "acme.json"
      [certificatesResolvers.leresolver.acme.httpChallenge]
        entryPoint = "web"
    
    [api]
      dashboard = true
    
    [global]
      sendAnonymousUsage = false
      checkNewVersion = false
    
    [accessLog]
      filePath = "/var/log/access.log"
      format = "json"
    ```


**[providers.docker]** : cette directive permet tout simplement d’activer le support Docker.
**endpoint = « unix:///var/run/docker.sock »** : précise le socket Unix de votre Docker à Traefik. Laissez la valeur par défaut.

**watch = true** : permet de déployer à chaud les containers dès qu’un changement dans la configuration est détectée.

**exposedByDefault = false** : n’expose pas par défaut les containers au monde extérieur. Il est préférable d’activer cette option dans le *docker-compose.yml* pour chacun des containers dans le cas où, si pour une raison ou une autre, vous ne souhaiteriez plus rendre accessible un container de l’extérieur.

**[entryPoints.web]** et **[entryPoints.websecure]** : ces directives définissent les points d’entrée de votre Traefik. **web** et **websecure** sont purement indicatifs, vous pouvez les nommer autrement mais ils doivent correspondre aux labels que nous définirons plus tard dans le *docker-compose.yml*. Les propriétés **adress** précisent les ports sur lesquels Traefik « écoute » en fonction du point d’entrée.

**[entryPoints.web.http.redirections.entryPoint]** : force la redirection du port 80 vers le port 443 donc force le HTTP en HTTPS.

**[certificatesResolvers.[*NOM_RESOLVEUR*].acme]** : nous indiquons ici que nous souhaitons utiliser le protocole ACME (donc le service Let’s Encrypt) pour obtenir un certificat. Le nom du
résolveur est purement indicatif, il doit simplement correspondre aux labels que nous définirons plus tard dans le *docker-compose.yml*.
**storage = « acme.json »** : la propriété **storage** précise à Traefik le fichier (que nous avons créé précédemment) où seront stockées les informations relatives aux certificats. L’erreur commune est d’indiquer le chemin absolu (dans notre cas */home/traefik/acme.json*), ici le chemin à renseigner est celui au sein du container Traefik, il faut donc laisser cette valeur par défaut.

**[certificatesResolvers.[*NOM_RESOLVEUR*].acme.httpChallenge]** : cette directive précise tout simplement par quel point d’entrée (dans notre cas le port 80) Traefik peut obtenir un certificat.

- Déplacez-vous dans le répertoire personnel de l’utilisateur *traefik* et démarrez le container :

```bash
cd
docker-compose up -d
```

<aside>
✅ Traefik devrait se lancer sans aucun probleme

</aside>

# Droits Unix et arborescence

Nous allons maintenant nous occuper des outils multimedia.

Il est recommandé, pour des raisons de sécurité, de créer un  utilisateur dédié à la gestion des volumes Docker (Deluge, Jackett, Sonarr, Radarr et Lidarr) et de ne pas les lancer sous votre
super-utilisateur.

1. Créez un utilisateur ***media*** :

```bash
sudo adduser media
```

- Ajoutez-le au groupe *docker* :

```bash
sudo adduser media docker
```

L’arborescence utilisée sur le filesystem sera la suivante :

```
data
├── movies
├── music
├── torrents
└── tv
```

Le choix de spécifier un répertoire par type de média sera très utile pour la gestion des librairies au sein de Jellyfin.

Un fichier téléchargé via Deluge restera dans le dossier ***/data/torrents*** tant qu’il sera en cours de téléchargement. Une fois terminé, celui-ci  sera automatiquement déplacé par Deluge dans le répertoire ***/data/movies***, ***/data/tv*** ou ***/data/music*** selon si le téléchargement a été ajouté par Radarr, Sonarr ou Lidarr.
Nous verrons plus tard comment configurer automatiquement le déplacement d’un média dans le bon répertoire.

1. Créez l’arboresence dédiée aux téléchargements :

```bash
sudo mkdir -p /data/torrents /data/movies /data/music /data/tv
```

- Attribuez ces répertoires à l’utilisateur ***media*** pour éviter de futurs problèmes de permissions :

    ```bash
    sudo chown media:media /data/torrents /data/movies /data/music /data/tv
    ```


# Configuration avec OpenVPN

Connectez-vous sous l’utilisateur *media* :

```bash
su media
```

Déplacez-vous dans le répertoire personnel de cet utilisateur :

```bash
cd
```

Créez le fichier ***/home/media/.env*** et modifiez les valeurs en fonction de votre configuration :

```bash
PUID=1001
PGID=1001
PATH_MEDIA= /PATH/TO/MEDIA
VPN_USER= piauser
VPN_PASS= piapwd
NETWORK= treefik_network
BASE_HOST= dns.fr
LAN_NETWORK=192.168.1.0
```

- **PUID** et **GUID** : ces deux variables représentent respectivement l’identifiant et le groupe de votre utilisateur *media*. Ces valeurs peuvent être différentes d’un système à l’autre. Tapez la commande suivante pour obtenir le PUID et GUID de votre utilisateur *media* :

```
id media
uid=1001(media) gid=1001(media) groups=1001(media),999(docker)
```

- Dans cet exemple, le ***uid*** correspond au **PUID** et le ***gid*** correspond au **PGID**.
- **PATH_MEDIA** : chemin absolu du **dossier parent** des dossiers */data/torrents*, */data/movies*, */data/music* et */data/tv* où seront stockés vos médias. **Si vous choisissez un chemin autre que celui-ci, assurez-vous que l’utilisateur *media* ait les droits de lecture et d’écriture dans le répertoire spécifié.**
- **VPN_USER=*pia_user*** et **VPN_PASS=*pia_password*** : utilisateur et mot de passe de votre compte PIA.
- **LAN_NETWORK** : adresse IP (notation CIDR) de votre réseau local.

  Si vous êtes sur un kimsufi, serveur dédié ou VPS, cette valeur n’a pas d’importance. Laissez la valeur par défaut 192.168.1.0.

  Si vous réalisez l’installation sur votre système ou un système de
  votre réseau local, cette valeur doit être correctement renseignée.
  Tapez la commande `ifconfig` et identifiez votre réseau local (***eth0*** le plus souvent) :


```
ifconfig
eth0: flags=4163  mtu 1500

        inet 192.168.1.10  netmask 255.255.255.0  broadcast 192.168.1.255
        ether 68:05:ca:0b:fe:25  txqueuelen 0  (Ethernet)
        RX packets 28203743  bytes 36171326044 (33.6 GiB)
        RX errors 0  dropped 19925  overruns 0  frame 0
        TX packets 26710466  bytes 165269242671 (153.9 GiB)

        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

Téléchargez l’image Deluge et démarrez-la une première fois pour
forcer la création du répertoire dédié aux fichiers de configuration
avec la commande suivante :

```bash
docker-compose up -d deluge
```

Arrêtez le container *deluge* :

```bash
docker-compose stop deluge
```

- Téléchargez les certificats et fichiers de configuration de PIA à l’adresse suivante : [https://www.privateinternetaccess.com/openvpn/openvpn.zip](https://www.privateinternetaccess.com/openvpn/openvpn.zip).
- Décompressez l’archive et copiez les certificats ***crl.rsa.2048.pem***, ***ca.rsa.2048.crt*** et le fichier de configuration OpenVPN ***.ovpn*** de votre choix dans le répertoire ***/home/media/deluge/config/openvpn***.**Vérifiez bien que vous ayez uniquement ces 3 fichiers dans le répertoire. Un seul fichier de configuration OpenVPN *.ovpn* doit être présent.**
- Ces fichiers doivent appartenir à l’utilisateur et au groupe *media* et doivent avoir les permissions 755. Revenez sous votre super utilisateur et modifiez le propriétaire et les permissions du fichier *.ovpn* :

```bash
exit
```

```bash
sudo chown media:media /home/media/deluge/config/openvpn/*
sudo chmod 755 /home/media/deluge/config/openvpn/*
```

Connectez-vous à nouveau sous l’utilisateur *media* et déplacez-vous dans le répertoire personnel de cet utilisateur :

```bash
su media
```

```bash
cd
```

Démarrez les containers *deluge* :

```bash
docker-compose up -d
```
