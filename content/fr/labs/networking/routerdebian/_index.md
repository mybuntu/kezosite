---
title: "Créer un Routeur sur debian"
description: "Guide complet pour déployer un Routeur sur debian 12."
date: 2025-08-18
draft: false
---

## Objectif
Mettre en place une VM Debian jouant le rôle de **routeur/NAT** entre deux réseaux privés dans Proxmox, en utilisant **nftables** pour le filtrage et la traduction d’adresses.  
Ce lab permet de comprendre le rôle d’un routeur, du NAT, et de manipuler nftables.  

---

## Étape 1 – Création du réseau et ajout des interfaces
Dans Proxmox :  
- Créer un **bridge réseau supplémentaire** (`vmbr1`) qui servira de réseau privé derrière le routeur.  
- La VM `RouteurDebian` aura donc **deux interfaces** :  
  - `vmbr0` = Réseau principal (WAN)  
  - `vmbr1` = Réseau routé par la VM (LAN)  

> 💡 **Astuce** : faire un petit schéma de topologie aide à visualiser les flux :  
- [Internet/Proxmox] – vmbr0 –> [ens18 - Debian Routeur - ens19] –> vmbr1 –> [VMs LAN]

---

## Étape 2 – Configuration des interfaces réseau
Modifier le fichier `/etc/network/interfaces` de Debian :  

⚠️ Privilegier la configuration **static** pour l'interface `WAN`  

- **WAN (ex: ens18)** : peut être en DHCP ou statique.  
- **LAN (ex: ens19)** : doit être **statique**, sans passerelle.  

Exemple de config :  
```bash
auto ens18
iface ens18 inet static
    address 192.168.1.254/24
    gateway 192.168.1.1

auto ens19
iface ens19 inet static
    address 10.10.0.1/16
```
## Étape 3 -  Configuration de nftables
#### 📝 Editer le fichier `/etc/nftables.conf` :
```bash
#!/usr/sbin/nft -f
flush ruleset

table nat-pat {
    chain nat {
        type nat hook postrouting priority srcnat;
        oifname "ens18" masquerade;
    }
}
```
### Explications : 
- flush ruleset : vide toute règle existante.
- table nat-pat : création d’une table dédiée au NAT.
- chain nat : définit une chaîne de type nat.
- hook postrouting : applique les règles après le routage (sortie du paquet).
- priority srcnat : définit la priorité, ici pour les translations source.
- oifname "ens18" : cible l’interface de sortie (WAN).
- masquerade : remplace l’IP source interne par l’IP publique de la machine routeur (NAT).

## Étape 4 - Activation et redémarrage des services
#### ✅ Appliquer la configuration :
```bash
sudo systemctl restart networking
sudo systemctl restart nftables
sudo systemctl enable nftables
```

#### 👀 Vérifier l’état :
```bash
sudo nft list ruleset
```

## Résultat attendu
- Les VMs connectées à vmbr1 auront comme passerelle : 192.168.100.1
- Elles pourront sortir vers internet via l’interface WAN du routeur Debian.

## Pour tester : 
- Création d'une VM sur le réseau `vmbr1` a addresser manuellement 
  Exemple en fonction des configs précedente :  
```bash
auto ens18
iface ens18 inet static
    address 10.10.10.10/16
    gateway 10.10.0.1

```
Puis pour tester nous executerons la commande 
```bash
ping 8.8.8.8 
```

```bash
ping google.com
```
#### Si tout a bien été configuré, les deux pings devraient être concluants. ✅
`SCREENSHOTS POC`
# BONUS
### Script de déploiement simplifié
```bash
#!/bin/bash
#
#####SUDO_INSTALL#######
apt update
apt install sudo -y

####################
#INET CONFIGURATION#
####################
printf "Désirez-vous configurer le réseau maintenant ?(Y/n)\n "
read response
if [[ $response == "n" ]]; then {
	printf "Dans ce cas, tout est finit... \n "
	exit 0
}
fi
#####################

sed -i '/ip_forward/ s/^#//' /etc/sysctl.conf
sysctl -p

ip -br address

echo "Enter WAN Interface name(ensX): "
read WAN
echo "Enter LAN Interface name(ensX): "
read LAN
echo "Confirm (Y/n)? "
read confirm

if [[ $confirm == "n" ]]; then {
    echo "CANCELING..."
    exit 1
}
fi

echo "Configre NIC $WAN (d)hcp ou (s)tatic ?"
read WAN_CONF
if [[ $WAN_CONF == "s" ]]; then {
        echo "Enter IP Address (192.168.100.100/24): "
        read WAN_IP
        echo "Enter IP Address of the Default Gateway for $WAN: "
        read GW
        echo "Configre NIC $LAN: "
        echo "Enter IP Address (192.168.100.100/24): "
        read LAN_IP
        echo "auto lo
iface lo inet loopback

auto $WAN
iface $WAN inet static
        address $WAN_IP
        gateway $GW

auto $LAN
iface $LAN inet static
        address $LAN_IP
" > /etc/network/interfaces
}

else {
        echo "Configre NIC $LAN: "
        echo "Enter IP Address (192.168.100.100/24): "
        read LAN_IP
		echo "auto lo
iface lo inet loopback

auto $WAN
iface $WAN inet dhcp

auto $LAN
iface $LAN inet static
        address $LAN_IP

" > /etc/network/interfaces
}
fi

echo "#!/usr/sbin/nft -f

flush ruleset
table nat-pat {
        chain nat {
                type nat hook postrouting priority srcnat;
                oifname $WAN masquerade;
        }
}
" > /etc/nftables.conf
systemctl restart networking
systemctl restart nftables
systemctl enable nftables
#####################
cat /etc/network/interfaces
ip -br a
```