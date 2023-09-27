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
### 3.5 download free5gc helm charts and configurations:
download [free5gc helm charts](https://github.com/Orange-OpenSource/towards5gs-helm)
```
git clone https://github.com/Orange-OpenSource/towards5gs-helm.git
```












