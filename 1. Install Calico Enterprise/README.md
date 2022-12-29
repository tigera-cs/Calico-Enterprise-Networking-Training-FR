# In this lab

This lab provides the instructions to:

* [Install Calico Enterprise](https://github.com/Pooriya-a/CalicoEnterprise-Networking-Training/blob/main/1.%20Install%20Calico%20Enterprise/README.md#install-calico-enterprise)
* Deploy a three-tier sample application called "yaobank" (Yet Another Online Bank)



Before getting started, let's enable bash autocomplete for kubectl so that we can easier interact with kubectl.

```
sudo apt-get install bash-completion
source /usr/share/bash-completion/bash_completion
echo 'source <(kubectl completion bash)' >>~/.bashrc
source ~/.bashrc

```

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

2. Calico Enterprise uses ElasticSearch to store various logs such as flowlogs, DNS logs, and all others that it collects over the network. ElasticSearch requires persistent storage to store the data. This lab uses host path storage provisioner, which is not suitable for production enviroment and can result in scalability issues, instability, and data loss. 

```
kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: tigera-elasticsearch
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: tigera-elasticsearch
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /var/tigera/elastic-data/1
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: tigera-elasticsearch
EOF

```

3. Make sure the StorageClass has been created successfully.

```
kubectl get storageclass
```
```
NAME                   PROVISIONER                    RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
tigera-elasticsearch   kubernetes.io/no-provisioner   Delete          WaitForFirstConsumer   false                  25s
```

4. The Tigera Operator is a Kubernetes operator and provides a well-defined API for how you install, configure, and run Calico Enterprise. Tigera operator also automates and controls the the lifecycle of a Calico Enterprise deployment. Tigera operator manifest configures the necessary resources such as custom resource definitions, namespaces, services accounts, clusterroles, etc so that cluster is ready to deploy other calico Enterprise components. Get yourself familiar with the content of the manifest and create it in the cluster.

```
kubectl create -f https://docs.tigera.io/manifests/tigera-operator.yaml
```
6. Validate that the tigera-operator is running in the cluster. Note that the tigera-operator is running even though there is no CNI plugin deployed in the cluster. This is because tigera-operator is host networked meaning that it uses the host IP address to communicate over the network.

```
kubectl get pods -n tigera-operator
```

```
NAME                               READY   STATUS    RESTARTS   AGE   IP          NODE                                      NOMINATED NODE   READINESS GATES
tigera-operator-6cb8864f87-w9bb5   1/1     Running   0          17m   10.0.1.31   ip-10-0-1-31.us-west-1.compute.internal   <none>           <none>
```

7. Calico Enterprise Manager UI uses Prometheus to provide out-of-the-box metrics in the various sections of CE Manager UI such as the dashboard page, security policies, and others. Calico Enterprise uses Prometheus operator to deploy Prometheus server and Alertmanager. Apply the following manifest to deploy the Prometheus operator.

  **Note:** Prometheus operator that ships with Calico Enterprise is an optional component and can be replaced with the customer's Prometheus operator if needed.   However, a Prometheus operator must be deployed in the cluster and watch the "tigera-prometheus" namespace to deploy the required Prometheus and Alertmanager   resources. Customer's instance of Prometheus operator can be deployed in any namesapce.


```
kubectl create -f https://docs.tigera.io/manifests/tigera-prometheus-operator.yaml
```

8. Check the tigera-prometheus pod status. 

```
kubectl get pods -n tigera-prometheus
```
```
NAME                                          READY   STATUS    RESTARTS   AGE
calico-prometheus-operator-6557c5bc57-6dbqk   0/1     Pending   0          81s
```

9. tigera-prometheus operator should be in the Pending STATUS. Let's run the following command and check the **Events:** section of the pod. As indicated in the Warning message, the pod is not running because two nodes have the taint "node.kubernetes.io/not-ready". The reason for this taint is because the CNI plugin is not running in this cluster yet.

```
kubectl describe pods -n tigera-prometheus
```

```
Events:
  Type     Reason            Age                  From               Message
  ----     ------            ----                 ----               -------
  Warning  FailedScheduling  43s (x4 over 3m47s)  default-scheduler  0/3 nodes are available: 1 node(s) had taint {node-role.kubernetes.io/master: }, that the pod didn't tolerate, 2 node(s) had taint {node.kubernetes.io/not-ready: }, that the pod didn't tolerate.
```

10. This lab directly downloads the images from quay.io/tigera, which requires authentication. Run the following command to create the secret necessary to pull the images.

```
kubectl create secret generic tigera-pull-secret \
    --from-file=.dockerconfigjson=/home/tigera/config.json \
    --type=kubernetes.io/dockerconfigjson -n tigera-operator
```

11. We also need to create the pull secret in the tigera-prometheus namespace and patch the prometheus operator deployment to pull the images. For all the other Calico Enterprise components such as calico-node, typha, and others to pull the images, tigera-operator copies "tigera-pull-secret" secret to the relevant namespaces.

```
kubectl create secret generic tigera-pull-secret \
    --type=kubernetes.io/dockerconfigjson -n tigera-prometheus \
    --from-file=.dockerconfigjson=/home/tigera/config.json
```
```
kubectl patch deployment -n tigera-prometheus calico-prometheus-operator \
    -p '{"spec":{"template":{"spec":{"imagePullSecrets":[{"name": "tigera-pull-secret"}]}}}}'
```

12. Get yourself familiar with the resources in the following manifest and then install the Tigera custom resources. As discussed in the training, the Installation resource deploy the required resources in the calico-system namespace, which enables the CNI functionality in the cluster. 

```
kubectl create -f https://docs.tigera.io/manifests/custom-resources.yaml
```

13. Watch the status of various components progressing. We need to wait for at least one of the tigera-apiserver pods in the tigera-system namespace to be running before applying the tigera licensekey. The reasons is that the LicenseKey resource uses "projectcalico.org/v3" api, which is managed by the tigera apiserver.

```
watch kubectl get pods -A
```
```
Every 2.0s: kubectl get pods -A                                                                                                                                                                                                                                                                                                         bastion: Wed Dec 28 23:50:12 2022

NAMESPACE              NAME                                                              READY   STATUS    RESTARTS   AGE
calico-system          calico-kube-controllers-7b4fd8c4dc-smcrl                          1/1     Running   0          2m14s
calico-system          calico-node-fh6mk                                                 1/1     Running   0          2m9s
calico-system          calico-node-slj56                                                 1/1     Running   0          2m2s
calico-system          calico-node-t26dd                                                 1/1     Running   0          2m7s
calico-system          calico-typha-6b4d56986f-48f4s                                     1/1     Running   0          2m9s
calico-system          calico-typha-6b4d56986f-ndtv2                                     1/1     Running   0          2m15s
calico-system          csi-node-driver-5gh77                                             2/2     Running   0          104s
calico-system          csi-node-driver-98jfd                                             2/2     Running   0          100s
calico-system          csi-node-driver-wpzsh                                             2/2     Running   0          100s
kube-system            coredns-64897985d-89vz2                                           1/1     Running   0          165m
kube-system            coredns-64897985d-cctns                                           1/1     Running   0          165m
kube-system            etcd-ip-10-0-1-20.us-west-1.compute.internal                      1/1     Running   0          166m
kube-system            kube-apiserver-ip-10-0-1-20.us-west-1.compute.internal            1/1     Running   0          166m
kube-system            kube-controller-manager-ip-10-0-1-20.us-west-1.compute.internal   1/1     Running   0          166m
kube-system            kube-proxy-5ktgw                                                  1/1     Running   0          165m
kube-system            kube-proxy-6pldx                                                  1/1     Running   0          165m
kube-system            kube-proxy-djrv5                                                  1/1     Running   0          165m
kube-system            kube-scheduler-ip-10-0-1-20.us-west-1.compute.internal            1/1     Running   0          166m
tigera-operator        tigera-operator-74575475cc-95b2b                                  1/1     Running   0          9m18s
tigera-packetcapture   tigera-packetcapture-89666fc9d-t6bf2                              1/1     Running   0          83s
tigera-prometheus      alertmanager-calico-node-alertmanager-0                           2/2     Running   0          37s
tigera-prometheus      alertmanager-calico-node-alertmanager-1                           2/2     Running   0          37s
tigera-prometheus      alertmanager-calico-node-alertmanager-2                           2/2     Running   0          36s
tigera-prometheus      calico-prometheus-operator-5798884f64-qflkx                       1/1     Running   0          8m45s
tigera-prometheus      prometheus-calico-node-prometheus-0                               2/3     Running   0          36s
tigera-system          tigera-apiserver-69b9886985-cg6lv                                 2/2     Running   0          83s
tigera-system          tigera-apiserver-69b9886985-sg9ht                                 2/2     Running   0          83s
```
Alternatively, you could use the following command to watch the status of various Calico Enteprise components progress.

```
watch kubectl get tigerastatus
```
```
Every 2.0s: kubectl get tigerastatus                                                                                                                                                                                                                                                                                                    bastion: Wed Dec 28 23:57:31 2022

NAME                  AVAILABLE   PROGRESSING   DEGRADED   SINCE
apiserver             True        False         False      7m48s
calico                True        False         False      8m43s
compliance                                      True
intrusion-detection                             True
log-collector                                   True
log-storage                                     True
manager                                         True
monitor               True        False         False      7m53s
```

14. Apply the LicenseKey to unblock enterprise features of Calico Enterprise.

```
kubectl create -f /home/tigera/license.yaml
```

15. Watch the status of various components progressing. Ensure that all the components are AVAILABLE and there is no components in PROGRESSING or DEGRADED before moving forward. This can take few minutes.

```
watch kubectl get tigerastatus
```

You should an output similar to the following.
```
NAME                  AVAILABLE   PROGRESSING   DEGRADED   SINCE
apiserver             True        False         False      53m
calico                True        False         False      54m
compliance            True        False         False      46s
intrusion-detection   True        False         False      56s
log-collector         True        False         False      86s
log-storage           True        False         False      51s
manager               True        False         False      11s
monitor               True        False         False      53m
```

### 1.1.5. Secure calico system components

As part of the installtion process, we will implement Network security Policies to protect calico components but allow the communication between them, so we can follow a zero trust security approach. Implement the following calico network policies to the environment:

```
kubectl create -f https://docs.tigera.io/manifests/tigera-policies.yaml
```

### 1.1.6. Install the calicoctl utility

Perform the commands below to install the calicoctl client:

```
curl -o calicoctl -O -L https://downloads.tigera.io/ee/binaries/v3.12.0/calicoctl
```
```
chmod +x calicoctl
```
```
sudo mv calicoctl /usr/local/bin/
```

Finally, check the installed Calico version in your lab:

```
calicoctl version
```

Please confirm the "Cluster Calico Enterprise Version" matches the calicoctl version in "Client Version", otherwise please raise this to your instructor.

Configure autocomplete for kubectl.

```
sudo apt-get install bash-completion
source /usr/share/bash-completion/bash_completion
echo 'source <(kubectl completion bash)' >>~/.bashrc
source ~/.bashrc
```
