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

2. Calico Enterprise uses ElasticSearch to store various logs such as flowlogs, DNS logs, and all others that it collects over the network. ElasticSearch requires persistent storage to store the data. This lab uses AWS EBS to provide persistent storage for ElasticSearch. Apply the following manifest to create the storage class, which enables us to dynamically provision AWS EBS storage.

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

3. Make sure the StorageClass has been created successfully.

```
kubectl get storageclass
```
```
NAME                   PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
tigera-elasticsearch   kubernetes.io/aws-ebs   Retain          WaitForFirstConsumer   true                   10s
```

4. The Tigera Operator is a Kubernetes operator and provides a well-defined API for how you install, configure, and run Calico Enterprise. Tigera operator also automates and controls the the lifecycle of a Calico Enterprise deployment. Tigera operator manifest configures the necessary resources such as custom resource definitions, namespaces, services accounts, clusterroles, etc so that cluster is ready to deploy other calico Enterprise components. Get yourself familiar with the content of the manifest and create it in the cluster.

```
kubectl create -f https://docs.tigera.io/manifests/tigera-operator.yaml

```

5. Calico Enterprise Manager UI uses Prometheus to provide out-of-the-box metrics in the various sections of CE Manager UI such as the dashboard page, security policies, and others. Calico Enterprise uses Prometheus operator to deploy Prometheus server and Alertmanager. Apply the following manifest to deploy the Prometheus operator.

**Note:** Prometheus operator that ships with Calico Enterprise is an optional component and can be replaced with the customer's Prometheus operator if needed. However, a Prometheus operator must be deployed in the cluster and watch the "tigera-prometheus" namespace to deploy the required Prometheus and Alertmanager resources. Customer's instance of Prometheus operator can be deployed in any namesapce.


```
kubectl create -f https://docs.tigera.io/manifests/tigera-prometheus-operator.yaml

```

Check tigera-operator has been successfully rolled out:

```
kubectl rollout status -n tigera-operator deployment tigera-operator
```

Now, we must specify the pull secret we will use to download the required images from tigera:

```
kubectl create secret generic tigera-pull-secret \
    --from-file=.dockerconfigjson=/home/tigera/config.json \
    --type=kubernetes.io/dockerconfigjson -n tigera-operator
```

We will apply now the Custom Resource Definitions:

```
kubectl apply -f training-lab-workbooks/advanced/1-initial-setup/lab_manifests/custom-resources.yaml
```

And check the components start progressing:

```
watch kubectl get tigerastatus
```

### 1.1.4. Apply the license

Wait util the `apiserver` shows an status of `True` under the `Available` column, then press `Ctrl+C` to return to the prompt, and apply the license file:

```
kubectl create -f /home/tigera/license.yaml
```

Check all components become available before proceding further (this can take few minutes):

```
watch kubectl get tigerastatus
```

You should an output similar to the following:
```
NAME                  AVAILABLE   PROGRESSING   DEGRADED   SINCE
apiserver             True        False         False      3m50s
calico                True        False         False      35s
compliance            True        False         False      65s
intrusion-detection   True        False         False      85s
log-collector         True        False         False      55s
log-storage           True        False         False      2m5s
manager               True        False         False      50s
monitor               True        False         False      4m50s
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
