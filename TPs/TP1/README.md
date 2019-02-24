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


# I. Communication simple avec deux machines
## 1. Mise en place
* Nouvelle VM, nouvelle configuration, ip: ```10.1.1.3```. On fait... comme d'habitude (edit ```ifcfg-enp0s8``` puis ```ifdown``` et ```ìfup```).
* **client 2** ```sudo hostname tp1.vm2``` et pareil dans ```/etc/hostname```
* **client 1** mise à jour du fichier hosts, ```sudo vi /etc/hosts```



## 2. Basics

### ```Ping``` et ARP

* On vide les deux tables ARP :

    ```sudo ip n flush all``` (ou ip neigh)

* Avec **client 1** on ping **client 2** :

    ```ping -c 4 client2 (10.1.1.3)``` (on ping 4x parceque c'est cool et ça s'arrete tout seul)
    * On regarde la table ARP : 

        * ```ip n s```
        et on a la bonne ligne dans la table :
        * ```10.1.1.3 dev enp0s8 lladdr 08:00:27:26:2e:dc REACHABLE```

* On refait pareil mais avec la capture réseau : 

    * Sur **client 1**, 1er SSH :

        ```sudo tcpdump -i enp0s8 -w ping_v2.pcap``` (comme la dernière fois)

     * Sur **client1**, 2nd SSH :

        ```ping -c 4 10.1.1.3```

    * Sur **client1**, 1er SSH : 

        ```
        ^C10 packets captured
        10 packets received by filter
        0 packets dropped by kernel
        ```
        on a récup les 10 packets, c'est cool

    * On récupère le fichier ping-2.pcap comme la dernière fois

        ```scp it4@10.1.1.2:/home/it4/ping.pcap C:/Users/arnau/Desktop```

* On passe le fichier dans Wireshark : 

    ![lien de la capture](./relatives/ping_v2.PNG 'lien de la capture')

    Comme la première fois, on a la requêtte ARP pour récupperer la MAC de **client 2**, sa réponse puis les ping-pong (dans l'ordre des lignes, j'ai juste oublier mettre le tableau à "l'endroit")

___
### UDP

* **client 1** :

    * Ouverture du port 8888 (et on relance le firewall pour prendre en compte la nouvelle conf) : 

        ```
        sudo firewall-cmd --add-port=8888/udp --permanent
        sudo firewall-cmd --reload
        ```

    * Et on lance netcat pour ecouter l'UDP sur le port (8888) : 

        ```nc -u -l 8888```

* **client 2**, on se connecte sur le port 8888 de **client 1** : 

    ```nc -u 10.1.1.2 8888```
    Le tchat marche bien (vaut mieux pas mettre de screen :D )

* **client 1**, 2nd SSH, on va voir ce qu'il s'est passer... 

    ```
    ss -unp
    Recv-Q Send-Q Local Address:Port               Peer Address:Port              
    0      0      10.1.1.2:8888               10.1.1.3:43998              users:(("nc",pid=1512,fd=4))
    ```

* **client 2**,2nd SSH, on a la chose inverse c'est donc... bien : 

    ```
    ss -unp
    Recv-Q Send-Q Local Address:Port               Peer Address:Port              
    0      0      10.1.1.3:43998             10.1.1.2:8888                users:(("nc",pid=1494,fd=3))
    ```

* **client 1**, 3eme SSH, on a capturé : 

    ```
    sudo tcpdump -i enp0s8 -w nc-udp.pcap
    tcpdump: listening on enp0s8, link-type EN10MB (Ethernet), capture size 262144 bytes
    ^C15 packets captured
    15 packets received by filter
    0 packets dropped by kernel
    ```

    ![lien de la capture](./relatives/udp.PNG 'lien de la capture')

    * Ba.... ça transfert des données.

___
### TCP 

* Pur le TCP, on fait la même chose. Ca marche comme l'UDP mais avec en plus un 'accusé de reception'.

* Grossièrement, le **client 1** rentre dans une salle, 'Salut!', **client 2** répond, 'Yep, Salut' et enfin, **client 1** finit par répondre 'OK nice! (quelqu'un m'a parler! !!)'

![lien de la capture](./relatives/tcp.PNG 'lien de la capture')


## 3. ARP Spoofin
J'ai regarder mais j'ai pas essayer :)

# III. Routage statique simple 

* **client 1** : 

    * Transformation de notre machine en routeur, on active l'ipv4-forwarding

        ```sudo sysctl -w net.ipv4.ip_forward=1```

* **client 2** : 

    * Ajout d'une route statique sur net2 :

        ```sudo vi /etc/sysconfig/network-scripts/route-enp0s9```

        ```10.1.2.0/30 via 10.1.1.2 dev enp0s9```

    * Ping l'ip de l'hôte dans net2 : 

        ```ping 10.1.2.1```
        Ok, it works!

    * Et maintenant, on regarde par où ça passer : 

        ```
        traceroute 10.1.2.1
        traceroute to 10.1.2.1 (10.1.2.1), 30 hops max, 60 byte packets
        1  gateway (10.0.2.2)  0.658 ms  0.205 ms  0.258 ms
        2  10.1.2.1 (10.1.2.1)  0.285 ms  0.243 ms  0.356 ms
        ``` 

        Et bien, on passe par la gateway (10.1.2.1) puis par net2 (10.1.2.1).

        


