
# In this lab

This lab provides the instructions to:

* [Examine pod network connectivity across cluster nodes using CE VXLAN mode](https://github.com/Pooriya-a/CalicoEnterprise-Networking-Training/blob/main/3.%20Cross%20Node%20Connectivity/README.md#examine-pod-network-connectivity-across-cluster-nodes-using-ce-vxlan-mode)

## Overview

When it comes to pod networking, a pod either needs to talk to other pods that are running on the same node or different nodes. If the communicating pods are collocated on the same node, it is called intrahost networking. In this scenario, pods are connected together through a layer 2 or layer 3 device on the host depending on the CNI solution used. However, pods traffic stays within the same node no matter if the pods are connected using a layer 2 or layer 3 device.

If the communicating pods are located on the different nodes, it is called interhost networking. In this case, pods communicate with each other using their IP addresses without any nat. In this scenario, network must be configured such that pods located on different nodes can communicate with each other. Depending on the CNI provider, network can be configured in layer 2, layer 3, or overlay networking mode for the pods to communicate with each. Calico uses the same logic for intra-host POD traffic and Inter-host POD traffic. All traffic between the PODs must follow a L3 routed path.

Calico has got a modular architecture and supports various networking deployment options so that you can select the best networking approach for your specific environment and needs. 

Calico can operate in two networking modes:

* Non-Overlay or flat mode uses bgp and is typically used in on-premise deployments
* Overlay network is a network that is layered on top of another network. In the context of Kubernetes, an overlay network can be used to handle pod-to-pod traffic between nodes on top of an underlying network that is not aware of pod IP addresses. Calico supports two different overlay technologies: IPIP and VXLAN



### Examine pod network connectivity across cluster nodes using CE VXLAN mode

VXLAN is one of the two supported overlay technologies in Calico Enterprise CNI implementation. VXLAN implementation in Calico Enterprise can use bird (BGP agent) or Felix as the control plane to distribute routing information. Calico Enterprise by default deploys VXLAN overlay using Felix. In this section, we will learn about Calico Enterprise VXLAN implementation by examining the relevant configurations in the lab.

1. Networking mode used in the cluster is a configuration specified in the IPPool resource. Let's examine the default IPPool. The two configuration parameters `ipipMode: Never` and `vxlanMode: Never` tells us that this cluster is configured to use native bgp to exchange routing information.

```
kubectl get ippools default-ipv4-ippool -o yaml
```

```
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  creationTimestamp: "2022-12-29T22:21:11Z"
  name: default-ipv4-ippool
  resourceVersion: "1722"
  uid: 3f053fa7-0d94-4d6d-94e6-e6582d80c0b0
spec:
  allowedUses:
  - Workload
  - Tunnel
  blockSize: 26
  cidr: 10.48.0.0/24
  ipipMode: Never
  natOutgoing: true
  nodeSelector: all()
  vxlanMode: Never
```

2. Let's ssh into one of the cluster nodes and examine the current networking configurations of the node.

```
ssh control1
```

3. Check the network interface on the node and make sure there is no VXLAN interface.

```
ip -c link
```
```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: ens5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc mq state UP mode DEFAULT group default qlen 1000
    link/ether 02:79:d5:90:06:0b brd ff:ff:ff:ff:ff:ff
    altname enp0s5
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN mode DEFAULT group default 
    link/ether 02:42:ba:ae:65:54 brd ff:ff:ff:ff:ff:ff
4: cali1bff47cba53@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 0
10: calic628ed26868@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 1
11: calic7ceee6b788@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 2
12: cali4b871599db0@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 3
13: caliae0f159e5b3@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 4
14: cali2135bcd7981@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 5
```

4. Let's examine the node routing table. We do not see any routes related to VXLAN in the following routing table (You can compare this output with the output of the node routing table when VXLAN is enabled). We also see that bird is used to exchange routing information on this node. One example of such route is `10.48.0.192/26 via 10.0.1.30 dev ens5 proto bird `. 

```
default via 10.0.1.1 dev ens5 proto dhcp src 10.0.1.20 metric 100 
10.0.1.0/24 dev ens5 proto kernel scope link src 10.0.1.20 
10.0.1.1 dev ens5 proto dhcp scope link src 10.0.1.20 metric 100 
10.48.0.0/26 via 10.0.1.31 dev ens5 proto bird 
10.48.0.128 dev cali1bff47cba53 scope link 
blackhole 10.48.0.128/26 proto bird 
10.48.0.130 dev calic628ed26868 scope link 
10.48.0.131 dev calic7ceee6b788 scope link 
10.48.0.132 dev cali4b871599db0 scope link 
10.48.0.133 dev caliae0f159e5b3 scope link 
10.48.0.134 dev cali2135bcd7981 scope link 
10.48.0.192/26 via 10.0.1.30 dev ens5 proto bird 
10.48.128.0/26 via 10.0.1.31 dev ens5 proto bird 
10.48.128.192/26 via 10.0.1.30 dev ens5 proto bird 
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 linkdown 
```
4. Exit the ssh mode from control1.

```
exit
```

5. Use the following command to configure the default IPPool to use vxlanMode by setting `vxlanMode: Always`. Then save and exit.

```
kubectl edit ippools default-ipv4-ippool
```
```
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  creationTimestamp: "2022-12-29T22:21:11Z"
  name: default-ipv4-ippool
  resourceVersion: "232027"
  uid: 3f053fa7-0d94-4d6d-94e6-e6582d80c0b0
spec:
  allowedUses:
  - Workload
  - Tunnel
  blockSize: 26
  cidr: 10.48.0.0/24
  ipipMode: Never
  natOutgoing: true
  nodeSelector: all()
  vxlanMode: Always
```
