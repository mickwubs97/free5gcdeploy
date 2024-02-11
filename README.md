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
enable kube-ovn:
```
sudo microk8s enable kube-ovn --force
```
here I use kube-ovn CNI to avoid the [ipv4_forwarding problem](https://github.com/canonical/microk8s/issues/1989) in the calico CNI

join Microk8s cluster (make sure to join the cluster _after_ enable kube-ove otherwise you may lose control of the cluster):
```
sudo usermod -a -G microk8s $USER\
&& sudo chown -f -R $USER ~/.kube
```
```
su - $USER
```
enable multus:
```
microk8s enable community\
&& microk8s enable multus
```
(re)enable dns:
```
microk8s disable dns\
&& microk8s enable dns
```
when Microk8s is ready, type ``` microk8s status``` to check, it should have these addons enabled:
```
microk8s is running
high-availability: no
datastore master nodes: 127.0.0.1:19001
datastore standby nodes: none
addons:
  enabled:
    multus               # (community) Multus CNI enables attaching multiple network interfaces to pods
    community            # (core) The community addons repository
    dns                  # (core) CoreDNS
    ha-cluster           # (core) Configure high availability on the current node
    helm                 # (core) Helm - the package manager for Kubernetes
    helm3                # (core) Helm 3 - the package manager for Kubernetes
    kube-ovn             # (core) An advanced network fabric for Kubernetes
  disabled:
    ...
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
for now I leave the default IP configurations (10.100.50.0/24) of n2, n3, n4 and n9 unchanged because I deploy all 5g core and gNB functions on one kubernetes node/VM. This may change if you need to deploy them over a kubernetes cluster.

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

#### test upf:
check upf interfces:
```
kubectl exec -it -n free5gc [upf pod id] -- ip a
```
it should give somwthing like this (with different ip addresses):
```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
3: eth0@if166: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default
    link/ether 2e:83:7a:d0:2c:c5 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.1.214.169/32 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::2c83:7aff:fed0:2cc5/64 scope link
       valid_lft forever preferred_lft forever
4: n3@if2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default
    link/ether 08:00:27:df:4e:37 brd ff:ff:ff:ff:ff:ff
    inet 10.100.50.233/29 brd 10.100.50.239 scope global n3
       valid_lft forever preferred_lft forever
    inet6 fe80::800:2700:3df:4e37/64 scope link
       valid_lft forever preferred_lft forever
5: n6@eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default
    link/ether 08:00:27:3d:1c:53 brd ff:ff:ff:ff:ff:ff
    inet 10.0.3.12/24 brd 10.0.3.255 scope global n6
       valid_lft forever preferred_lft forever
    inet6 fe80::800:2700:13d:1c53/64 scope link
       valid_lft forever preferred_lft forever
6: n4@if2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default
    link/ether 08:00:27:df:4e:37 brd ff:ff:ff:ff:ff:ff
    inet 10.100.50.241/29 brd 10.100.50.247 scope global n4
       valid_lft forever preferred_lft forever
    inet6 fe80::800:2700:4df:4e37/64 scope link
       valid_lft forever preferred_lft forever
7: upfgtp: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> mtu 1464 qdisc noqueue state UNKNOWN group default qlen 1000
    link/none
    inet6 fe80::8ec6:e4da:9c15:9866/64 scope link stable-privacy
       valid_lft forever preferred_lft forever
```
check n6 connection to the DN:
```
kubectl exec -it -n free5gc [upf pod id] -- ping -I n6 8.8.8.8
```
it should work:
```
PING 8.8.8.8 (8.8.8.8): 56 data bytes
64 bytes from 8.8.8.8: seq=0 ttl=112 time=10.474 ms
64 bytes from 8.8.8.8: seq=1 ttl=112 time=17.193 ms
...
```
to test uesimtun0 form ue, add ue profile to amf with webui service.\
[forward webui service to host machine](https://free5gc.org/blog/IntroduceKubernetesAndDeploymentfree5GConKubernetesWithHelm/main/#start-webconsole)
```
kubectl port-forward --namespace free5gc svc/webui-service 5000:5000
```
listen to port 5000 from host:
```
ssh -L localhost:5000:localhost:5000 -p 8022 ubuntu@127.0.0.1
```
access webui form host's broswer: \
http://localhost:5000

login and add a new subscriber.

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
deploy ueransim:
```
cd --\
&& cd towards5gs-helm/charts/ \
&& helm -n free5gc install ueransim-v1 ./ueransim/
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

[check ipv4_forwarding](https://github.com/Orange-OpenSource/towards5gs-helm/blob/main/docs/demo/Setup-free5gc-and-test-with-UERANSIM.md#tun-interface-correctly-created-on-the-ue-but-internet):
```
kubectl exec -it -n [upf pod id] -- cat /proc/sys/net/ipv4/ip_forward
```
it should give:
```
1
```
otherwise, check document [here](https://github.com/Orange-OpenSource/towards5gs-helm/blob/main/docs/demo/Setup-free5gc-and-test-with-UERANSIM.md#tun-interface-correctly-created-on-the-ue-but-internet) and solution [here](https://github.com/canonical/microk8s/issues/1989) or use microk8s kube-ovn as CNI.
check uesimtun0:
```
kubectl exec -it -n [ue pod id] -- ip a
```
output should have uesimtun0 in the interfeaces
test uesimtun0:
```
kubectl exec -it -n free5gc ueransim-v1-ue-65985c8bfd-8w5x5 -- ping -I uesimtun0 8.8.8.8
```
and it should work:
```
PING 8.8.8.8 (8.8.8.8) from 10.1.0.1 uesimtun0: 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=111 time=12.5 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=111 time=10.7 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=111 time=12.2 ms
64 bytes from 8.8.8.8: icmp_seq=4 ttl=111 time=13.3 ms
...
```

## Additional
to stop free5gc for this deployment, run:
```
kubectl delete -n free5gc pod,svc --all \
&& kubectl delete -n free5gc deployment --all \
&& kubectl delete ns free5gc \
&& kubectl delete PersistentVolume free5gc-pv0 \
&& cd -- \
&& sudo rm -rf kubedata/
```



















