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
`10` | `admins` | `10.4.102.0/28` | 
`20` | `serveurs1` | `10.4.102.68/27` | `serveurs publiques (pour les étudiants)`
`21` | `serveurs2` | `10.4.102.124/27 ` | `serveurs privés (pour les profs/admins)`
`30` | `cameras` | `10.4.102.192/27` | 













## Matériel requis




## Installation de l'infra

### Routeurs
Comme pour le dernier TP on va faire une boucle avec les **routeurs** mais cette fois on va mettre en place la redondance via **HSRP** (entre les **router2** et **router3**)

* Configuration de base (ip et OSPF)


* Mise en place d'**HSRP**


### Switchs
Configuration d'**STP**



## Trucs à pas oublier
* STP pour bloquer les tempêtes de braodcast (https://www.cisco.com/c/en/us/support/docs/lan-switching/spanning-tree-protocol/5234-5.html)