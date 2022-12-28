# CalicoEnterprise-Networking-Training
This is the hands-on lab guide for Calico Enterprise networking training.

## Lab setup

The following diagram depicts the lab setup used for the entire training.

![image](https://user-images.githubusercontent.com/29644478/209869545-03ae6c68-940d-4570-887e-a25dd7223eae.png)


Followings is a short description of each node in the lab.

* **Control1** is Kubernetes master node and runs Kubernetes control plane pods.
* **Worker1** is a Kubernetes worker node.
* **Worker2** is a Kubernetes worker node.
* **Bastion** is a node outside the Kubernetes cluster and is used to run kubectl/calicoctl commands. This node also runs a standalone instance of bird and is used for bgppeering in the relevant lab modules. Finally, since this node is outside the cluster, it is used to simulate external connectivity when needed.

Followings are the CIDRs used to build the trainibg lab environment.

* 10.0.1.0/24 host CIDR
* 10.48.0.0/16 Kubernetes Pod Network (via kubeadm --pod-network-cidr)
* 10.49.0.0/16 Kubernetes Service CIDR (via kubeadm --service-cidr)

## SSH access to the nodes

To ssh into the lab instances, type the following commands.

* ssh control1 
* ssh worker1
* ssh worker2


## Lab modules
