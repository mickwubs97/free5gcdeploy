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
port forwarding:\\
Host Address: 
Host Port: 8022
Guest Address: [address for default network interface] ``` ip a ```
Guest Port: 22




