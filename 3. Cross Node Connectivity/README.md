
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

