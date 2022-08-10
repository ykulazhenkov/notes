# Notes for accelerated-bridge-cni multibridge feature demo

## Use-case

Before multibridge feature was implemented in the accelerated-bridge CNI it was possible to
specify only one bridge name in the CNI config. 
That's mean that one NetworkAttachDefinition was pointing to exact one linux bridge.
Some users may have 2 or more NICs attached to the same network. In this case a separate linux bridge
exist for each NIC.
Users want to configure these bridges as a same logical network without need to create an
extra NetworkAttachDefinitions. 

The solution is to support multiple bridge in the CNI config.
Plugin checks to which Linux bridge uplink for VF is attached and uses that bridge to add a VF representor.
Automatic bridge selection logic requires uplink to be added to a bridge before calling the CNI.

Supported configurations for auto bridge selection:

- uplink is a direct member of a Linux bridge 
- uplink is a part of a bond interface, bond interface is a member of a Linux bridge

```
┌────────────────────────────────────────┐
│                 Host                   │
│                                        │
│                                        │
│ ┌────────────┐                         │
│ │            │                         │
│ │            ├───────────┐             │           ...........
│ │            │           ├─────────────┤         ...          ...
│ │  Logical   │ bridge1   │ Nic 1       ├───────►..              ...
│ │  network   │           ├─────────────┤       .                  ..
│ │            ├───────────┘             │       .      PHYSICAL     .
│ │   same     │                         │       .      NETWORK     ..
│ │net-attach- ├───────────┬─────────────┤       .                  .
│ │    def     │ bridge2   │ Nic 2       ├───────►..               ..
│ │            │           ├─────────────┤         .....          ..
│ │            ├───────────┘             │             .. .........
│ └────────────┘                         │
│                                        │
│                                        │
└────────────────────────────────────────┘

```
## Changes in API
Before
```
bridge (string, optional): Linux Bridge to use. e.g. "br1", default value is "cni0"
```
After
```
bridge (string, optional): single or comma separated list of linux bridges to use e.g. "br1" or "br1, br2",
 default value is "cni0". CNI will use automatic bridge selection logic if multiple bridges are set.
```



## Environment
```
OS: CentOS Linux release 8.5.2111
Kernel: 5.19.0-1.el8.elrepo.x86_64
Kubernetes: v1.24.3
containerd.io: 1.6.7

Multus: v3.9
SRIOV-device-plugin: v3.5.1
Whereabouts v0.5.4

accelerated-bridge-cni sha-89e6d84e4704a24c148b661f826869e49f8708d9
```

## Hardware
```
[root@host ~]# lshw -c network -businfo | grep ConnectX-6
pci@0000:d8:00.0  ens3f0np0     network        MT2892 Family [ConnectX-6 Dx]
pci@0000:d8:00.1  ens3f1np1     network        MT2892 Family [ConnectX-6 Dx]

```

### FW settings
Install mstconfig tool
```
yum install mstflint
```
Enable SRIOV:
```
mstconfig -d 0000:d8:00.0 set SRIOV_EN=1
```
Set number of VFs:
```
mstconfig -d 0000:d8:00.0 set NUM_OF_VFS=2
mstconfig -d 0000:d8:00.1  set NUM_OF_VFS=2
```

Reboot the host if changes were applied

### Configure VFs
```
echo 2 > /sys/class/net/ens3f0np0/device/sriov_numvfs
echo 2 > /sys/class/net/ens3f1np1/device/sriov_numvfs
```

Unbind VFs
```
VFS_PCI=($(lspci | grep "Mellanox" | grep "Virtual" | cut -d " " -f 1));
     for i in ${VFS_PCI[@]};
     do
         echo "unbinding VF $i";
         echo "0000:${i}" >> /sys/bus/pci/drivers/mlx5_core/unbind;
     done
```

Switch PF to switchdev mode

```
devlink dev eswitch set pci/0000:d8:00.0 mode switchdev
devlink dev eswitch set pci/0000:d8:00.1 mode switchdev
```

Bind VFs
```
VFS_PCI=($(lspci | grep "Mellanox" | grep "Virtual" | cut -d " " -f 1));
     for i in ${VFS_PCI[@]};
     do
         echo "binding VF $i";
         echo "0000:${i}" >> /sys/bus/pci/drivers/mlx5_core/bind;
     done
```

### Configure Bridges

Create a separate bridge for each PF

```
ip link add name accbr-pf1 type bridge vlan_filtering 1
ip link add name accbr-pf2 type bridge vlan_filtering 1
ip link set dev accbr-pf1 up
ip link set dev accbr-pf2 up
ip link set dev ens3f0np0 master accbr-pf1
ip link set dev ens3f1np1 master accbr-pf2
ip link set dev ens3f0np0 up
ip link set dev ens3f1np1 up
```

## Deploy dependencies

### SRIOV-device-plugin
Deploy plugin config
```
cat << 'EOF' | kubectl apply -f -
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: sriovdp-config
  namespace: kube-system
data:
  config.json: |
    {
        "resourceList": [
            {
                "resourceName": "cloudxdx6vf",
                "resourcePrefix": "nvidia.com",
                "selectors": {
                    "vendors": ["15b3"],
                    "devices": ["101e"],
                    "isRdma": true
                }
            }
        ]
    }
EOF
```

