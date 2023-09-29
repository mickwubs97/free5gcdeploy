delete:
```
kubectl delete -n free5gc pod,svc --all \
&& kubectl delete ns free5gc \
&& kubectl delete PersistentVolume free5gc-pv0 \
&& cd -- \
&& sudo rm -rf kubedata/
```
start free5gc:
```
mkdir kubedata \
&& kubectl create ns free5gc \
&& kubectl apply -f persistentvolume-definition.yml \
&& cd towards5gs-helm/charts/ \
&& helm -n free5gc install free5gc-v1 ./free5gc/ \
&& watch kubectl get pods -n free5gc
```
start free5gc&ueransim:
```
mkdir kubedata \
&& kubectl apply -f persistentvolume-definition.yml \
&& kubectl create ns free5gc \
&& cd towards5gs-helm/charts/ \
&& helm -n free5gc install free5gc-v1 ./free5gc/ \
&& helm -n free5gc install ueransim-v1 ./ueransim/ \
&& watch kubectl get pods -n free5gc
```
start ueransim only:
```
cd --\
&& cd towards5gs-helm/charts/ \
&& helm -n free5gc install ueransim-v1 ./ueransim/
```



