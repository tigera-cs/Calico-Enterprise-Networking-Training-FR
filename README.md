# CalicoEnterprise-Networking-Training
This is the hands-on lab guide for Calico Enterprise networking training.

## Lab setup

The following diagram depicts the lab setup used for the entire training.

![image](https://user-images.githubusercontent.com/29644478/209869545-03ae6c68-940d-4570-887e-a25dd7223eae.png)



Followings are the CIDRs used to build the trainibg lab environment.

* 10.0.1.0/24 host CIDR
* 10.48.0.0/16 Kubernetes Pod Network (via kubeadm --pod-network-cidr)
* 10.49.0.0/16 Kubernetes Service CIDR (via kubeadm --service-cidr)


## Hands-on modules
