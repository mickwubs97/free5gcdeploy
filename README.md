# free5gcdeploy
A helm deployment procedure for [free5gc](https://github.com/Orange-OpenSource/towards5gs-helm)

## 1. VM Image:
[Ubuntu 20.04 LTS Server] (https://www.releases.ubuntu.com/focal/)

## 2. VM Setup:
4 GPU, 10GB RAM, 25GB virtual hard disk, two NAT network adapters

## 3. VM Configurations:
### 3.1 Kernal Version Check:
```
uname -r
```
output:
```
5.4.0-163-generic
```
### 3.2 Network Configuration:
port forwarding:\
Host Address:\
Host Port: 8022\
Guest Address: [address for enp0s3/eth0/ens3]\
Guest Port: 22

configure _another_ NAT network interface

ssh into Ubuntu Server:
```
ssh -p 8022 ubuntu@127.0.0.1
```
check VM networks:
```
ip a
```
output:
```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:df:4e:37 brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic enp0s3
       valid_lft 86364sec preferred_lft 86364sec
    inet6 fe80::a00:27ff:fedf:4e37/64 scope link
       valid_lft forever preferred_lft forever
3: enp0s8: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 08:00:27:3d:1c:53 brd ff:ff:ff:ff:ff:ff
```
configure netplan:
```
sudo vim /etc/netplan/00-installer-config.yaml
```
```
network:
  ethernets:
    enp0s3:
      dhcp4: true
    enp0s8:
      dhcp4: true
  version: 2
```
```
sudo netplan try
```
check configurations:
```
ip a
```
output:
```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:df:4e:37 brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic enp0s3
       valid_lft 86396sec preferred_lft 86396sec
    inet6 fe80::a00:27ff:fedf:4e37/64 scope link
       valid_lft forever preferred_lft forever
3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:3d:1c:53 brd ff:ff:ff:ff:ff:ff
    inet 10.0.3.15/24 brd 10.0.3.255 scope global dynamic enp0s8
       valid_lft 86396sec preferred_lft 86396sec
    inet6 fe80::a00:27ff:fe3d:1c53/64 scope link
       valid_lft forever preferred_lft forever
```
now we have two VM network interfaces belong to two _different_ subnets.

### 3.3 Ready Microk8s and Multus
install [Microk8s](https://microk8s.io/docs/getting-started)
```
sudo snap install microk8s --classic --channel=1.28
```
configure Microk8s:
```
sudo usermod -a -G microk8s $USER\
&& sudo chown -f -R $USER ~/.kube
```
```
su - $USER
```
install kubectl:
```
 sudo snap install kubectl --classic
```
install helm:
```
sudo snap install helm --classic
```
[enable microk8s to work with VM's kubectl](https://microk8s.io/docs/working-with-kubectl):
```
cd $HOME
cd .kube
microk8s config > config
```
enable multus:
```
microk8s enable community\
&& microk8s enable multus
```
## 4. Free5GC Deplyoyment
### 4.1 gtp5g installation
install gtp5g v8.0.1 (need to be compatible with _kernel version_)
```
sudo apt install gcc
sudo apt install make
git clone -b v0.8.1 https://github.com/free5gc/gtp5g.git
cd gtp5g
make
sudo make install
```
```
cd --
```
see also [install-gtp5g](https://free5gc.org/blog/IntroduceKubernetesAndDeploymentfree5GConKubernetesWithHelm/main/#install-gtp5g)

### 4.2 download free5gc helm charts and configurations:
download [free5gc helm charts](https://github.com/Orange-OpenSource/towards5gs-helm)
```
git clone https://github.com/Orange-OpenSource/towards5gs-helm.git
```
check the _gateway_ of VM network interfeces:
```
ip r
```
output:
```
default via 10.0.3.2 dev enp0s8 proto dhcp src 10.0.3.15 metric 100
default via 10.0.2.2 dev enp0s3 proto dhcp src 10.0.2.15 metric 100
10.0.2.0/24 dev enp0s3 proto kernel scope link src 10.0.2.15
10.0.2.2 dev enp0s3 proto dhcp scope link src 10.0.2.15 metric 100
10.0.3.0/24 dev enp0s8 proto kernel scope link src 10.0.3.15
10.0.3.2 dev enp0s8 proto dhcp scope link src 10.0.3.15 metric 100
blackhole 10.1.214.128/26 proto 80
10.1.214.135 dev cali032701d8fbd scope link
10.1.214.136 dev calicd7aa8cce63 scope link
```
configure n6 _network_ for upf DN connection:
```
cd --\
&& vim towards5gs-helm/charts/free5gc/values.yaml 
```
```
n6network:
 enabled: true
 name: n6network
 type: ipvlan
 masterIf: enp0s8
 subnetIP: 10.0.3.0
 cidr: 24
 gatewayIP: 10.0.3.2
 excludeIP: 10.0.3.254
```
masterIf should be the name of the second interface, subnetIP should be the subnet directly connected to the second interfece, gatewayIP shoud be the _gateway IP_ of the second interface (not the ip of the second interface).

configure the name of the default interface (first interface) as the masterIf for other(n2, n3, n4 and n9):
```
:%s/eth0/enp0s3/gc
```
for now I leave the default IP configurations (10.100.50.0/24) of n2, n3, n4 and n9 unchanged because I deploy all 5g core and gNB functions on one kubernetes node/VM. This may change if you need to deploy them over a cubernetes cluster.

also remember to configure ip for n6 interface, it needs to be an ip address in the n6 network/subnet.
```
vim
```
vim towards5gs-helm/charts/free5gc/charts/free5gc-upf/values.yaml
```
n3if:  # GTP-U
    ipAddress: 10.100.50.233
  n4if:  # PFCP
    ipAddress: 10.100.50.241
  n6if:  # DN
    ipAddress: 10.0.3.12
```

### 4.3 configure persistent volume:
configure 8G of persistent volume with [this file](https://github.com/Orange-OpenSource/towards5gs-helm/blob/main/docs/demo/Setup-free5gc-and-test-with-UERANSIM.md#create-a-persistent-volume)
```
cd --
vim persistentvolume-definition.yml
```
persistentvolume-definition.yml:
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: [persistent volume name]
  labels:
    project: free5gc
spec:
  capacity:
    storage: 8Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  local:
    path: /home/[username]/kubedata
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - [node name]
```
in my case, [persistent volume name] is free5gc-pv0, replace [username] with your username and [node name] with the name of your kubernetes worker node.

### deployment and test with [ueransim](https://github.com/Orange-OpenSource/towards5gs-helm/tree/main/charts/ueransim)

now, deploy free5gc:
```
cd --
```
```
mkdir kubedata \
&& kubectl create ns free5gc \
&& kubectl apply -f persistentvolume-definition.yml \
&& cd towards5gs-helm/charts/ \
&& helm -n free5gc install free5gc-v1 ./free5gc/ \
&& watch kubectl get pods -n free5gc
```
if successful:
```
Every 2.0s: kubectl get pods -n free5gc                                                                                                                        free5gc: Wed Sep 27 11:28:47 2023
NAME                                                    READY   STATUS    RESTARTS   AGE
free5gc-v1-free5gc-upf-upf-555cfb4d7f-k9lxq             1/1     Running   0          64s
mongodb-0                                               1/1     Running   0          64s
free5gc-v1-free5gc-nrf-nrf-6699c64fd7-8xt9g             1/1     Running   0          64s
free5gc-v1-free5gc-dbpython-dbpython-5d46c79f86-vjpjm   1/1     Running   0          64s
free5gc-v1-free5gc-webui-webui-54f45bb69f-wvsjg         1/1     Running   0          64s
free5gc-v1-free5gc-pcf-pcf-5fc9b5f98f-5pgxh             1/1     Running   0          64s
free5gc-v1-free5gc-udm-udm-5d8d96c57b-qkcvh             1/1     Running   0          64s
free5gc-v1-free5gc-amf-amf-6fc4bf584-pbrf6              1/1     Running   0          64s
free5gc-v1-free5gc-udr-udr-6dfc5f6f74-9bwch             1/1     Running   0          64s
free5gc-v1-free5gc-smf-smf-58b7874499-72hb8             1/1     Running   0          64s
free5gc-v1-free5gc-nssf-nssf-58c6cddc69-bxq9r           1/1     Running   0          64s
free5gc-v1-free5gc-ausf-ausf-75586797cf-wpg57           1/1     Running   0          64s
```
if not, check the configrations or post an issue. 

to stop free5gc for this deployment, run:
```
kubectl delete -n free5gc pod,svc --all \
&& kubectl delete ns free5gc \
&& kubectl delete PersistentVolume free5gc-pv0 \
&& cd -- \
&& sudo rm -rf kubedata/
```
configure ueransim (n2, n3 interface):
```
vim towards5gs-helm/charts/ueransim/values.yaml
```
```
n2network:
    enabled: true
    name: n2network
    type: ipvlan
    masterIf: enp0s3
    subnetIP: 10.100.50.248
    cidr: 29
    gatewayIP: 10.100.50.254
    excludeIP: 10.100.50.254
  n3network:
    enabled: true
    name: n3network
    type: ipvlan
    masterIf: enp0s3
    subnetIP: 10.100.50.232
    cidr: 29
    gatewayIP: 10.100.50.238
    excludeIP: 10.100.50.238
```
now, deploy free5gc with ueransim:
```
mkdir kubedata \
&& kubectl apply -f persistentvolume-definition.yml \
&& kubectl create ns free5gc \
&& cd towards5gs-helm/charts/ \
&& helm -n free5gc install free5gc-v1 ./free5gc/ \
&& helm -n free5gc install ueransim-v1 ./ueransim/ \
&& watch kubectl get pods -n free5gc
```
if successful:
```
Every 2.0s: kubectl get pods -n free5gc                                                                                                                        free5gc: Wed Sep 27 11:40:41 2023
NAME                                                    READY   STATUS    RESTARTS   AGE
free5gc-v1-free5gc-upf-upf-555cfb4d7f-fk7xw             1/1     Running   0          33s
mongodb-0                                               1/1     Running   0          33s
free5gc-v1-free5gc-webui-webui-54f45bb69f-wg5rn         1/1     Running   0          33s
free5gc-v1-free5gc-dbpython-dbpython-5d46c79f86-m5lnz   1/1     Running   0          33s
free5gc-v1-free5gc-nrf-nrf-6699c64fd7-vs8b9             1/1     Running   0          33s
free5gc-v1-free5gc-pcf-pcf-5fc9b5f98f-vkhbx             1/1     Running   0          33s
free5gc-v1-free5gc-nssf-nssf-58c6cddc69-7l7j5           1/1     Running   0          33s
free5gc-v1-free5gc-ausf-ausf-75586797cf-r8xb6           1/1     Running   0          33s
free5gc-v1-free5gc-udr-udr-6dfc5f6f74-thqs7             1/1     Running   0          33s
free5gc-v1-free5gc-udm-udm-5d8d96c57b-jz6cm             1/1     Running   0          33s
free5gc-v1-free5gc-smf-smf-58b7874499-4pfc2             1/1     Running   0          33s
ueransim-v1-ue-65985c8bfd-4x9dq                         1/1     Running   0          32s
free5gc-v1-free5gc-amf-amf-6fc4bf584-nwt4r              1/1     Running   0          33s
ueransim-v1-gnb-7ccf5f7bf9-xx2rw                        1/1     Running   0          32s
```
if not, check the configrations or post an issue. 

test 

## Additional




















