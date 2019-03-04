# B2 Réseau 2018 - TP2

# TP 2 - Routage statique et services d'infra 

Deuxième TP : on va remettre en place du routage statique avec des VMs et quelques services réseau très utiles dans n'importe quelle infra. Encore des choses qu'on a vu l'an dernier, mais ça va nous servir pour construire des choses plus complexes.  

Au menu :
* des VMs (avec du SSH)
* revue de IP, MAC, ports, ARP
* routage statique
* DHCP
* NAT

# Sommaire

* [I. Mise en place du lab](#i-mise-en-place-du-lab)
  * [1. Création des VMs et adressage IP](#1-création-des-vms-et-adressage-ip)
  * [2. Routage statique](#2-routage-statique)
  * [3. Visualisation du routage avec Wireshark](#3-visualisation-du-routage-avec-wireshark)
* [II. NAT et services d'infra](#ii-nat-et-services-dinfra)
  * [1. Mise en place du NAT](#1-mise-en-place-du-nat)
  * [2. Serveur DHCP](#2-dhcp-server)
  * [3. Serveur NTP](#3-ntp-server)
  * [4. Serveur Web](#4-web-server)

---

# I. Mise en place du lab 

## 1. Création des VMs et adressage IP

* vous devez posséder 3 réseaux host-only
  * `net1` : `10.2.1.0/24`
    * réseau des clients
    * vbox : 10.2.1.1 / 255.255.255.0
  * `net2` : `10.2.2.0/24`
    * réseau des serveurs
    * vbox : 10.2.2.1 / 255.255.255.0
  * `net12` : `10.2.12.0/29`
    * réseau des routeurs ("backbone")
    * vbox : 10.2.12.1 / 255.255.255.248

* créer 4 clones du patron de VM
  * `client1.net1.b2`
    * **SANS** carte NAT
  * `server1.net2.b2`
    * **SANS** carte NAT
  * `router1.net12.b2`
    * **AVEC** carte NAT
  * `router2.net12.b2`
    * **SANS** carte NAT

### Tableau d'adressage IP

Machine | `net1` | `net2` | `net12`
--- | --- | --- | ---
PC | `10.2.1.1/24` | `10.2.2.1/24` | `10.2.12.1/29`
`client1.net1.b2` | `10.2.1.10/24` | X | X
`server1.net2.b2` | X | `10.2.2.10/24` | X
`router1.net12.b2` | `10.2.1.254/24` | X | `10.2.12.2/29`
`router2.net12.b2` | X | `10.2.2.254/24` | `10.2.12.3/29`

### Schéma

```
        router1.net12.b2                   router2.net12.b2
            +-----+                             +-----+
            |     |10.2.12.2/29     10.2.12.3/29|     |
            |     +-----------------------------+     |
            +-----+                             +-----+
   10.2.1.254/24|                                   |10.2.2.254/24
                |                                   |
                |                                   |
                |                                   |
                |                                   |
                |                                   |
    10.2.1.10/24|                                   |10.2.2.10/24
            +-----+                             +-----+
            |     |                             |     |
            |     |                             |     |
            +-----+                             +-----+
        client1.net1.b2                   server1.net2.b2
```

### Checklist IP VMs 

On parle de toutes les VMs :
* [X] Désactiver SELinux
  * déja fait dans le patron
* [X] Installation de certains paquets réseau
  * déja fait dans le patron
* [X] Désactivation de la carte NAT
  * déja fait dans le patron
* [X] Définition des IPs statiques (comme d'habitude mais là, on y passe 10min)
* [X] La connexion SSH doit être fonctionnelle
* [X] [Définition du nom de domaine]
    * ex:  `sudo hostname server1.net1.b2 && echo 'server1.net2.b2' | sudo tee /etc/hostname`

* [X] Remplissage du fichier `/etc/hosts`
    * exemple pour **client 1** :
    ```
    10.2.1.10   client1 client1.net1 client1.net1.b2
    10.2.2.10   server1 server1.net2 server1.net2.b2
    10.2.1.254  router1 router1.net12 router1.net12.b2
    10.2.12.3   router2 router2.net12 router2.net12.b2
    ```
    * il y a toute fois quelques trucs à changer pour chaques machines (on changera juste les ip des routers pour passer par les bonnes routes)

J'Y SUIS LA..

---

## 2. Routage statique

Vous allez ajouter des routes statiques sur les machines agissant comme routeurs, mais aussi sur les clients. 

Avant tout, sur **router1** et **router2**, activez l'IPv4 forwarding qui permettra à votre VM de traiter des paquets IP qui ne lui sont pas destinés. En d'autres termes **vous transformez votre machine en routeur**. GG. :fire:  

* activation de l'IPv4 Forwarding
  * `sudo sysctl -w net.ipv4.conf.all.forwarding=1`

Pour ajouter des routes statiques, vous pouvez vous référer à [la section dédiée dans page de procédures CentOS 7](../../cours/procedures.md#ajouter-une-route-statique).

* sur `router1`
  * ajouter une route vers `net2`
  * en vous servant de l'IP de `router2` dans `net12` comme passerelle
  * `sudo ip route add 10.2.2.0/24 via 10.2.12.3 dev enp0s8`
  * `10.2.2.0/24 via 10.2.12.3 dev enp0s8`
* sur `router2`
  * ajouter une route vers `net1`
  * en vous servant de l'IP de `router1` dans `net12` comme passerelle
* sur `client1`
  * ajouter une route vers `net2`
* sur `server1`
  * ajouter une route vers `net1`

Une fois en place :
  * `client1` et `server1` peuvent se `ping`
    * alors qu'ils ne connaissent pas l'adresse de `net12`
    * pourtant leurs messages passent bien par `net12`
    * :fire:

---

## 3. Visualisation du routage avec Wireshark

> Pour rappel, lorsque vous utilisez Wireshark, évitez de faire du SSH sur l'interface où vous capturez le trafic, ça évitera la pollution dans vos captures.

Vos captures captureront un `ping` entre `router1` et `server1`. **Le `ping` devra donc passer par `router2`.**

But : 
* depuis `router1`
  * `ping server1`
* capturer le trafic qui passe sur `router2`
  * en utilisant `tcpdump`
  * une capture pour l'interface qui est dans `net12`
  * une autre capture pour l'interface qui est dans `net2`
  * expliquer la différence

> Les captures devraient être très simples : uniquement les ping/pong et éventuellement un peu d'ARP

> Les deux captures doivent figurer dans le rendu

---

# II. NAT et services d'infra

Dans cette partie on va se rapprocher un peu de ce qu'on peut voir dans des infras réelles. Quelques services récurrents et indispensables, ainsi qu ele concept de NAT.

* le [**NAT**](#1-mise-en-place-du-nat) permet à un réseau d'IP privées de joindre des IPs publiques
  * on en a donc besoin dans tous les LANs du monde
  * chez vous
  * n'importe quelle entreprise
  * etc

* un serveur [**DHCP**](#2-dhcp-server) permet d'attribuer automatiquement des IPs aux clients
  * cela permet de ne pas avoir à les saisir à la main, de façon *statique*
  * un serveur DHCP peut aussi donner d'autres infos (cf. la partie du TP dédiée)

* le [**NTP**](#3-ntp-server) est un protocole qui permet de synchroniser l'heure des serveurs

* les **[serveurs Web](#4-web-server)** sont partout de nos jours, on en montera régulièrement pour symboliser/simuler un serveur d'infra
  * on en trouve partout sur internet
  * mais aussi dans les services d'infra, le pilotage des outils se fait de plus en plus via HTTP (= serveur web)

## 1. Mise en place du NAT

### Présentation

**"Faire du NAT" est un rôle que peut jouer un [routeur]../../cours/lexique.md#routeur)** :
* qui possède une interface qui peut joindre le WAN
  * dans lequel on trouve des IPs publiques
  * comme internet
* qui possède une interface dans le LAN
  * dans lequel on trouve des réseaux privés

Dans le cadre du TP : 
* `router1` a une interface qui lui permet de joindre le WAN, internet
  * c'est sa carte VitualBox NAT
  * il a une autre interface host-only, qui possède une IP dans un LAN
    * le LAN `net12`
    * le LAN `net1`
  * **`router1` va faire du NAT pour permettre à toutes les autres machines d'accéder à internet
* afin de renforcer un peu la sécurité de notre mini-infra :
  * `net1` (clients) pourra joindre internet
  * `net2` (serveurs) **ne pourra pas** joindre internet

```
                VirtualBox
                   NAT
                    |
                    |
           10.0.2.15|
            (public)|                          router2.net12.tp2
                 +--+--+(internal)                   +-----+
router1.net12.tp2|     |10.2.12.2/29     10.2.12.3/29|     |
                 |     +-----------------------------+     |
                 +-----+                             +-----+
       10.2.1.254/24|                                   |10.2.2.254/24
            (public)|                                   |
                    |                                   |
                    |                                   |
                    |                                   |
                    |                                   |
        10.2.1.10/24|                                   |10.2.2.10/24
                 +-----+                             +-----+
                 |     |                             |     |
                 |     |                             |     |
                 +-----+                             +-----+
             client1.net1.tp2                   server1.net2.tp2
```

### Mise en place

S'assurer que la VM `router1` peut joindre internet :
* faire un `ping` vers une IP connue sur internet
* ou un [`curl`](../../cours/lexique.md#curl-et-wget) si vous êtes à YNOV

Pour ce faire, on va utiliser les *zones* du Firewall CentOS. 
* mettre l'interface NAT dans la zone `public`
  * dans le fichier de config de l'interface, ajouter `ZONE=public`
* mettre l'interface de `net1` dans la zone `public`
  * dans le fichier de config de l'interface, ajouter `ZONE=public`
* mettre l'interface de `net12` dans la zone `internal`
  * dans le fichier de config de l'interface, ajouter `ZONE=internal`
* il faut redémarrer les interfaces avec `ifdown` puis `ifup` pour que le changement prenne effet

Activer le NAT (ou *masquerading*) dans la zone `public`
* `sudo firewall-cmd --add-masquerade --zone=public --permanent`
* reload le firewall (vous connaissez la commande !)

Ajouter une route aux autres machines pour qu'elles récup un accès Internet
* d'abord sur `router2` pour tester
  * ajouter une route par défaut
  * `sudo ip route add default via <IP_GATEWAY> dev <INTERFACE_NAME>`
  * `ping` ou [`curl`](../../cours/lexique.md#curl-et-wget) pour tester l'accès internet
* ensuite idem sur `client1`
* vérifier que `server1` n'a pas accès à internet

---

## 2. DHCP server

Un serveur DHCP, ça permet de :
* donner une IP à des clients qui le demandent
* donner d'autres infos
  * comme l'adresse de la passerelle par défaut

On va s'en servir pour que de nouveaux clients puissent récupérer une IP et l'adresse de la passerelle directement dans `net1` (réseau clients).  

Pour faire ça, on va recycler `client1` :
* renommer `client1.net1.tp2` en `dhcp-server.net1.tp2`
* puis : `sudo yum install -y dhcp`
* récupérer [le fichier d'exemple de configuration dhcpd](./dhcp/dhcpd.conf)
  * **comprendre le fichier**
  * il est très minimaliste, c'est à la portée de tous
  * le mettre dans `/etc/dhcp/dhcpd.conf`
* démarrer le serveur DHCP 
  * `sudo systemctl start dhcpd`
  * appelez-moi en cas de pb

Pour tester : 
* clonez une nouvelle fois votre VM patron, ce sera notre `client2.tp2.b1`
  * [configurer l'interface en DHCP, en dynamique (pas en statique)](../../cours/procedures.md#définir-une-ip-dynamique-dhcp)
  * utiliser [`dhclient`](../../cours/lexique.md#dhclient-linux-only)
* dans un cas comme dans l'autre, vous devriez récupérer une IP dans la plage d'IP définie dans `dhcpd.conf`
  * et une route par défaut  

---

## 3. NTP server

### Présentation

NTP (pour *Network Time Protocol*) est le protocole (= la langue) que l'on utilise pour permettr eà plusieurs serveurs d'être synchronisés au niveau de la date et de l'heure. Le problème est loin d'être trivial lorsque l'on s'y intéresse de près.  

Il existe des [serveurs NTP publics](https://www.pool.ntp.org), sur lesquels n'importe qui peut se synchroniser. Ils servent souvent de référence.  

Dans notre cas :
* on va demander à `router1` de se synchroniser sur un serveur externe
* et on va demander à toutes les autres machines de se synchroniser sur `router1`

Dernier détail : sur CentOS, le service qui gère NTP s'appelle `chrony` :
* le démon systemd s'appelle `chronyd`
  * donc `sudo systemctl restart chronyd` par exemple
* la commande pour avoir des infos est `chronyc`
  * `chronyc sources` pour voir les serveurs pris en compte par `cronyd`
  * `chronyc tracking` pour voir l'état de la synchronisation
* la configuration se trouve dans `/etc/chrony.conf`
* présent par défaut sur CentOS

### Mise en place

Sur `router1` : 
* éditer le fichier `/etc/chrony.conf`
  * [un contenu modèle se trouve ici](./chrony/serveur/chrony.conf)
  * choisissez le pool de serveurs français sur [le site des serveurs externes de référence](https://www.pool.ntp.org) et ajoutez le à la configuration
* [ouvrir le port utilisé par NTP](../../cours/procedures.md#interagir-avec-le-firewall)
  * c'est le port 123/UDP
* démarrer le service `chronyd`
  * `sudo systemctl start chronyd`
* vérifier l'état de la synchronisation NTP
  * `chronyc sources`
  * `chronyc tracking`

Sur toutes les autres machines : 
* éditer le fichier `/etc/chrony.conf`
  * [un contenu modèle se trouve ici](./chrony/client/chrony.conf)
* [ouvrir le port utilisé par NTP](../../cours/procedures.md#interagir-avec-le-firewall)
  * c'est le port 123/UDP
* démarrer le service `chronyd`
  * `sudo systemctl start chronyd`
* vérifier l'état de la synchronisation NTP

---

## 4. Web server

Le serveur web tournera sur la machine `server1.net2.tp2`. **Ce sera notre "service d'infra".** Dans une vraie infra, on peut trouver tout un tas de services utiles à l'infra en interne :
* dépôts git pour les dévs
* partage de fichiers
* serveur mail
* etc.  

On va utiliser le serveur web [NGINX](https://www.nginx.com/) pour faire ça simplement :
* activation des dépôts EPEL (plus d'infos sur le net, ou posez moi la question)
  * `sudo yum install -y epel-release`
* installation du serveur web NGINX
  * `sudo yum install -y nginx`
* ouverture du port firewall pour l'HTTP
  * [ouvrez le port 80 en TCP](../../cours/procedures.md#interagir-avec-le-firewall)
* lancer le serveur web
  * `sudo systemctl start nginx`
* pour vérifier qu'il est lancé
  * `sudo systemctl status nginx`
  * [`sudo ss -altnp4`](../../cours/lexique.md#netstat-ou-ss)
* (optionnel) personnaliser la page d'accueil par défaut
  * c'est le fichier `/usr/share/nginx/html/index.html`

Vous devriez pouvoir accéder au serveur web depuis vos clients avec un [`curl`](../../cours/lexique.md#curl-et-wget). 
