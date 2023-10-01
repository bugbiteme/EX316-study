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
## Creating a service between two running VMIs
- given that each VMI has the label app=web

(example vm config under "template")
```
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: web <----------(LOOK)
        flavor.template.kubevirt.io/small: "true"
```
- create a `service` of type `ClusterIP` with a selector based on `app: web`

```
apiVersion: v1
kind: Service
metadata:
  name: web
  namespace: my-namespace
spec:
  type: ClusterIP
  selector:
    app: web
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
```

Verify two endpoints

```
$ oc get endpoints
NAME   ENDPOINTS                    AGE
web    10.11.0.28:80,10.8.2.20:80   2m13s
```
## add a cookie annotation to a route object
```
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  annotations:
    router.openshift.io/cookie_name: web
```

- calling the url using the cookie with curl
```
(create cookie)
$ curl web.apps.ocp4.example.com -c /tmp/my_cookie
Welcome to www1

(use cookie)
$ curl web.apps.ocp4.example.com -b /tmp/my_cookie
Welcome to www1
```
## Adding a readiness probe to a VM that uses HTTP GET requests to test the /health 
https://docs.openshift.com/container-platform/4.10/applications/application-health.html
```
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: health-check
  name: my-application
spec:
  containers:
  - name: my-container 
    args:
    image: k8s.gcr.io/goproxy:0.1 
    readinessProbe: 
      httpGet: 
        path: /health
        port: 80
      failureThreshold: 2
      successThreshold: 1
      periodSeconds: 5
      timeoutSeconds: 2 
```

## Adding a i6300esb watchdog device to a VM to power off if OS is unresponsive
https://docs.openshift.com/container-platform/4.10/virt/virtual_machines/advanced_vm_management/virt-configuring-a-watchdog.html

```
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  labels:
    kubevirt.io/vm: vm2-rhel84-watchdog
  name: <vm-name>
spec:
  running: false
  template:
    spec:
      domain:
        devices:
          watchdog:
            name: <watchdog>
            i6300esb:
              action: "poweroff" 
...
```

## A VM listens on TCP port 3306. Add a liveness probe that tests the service by sending requests to the TCP socket
https://docs.openshift.com/container-platform/4.10/applications/application-health.html
```
# ...
spec:
  domain:
     ....stuff....
  livenessProbe:
    initialDelaySeconds: 2 
    periodSeconds: 2
    tcpSocket: 
      port: 3306 
    timeoutSeconds: 10 
# ...
```

## Extra things you may want to brush up on:
- whenever you make changes to a VMs yaml config, save and reload to ensure your changes are still there.  
  Sometimes, if there is a mistake, it will be erased, and you will wonder why your probes aren't working.
- one trick for creating readiness and liveness probes for VMs is to create a deployment, and use the form to creat
  them, and then copy/paste it to vm config. Make sure `ReadienessProbe` is lowercase `readinessProbe`
- Installing a yum repo in Linux
```
$ sudo yum-config-manager --add-repo http://www.example.com/example.repo
```
- user and group policy management in OpenShift
