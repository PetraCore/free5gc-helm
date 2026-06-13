# UERANSIM Machine Configuration

Setup guide for a machine running **UERANSIM** — a simulated 5G UE and gNB, connected to the free5GC core.

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [System Update & Dependencies](#system-update--dependencies)
3. [Network Configuration](#network-configuration)
4. [Kernel Module Installation](#kernel-module-installation)
5. [MicroK8s Setup](#microk8s-setup)
   - [Install & Configure](#install--configure)
   - [Post-NAT Removal Fix](#post-nat-removal-fix)
   - [CNI Network Config](#cni-network-config)
   - [Joining the Cluster](#joining-the-cluster)
6. [UE Namespace Setup](#ue-namespace-setup)
   - [Cleanup](#cleanup)
   - [UE1](#ue1)
   - [UE2](#ue2)
   - [NAT & Forwarding](#nat--forwarding)
7. [Starting UERANSIM](#starting-ueransim)
8. [Tests](#tests)
9. [Scripts](#scripts)
   - [setup-ues.sh](#setup-uesh)
   - [netchat.sh](#netchatsh)

---

## Prerequisites

- Static IP: `192.168.67.42/24`
- Gateway (UPF): `192.168.67.67`
- Ethernet interface: `eth0` (or `enp0s8` — see note below)
- UERANSIM built at `$HOME/UERANSIM/build/`
- Config files at `$HOME/ueransim/`

## System Update & Dependencies

```bash
sudo apt update && sudo apt upgrade
sudo apt install make gcc g++ libsctp-dev lksctp-tools iproute2
```

## Network Configuration

Create a static IP Netplan config:

```bash
sudo vim /etc/netplan/99-static-ip.yaml
```

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      match:
        name: "enp0s8"
      set-name: eth0
      addresses:
        - 192.168.67.42/24
      routes:
        - to: default
          via: 192.168.67.67
      nameservers:
        addresses: [8.8.8.8]
      dhcp4: false
      dhcp6: false
```

> **Alternative — skip the `match`/`set-name` renaming:**
> If you prefer to keep the interface as `enp0s8`, omit the `match` and `set-name` fields and use `enp0s8` as the key instead. However, you must then also update all UERANSIM config files to reference `enp0s8` instead of `eth0`.

```bash
sudo chmod 600 /etc/netplan/99-static-ip.yaml
sudo netplan try
```

> **Tip:** Once the UPF is configured and routing is set up, remove the NAT network card so all traffic goes through the 5G core.

## Kernel Module Installation

```bash
sudo modprobe ipvlan
echo "ipvlan" | sudo tee /etc/modules-load.d/ipvlan.conf
```

## Build UERANSIM

```bash
git clone https://github.com/aligungr/UERANSIM.git
cd UERANSIM
make
```
### Adding auto export of ueransim
```bash
echo 'export PATH="$PATH:/home/student/net5g/UERANSIM/build"' >> .bashrc
```

```bash
source ~/.bashrc
```

## Pull the UERANSIM configuration:

```bash
git clone --filter=blob:none --no-checkout https://github.com/PetraCore/free5gc-helm.git
cd free5gc-helm/
git sparse-checkout init --cone
git sparse-checkout set ueransim/
git checkout
cp ~/free5gc-helm/ueransim/ ~/ueransim-config -r
```

## Starting the system

After the UPF and CP are working, in order to test if this system works, we need to start one gNB and one or two UEs.

### Starting gNB

```bash
sudo $HOME/UERANSIM/build/nr-gnb -c $HOME/ueransim/gnbconfig.yaml
```

### Starting UE(s)

In order to start one UE:

```bash
$UERANSIM_DIR/build/nr-ue -c $CONFIG_DIR/ueconfig.yaml
```

and we can add a -n flad to add more UE's (the ism number will be incremented by one)

```bash
$UERANSIM_DIR/build/nr-ue -c $CONFIG_DIR/ueconfig.yaml -n 2
```

> A good practice is to run each UE on a different namespace because Linux might have some problems with routing

## Tests

Check which device is assigned to UEs and test the using ping (this example is for UEs in network namespaces)

```bash
# UE1 → UE2 through the 5G core
sudo ip netns exec ue1 ping -I uesimtun0 10.1.0.2

# UE2 → UE1
sudo ip netns exec ue2 ping -I uesimtun0 10.1.0.1

# UE1 → internet
sudo ip netns exec ue1 ping -I uesimtun0 8.8.8.8

# UE2 → internet
sudo ip netns exec ue2 ping -I uesimtun0 8.8.8.8
```
> **Disclaimer:** Note that UEs in your setup might be assigned different IPs if you started them in different order or restarted them.