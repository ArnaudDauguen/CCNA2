# Le projet : Menu 2, Infra Campus

J'ai choisi ce projet car il me semblais être celui le plus à ma porté et puis je trouve sympa de monter un gros truc comme une infra de campus, ça me fait penser à des Légo(r) je sais pas pourquoi.
Mais je trouver sa fun donc c'est partit.

Etant donné que l'infra est tout de même assez massive pour être virtualisé completement sur un seul PC, j'ai choisi de la rétrécir une peu.

2PC par salles (1 prof, un étudiant), 1 salle par batiment et 3 batiments plus 2 serveurs, 1 admin et 1 caméra.
Ainsi, on passe de près de 300 machines à une dizaine de machines, 4 routeurs et quelques switchs.


## Objectif

<br><p align="center">
  <img src="./relatives/infra.png" title="TP 4 Campus : Infra">
</p>










## Tableaux d'adressage

Hosts | `10.4.100.0/30` | `10.4.100.4/30` | `10.4.100.8/30` | `10.4.100.12/30`
--- | --- | --- | --- | ---
`router1` | `10.4.100.1` | x | x | `10.4.100.14`
`router2` | `10.4.100.2` | `10.4.100.5` | x | x
`router3` | x | `10.4.100.6` | `10.4.100.9` | x
`router4` | x | x | `10.4.100.10` | `10.4.100.13`





* Les salles sont isolée de façon à empecher les petits génies d'embeter leurs collègues.
* On va faire des (trop) gros réseau, comme ça on a de la place pour plus tard, si on veux rajouter des machines





VLAN | nom | réseau | description
--- | --- | --- | ---
`101` | `etu101` | `10.4.11.0/25` | `étudiants batiment 1 salle 1`
`102` | `etu102` | `10.4.12.0/25` | `étudiants batiment 1 salle 2`
`103` | `etu103` | `10.4.13.0/25` | `étudiants batiment 1 salle 3`
`...` | `...` | `...` | `...`
`309` | `etu309` | `10.4.39.0/25` | `étudiants batiment 3 salle 9`
`310` | `etu310` | `10.4.40.0/25` | `étudiants batiment 3 salle 10`
`501` | `profs` | `10.4.101.0/24` | 
`10` | `admins` | `10.4.201.0/28` | 
`20` | `serveurs1` | `10.4.202.0/27` | `serveurs publiques (pour les étudiants)`
`21` | `serveurs2` | `10.4.203.0/27 ` | `serveurs privés (pour les profs/admins)`
`30` | `cameras` | `10.4.204.0/27` | 







## Matériel requis
  * des cables (pleins, à compter)
  * autant de bornes Wifi que de salles (20aine)



## Installation de l'infra

### Routeurs
Comme pour le dernier TP on va faire une boucle avec les **routeurs** mais cette fois on va mettre en place la redondance via **HSRP** (entre les **router2** et **router3**)

