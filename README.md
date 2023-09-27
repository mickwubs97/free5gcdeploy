# free5gcdeploy
A helm deployment procedure for [free5gc](https://github.com/Orange-OpenSource/towards5gs-helm)

## 1. VM Image:
[Ubuntu 20.04 LTS Server] (https://www.releases.ubuntu.com/focal/)

## 2. VM Setup:
4 GPU, 10GB RAM, 25GB virtual hard disk, two NAT network adapters

## 3. VM Configurations:
### 3.1 Version Check:
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

ssh into Ubuntu Server:
```
ssh -p 8022 ubuntu@127.0.0.1
```

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








