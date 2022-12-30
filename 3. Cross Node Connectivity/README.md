
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
ip route
```
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
5. Exit the ssh mode from control1.

```
exit
```

6. Use the following command to configure the default IPPool to use vxlanMode by setting `vxlanMode: Always`. Then save and exit.

```
kubectl edit ippools default-ipv4-ippool
```

You should see an output similar to the following.

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

7. Let's check on the configured interfaces on the node resource. We can see that all the nodes have got a VXLAN interface. This interface was added after we enabled VXLAN networking in the IPPool resource. calico-node watches the IPPool resource and as soon as the resource is configured to use VXLAN, it will configure the VXLAN device on the nodes. Make note of each node's VXLAN interface IP address. We will use them in the next steps.

```
kubectl get nodes -o yaml | grep -B1 -i vxlan
```
```
      projectcalico.org/IPv4Address: 10.0.1.20/24
      projectcalico.org/IPv4VXLANTunnelAddr: 10.48.0.135
--
      projectcalico.org/IPv4Address: 10.0.1.30/24
      projectcalico.org/IPv4VXLANTunnelAddr: 10.48.0.206
--
      projectcalico.org/IPv4Address: 10.0.1.31/24
      projectcalico.org/IPv4VXLANTunnelAddr: 10.48.0.46
```


8. ssh into control1 again.

```
ssh control1
```

9. Check the network interface on the node again. This time you should see a vxlan interface named `vxlan.calico`. 

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
17: vxlan.calico: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 8951 qdisc noqueue state UNKNOWN mode DEFAULT group default 
    link/ether 66:c9:ee:b1:9b:47 brd ff:ff:ff:ff:ff:ff
```

10. Let's examine the node routing table again. We can routes related to VXLAN overlay networking as indicted by `10.48.0.192/26 via 10.48.0.206 dev vxlan.calico onlink ` and routes related to native bgp networking as indicated by `10.48.128.0/26 via 10.0.1.31 dev ens5 proto bird `. The reason for having both VXLAN and native bgp networking in this cluster is that we have two IPPools one of which is configured to use VXLAN and the other is configured to use native bgp.

```
ip route
```
```
default via 10.0.1.1 dev ens5 proto dhcp src 10.0.1.20 metric 100 
10.0.1.0/24 dev ens5 proto kernel scope link src 10.0.1.20 
10.0.1.1 dev ens5 proto dhcp scope link src 10.0.1.20 metric 100 
10.48.0.0/26 via 10.48.0.46 dev vxlan.calico onlink 
10.48.0.128 dev cali1bff47cba53 scope link 
blackhole 10.48.0.128/26 proto 80 
10.48.0.130 dev calic628ed26868 scope link 
10.48.0.131 dev calic7ceee6b788 scope link 
10.48.0.132 dev cali4b871599db0 scope link 
10.48.0.133 dev caliae0f159e5b3 scope link 
10.48.0.134 dev cali2135bcd7981 scope link 
10.48.0.192/26 via 10.48.0.206 dev vxlan.calico onlink 
10.48.128.0/26 via 10.0.1.31 dev ens5 proto bird 
10.48.128.192/26 via 10.0.1.30 dev ens5 proto bird 
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 linkdown 
```
11. Felix on each node is watching the node objects in the datastore to learn about the VXLAN tunnel endpoints on remote nodes as well as to learn their IP addresses and MAC addresses. Felix will then program the local node with the following information on each node in the cluster:

* A static ARP entry, visible with `ip neigh show`
* Static Bridge FDB entry, visible with `bridge fdb show`

Run the following command to see the relevant `VXLAN` ARP entries. Note the followings:
* `10.48.0.206` is the IP address and `66:14:9d:f4:2c:1f` is the mac address of `vxlan.calico` on worker1
* `10.48.0.46` is the IP address and `66:4b:ff:e1:ef:01` is the mac address of `vxlan.calico` on worker2

