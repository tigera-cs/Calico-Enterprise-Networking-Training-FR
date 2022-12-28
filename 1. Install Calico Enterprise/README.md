# In this lab

This lab provides the instructions to:

* [Install Calico Enterprise](https://github.com/Pooriya-a/CalicoEnterprise-Networking-Training/blob/main/1.%20Install%20Calico%20Enterprise/README.md#install-calico-enterprise)
* Deploy a three-tier sample application called "yaobank" (Yet Another Online Bank)


### Install Calico Enterprise

Upstream Kubernetes by default does not provide a network interface plugin. In this lab environment, Kubernetes has been preinstalled, but as your network plugin is not installed yet, your nodes will appear as NotReady. This section walks you through the required steps to install Calico Enteprise as both the CNI provider and network policy engine provider.

1. To validate that the CNI plugin is not installed in this cluster, run the following command. All the cluster nodes' STATUS should appear as NotReady.

```
kubectl get nodes
```
```
NAME                                      STATUS     ROLES                  AGE   VERSION
ip-10-0-1-20.us-west-1.compute.internal   NotReady   control-plane,master   95m   v1.23.14
ip-10-0-1-30.us-west-1.compute.internal   NotReady   <none>                 94m   v1.23.14
ip-10-0-1-31.us-west-1.compute.internal   NotReady   <none>                 94m   v1.23.14
```
### 1.1.2. Prepare the storage

First configure storage for Calico Enterprise, we will use AWS EBS provided by the AWS cloud provider integration.

```
kubectl apply -f -<<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: tigera-elasticsearch
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
  fsType: ext4
reclaimPolicy: Retain
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
EOF
```

Make sure the storageclass has been created successfully:

```
kubectl get storageclass
```
```
NAME                   PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
tigera-elasticsearch   kubernetes.io/aws-ebs   Retain          WaitForFirstConsumer   true                   10s
```