Deploy plugin
```
cat << 'EOF' | kubectl apply -f -
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: sriov-device-plugin
  namespace: kube-system

---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: kube-sriov-device-plugin-amd64
  namespace: kube-system
  labels:
    tier: node
    app: sriovdp
spec:
  selector:
    matchLabels:
      name: sriov-device-plugin
  template:
    metadata:
      labels:
        name: sriov-device-plugin
        tier: node
        app: sriovdp
    spec:
      hostNetwork: true
      nodeSelector:
        beta.kubernetes.io/arch: amd64
      tolerations:
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: NoSchedule
      serviceAccountName: sriov-device-plugin
      containers:
      - name: kube-sriovdp
        image: ghcr.io/k8snetworkplumbingwg/sriov-network-device-plugin:v3.5.1
        imagePullPolicy: IfNotPresent
        args:
        - --log-dir=sriovdp
        - --log-level=10
        securityContext:
          privileged: true
        resources:
          requests:
            cpu: "250m"
            memory: "40Mi"
          limits:
            cpu: 1
            memory: "200Mi"
        volumeMounts:
        - name: devicesock
          mountPath: /var/lib/kubelet/
          readOnly: false
        - name: log
          mountPath: /var/log
        - name: config-volume
          mountPath: /etc/pcidp
        - name: device-info
          mountPath: /var/run/k8s.cni.cncf.io/devinfo/dp
      volumes:
        - name: devicesock
          hostPath:
            path: /var/lib/kubelet/
        - name: log
          hostPath:
            path: /var/log
        - name: device-info
          hostPath:
            path: /var/run/k8s.cni.cncf.io/devinfo/dp
            type: DirectoryOrCreate
        - name: config-volume
          configMap:
            name: sriovdp-config
            items:
            - key: config.json
              path: config.json
EOF
```

Check that VFs detected by device plugin
```
kubectl get nodes -o yaml | grep cloudxdx6vf
      nvidia.com/cloudxdx6vf: "4"
      nvidia.com/cloudxdx6vf: "4"
```

### Whereabouts

Deploy Whereabouts

```
kubectl apply \
    -f https://raw.githubusercontent.com/k8snetworkplumbingwg/whereabouts/v0.5.4/doc/crds/daemonset-install.yaml \
    -f https://raw.githubusercontent.com/k8snetworkplumbingwg/whereabouts/v0.5.4/doc/crds/whereabouts.cni.cncf.io_ippools.yaml \
    -f https://raw.githubusercontent.com/k8snetworkplumbingwg/whereabouts/v0.5.4/doc/crds/whereabouts.cni.cncf.io_overlappingrangeipreservations.yaml \
    -f https://raw.githubusercontent.com/k8snetworkplumbingwg/whereabouts/v0.5.4/doc/crds/ip-reconciler-job.yaml
```

### Multus

Deploy mutlus

```
kubectl apply -f https://raw.githubusercontent.com/k8snetworkplumbingwg/multus-cni/v3.9/deployments/multus-daemonset.yml
```

Create NetworkAttachmentDefinition for network

```
cat << 'EOF' | kubectl apply -f -
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: accbr
  annotations:
    k8s.v1.cni.cncf.io/resourceName: nvidia.com/cloudxdx6vf
spec:
  config: '{
  "type": "accelerated-bridge",
  "cniVersion": "0.3.1",
  "name": "accbr",
  "bridge": "accbr-pf1, accbr-pf2",
  "ipam": {
    "type": "whereabouts",
    "range": "192.168.33.0/24"
  }
}'
EOF
```

## Accelerated-bridge-cni

Deploy

```
cat << 'EOF' | kubectl apply -f -
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: kube-accelerated-bridge-cni-ds-amd64
  namespace: kube-system
  labels:
    tier: node
    app: accelerated-bridge
spec:
  selector:
    matchLabels:
      name: accelerated-bridge-cni
  template:
    metadata:
      labels:
        name: accelerated-bridge-cni
        tier: node
        app: accelerated-bridge
    spec:
      hostNetwork: true
      nodeSelector:
        beta.kubernetes.io/arch: amd64
      tolerations:
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: NoSchedule
      containers:
      - name: kube-accelerated-bridge-cni
        image: ghcr.io/k8snetworkplumbingwg/accelerated-bridge-cni:sha-89e6d84e4704a24c148b661f826869e49f8708d9
        imagePullPolicy: IfNotPresent
        securityContext:
          privileged: true
        resources:
          requests:
            cpu: "100m"
            memory: "50Mi"
          limits:
            cpu: "100m"
            memory: "50Mi"
        volumeMounts:
        - name: cnibin
          mountPath: /host/opt/cni/bin
      volumes:
        - name: cnibin
          hostPath:
            path: /opt/cni/bin
EOF
```


## Create PODs which consume VFs

```
for i in {1..4};do
    cat << EOF | kubectl create -f -
    apiVersion: v1
    kind: Pod
    metadata:
      name: accbr-test-pod-$i
      annotations:
        k8s.v1.cni.cncf.io/networks: accbr
    spec:
      containers:
        - name: test-pod
          image: nicolaka/netshoot
          imagePullPolicy: IfNotPresent
          command: [ "/bin/bash", "-c", "--" ]
          args: [ "_term() { echo shutdown; exit 0; }; trap _term SIGTERM; while true; do sleep 1; date; done" ]
          securityContext:
            privileged: true
          resources:
            requests:
              nvidia.com/cloudxdx6vf: '1'
            limits:
              nvidia.com/cloudxdx6vf: '1'
EOF
done
```


Check that VF representors are attached to bridges

```
bridge link | grep accbr-pf1
bridge link | grep accbr-pf2

```

Check PODs IPs

```
kubectl exec accbr-test-pod-1 -- ip add show net1 | grep "altname\|inet "
kubectl exec accbr-test-pod-2 -- ip add show net1 | grep "altname\|inet "
kubectl exec accbr-test-pod-3 -- ip add show net1 | grep "altname\|inet "
kubectl exec accbr-test-pod-4 -- ip add show net1 | grep "altname\|inet "

```