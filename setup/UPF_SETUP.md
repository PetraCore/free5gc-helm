# UPF Machine Configuration

Setup guide for a Raspberry Pi running **Ubuntu Server 24.04 LTS** as a User Plane Function (UPF) node.

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [System Update](#system-update)
3. [Network Configuration](#network-configuration)
   - [Static IP (Ethernet)](#static-ip-ethernet)
   - [Wi-Fi — Enterprise (WEITI) and Home Networks](#wi-fi--enterprise-weiti-and-home-networks)
   - [Optional: Netplan-based Wi-Fi](#optional-netplan-based-wi-fi)
   - [IP Forwarding & NAT](#ip-forwarding--nat)
4. [Kernel Module Installation](#kernel-module-installation)
   - [GTP5G](#gtp5g)
   - [IPvlan](#ipvlan)
5. [MicroK8s Setup](#microk8s-setup)
   - [Install & Configure](#install--configure)
   - [Offline / Static IP Mode](#offline--static-ip-mode)
   - [CNI Network Config](#cni-network-config)

---

## Prerequisites

- Raspberry Pi with Ubuntu Server 24.04 LTS installed
- Ethernet interface: `eth0`
- Wi-Fi interface (optional): `wlan0`
- Target static IP: `192.168.67.67/24`

---

## System Update

```bash
sudo apt update && sudo apt upgrade
```

---

## Network Configuration

### Static IP (Ethernet)

**1. Disable cloud-init network management:**

```bash
sudo vim /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg
```

```yaml
network: {config: disabled}
```

**2. Create a static IP Netplan config:**

```bash
sudo vim /etc/netplan/99-static-ip.yaml
```

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      addresses:
        - 192.168.67.67/24
      dhcp4: false
      dhcp6: false
```

**3. Apply the config:**

```bash
sudo chmod 600 /etc/netplan/99-static-ip.yaml
sudo netplan try
```

**4. Back up and disable the original cloud-init netplan file:**

```bash
sudo mv /etc/netplan/50-cloud-init.yaml /etc/netplan/50-cloud-init.yaml.back
```

---

### Wi-Fi — Enterprise (WEITI) and Home Networks

Use this method for networks requiring WPA-EAP (e.g. eduroam-style enterprise Wi-Fi).

**1. Configure wpa_supplicant:**

```bash
sudo vim /etc/wpa_supplicant/wpa_supplicant-wlan0.conf
```

```
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1
country=PL

network={
    ssid="WEITI"
    scan_ssid=1
    key_mgmt=WPA-EAP
    eap=PEAP
    identity="YOUR_WEITI_USERNAME"
    password="YOUR_WEITI_PASSWORD"
    phase2="auth=MSCHAPV2"
}

network={
    ssid="YOUR_WIFI"
    psk="YOUR_WIFI_PASSWORD"
}
```

**2. Configure systemd-networkd for wlan0:**

```bash
sudo vim /etc/systemd/network/25-wlan0.network
```

```ini
[Match]
Name=wlan0

[Network]
DHCP=ipv4

[DHCPv4]
RouteMetric=10
UseDNS=true
```

**3. Enable and start wpa_supplicant:**

```bash
sudo systemctl unmask wpa_supplicant@wlan0
sudo systemctl enable wpa_supplicant@wlan0
sudo systemctl restart wpa_supplicant@wlan0
sudo systemctl restart systemd-networkd
```

---

### Optional: Netplan-based Wi-Fi

An alternative to the wpa_supplicant method above. Useful for simpler setups.

```bash
sudo vim /etc/netplan/99-wifi-conn.yaml
```

```yaml
network:
  version: 2
  renderer: networkd
  wifis:
    wlan0:
      dhcp4: true
      optional: true
      regulatory-domain: "PL"
      access-points:
        "YOUR_WIFI":
          auth:
            key-management: "psk"
            password: "YOUR_WIFI_PASSWORD"
        "WEITI":
          hidden: false
          auth:
            key-management: eap
            method: peap
            identity: "YOUR_WEITI_USERNAME"
            password: "YOUR_WEITI_PASSWORD"
            phase2-auth: mschapv2
```

```bash
sudo chmod 600 /etc/netplan/99-wifi-conn.yaml
```

---

### IP Forwarding & NAT

Enable IP forwarding and set up NAT so the Ethernet-connected cluster can route through Wi-Fi.

```bash
# Enable IP forwarding
sudo sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf

# Set up NAT and forwarding rules
sudo iptables -t nat -A POSTROUTING -s 192.168.67.0/24 -o wlan0 -j MASQUERADE
sudo iptables -A FORWARD -i eth0 -o wlan0 -j ACCEPT
sudo iptables -A FORWARD -i wlan0 -o eth0 -m state --state ESTABLISHED,RELATED -j ACCEPT
```

> **Persist iptables rules across reboots**
>
> Run the following to save rules persistently. **Do not save while MicroK8s is running** — it adds its own rules that should not be persisted.
>
> ```bash
> sudo apt install iptables-persistent
> sudo netfilter-persistent save
> ```

---

## Kernel Module Installation

### GTP5G

```bash
# Install dependencies
sudo apt -y install gcc g++ cmake autoconf libtool pkg-config libmnl-dev libyaml-dev
sudo apt -y install linux-headers-$(uname -r)

# Clone and build
git clone -b v0.9.15 https://github.com/free5gc/gtp5g.git
cd gtp5g
make clean && make
sudo make install

# Load module
sudo modprobe gtp5g

# Persist across reboots
echo "gtp5g" | sudo tee /etc/modules-load.d/gtp5g.conf
```

### IPvlan

```bash
sudo modprobe ipvlan
echo "ipvlan" | sudo tee /etc/modules-load.d/ipvlan.conf
```

**Verify both modules are loaded:**

```bash
lsmod | grep ipvlan
lsmod | grep gtp5g
```

---

## MicroK8s Setup

### Install & Configure

```bash
# Install MicroK8s and kubectl
sudo snap install microk8s --classic --channel=1.28/stable
sudo snap install kubectl --classic

# Add user to microk8s group
sudo usermod -aG microk8s $USER
newgrp microk8s

# Enable required addons
git config --global --add safe.directory /snap/microk8s/current/addons/community/.git
microk8s enable community
microk8s enable multus
microk8s enable hostpath-storage

# Set up kubeconfig
mkdir -p ~/.kube
chmod 0700 ~/.kube
microk8s config > ~/.kube/config
```

---

### Offline / Static IP Mode

Use this when the machine has no internet access or must bind to the static IP only.

```bash
# Refresh TLS certificates
sudo microk8s refresh-certs --cert server.crt

# Point kubeconfig to the static IP
sudo microk8s config | sed 's/https:\/\/.*:16443/https:\/\/192.168.67.67:16443/' > ~/.kube/config

# Bind kubelet and API server to the static IP
echo "--node-ip=192.168.67.67" | sudo tee -a /var/snap/microk8s/current/args/kubelet > /dev/null
echo "--advertise-address=192.168.67.67" | sudo tee -a /var/snap/microk8s/current/args/kube-apiserver > /dev/null

# Restart MicroK8s to apply
microk8s stop && microk8s start

# Set Calico to use eth0 for IP autodetection
kubectl set env daemonset/calico-node -n kube-system IP_AUTODETECTION_METHOD=interface=eth0
```

---

### CNI Network Config

Edit the Calico CNI config to allow IP forwarding inside pods:

```bash
vim /var/snap/microk8s/current/args/cni-network/cni.yaml
```

Locate the `plugins` array inside the `cni_network_config` field of the `ConfigMap` and append the `container_settings` block:

```json
{
  "type": "calico",
  "kubernetes": {
    "kubeconfig": "__KUBECONFIG_FILEPATH__"
  },
  "container_settings": {
    "allow_ip_forwarding": true
  }
}
```

Then allow unsafe sysctls for IP forwarding in kubelet:

```bash
echo "--allowed-unsafe-sysctls=net.ipv4.ip_forward" | sudo tee -a /var/snap/microk8s/current/args/kubelet > /dev/null
```

Apply the updated CNI config:

```bash
kubectl apply -f /var/snap/microk8s/current/args/cni-network/cni.yaml
```

---

### Adding a Worker Node

On the primary node, generate a join command:

```bash
microk8s add-node
```

On the worker node, run the generated command with the `--worker` flag:

```bash
microk8s join <generated-join-address> --worker
```
