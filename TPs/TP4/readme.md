## Objectif

<br><p align="center">
  <img src="./relatives/infra.png" title="TP 4 Campus : Infra">
</p>

## Tableaux d'adressage

Hosts | `10.4.100.0` | `10.4.100.4` | `10.4.100.8` | `10.4.100.12`
--- | --- | --- | --- | ---
`router1` | `10.4.100.1` | x | x | `10.4.100.14`
`router2` | `10.4.100.2` | `10.4.100.5` | x | x
`router3` | x | `10.4.100.6` | `10.4.100.9` | x
`router4` | x | x | `10.4.100.10` | `10.4.100.13`




VLAN | nom | réseau | description
--- | --- | --- | ---
`101` | `client101` | `10.4.10.0/25` | `clients batiment 1 salle 1`
`102` | `client102` | `10.4.11.0/25` | `clients batiment 1 salle 2`
`103` | `client103` | `10.4.12.0/25` | `clients batiment 1 salle 3`
`...` | `...` | `...` | `...`
`309` | `client309` | `10.4.38.0/25` | `clients batiment 3 salle 9`
`310` | `client310` | `10.4.39.0/25` | `clients batiment 3 salle 10`
`` | `` | `` | ``
`` | `` | `` | ``
`` | `` | `` | ``
`` | `` | `` | ``
`` | `` | `` | ``
`` | `` | `` | ``
`` | `` | `` | ``
`` | `` | `` | ``
`` | `` | `` | ``
`` | `` | `` | ``
`` | `` | `` | ``



## Matériel requis




## Installation de l'infra

### Routeurs
Comme pour le dernier TP on va faire une boucle avec les **routeurs** mais cette fois on va mettre en place la redondance via **HSRP** (entre les **router2** et **router3**)

* Configuration de base (ip et OSPF)


* Mise en place d'**HSRP**


### Switchs
Configuration d'**STP** sur le/les **router** de backbone



## Trucs à pas oublier
* STP pour bloquer les tempêtes de braodcast (https://www.cisco.com/c/en/us/support/docs/lan-switching/spanning-tree-protocol/5234-5.html)