* Configuration de base (ip et OSPF (OSPF sert a rien ici mais je sais faire donc je le met pour le TP :D ))
    ```
    R1#conf t
    R1(config)#interface f0/0
    R1(config-if)#ip address 10.4.100.1 255.255.255.252
    R1(config-if)#no shut
    R1(config-if)#ip nat inside
    R1(config-if)#exit

    R1(config)#interface f1/0
    R1(config-if)#ip address 10.4.100.14 255.255.255.252
    R1(config-if)#no shut
    R1(config-if)#ip nat inside
    R1(config-if)#exit

    R1(config)#interface f3/0
    R1(config-if)#ip address dhcp
    R1(config-if)#no shut
    R1(config-if)#ip nat outside
    R1(config-if)#exit

    R1(config)#ip nat inside source list 1 interface f3/0 overload
    R1(config)#access-list 1 permit any

    R1(config)#router ospf 1
    R1(config-router)#router-id 1.1.1.1
    R1(config-router)#network 10.4.100.0 0.0.0.3 area 0
    R1(config-router)#network 10.4.100.12 0.0.0.3 area 0
    // R1 routeur par défaut (pour internet) via OSPF
    R1(config-router)#default-information originate

    R1#copy running-config startup-config
    ```

    [Tester internet](https://github.com/It4lik/B2-Reseau-2018/tree/master/tp/3#Annex)

* Configuration des VLANs (sur **R3** et **R2**)

  ```
  R3#conf t
  R3(config)#interface f2/0.101
  R3(config-subif)#encap dot1q 101
  R3(config-subif)#ip add 10.4.11.125 255.255.255.128
  R3(config-subif)#no shut
  R3(config-subif)#exit
  
  R3(config)#interface f2/0.201
  R3(config-subif)#encap dot1q 201
  R3(config-subif)#ip add 10.4.21.125 255.255.255.128
  R3(config-subif)#no shut
  R3(config-subif)#exit

  // et de même pour les autres VLANs
  ```



* Mise en place d'**HSRP**

  **HSRP** à pour objectif de faire un peu de magie (c'est plus clair une fois compris). Le but c'est de faire, en gros, fusionner 2 routeurs pour que s'il y en a un qui meurt l'autre prenne sa place.
  
  Un routeur fait son job, l'autre attend, avec plus ou moins d'impatience selon affinitées, que son pote décède pour prendre sa place.
  La magie est toute simple, on donne une même ip virtuelle aux deux routeurs. Juste ça...
  
  Sauf que nous, on a un autre problème. On doit faire 8 sous-interface (mais normalement 33)
  Donc on va "s'amuser" à faire 8 ips virtuelles

  8 vIPs pour 8 VLANs : 3 salles, admins, profs, serveurs1, serveurs2 et cameras

  * Exemple pour le premier réseau
  ```
  R2(config)#interface f2/0.10
    // priorité du routeur (pour savoir lequel sera utilisé en premier) (on met 110 comme ça on se garde de la place au cas ou)
  R2(config-subif)#standby 10 priority 110
    // forcer les éléctions de routeur
  R2(config-subif)#standby 10 preempt
    // définition de l'ip virtuelle, commune aux deux routeurs
  R2(config-subif)#standby 10 ip 10.4.201.14

  R3(config)#interface f2/0.10
  R3(config-subif)#standby 10 priority 120
  R3(config-subif)#standby 10 preempt
  R3(config-subif)#standby 10 ip 10.4.201.14
  
  ```
  * On a disposition que les groupes HSRP de 0 (défaut) ) 255. Donc les deux derniers réseaux ne seront pas rangés de la même façon que les VLANs.

### Switchs

( J'ai perdu un temps fou avec les switches qui lagent. 2h pour juste configurer access/trunk -__- )

#### Switchs de Core
* Afin d'éviter les désagréments d'une tempête de broadcast notamment, nous allons mettre en place **STP** (Spanning Tree Protocol). Il va s'occuper de dconnecter automatiquement les ports en trop.

* Conf de **IOU2** (puisqu'il est sous **R2**)
  ```


  ```

#### Switchs de Distribution
* Comme leurs noms l'indiquent, il s'agit de switchs de distribution donc on rien à faire à part configurer tous les ports en mode **trunk**.
* voila l'exemple pour la batiment 1, **Bat 1**


  ```
  Bat1#conf t
  Bat1(config)#int e0/0
  Bat1(config-if)#switchport trunk encapsulation dot1q
  Bat1(config-if)#switchport mode trunk
  Bat1(config-if)#exit

  Bat1(config)#int e0/1
  Bat1(config-if)#switchport trunk encapsulation dot1q
  Bat1(config-if)#switchport mode trunk
  Bat1(config-if)#exit

  Bat1(config)#int e0/3
  Bat1(config-if)#switchport trunk encapsulation dot1q
  Bat1(config-if)#switchport mode trunk
  Bat1(config-if)#exit

  Bat1(config)#int e1/0
  Bat1(config-if)#switchport trunk encapsulation dot1q
  Bat1(config-if)#switchport mode trunk
  Bat1(config-if)#exit

  Bat1(config)#int e1/1
  Bat1(config-if)#switchport trunk encapsulation dot1q
  Bat1(config-if)#switchport mode trunk
  Bat1(config-if)#exit
  ```


#### Switchss d'Access
* Le port d'entré (celui relié au switch du batiment) est en mode **trunk**
* Les ports pour les élèves sont en mode **access** et configurés sur le VLAN de la salle
* Le port du professeur est en mode **access** lui aussi mais configuré sur le VLAN des professeurs.
* Dans notre cas
  * on utilise un switch à la place d'une borne Wifi mais ça marche snesiblement pareil (juste que la on est branché au lieu de regarder les MACs)
  * pour le plaisir et la survie de mon PC, on ne simule qu'une salle par batiment et chaque salle contient 1 elève et 1 professeur.

Et voici la conf du switch de la 1ere salle du Batiment 1, **Wifi101**


conf t
int e0/0
switchport trunk encapsulation dot1q
switchport mode trunk
exit
int e0/2
switchport mode access
switchport access vlan 101
exit
int e3/3
switchport mode access
switchport access vlan 501
exit





  ```
  Wifi101#conf t

  Wifi101(config)#int e0/0
  Wifi101(config-if)#switchport trunk encapsulation dot1q
  Wifi101(config-if)#switchport mode trunk
  Wifi101(config-if)#exit

  Wifi101(config)#int e0/2
  Wifi101(config-if)#switchport mode access
  Wifi101(config-if)#switchport access vlan 101

  
  Wifi101(config)#interface e3/3
  Wifi101(config-if)#switchport mode access
  Wifi101(config-if)#switchport access vlan 501

  ```











## Trucs à pas oublier
* STP pour bloquer les tempêtes de braodcast (https://www.cisco.com/c/en/us/support/docs/lan-switching/spanning-tree-protocol/5234-5.html)
* DHCP sur un des routeurs pour faire toutes les adresses (routeur en haut peut etre)