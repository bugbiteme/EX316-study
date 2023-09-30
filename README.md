# EX316-study

## useful commands

- retrieve performance metrics
```
  watch vmstat -S M

  swpd - amount of memory used in swap space

  free - available free memory

  wa - CPU wait  
```
## Configuring Kubernetes Networking for Virtual Machines

## Connecting Virtual Machines to External Networks
Creating a Linux Bridge network on worker nodes

https://docs.openshift.com/container-platform/4.10/virt/virtual_machines/vm_networking/virt-attaching-vm-multiple-networks.html

may need to label worker nodes first and add a nodeSelector in the spec
```
oc label node worker01 network-type=external

(and then)

apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
   name: br-policy
spec:
  nodeSelector:
    network-type: "external" <-----!!!!!!
  desiredState:
    interfaces:
.
.
.
```
Creating a network attachment definition  
  
https://docs.openshift.com/container-platform/4.10/virt/virtual_machines/vm_networking/virt-attaching-vm-multiple-networks.html  
  
```
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: <bridge-network> 
  annotations:
    k8s.v1.cni.cncf.io/resourceName: bridge.network.kubevirt.io/<bridge-interface> 
spec:
  config: '{
    "cniVersion": "0.3.1",
    "name": "<bridge-network>", 
    "type": "cnv-bridge", 
    "bridge": "<bridge-interface>", 
    "macspoofchk": true, 
    "vlan": 1 
  }'
```
## Configuring Storage for Virtual Machines
- Creating a PV that declares an external NFS share
- Need to have IP or FQDN of external storage
- Need to have export path as well

- OpenShift provides the following template when you create a pv via the web console
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: example
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: slow
  nfs:
    path: /tmp
    server: 172.17.0.2
```
This needs to be updated 
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: <pv name>
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnly <------ this may need to be updated based on the spcification
  persistentVolumeReclaimPolicy: Retain  <----- remove
  storageClassName: slow <----- remove
  volumeMode: Filesystem <----- ADD!!!
  claimRef:    <----- ADD!!! directives for attaching to a particular PVC
    name: <name of PVC> <----- ADD!!! PVC name to attach to
    namespace: <name space where PVC will reside> <----- ADD!!!
  nfs:
    path: /tmp
    server: 172.17.0.2
```

Final form for PV
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: <pv name>
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteMany 
  volumeMode:  Filesystem 
  claimRef:   
    name: <name of PVC> 
    namespace: <name space where PVC will reside> 
  nfs:
    path: /<path>
    server: <ip or fqdn of nfs system>
```

for PVC creation, use the yaml template in the console to fill out the information.   
DO NOT USE `storageclass` when using external fileshare

## Restoring a snapshot volume to a PVC

- Go to Storage -> VolumeSnapshots to find the volume and restore as new PVC

## Setting a node to maintenance mode and draining workloads to remaining nodes

```
$ oc adm cordon worker02
node/worker02 cordoned

$ oc adm drain worker02 --ignore-daemonsets --delete-emptydir-data --force
.
.
.
(wait a sec)
.
node/worker02 drained
```

## Extra things you may want to brush up on:
- Installing a yum repo in Linux
- user and group policy management in OpenShift
