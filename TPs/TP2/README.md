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
    * il y a toute fois quelques trucs à changer pour chaques machines (on changera juste les ip pour les routers pour passer par les bonnes routes)

---

## 2. Routage statique 

* activation de l'IPv4 Forwarding (**router1** et **router2**) pour faire voyager des packets

* sur `router1`
  * ajouter une route vers `net2`
  * en vous servant de l'IP de `router2` dans `net12` comme passerelle
  * `sudo ip route add 10.2.2.0/24 via 10.2.12.3 dev enp0s8`
  * `10.2.2.0/24 via 10.2.12.3 dev enp0s8`
* sur `router2`
  * ajouter une route vers `net1`
  * en vous servant de l'IP de `router1` dans `net12` comme passerelle
  * `sudo ip route add 10.2.1.0/24 via 10.2.12.2 dev enp0s8`
* sur `client1`
  * ajouter une route vers `net2`
  * `sudo ip route add 10.2.2.0/24 via 10.2.1.254 dev enp0s8`
* sur `server1`
  * ajouter une route vers `net1`
  * `sudo ip route add 10.2.1.0/24 via 10.2.2.254 dev enp0s8`

Une fois en place :
  ```
  [it4@server1 ~]$ ping client1
  PING client1 (10.2.1.10) 56(84) bytes of data.
  64 bytes from client1 (10.2.1.10): icmp_seq=1 ttl=62 time=3.67 ms
  64 bytes from client1 (10.2.1.10): icmp_seq=2 ttl=62 time=5.35 ms
  64 bytes from client1 (10.2.1.10): icmp_seq=3 ttl=62 time=4.37 ms
  ```

---

## 3. Visualisation du routage avec Wireshark

Vos captures captureront un `ping` entre `router1` et `server1`. **Le `ping` devra donc passer par `router2`.**

But : 
* depuis `router1`
  * `ping server1`
