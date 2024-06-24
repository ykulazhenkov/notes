# Notes for nvidia-ipam (deployed by the network-operator) static IP feature

## Links

* [nvidia-ipam: documentation](https://github.com/Mellanox/nvidia-k8s-ipam/blob/main/README.md)

* [nvidia-ipam: static IP example](https://github.com/Mellanox/nvidia-k8s-ipam/blob/main/docs/static-ip.md)


## Environment

**Kubernetes cluster:** Kind cluster (1 control-plane node + 2 workers)

**Kind version:** v0.23.0

**Kubernetes:** v1.30.0

**network-operator version:** dev (commit cd0c86961c0d8bc4845d2db196418d2edd2ccc22)

**nvidia-ipam version:** v0.2.0

### Nodes

```bash
kubectl get node
NAME                 STATUS   ROLES                  AGE   VERSION
kind-control-plane   Ready    control-plane,worker   72m   v1.30.0
kind-worker          Ready    worker                 72m   v1.30.0
kind-worker2         Ready    worker                 72m   v1.30.0

```

### Network-operator Pods

```bash
kubectl get po -n nvidia-network-operator 
NAME                                                              READY   STATUS    RESTARTS   AGE
cni-plugins-ds-7n44v                                              1/1     Running   0          57m
cni-plugins-ds-n7vsz                                              1/1     Running   0          57m
cni-plugins-ds-t4s54                                              1/1     Running   0          57m
kube-multus-ds-hmcfj                                              1/1     Running   0          57m
kube-multus-ds-mrn52                                              1/1     Running   0          57m
kube-multus-ds-sbrdv                                              1/1     Running   0          57m
network-operator-57b5d988c9-gnkml                                 1/1     Running   0          57m
network-operator-node-feature-discovery-master-5c47cc5574-52nwb   1/1     Running   0          57m
network-operator-node-feature-discovery-worker-f424g              1/1     Running   0          57m
network-operator-node-feature-discovery-worker-kqrx5              1/1     Running   0          57m
network-operator-node-feature-discovery-worker-zsjrs              1/1     Running   0          57m
nv-ipam-controller-67556c846b-d4rsq                               1/1     Running   0          57m
nv-ipam-controller-67556c846b-qx79v                               1/1     Running   0          57m
nv-ipam-node-8hlb7                                                1/1     Running   0          57m
nv-ipam-node-vk6k8                                                1/1     Running   0          57m
nv-ipam-node-wxbqc                                                1/1     Running   0          57m
rdma-shared-dp-ds-6947s                                           1/1     Running   0          57m
rdma-shared-dp-ds-7hsht                                           1/1     Running   0          57m
rdma-shared-dp-ds-ttzpr                                           1/1     Running   0          57m

```

## Demo

### Create IPPool CR for nv-ipam

```yaml
apiVersion: nv-ipam.nvidia.com/v1alpha1
kind: IPPool
metadata:
  name: pool1
  namespace: nvidia-network-operator
spec:
  subnet: 192.168.0.0/16
  perNodeBlockSize: 100
  gateway: 192.168.0.1
  exclusions:
  - startIP: 192.168.0.0
    endIP: 192.168.0.10
  nodeSelector:
    nodeSelectorTerms:
    - matchExpressions:
        - key: node-role.kubernetes.io/worker
          operator: Exists
```

### Check IP ranges allocated for nodes

```bash
kubectl get ippools.nv-ipam.nvidia.com -A -o jsonpath='{range .items[*]}{.metadata.name}{"\n"} {range .status.allocations[*]}{"\t"}{.nodeName} => Start IP: {.startIP} End IP: {.endIP}{"\n"}{end}{"\n"}{end}'
pool1
        kind-control-plane => Start IP: 192.168.0.1 End IP: 192.168.0.100
        kind-worker => Start IP: 192.168.0.101 End IP: 192.168.0.200
        kind-worker2 => Start IP: 192.168.0.201 End IP: 192.168.1.44
```

### Create MacVlanNetwork CR

```yaml
apiVersion: mellanox.com/v1alpha1
kind: MacvlanNetwork
metadata:
  name: example-macvlannetwork
spec:
  networkNamespace: "default"
  master: "eth0"
  mode: "bridge"
  mtu: 1500
  ipam: |
    {
      "type": "nv-ipam",
      "poolName": "pool1"
    }
```

### Create first Pod with dynamic IP

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dynamic-ip-pod1
  annotations:
    k8s.v1.cni.cncf.io/networks: example-macvlannetwork
spec:
  nodeSelector:
    kubernetes.io/hostname: kind-control-plane
  terminationGracePeriodSeconds: 1
  containers:
    - name: test
      image: centos/tools
      command: [ "/bin/bash", "-c", "--" ]
      args: [ "while true; do sleep 300; done;" ]

```

Check allocated IP

```bash
kubectl get pod dynamic-ip-pod1 -o=custom-columns='NETWORK-STATUS:metadata.annotations.k8s\.v1\.cni\.cncf\.io/network-status'
NETWORK-STATUS
[{
    "name": "kindnet",
    "interface": "eth0",
    "ips": [
        "10.244.0.14"
    ],
    "mac": "be:75:d3:da:da:c6",
    "default": true,
    "dns": {}
},{
    "name": "default/example-macvlannetwork",
    "interface": "net1",
    "ips": [
        "192.168.0.13"
    ],
    "mac": "8a:ce:ae:9d:03:18",
    "dns": {}
}]
```

### Create second Pod with dynamic IP

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dynamic-ip-pod2
  annotations:
    k8s.v1.cni.cncf.io/networks: example-macvlannetwork
spec:
  terminationGracePeriodSeconds: 1
  containers:
    - name: test
      image: centos/tools
      command: [ "/bin/bash", "-c", "--" ]
      args: [ "while true; do sleep 300; done;" ]
```

Check allocated IP

```bash
kubectl get pod dynamic-ip-pod2 -o=custom-columns='NETWORK-STATUS:metadata.annotations.k8s\.v1\.cni\.cncf\.io/network-status'
NETWORK-STATUS
[{
    "name": "kindnet",
    "interface": "eth0",
    "ips": [
        "10.244.1.21"
    ],
    "mac": "92:88:ee:53:67:bf",
    "default": true,
    "dns": {}
},{
    "name": "default/example-macvlannetwork",
    "interface": "net1",
    "ips": [
        "192.168.0.104"
    ],
    "mac": "c6:3c:91:d5:c6:4e",
    "dns": {}
}]

```

### Create Pod with static IP

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: static-ip-pod1
  annotations:
    k8s.v1.cni.cncf.io/networks: '[{"name": "example-macvlannetwork", "ips": ["192.168.0.5"]}]'
spec:
  nodeSelector:
    kubernetes.io/hostname: kind-worker2
  terminationGracePeriodSeconds: 1
  containers:
    - name: test
      image: centos/tools
      command: [ "/bin/bash", "-c", "--" ]
      args: [ "while true; do sleep 300; done;" ]
```

Check allocated IP

```bash
kubectl get pod static-ip-pod1 -o=custom-columns='NETWORK-STATUS:metadata.annotations.k8s\.v1\.cni\.cncf\.io/network-status'
NETWORK-STATUS
[{
    "name": "kindnet",
    "interface": "eth0",
    "ips": [
        "10.244.2.11"
    ],
    "mac": "0e:7f:e6:7b:56:49",
    "default": true,
    "dns": {}
},{
    "name": "default/example-macvlannetwork",
    "interface": "net1",
    "ips": [
        "192.168.0.5"
    ],
    "mac": "72:1d:13:3c:d1:20",
    "dns": {}
}]
```

### Statically allocate wrong IP (not in the pool's IP range)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: static-ip-pod2-wrong-ip
  annotations:
    k8s.v1.cni.cncf.io/networks: '[{"name": "example-macvlannetwork", "ips": ["10.10.10.10"]}]'
spec:
  terminationGracePeriodSeconds: 1
  containers:
    - name: test
      image: centos/tools
      command: [ "/bin/bash", "-c", "--" ]
      args: [ "while true; do sleep 300; done;" ]
```

Pod will not start with the following error

```bash
Events:
  Type     Reason                  Age   From               Message
  ----     ------                  ----  ----               -------
  Normal   Scheduled               5s    default-scheduler  Successfully assigned default/static-ip-pod2-wrong-ip to kind-worker
  Normal   AddedInterface          5s    multus             Add eth0 [10.244.1.58/24] from kindnet
  Warning  FailedCreatePodSandBox  5s    kubelet            Failed to create pod sandbox: rpc error: code = Unknown desc = failed to setup network for sandbox "f5e3614bd99aedbafadd7f811b9bc58dd20c99f74891326b4ac02d2d871e4d1c": plugin type="multus" name="multus-cni-network" failed (add): [default/static-ip-pod2-wrong-ip/0598c0e7-29fc-4abd-8dbe-eee843f0d9ce:example-macvlannetwork]: error adding container to network "example-macvlannetwork": grpc call failed: rpc error: code = InvalidArgument desc = not all requested static IPs can be allocated from the ranges available on the node, ip 10.10.10.10 has no matching Pool
```