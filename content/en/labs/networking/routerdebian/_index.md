---
title: "Create a Router on Debian"
description: "Complete guide to deploy a router on Debian 12 using nftables."
date: 2025-08-18
draft: false
---

## Objective
Set up a Debian VM acting as a **router/NAT** between two private networks in Proxmox, using **nftables** for filtering and address translation.  
This lab helps to understand the role of a router, NAT, and how to configure nftables.  

---

## Step 1 – Create the network and add interfaces
In Proxmox:  
- Create an **additional network bridge** (`vmbr1`) that will serve as the private network behind the router.  
- The VM `RouteurDebian` will therefore have **two interfaces**:  
  - `vmbr0` = Main network (WAN)  
  - `vmbr1` = Network routed by the VM (LAN)  

💡 **Tip**: A simple topology diagram helps to visualize traffic:

---

## Step 2 – Configure the network interfaces
Edit the file `/etc/network/interfaces` on Debian.  

⚠️ It is recommended to use a **static configuration** for the `WAN` interface.  

- **WAN (ex: ens18)**: can be DHCP or static.  
- **LAN (ex: ens19)**: must be **static**, without a gateway.  

Example configuration:  
```bash
auto ens18
iface ens18 inet static
    address 192.168.1.254/24
    gateway 192.168.1.1

auto ens19
iface ens19 inet static
    address 10.10.0.1/16
```

## Step 3 – Configure nftables
#### 📝 Edit the file /etc/nftables.conf:
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
Explanation of the configuration:
- flush ruleset → clears all existing rules.
- table nat-pat → creates a dedicated NAT table.
- chain nat → defines a chain of type NAT.
- hook postrouting → applies rules after routing (packet output).
- priority srcnat → sets the priority, here for source NAT.
- oifname “ens18” → targets the output interface (WAN).
- masquerade → replaces the internal source IP with the router’s public IP (NAT).

## Step 4 – Enable and restart services
#### Apply the configuration:
```bash
sudo systemctl restart networking
sudo systemctl restart nftables
sudo systemctl enable nftables
```
#### 👀 Check the status:
```bash
sudo nft list ruleset
```

## Expected Result
- VMs connected to vmbr1 will have the gateway: 192.168.100.1.
- They will be able to reach the internet through the Debian router’s WAN interface.

## Testing

💻 Create a VM connected to vmbr1 with a manual IP configuration.
Example, based on the configuration above:
```bash 
auto ens18
iface ens18 inet static
    address 10.10.10.10/16
    gateway 10.10.0.1
```
Run connectivity tests:
```bash
ping 8.8.8.8 
```

```bash
ping google.com
```
#### If everything is configured correctly, both pings should succeed. ✅
`SCREENSHOTS POC`
# BONUS
###  Deployment script
Here is a simple Bash script to automate the router setup:
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