* capturer le trafic qui passe sur `router2`
  * en utilisant `tcpdump`
  * une capture pour l'interface qui est dans `net12`
  ![lien de la capture](./relatives/net12_packets.PNG 'lien de la capture')
  * une autre capture pour l'interface qui est dans `net2`
  ![lien de la capture](./relatives/net2_packets.PNG 'lien de la capture')
  * expliquer la différence
    * Les MACs de destination et de source changent entre les deux captures, le routeur à donc fait son job (petit screen à l'appuie).
    ![lien de la capture](./relatives/net2.PNG 'lien de la capture')
    ![lien de la capture](./relatives/net12.PNG 'lien de la capture')
---

# II. NAT et services d'infra

Dans cette partie on va se rapprocher un peu de ce qu'on peut voir dans des infras réelles. Quelques services récurrents et indispensables, ainsi qu ele concept de NAT.

## 1. Mise en place du NAT

### Présentation

**"Faire du NAT" est un rôle que peut jouer un routeur"** :
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
  * **`router1`** va faire du NAT pour permettre à toutes les autres machines d'accéder à internet
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
```
[it4@router1 ~]$ curl google.com
<HTML><HEAD><meta http-equiv="content-type" content="text/html;charset=utf-8">
<TITLE>301 Moved</TITLE></HEAD><BODY>
<H1>301 Moved</H1>
The document has moved
<A HREF="http://www.google.com/">here</A>.
</BODY></HTML>
```

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
* reload le firewall

Ajouter une route aux autres machines pour qu'elles récup un accès Internet
* d'abord sur **router2** pour tester
  * ajouter une route par défaut
  * `sudo ip route add default via <IP_GATEWAY> dev <INTERFACE_NAME>`
  * `sudo ip route add default via 10.2.12.2 dev enp0s9`
* ensuite idem sur `client1`
  * on rajoute un petit DNS pour curl google
  ```
  [it4@client1 ~]$ cat /etc/resolv.conf 
  # Generated by NetworkManager
  nameserver 8.8.8.8
  ```
  ```
  [it4@router1 ~]$ curl google.com
  <HTML><HEAD><meta http-equiv="content-type" content="text/html;charset=utf-8">
  <TITLE>301 Moved</TITLE></HEAD><BODY>
  <H1>301 Moved</H1>
  The document has moved
  <A HREF="http://www.google.com/">here</A>.
  </BODY></HTML>
  ```
* vérifier que `server1` n'a pas accès à internet
    ```
    [it4@server1 ~]$ curl google.com
    curl: (6) Could not resolve host: google.com; Unknown error
    ```

---

## 2. DHCP server

On va s'en servir pour que de nouveaux clients puissent récupérer une IP et l'adresse de la passerelle directement dans `net1` (réseau clients).  

Pour faire ça, on va recycler `client1` :
* renommer `client1.net1.tp2` en `dhcp-server.net1.tp2`

* récupérer le fichier d'exemple de configuration dhcpd
  * **comprendre le fichier**
    * Bon... Ca marche comme une location d'appart
    * On signe un contrat d'un certaine durée (ici, 10min au moins et 2h max)
    * Ensuite on choisi la plage des ips à louer dans un sous réseau de celui actuel (appart 1, 2, 3, 4)
    * puis on baptise le batiment (domain name), on donne la boite aux lettres publique (broadcast) et la porte de sortie (router)
  * il est très minimaliste, c'est à la portée de tous
    * Yep, j'ai compris
  * le mettre dans `/etc/dhcp/dhcpd.conf`
  * puis on démarre `sudo systemctl start dhcpd`

Pour tester : 
* on craft `client2.tp2.b1`
  * on viens conf son interface **enp0s8**, celle dans **net1**
    ```
    TYPE=Ethernet
    BOOTPROTO=dhcp
    NAME=enp0s8
    DEVICE=enp0s8
    ONBOOT=yes
    ```
  * `ifdown`, `ifup` et on check qu'on a bien une nouvelle ip et une route par defaut
    ```
    [it4@client2 ~]$ ip a | grep "inet 10.2.1"
    inet 10.2.1.50/24 brd 10.2.1.255 scope global dynamic enp0s8
    ```
    ```
    [it4@client2 ~]$ ip r s
    default via 10.2.1.254 dev enp0s8 proto static metric 100 
    10.2.1.0/24 dev enp0s8 proto kernel scope link src 10.2.1.50 metric 100
    ```

---

## 3. NTP server

### Mise en place
On va ouvrir internet à **router2** et aux *servers*
* **router1**


Conf, sur **router1**, **router2** et **server1** : 
* install
  * `sudo yum install -y chrony`
* éditer le fichier `/etc/chrony.conf`, on lui donne les servers français.
  ```
  server 0.europe.pool.ntp.org
  server 1.europe.pool.ntp.org
  server 2.europe.pool.ntp.org
  server 3.europe.pool.ntp.org
  ```
  
* on ouvre son port dans le firewall puis on le lance
  ```
  sudo firewall-cmd --add-port=123/udp --permanent
  sudo firewall-cmd --reload
  ```
  ```
  sudo systemctl start chronyd
  ```
* vérifier l'état de la synchronisation NTP
  * **router1**
    * *chronyc sources*
      ```
      [it4@router1 ~]$ chronyc sources
      210 Number of sources = 4
      MS Name/IP address         Stratum Poll Reach LastRx Last sample
      ===============================================================================
      ^? server.spnr.de                2   6     1    15  +3122us[+3122us] +/-   34ms
      ^? ns2.admincmd.com              2   6     1    16  +1261us[+1261us] +/-   35ms
      ^? server.samoylyk.net           2   6     1    16  +5831us[+5831us] +/-   56ms
      ^? meg.magnet.ie                 2   6     1    17  -2184us[-2184us] +/-   72ms
      ```
    * *chronyc tracking*
      ```
      [it4@router1 ~]$ chronyc tracking
      Reference ID    : 2EE8FABC (server.spnr.de)
      Stratum         : 3
      Ref time (UTC)  : Tue Mar 05 16:23:38 2019
      System time     : 0.000466082 seconds fast of NTP time
      Last offset     : +0.000668030 seconds
      RMS offset      : 0.000668030 seconds
      Frequency       : 0.629 ppm fast
      Residual freq   : +0.789 ppm
      Skew            : 140.169 ppm
      Root delay      : 0.045167409 seconds
      Root dispersion : 0.009137428 seconds
      Update interval : 64.3 seconds
      Leap status     : Normal
      ```
  * exemple de **router2**
    * *chronyc sources*
      ```
      [it4@router2 ~]$ chronyc sources
      210 Number of sources = 1
      MS Name/IP address         Stratum Poll Reach LastRx Last sample               
      ===============================================================================
      ^? router1                       2   6     1     2    +87us[  +87us] +/-   21ms
      ```
      On voit qu'il passe par **routeur1**, c'est good
    * *chronyc tracking*
      ```
      [it4@routeur2 ~]$ chronyc tracking
      Reference ID    : 2EE8FABC ()
      Stratum         : 10
      Ref time (UTC)  : Mon Mar 04 10:41:18 2019
      System time     : 0.000000000 seconds slow of NTP time
      Last offset     : +0.000000000 seconds
      RMS offset      : 0.000000000 seconds
      Frequency       : 0.000 ppm slow
      Residual freq   : +0.000 ppm
      Skew            : 0.000 ppm
      Root delay      : 0.000000000 seconds
      Root dispersion : 0.000000000 seconds
      Update interval : 0.0 seconds
      Leap status     : Normal
      ```
      Tout est à 0, on va dire que c'est normal, il sont censé être synchro mais je ne m'attendais pas à une telle précision.

---

## 4. Web server

Le serveur web tournera sur la machine `server1.net2.b2`. **Ce sera notre "service d'infra".** Dans une vraie infra, on peut trouver tout un tas de services utiles à l'infra en interne :
* dépôts git pour les dévs
* partage de fichiers
* serveur mail
* etc.  

On va utiliser le serveur web [NGINX](https://www.nginx.com/) pour faire ça simplement :
* activation des dépôts EPEL
  * `sudo yum install -y epel-release`
* installation du serveur web NGINX
  * `sudo yum install -y nginx`
* ouverture du port firewall pour l'HTTP
  ```
  sudo firewall-cmd --add-port=80/tcp --permanent
  sudo firewall-cmd --reload
  ```
* lancer le serveur web
  * `sudo systemctl start nginx`
* pour vérifier qu'il est lancé
  * `sudo systemctl status nginx`
  * `sudo ss -altnp4`
    ```
    [it4@server1 ~]$ sudo ss -altnp4
    State       Recv-Q Send-Q Local Address:Port               Peer Address:Port              
    LISTEN      0      128      *:80                   *:*                   users:(("nginx",pid=2423,fd=6),("nginx",pid=2422,fd=6))
    LISTEN      0      128      *:22                   *:*                   users:(("sshd",pid=940,fd=3))
    LISTEN      0      100    127.0.0.1:25                   *:*                   users:(("master",pid=1040,fd=13))
    ```
    Nginx est là et il ecoute sur le port 80, c'est parfait.
* (optionnel) personnaliser la page d'accueil par défaut
  * c'est le fichier `/usr/share/nginx/html/index.html`
    * Afin d'éviter tout risque de propagande (Soviétique ou autre) nous n'irons pas voir ce fichier :)

On curl pour tester :
  ```
  [it4@client2 ~]$ curl 10.2.2.10
  <!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.1//EN" "http://www.w3.org/TR/xhtml11/DTD/xhtml11.dtd">

  <html xmlns="http://www.w3.org/1999/xhtml" xml:lang="en">
      <head>
          <title>Test Page for the Nginx HTTP Server on Fedora</title>
          <meta http-equiv="Content-Type" content="text/html; charset=UTF-8" />
      </head>

      <body>
          <h1>What's wrong, Comrade? You look unwell. I'll get you some Russian tea. Wait a moment.</h1>

      </body>
  </html>
  ```

  J'ai pas pu résister :D

# Bilan
* on a des **clients** 
  * dans un réseau dédié
  * qui récupèrent tout leur réseau automatiquement
    * une IP
    * une passerelle par défaut
    * un serveur DNS
  * qui peuvent aller sur internet normalement (avec firefox ou quoi) directement
* on a des **serveurs**
  * dans un réseau dédié
  * qui ont une IP fixe
  * qui hébergent des services d'infra (un simple `nginx` chez nous)
* on a des **routeurs** 
  * qui peuvent faire passer le trafic d'un réseau à l'autre
  * qui possèdent un lien dédié (le `/30`)
  * l'un d'eux fait du **NAT** pour permettre à tout le monde d'accéder à Internet

:o ! Et nos machines sont synchro pour ce qui est de l'heure. 

:fire: