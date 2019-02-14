# TP_1

# I. Exploration du réseau d'une machine CentOS
## 1. Mise en place
### Configuration de VirtualBox
* `net1` : `10.1.1.0/24`
    * /24 : 8bits à 0 donc 2^8 -2 = 256 -2 = 254 adresses disponnibles
* `net2` : `10.1.2.0/30`
    * /30 : 2bits à 0 donc 2^2 -2 = 4 -2 = 4 adresses disponnibles

### Allumage et configuration de la VM
* [X] Désactiver SELinux
    * déja fait dans le patron
* [X] Installation de certains paquets réseau
    * déja fait dans le patron
* [X] définition d'IP statique sur les deux cartes host-only
    * `vi /etc/sysconfig/network-scripts/ifcfg-enp0s8` (puis enp0s9)
        ```
        NAME=enp0s8
        DEVICE=enp0s8
        BOOTPROTO=static
        ONBOOT=yes
        IPADDR=10.1.1.2
        NETMASK=255.255.255.0
        ```
        ```
        NAME=enp0s9
        DEVICE=enp0s9
        BOOTPROTO=static
        ONBOOT=yes
        IPADDR=10.1.2.2
        NETMASK=255.255.255.252
        ```
    * on redémrre les interfaces avec `ifdown / ifup`
    * `ping 10.1.1.1` et `ping 10.1.2.1`, ça marche, c'est cool
* [X] connexion en SSH
    * `ssh it4@10.1.1.2`
* [X] définition d'un nom de domaine
    ```
    sudo hostname vm1.tp1.b2
    hostname --fqdn
        vm1.tp1.b2

    sudo vi /etc/hostame
        vm1.tp1.b2
    ```
* [X] compléter le fichier hosts de la VM
    * `sudo vi /etc/hosts`
    * avec toutes ses IPs et les IPs de l'hôte, on rajoute...
        ```
        10.1.1.2    tp1.vm1
        10.1.2.2    tp1.vm1
        10.1.1.1    net1.tp1
        10.1.2.1    net2.tp1
        ```

* [X] s'assurer que les 3 cartes réseaux fonctionnent :
  * `NAT` : `ping google.com`
    * OK
  * `net1` : `ping net1.tp1`
    * OK
  * `net2` : `ping net2.tp1`
    * OK


## 2. Basics

### Routes

* afficher les routes que connaît votre machine
  * `ip route show`
    ```
    default via 10.0.2.2 dev enp0s3 proto static metric 100
    ```
    ça c'est la gateway, route par défaut
    ```
    10.0.2.0/24 dev enp0s3 proto kernel scope link src 10.0.2.15 metric 100
    10.1.1.0/24 dev enp0s8 proto kernel scope link src 10.1.1.2 metric 100
    10.1.2.0/30 dev enp0s9 proto kernel scope link src 10.1.2.2 metric 100
    ```
    Ici, c'est les routes pour les 3 interfaces, `enp0s3` passe par `10.0.2.15`


* supprimer une route
  * `sudo ip route del 10.1.2.0/30` et `sudo ip route del default`
  * `ping net2.tp1` `connect: Network is unreachable`, OK

* remettre la route
  * `sudo ip route add 10.1.2.0/30 via 10.1.2.2 dev enp0s9` pour la mettre en temporaire
  * `ping net2.tp1` OK


### Table ARP

* afficher les voisins que connaît votre machine = la table ARP
  * `ip neigh show`
  * `10.1.1.1 dev enp0s8 lladdr 0a:00:27:00:00:51 REACHABLE`
    * La table ARP a trouvé mon PC (c'est bien la MAC de ma machine) comme voisin via l'interface enp0s8 (connexion en SSH) 

* vider la table ARP
  * `sudo ip neigh flush all`
  * `ip neigh show`
    * `10.1.1.1 dev enp0s8  FAILED`

* effectuer une requête simple vers l'hôte
  * `ping 10.1.2.1` par exemple
  * `10.1.1.1 dev enp0s8 lladdr 0a:00:27:00:00:51 REACHABLE`
  * Wait ?!? Il n'y a rien de nouveau la dedans...



### Capture réseau
Let's go :
* choisir une interface host-only que vous allez utilisez pour `ping` l'hôte (votre PC)
  * ce sera `enp0s9`
* vider la table ARP, checker, il n'y a que enp0s8 (SSH) donc c'es good
* ouvrir deux sessions SSH dans la VM
  * **1** : lancer une capture
    * `sudo tcpdump -i enp0s9 -w ping.pcap`
  * **2** : 
    * s'assurer que la table ARP est toujours vide, OK
    * envoyer 4 `ping`
      * `ping -c 4 10.1.2.1`
  * **1** : CTRL + C
    * vous devrez avoir capturé 10 paquets
      * 2 pour l'échange ARP
      * 4 ping
      * 4 pong
* récupérer le fichier `ping.pcap` sur l'hôte
  * `scp it4@10.1.1.2:/home/it4/ping.pcap C:/Users/arnau/Desktop`
* explorer la capture dans Wireshark
    * ![lien du fichier wireshark](./relatives/ping.pcap 'lien du fichier wireshark')
    * ![lien de la capture](./relatives/ping.PNG 'lien de la capture')
    * les lignes 1 à 8 sont juste des ping-pong
    * la ligne 10, c'est le broadcast de la table ARP qui cherche qui est 10.1.2.1 as <MAC>
    * la ligne 9, c'est la réponse ARP











    