# free5GC-helm

[![Helm charts linting](https://github.com/free5gc/free5gc-helm/actions/workflows/helm-charts-testing.yml/badge.svg)](https://github.com/free5gc/free5gc-helm/actions/workflows/helm-charts-testing.yml)

***free5gc-helm*** is an open-source project implemented to provide helm charts in order to deploy on one click a 5G system (RAN+SA 5G core) on top of Kubernetes. It currently relies on free5GC for the core network and UERANSIM to simulate Radio Access Network.

## Documentation
The documentation is available on the [free5GC official site](https://free5gc.org/guide/7-free5gc-helm/).

## Motivations
Please consult this [link](/motivations.md) to see the motivations that have led to this project.

## Contributing
Moving towards a Cloud native model for the 5G system is not a simple task. We welcome all new [contributions](./CONTRIBUTING.md) making this project better!

## Acknowledgement
Thanks to both Free5GC and UERANSIM teams for their great efforts.

## License
***free5gc-helm*** is under [Apache 2.0](./LICENSE) license.

## Citation
Text format:
```
A. Khichane, I. Fajjari, N. Aitsaadi and M. Gueroui, "Cloud Native 5G: an Efficient Orchestration of Cloud Native 5G System," NOMS 2022-2022 IEEE/IFIP Network Operations and Management Symposium, Budapest, Hungary, 2022, pp. 1-9, doi: 10.1109/NOMS54207.2022.9789856.
```
BibTex:
```
@INPROCEEDINGS{9789856,
  author={Khichane, Abderaouf and Fajjari, Ilhem and Aitsaadi, Nadjib and Gueroui, Mourad},
  booktitle={NOMS 2022-2022 IEEE/IFIP Network Operations and Management Symposium},
  title={Cloud Native 5G: an Efficient Orchestration of Cloud Native 5G System},
  year={2022},
  volume={},
  number={},
  pages={1-9},
  doi={10.1109/NOMS54207.2022.9789856}}
```

# Deployment tutorial

## Used VMs

free5gc_host:

    enp0s3: 15.0.2.0/24 (NAT)
    enp0s8: 192.168.0.0/24 (internal network)
    RAM: 6144 MB (can be adjusted)

ueransim_host:

    enp0s3: 15.0.2.0/24 (NAT)
    enp0s8: 192.168.0.0/24 (internal network, same as free5gc_host enp0s8)
    RAM: 4096 MB (can be adjusted)
    
On both hosts enable promiscous mode on enp0s8

-> Promiscous mode: "Allow all"

I set enp08s interface on both machines through Ubuntu UI. Use manual configuration:
- free5gc_host: 192.168.0.1 255.255.255.0
- ueransim_host: 
  - 192.168.0.2 255.255.255.0
  - 10.100.50.236 255.255.255.248
  - 10.100.50.250 255.255.255.248

## free5gc_host

Make sure you have your system updated:

```
sudo apt update && sudo apt upgrade
```

Check if you have AVX enabled (needed for MongoDB to work):
```
grep avx /proc/cpuinfo 
```

If not:
```
VBoxManage setextradata "[YOUR_VM_NAME]" VBoxInternal/CPUM/IsaExts/AVX 1
VBoxManage setextradata "[YOUR_VM_NAME]" VBoxInternal/CPUM/IsaExts/AVX2 1
```

If you are on Windows, you might have to disable hyper-v:
```
bcdedit /set hypervisorlaunchtype off
```
(after this step, restart your system)

Install basic tools:
```
sudo apt install vim git
```

Make a directory for all files related to free5gc:
```
cd
mkdir net5g
cd net5g
```

We will need [gtp5g](https://github.com/free5gc/gtp5g) for the UPF to work. Free5gc helm does not support the latest version so we will run v0.9.15:


```
git clone -b v0.9.15 https://github.com/free5gc/gtp5g.git

sudo apt -y install gcc g++ cmake autoconf libtool pkg-config libmnl-dev libyaml-dev

cd gtp5g
make clean && make
sudo make install
```

It should now be visible here:
```
lsmod | grep gtp
```

Now we can proceed to install free5gc. We will mostly be following [official intructions](https://free5gc.org/guide/7-free5gc-helm/): 

Install neccessary tools:
```
sudo snap install microk8s --classic --channel=1.28/stable
sudo snap install kubectl --classic
sudo snap install helm --classic
```

Set sudo group and join:
```
sudo groupadd microk8s
sudo usermod -aG microk8s $USER
newgrp microk8s
```

Set microk8s work with local kubectl:
```
mkdir -p ~/.kube
chmod 0700 ~/.kube
microk8s config > ~/.kube/config
```

Setup Calico CNI for IP forwarding:
```
vim /var/snap/microk8s/current/args/cni-network/cni.yaml
```

Here is a snippet of how the config should look:
```
...
kind: ConfigMap
...
data:
    ...
    cni_network_config: |-
        {
            ...
            "plugins": [
                {
                    "type": "calico",
                    ...
                    "kubernetes": {
                        "kubeconfig": "__KUBECONFIG_FILEPATH__"
                    },
                    # append IP forwarding settings
                    "container_settings": {
                        "allow_ip_forwarding": true
                    },
                }
            ]
        }
```

Setup kubelet args for IP fowarding:
```
vim /var/snap/microk8s/current/args/kubelet
```

Add this:
```
# append this arg
--allowed-unsafe-sysctls "net.ipv4.ip_forward"
```

Apply settings and restart MicroK8s:
```
# apply cni configuration
kubectl apply -f /var/snap/microk8s/current/args/cni-network/cni.yaml
# restart MicroK8s
microk8s stop
microk8s start
```

Enable addons:
```
# If enabling community does not work, use this:
git config --global --add safe.directory /snap/microk8s/current/addons/community/.git

microk8s enable community
microk8s enable multus
microk8s enable hostpath-storage
```


Setup for persistent volumes:
```
cd ~/net5g
mkdir mongo
mkdir cert
```

```
vim persistent-vol-for-mongodb.yaml
```

Add content:
```
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
    path: /home/student/net5g/mongo # put your own path here
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - proj # edit to you node name (= hostname)
```

```
vim persistent-vol-for-cert.yaml
```

Add content
```
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
    path: /home/student/net5g/cert # put your own path here
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - proj # edit to you node name (= hostname)
```

Apply via kubectl:
```
kubectl apply -f persistent-vol-for-mongodb.yaml
kubectl apply -f persistent-vol-for-cert.yaml
```

Check:
```
kubectl get pv
```

Clone this helm chart:
```
cd ~/net5g
git clone https://github.com/PetraCore/free5gc-helm.git
```

I will be showing how to run control-plane and user-plane on separate namespaces, but you are free to create just one (e.g. "free5gc") and deploy everything there

Create namespaces
```
kubectl create ns cp
kubectl create ns up
```

Install UPF on user-plane namespace
```
cd ~/net5g/free5gc-helm/charts/free5gc/charts/
helm install userplane -n up free5gc-upf
```

It should be set to "Running" after a while:
```
kubectl get pods -n up
```

Install control-plane network functions:
```
cd ~/net5g/free5gc-helm/charts/
helm upgrade --install controlplane -n cp \
--set deployUpf=false \
free5gc
```

Check if everything is "Running". It might take a while. When in doubt, you can always reboot your VM and/or restart microk8s.
```
kubectl get pods -n cp
```

Check port of webui-service (most probably 30500)
```
kubectl get services -n cp
```

Now open a web browser, connect to 127.0.0.1:[webui_port] and login:
- username: admin
- password: free5gc

Add a new subscriber with default parameters (subscribers -> create -> create).

And that is everything on this VM!

## ueransim_host

Make sure you have your system updated:

```
sudo apt update && sudo apt upgrade
```

Install basic tools:
```
sudo apt install vim git
```

Create directory for all the files:
```
cd ~
mkdir net5g
cd net5g
```

Now we will follow official [UERANSIM installation](https://github.com/aligungr/UERANSIM/wiki/Installation):

Install dependencies:
```
sudo apt install make gcc g++ libsctp-dev lksctp-tools iproute2
sudo snap install cmake --classic
```

Clone UERANSIM repo and build:
```
cd ~/net5g/
git clone https://github.com/aligungr/UERANSIM.git
cd UERANSIM
make
```

Add UERANSIM to Path (repeat for root user):
```
cd
vim .bashrc
```

Update content (repeat for root user):
```
# append this to the end
# you most probably need to change "student" to your own username
export PATH="$PATH:/home/student/net5g/UERANSIM/build"
```

Update path (repeat for root user):
```
source ~/.bashrc
```
When typing for example "nr-ue", you should be able to access the command.

Clone this repo for the config files:
```
cd ~/net5g/
git clone https://github.com/PetraCore/free5gc-helm.git
cp free5gc-helm/ueransim/* .
```

And your setup should be complete! Now you can try connecting to 5g core network (assuming free5gc_host is running along with free5gc itself)

Start the gNB:
```
cd ~/net5g
nr-gnb -c gnbconfig.yaml
```
It should successfully establish SCTP connection


Start a UE (in a separate terminal):
```
cd ~/net5g
sudo su
nr-ue -c ueconfig.yaml
```
It should successfully connect to gNB and create uesimtun0 interface.

Open another terminal and check UE's connection to the internet:
```
ping -I uesimtun0 8.8.8.8
ping -I uesimtun0 google.com
```

If those pings are successful, that means you properly configured the network. Congrats!