```
ip neigh show
```
```
10.48.0.206 dev vxlan.calico lladdr 66:14:9d:f4:2c:1f PERMANENT
10.0.1.10 dev ens5 lladdr 02:88:a0:32:73:4d REACHABLE
10.48.0.46 dev vxlan.calico lladdr 66:4b:ff:e1:ef:01 PERMANENT
10.0.1.1 dev ens5 lladdr 02:04:7a:86:55:c1 REACHABLE
10.48.0.134 dev cali2135bcd7981 lladdr fa:67:13:92:26:c8 REACHABLE
10.48.0.130 dev calic628ed26868 lladdr 52:12:d1:4c:04:20 REACHABLE
10.48.0.133 dev caliae0f159e5b3 lladdr a6:0b:a3:07:ca:a3 REACHABLE
10.0.1.30 dev ens5 lladdr 02:85:fc:c5:96:85 REACHABLE
10.48.0.131 dev calic7ceee6b788 lladdr d6:c0:bc:a2:8f:49 DELAY
10.0.1.31 dev ens5 lladdr 02:df:0c:da:8e:b1 REACHABLE
10.48.0.132 dev cali4b871599db0 lladdr 76:17:69:d0:15:04 REACHABLE
```

12. Run the following command to see the relevant bridge forward dabase enteries. Note the last two enteries that are relevant to the VXLAN networking in our scenario.
* `66:14:9d:f4:2c:1f` is the mac address of `vxlan.calico` on worker1 and is accessible via `10.0.1.30`, which is the fabric facing network interface of worker1
* `66:4b:ff:e1:ef:01` is the mac address of `vxlan.calico` on worker2 and is accessible via `10.0.1.31`, which is the fabric facing network interface of worker2

```
bridge fdb show
```
```
01:00:5e:00:00:01 dev ens5 self permanent
01:80:c2:00:00:00 dev ens5 self permanent
01:80:c2:00:00:03 dev ens5 self permanent
01:80:c2:00:00:0e dev ens5 self permanent
33:33:00:00:00:01 dev ens5 self permanent
33:33:ff:90:06:0b dev ens5 self permanent
33:33:00:00:00:01 dev docker0 self permanent
01:00:5e:00:00:6a dev docker0 self permanent
33:33:00:00:00:6a dev docker0 self permanent
01:00:5e:00:00:01 dev docker0 self permanent
02:42:ba:ae:65:54 dev docker0 vlan 1 master docker0 permanent
02:42:ba:ae:65:54 dev docker0 master docker0 permanent
33:33:00:00:00:01 dev cali1bff47cba53 self permanent
01:00:5e:00:00:01 dev cali1bff47cba53 self permanent
33:33:ff:ee:ee:ee dev cali1bff47cba53 self permanent
33:33:00:00:00:01 dev calic628ed26868 self permanent
01:00:5e:00:00:01 dev calic628ed26868 self permanent
33:33:ff:ee:ee:ee dev calic628ed26868 self permanent
33:33:00:00:00:01 dev calic7ceee6b788 self permanent
01:00:5e:00:00:01 dev calic7ceee6b788 self permanent
33:33:ff:ee:ee:ee dev calic7ceee6b788 self permanent
33:33:00:00:00:01 dev cali4b871599db0 self permanent
01:00:5e:00:00:01 dev cali4b871599db0 self permanent
33:33:ff:ee:ee:ee dev cali4b871599db0 self permanent
33:33:00:00:00:01 dev caliae0f159e5b3 self permanent
01:00:5e:00:00:01 dev caliae0f159e5b3 self permanent
33:33:ff:ee:ee:ee dev caliae0f159e5b3 self permanent
33:33:00:00:00:01 dev cali2135bcd7981 self permanent
01:00:5e:00:00:01 dev cali2135bcd7981 self permanent
33:33:ff:ee:ee:ee dev cali2135bcd7981 self permanent
66:14:9d:f4:2c:1f dev vxlan.calico dst 10.0.1.30 self permanent
66:4b:ff:e1:ef:01 dev vxlan.calico dst 10.0.1.31 self permanent
```
13. Finally, Felix on each node learns about the other nodes routing information through the information (IPAM blocks and pod IP address) stored in the datastore and updates its node's routing table.

14. Exit the ssh mode from control1.

```
exit
```

14. Reset the default IPPool configuration to native bgp again by setting `vxlanMode: Never`. Then save and exit.

```
kubectl edit ippools default-ipv4-ippool
```

You should see an output similar to the following.

```
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  creationTimestamp: "2022-12-29T22:21:11Z"
  name: default-ipv4-ippool
  resourceVersion: "245626"
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
