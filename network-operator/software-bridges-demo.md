# Notes for network-operator (sriov-network-operator) software bridge management demo

## Links

* [sriov-network-operator: switchdev mode refactoring](https://github.com/k8snetworkplumbingwg/sriov-network-operator/blob/2d2e5a45d453e9cd5b25495e8c9b1cf8c53840c6/doc/design/switchdev-refactoring.md)

* [sriov-network-operator: software bridge management](https://github.com/k8snetworkplumbingwg/sriov-network-operator/blob/2d2e5a45d453e9cd5b25495e8c9b1cf8c53840c6/doc/design/software-bridge-management.md)


## Demo environment

**Kubernetes cluster:** 1 control-plane node, 1 worker node

**OS:** Ubuntu 22.04.4 LTS

**Kernel:** 5.15.0-97-generic

**Kubernetes:** 1.29.3

**Containerd:** 1.6.28
 
**Adapters:** ConnectX-6 Dx 

**Openvswitch version:** 2.17.9 (upstream, ubuntu)

**Driver version:** inbox Ubuntu for 5.15.0-97-generic kernel

**network-operator version:** 24.4.0-beta3

## Worker node preparation

Install openvswitch-switch package

```bash
apt install openvswitch-switch -y
```

## Network-operator installation

Deploy network-operator 24.4.0-beta3 (by following the deployment guide for beta versions)

_following values for helm chart were used for the demo_
```yaml
# myvalues.yaml 

sriovNetworkOperator:
  enabled: true

sriov-network-operator:
  images:
    ovsCni: nvcr.io/nvstaging/mellanox/ovs-cni-plugin:61fd27c
  imagePullSecrets:
    - ngc-image-secret
  sriovOperatorConfig:
    configDaemonNodeSelector:
      node-role.kubernetes.io/worker: ""

imagePullSecrets: 
  - ngc-image-secret

deployCR: true

ofedDriver:
  deploy: false

nvIpam:
  deploy: true

```

```bash
helm install -n network-operator  -f myvalues.yaml network-operator network-operator
```

Create IPPool object for nv-ipam

```yaml
# pool1.yaml

apiVersion: nv-ipam.nvidia.com/v1alpha1
kind: IPPool
metadata:
  name: pool1
  namespace: network-operator
spec:
  subnet: 192.168.0.0/16
  perNodeBlockSize: 100
  gateway: 192.168.0.1
  nodeSelector:
    nodeSelectorTerms:
    - matchExpressions:
        - key: node-role.kubernetes.io/worker
          operator: Exists

```
```bash
kubectl apply -f pool1.yaml
```



Enable manageSoftwareBridges featureGate

```bash
kubectl patch sriovoperatorconfigs.sriovnetwork.openshift.io -n network-operator default --patch '{ "spec": { "featureGates": { "manageSoftwareBridges": true  } } }' --type='merge'
```


## Auto bridge creation in daemon mode

Create switchdev policy with OVS bridge template

```yaml
# policy-switchdev.yaml

kind: SriovNetworkNodePolicy
metadata:
  name: connectx6dx-switchdev
  namespace: network-operator
spec:
  eSwitchMode: switchdev
  mtu: 1500
  nicSelector:
    deviceID: 101d
    vendor: 15b3
  nodeSelector:
    node-role.kubernetes.io/worker: ""
  numVfs: 4
  resourceName: switchdev
  bridge:
    ovs: {}
```

Apply policy

```bash
kubectl apply -f policy-switchdev.yaml
```

Controller should update state of the node with the bridge configuration


```bash
kubectl get sriovnetworknodestates.sriovnetwork.openshift.io -n network-operator cloud-dev-16 -o yaml
```

```bash

apiVersion: sriovnetwork.openshift.io/v1
kind: SriovNetworkNodeState
metadata:
  # .... other fields
  name: cloud-dev-16
  namespace: network-operator
  # .... other fields
spec:
  bridges:
    ovs:
    - bridge: {}
      name: br-0000_d8_00.0
      uplinks:
      - interface: {}
        name: enp216s0f0np0
        pciAddress: 0000:d8:00.0
    - bridge: {}
      name: br-0000_d8_00.1
      uplinks:
      - interface: {}
        name: enp216s0f1np1
        pciAddress: 0000:d8:00.1
  interfaces:
    # .... other fields
```

When policy is applied, information about managed bridges will be reported
in the status field


### Validate bridges on the worker node

```
root@cloud-dev-16:~# ovs-vsctl show
d178a096-20dd-4aa1-931a-0a3e82613228
    Bridge br-0000_d8_00.0
        Port enp216s0f0np0
            Interface enp216s0f0np0
    Bridge br-0000_d8_00.1
        Port enp216s0f1np1
            Interface enp216s0f1np1
    ovs_version: "2.17.9"

```

### Create OVSNetwork and workloads

Create OVSNetwork CR for `switchdev` resource

```yaml
# ovs-network.yaml

apiVersion: sriovnetwork.openshift.io/v1
kind: OVSNetwork
metadata:
  name: ovs
  namespace: network-operator
spec:
  networkNamespace: default
  ipam: |
    {
      "type": "nv-ipam",
      "poolName": "pool1"
    }
  resourceName: switchdev
  vlan: 200
```

```
kubectl apply -f ovs-network.yaml
```

Create workload which uses OVSNetwork

```yaml
# test-pod-deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: ovs-offload
  labels:
    app: ovs-offload
spec:
  replicas: 2
  selector:
    matchLabels:
      app: ovs-offload
  template:
    metadata:
      labels:
        app: ovs-offload
      annotations:
        k8s.v1.cni.cncf.io/networks: ovs
    spec:
      containers:
      - name: ovs-offload-container
        command: ["/bin/bash", "-c"]
        args:
        - |
          while true; do sleep 1000; done
        image: nicolaka/netshoot
        resources:
          requests:
            nvidia.com/switchdev: 1
          limits:
            nvidia.com/switchdev: 1
      terminationGracePeriodSeconds: 1
```

```bash
kubectl apply -f test-pod-deployment.yaml
```

Validate that Pods are running.


## External bridge with VF-lag

### Reset operator config and switch to systemd mode

Remove workloads

```bash
kubectl delete -f test-pod-deployment.yaml
```

Delete policy

```bash
kubectl delete -f policy-switchdev.yaml
```

Change sriov-network-operator mode to `systemd`

```bash
kubectl patch sriovoperatorconfigs.sriovnetwork.openshift.io -n network-operator default --patch '{ "spec": { "configurationMode": "systemd" } }' --type='merge'
```

### Worker configuration

Prepare netplan configuration for bond on the worker node

```yaml
# /etc/netplan/01-uplink-bond.yaml 
network:
  version: 2
  renderer: networkd
  ethernets:
    enp216s0f0np0:
      dhcp4: no
      dhcp6: no
    enp216s0f1np1:
      dhcp4: no
      dhcp6: no
  bonds:
    bond0:
      dhcp4: no
      dhcp6: no
      interfaces:
        - enp216s0f0np0
        - enp216s0f1np1
      parameters:
        mode: 802.3ad
```

Apply config (ignore driver syndrome)
```
netplan apply
```

Create OVS bridge and add bond interface to it

```
ovs-vsctl add-br mybr
ovs-vsctl add-port mybr bond0
```

### Create switchdev policy

Create switchdev policy without bridge configuration

```yaml
# policy-switchdev.yaml

kind: SriovNetworkNodePolicy
metadata:
  name: connectx6dx-switchdev
  namespace: network-operator
spec:
  eSwitchMode: switchdev
  mtu: 1500
  nicSelector:
    deviceID: 101d
    vendor: 15b3
  nodeSelector:
    node-role.kubernetes.io/worker: ""
  numVfs: 4
  resourceName: switchdev
```

Apply policy

```bash
kubectl apply -f policy-switchdev.yaml
```

After reboot validate that workloads can be deployed

```bash
kubectl apply -f test-pod-deployment.yaml
```

Validate that Pods are running.