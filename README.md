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
Guest Port: 22\

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
configure n6 interface for upf DN connection:
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

configure default interface (first interface) as the masterIf for other(n2, n3, n4 and n9):
```
:%s/eth0/enp0s3/gc
```
for now I leave the default IP configurations (10.100.50.0/24) of n2, n3, n4 and n9 unchanged because I deploy all 5g core and gNB functions on one kubernetes node/VM. This may change if you need to deploy them over a cubernetes cluster.

### 4.3 configure persistent volume:
configure 8G of persistent volume with [this file](https://github.com/Orange-OpenSource/towards5gs-helm/blob/main/docs/demo/Setup-free5gc-and-test-with-UERANSIM.md#create-a-persistent-volume)
















