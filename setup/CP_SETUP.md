# Control Plane (CP) Machine Configuration

Setup guide for the **free5GC Control Plane** node — a Raspberry Pi running Ubuntu 24.04 at `192.168.67.69`, routing through the UPF at `192.168.67.67`.

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [System Update](#system-update)
3. [Network Configuration](#network-configuration)
   - [Static IP (Ethernet)](#static-ip-ethernet)
   - [Optional: Wi-Fi](#optional-wi-fi)
4. [Kernel Module Installation](#kernel-module-installation)
5. [MicroK8s Setup](#microk8s-setup)
   - [Install & Configure](#install--configure)
   - [Troubleshooting: Network Change Fix](#troubleshooting-network-change-fix)
   - [CNI Network Config](#cni-network-config)
6. [Persistent Storage](#persistent-storage)
7. [free5GC Helm Deployment](#free5gc-helm-deployment)
8. [Cluster Bootstrap](#cluster-bootstrap)
   - [Add Nodes & Label Them](#add-nodes--label-them)
   - [Deploy with Helm](#deploy-with-helm)
9. [Tests & Debugging](#tests--debugging)

---

## Prerequisites

- Static IP: `192.168.67.69/24`
- Gateway (UPF): `192.168.67.67`
- Ethernet interface: `eth0`
- Username used in paths below: `truskawka` — replace with your own

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
        - 192.168.67.69/24
      routes:
        - to: default
          via: 192.168.67.67
      nameservers:
        addresses: [8.8.8.8]
      dhcp4: false
      dhcp6: false
```

**3. Apply and clean up:**

```bash
sudo chmod 600 /etc/netplan/99-static-ip.yaml
sudo mv /etc/netplan/50-cloud-init.yaml /etc/netplan/50-cloud-init.yaml.backup
sudo netplan try
```

---

### Optional: Wi-Fi

> **Not recommended.** It is better to first set up the UPF and route internet through it. Only use this if the UPF is not yet available.

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
```

```bash
sudo chmod 600 /etc/netplan/99-wifi-conn.yaml
sudo netplan try
```

---

## Kernel Module Installation

```bash
# Install build dependencies
sudo apt -y install gcc g++ cmake autoconf libtool pkg-config libmnl-dev libyaml-dev
sudo apt -y install linux-headers-$(uname -r)

# Build and install GTP5G
git clone -b v0.9.15 https://github.com/free5gc/gtp5g.git
cd gtp5g
make clean && make
sudo make install

# Load all required modules
sudo modprobe gtp5g
sudo modprobe ipvlan
sudo modprobe sctp

# Persist across reboots
echo "ipvlan" | sudo tee /etc/modules-load.d/ipvlan.conf
echo "gtp5g"  | sudo tee /etc/modules-load.d/gtp5g.conf
echo "sctp"   | sudo tee /etc/modules-load.d/sctp.conf
```

> **Note:** The original file had typos in the module names (`gt5g`, `spctp`). The corrected names above (`gtp5g`, `sctp`) should be used.

**Verify all modules are loaded:**

```bash
lsmod | grep gtp5g
lsmod | grep ipvlan
lsmod | grep sctp
```

---

## MicroK8s Setup

### Install & Configure

```bash
sudo snap install microk8s --classic --channel=1.28/stable
sudo snap install kubectl --classic
sudo snap install helm --classic

sudo usermod -aG microk8s $USER
newgrp microk8s

git config --global --add safe.directory /snap/microk8s/current/addons/community/.git
microk8s enable community
microk8s enable multus
microk8s enable hostpath-storage
microk8s enable dns

mkdir -p ~/.kube
chmod 0700 ~/.kube
sudo microk8s config > ~/.kube/config
```

---

### Troubleshooting: Network Change Fix

If MicroK8s was installed on a different network than the one currently active (e.g. the Wi-Fi SSID changed), run the following to re-bind to the static IP:

```bash
sudo microk8s refresh-certs --cert server.crt

sudo microk8s config | sed 's/https:\/\/.*:16443/https:\/\/192.168.67.69:16443/' > ~/.kube/config
echo "--node-ip=192.168.67.69" | sudo tee -a /var/snap/microk8s/current/args/kubelet > /dev/null
echo "--advertise-address=192.168.67.69" | sudo tee -a /var/snap/microk8s/current/args/kube-apiserver > /dev/null

kubectl set env daemonset/calico-node -n kube-system IP_AUTODETECTION_METHOD=interface=eth0
kubectl rollout restart deployment coredns -n kube-system
microk8s stop && microk8s start
```

---

### CNI Network Config

Edit the Calico CNI config to allow IP forwarding inside pods:

```bash
vim /var/snap/microk8s/current/args/cni-network/cni.yaml
```

Locate the `plugins` array inside the `cni_network_config` field of the `ConfigMap` and append `container_settings`:

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

Permit the unsafe sysctl in kubelet and apply:

```bash
echo "--allowed-unsafe-sysctls=net.ipv4.ip_forward" | sudo tee -a /var/snap/microk8s/current/args/kubelet > /dev/null

kubectl apply -f /var/snap/microk8s/current/args/cni-network/cni.yaml
microk8s stop && microk8s start
```

---

## Persistent Storage

Create local directories and PersistentVolume manifests for MongoDB and TLS certificates.

```bash
mkdir -p ~/storage5g/mongo
mkdir -p ~/storage5g/cert
```

**MongoDB PersistentVolume** (`~/storage5g/persistent-vol-for-mongodb.yaml`):

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: free5gc-pv-mongo
  labels:
    project: free5gc
spec:
  capacity:
    storage: 8Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: microk8s-hostpath
  local:
    path: /home/truskawka/storage5g/mongo  # update to your username
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: free5gc-role
          operator: In
          values:
          - rpi-cp
```

**Certificate PersistentVolume** (`~/storage5g/persistent-vol-for-cert.yaml`):

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: free5gc-pv-cert
  labels:
    project: free5gc
spec:
  capacity:
    storage: 2Mi
  accessModes:
  - ReadOnlyMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: microk8s-hostpath
  local:
    path: /home/truskawka/storage5g/cert  # update to your username
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: free5gc-role
          operator: In
          values:
          - rpi-cp
```

Apply both:

```bash
kubectl apply -f ~/storage5g/persistent-vol-for-mongodb.yaml
kubectl apply -f ~/storage5g/persistent-vol-for-cert.yaml

# Verify
kubectl get pv
```

To remove:

```bash
kubectl delete -f ~/storage5g/persistent-vol-for-mongodb.yaml
kubectl delete -f ~/storage5g/persistent-vol-for-cert.yaml
```

---

## free5GC Helm Deployment

```bash
cd ~
git clone https://github.com/PetraCore/free5gc-helm.git
```

> **Important:** Before deploying, update the UPF IP address in the Helm chart values to match the UPF's Wi-Fi interface IP.

---

## Cluster Bootstrap

Run these steps **after all machines are up and joined to the cluster**.

### Add Nodes & Label Them

```bash
# Generate join token (run on this node)
microk8s add-node

# Verify all nodes are visible
microk8s kubectl get nodes

# Label nodes by role
kubectl label nodes truskawka5g free5gc-role=rpi-cp
kubectl label nodes malinka5g    free5gc-role=rpi-upf
kubectl label nodes ueransim1    free5gc-role=ueransim1
kubectl label nodes ueransim2    free5gc-role=ueransim2
```

---

### Deploy with Helm

```bash
# User Plane (UPF)
cd ~/free5gc-helm/charts/free5gc/charts/
helm upgrade --install userplane -n up \
             --create-namespace \
             free5gc-upf

# Control Plane
cd ~/free5gc-helm/charts/
helm upgrade --install controlplane -n cp \
             --create-namespace \
             free5gc

# UERANSIM (single UE)
helm upgrade --install gdbue -n ue \
             --create-namespace \
             ueransim

# UERANSIM (multiple UEs)
helm upgrade --install gdbues -n ue \
             --create-namespace \
             ueransim-many
```

Check pod status:

```bash
kubectl get pods -n up -o wide
kubectl get pods -n cp -o wide
kubectl get pods -n ue -o wide
```

Remove all namespaces (full teardown):

```bash
kubectl delete ns cp
kubectl delete ns up
kubectl delete ns ue
```

---

## Tests & Debugging

**Access the free5GC WebUI:**

```bash
kubectl port-forward -n cp deployment/controlplane-free5gc-webui-webui --address 0.0.0.0 5000:5000
```

Then open `http://192.168.67.69:5000` in a browser.

**Connectivity tests from UE pod:**

```bash
kubectl exec <ue-pod> -n ue -it -- traceroute -i uesimtun0 8.8.8.8
kubectl exec <ue-pod> -n ue -it -- ping -I uesimtun0 8.8.8.8
```

**Packet capture on UPF pod:**

```bash
kubectl exec <upf-pod> -n up -it -- tcpdump -i any icmp
```

**MongoDB issues — restart Multus:**

```bash
kubectl rollout restart daemonset kube-multus-ds -n kube-system
```

**TODO: apply to pod configs at deploy time (currently manual):**

```bash
kubectl exec <upf-pod> -n up -it -- sh -c "echo 'nameserver 8.8.8.8' > /etc/resolv.conf"
kubectl exec <upf-pod> -n up -it -- apt install tcpdump
kubectl exec <ue-pod>  -n ue -it -- apt install traceroute
```
